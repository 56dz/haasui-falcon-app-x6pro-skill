# 平台坑：屏幕、输入法、打包、系统限制

## 屏幕与保活

### `hal-screen` 的实际行为

```sh
# 点亮（三步都要，缺一不可）
hal-screen on
sleep 1
hal-screen reset
sleep 0.6
hal-screen set 40      # 亮度百分比

# 验证
adb shell "cat /sys/class/backlight/backlight/bl_power"    # 必须是 0
adb shell "cat /sys/class/backlight/backlight/brightness"  # 你自己的亮度值
```

关键点：

- **只 `hal-screen set N` 不会把 `bl_power` 从 4 改回 0**，屏幕仍是断电状态，截出来是全黑。
- **`hal-screen off` 只变暗不当真断电**。真正关屏要补：
  ```sh
  echo 4 > /sys/class/backlight/backlight/bl_power
  ```
- 截图大小可以用来判断屏幕状态：亮着时 PNG 几十 KB 到几百 KB，全黑时只有 **1086 字节**左右。
- `hal-screen` 会打印三行无害噪声（`there are 9 input devices` /
  `device fts_ts has evt type 3` / `device with event num=1 supports event type 3`），不是错误。

### 系统会自己息屏

`input-event-daemon` 按 `/data/cfg/input-event-daemon.conf` 的 `[Idle]` 段执行动作：

```ini
[Idle]
600s  = hal-screen down; hal-gsensor wakeup 1
1020s = len_onoff.sh off; hal-screen off; freq_adj low
1680s = system_sleep_wakeup
```

**跑自动化测试前一定要确认这段配置给的窗口够长**。用 30s/60s 的配置跑自检，会得到
`30s 内无帧输出` 的假失败——实际上是息屏触发了自动暂停。

改完要重启守护进程：

```sh
adb shell "killall -9 input-event-daemon"
```

注意 `system_sleep_wakeup` 会让进程被杀（不是单纯息屏）。长跑测试要避开它。

### 屏幕保活

播放中调平台的亮屏接口（一次性保活），间隔不要超过息屏窗口：

```js
/* 每 20 秒续一次 */
this._keepTimer = setInterval(() => {
  try { this.$falcon.brightness.keepScreenOn(); } catch (e) {}
}, 20000);
```

**必须在 `teardown` / `onHide` 里清掉这个 timer**。漏掉会让设备永远不睡。

### 防烧屏

词典笔是 OLED，长时间亮着固定画面会留残影。自动化测试时把亮度压到 40% 以下，测试
结束恢复原值。这不是"可选的优化"。

## 输入法（内容输入）

### 平台只有 `input` 能力，没有内嵌输入框

调用方式大致是"打开系统输入面板，用户输入完回调给你"：

```js
const ctx = this.makeCtx(this.gen);
const reply = await this.$falcon.input.open({ text: null, ctx });
/* reply.text 是用户输入的内容 */
```

### 键盘会盖住视频平面

系统的输入面板是另一个窗口/平面。KMS 视频平面没有"层级"概念，它和 UI 平面的 z-order
由内核决定，**默认在 UI 之上**。

所以：

```text
播放中打开输入法
  → 输入面板出现
  → 视频平面还在原位置送显
  → 视频盖住了键盘的一部分
```

必须在打开输入法前把视频平面缩到 `1x1`，关闭后恢复。恢复的方法见
[pipeline-video.md](pipeline-video.md)（1x1 无法放大，必须重建管线）。

### `onHide` 的坑

打开输入面板时页面会收到 `onHide`。如果 `onHide` 里的逻辑是"暂停播放"，那么：

```text
用户点输入框 → onHide → 暂停 → 面板关闭 → onShow
```

用户会看到视频莫名其妙暂停了。需要区分"因为打开输入法而 hide"和"真正离开页面"：

```js
onHide() {
  if (this._imOpen) {           /* 输入法导致的 hide，不暂停 */
    log('onHide during IM session -> skip');
    return;
  }
  this.pause();
}
```

## 打包与安装

### `.amr` 不是 zip

`.amr` 是这套平台的打包格式。**不要用普通 zip 替代**，也不要手工拼。必须用项目锁定的
`aiot-vue-cli`：

```sh
node <path-to>/aiot-vue-cli/src/cli.js -q -p
```

产物在项目根，文件名形如 `<appid>.<a>_<b>_<c>.amr`。

### 安装

```sh
adb push app.amr /tmp/app.amr
adb shell "miniapp_cli install /tmp/app.amr"
# 期望：{"appid": "<your-appid>", "ret": 0}
```

`ret` 不是 0 就是失败，不要看别的。

### 启动

```sh
adb shell "miniapp_cli start <appid> index"
```

- **页面名不带 `--` 前缀**。`start <appid> --index` 是错的。
- 起播前通常要先回桌面再启动一次（`miniapp_cli start <launcher-appid>`），否则可能黑屏。

### slot a/b 轮换

每次 install 会在 `a` 和 `b` 两个 slot 之间轮换：

```sh
adb shell "ls /userdisk/miniapp/data/mini_app/pkg/<appid>/"
# a  b
adb shell "ls -l /userdisk/miniapp/data/mini_app/pkg/<appid>/<slot>/libs/"
```

装完**必须确认实际生效的是哪个 slot 里的 `.so`**。曾经因为看错了 slot，拿旧库测了半天，
还得出过完全错误的结论。

### 换 appid 后数据不迁移

appid 就是数据目录名。换 appid 等于换了一个 app：

```sh
# 旧数据不会被自动搬过去
/userdisk/miniapp/data/mini_app/pkg/<old-appid>/data/sharedpreferences/preferences.json
```

要保留登录态，手动搬：

```sh
adb push backup.json /tmp/pref.json
adb shell "mkdir -p /userdisk/miniapp/data/mini_app/pkg/<new-appid>/data/sharedpreferences"
adb shell "cp /tmp/pref.json /userdisk/miniapp/data/mini_app/pkg/<new-appid>/data/sharedpreferences/preferences.json"
```

**不要把只含测试开关的 JSON 覆盖整个 `preferences.json`**，会把登录态一起清掉。正确做法
是读出来、改一个 key、写回去。

### 设备上没有 python

改配置文件只能 pull 到 PC、改了再 push 回去。别指望在设备上跑脚本。

```sh
adb pull /path/to/config .tmp/config.json     # 注意 MSYS 下要 MSYS_NO_PATHCONV=1
# 本地 python 改
adb push .tmp/config.json /tmp/config.json
adb shell "cp /tmp/config.json /path/to/config.json"
```

## 系统限制

### 进程数上限

默认 `pid_max` 是 4096。长时间反复 spawn ffmpeg 会撞上限，表现是"突然全都起不来"。

```sh
adb shell "cat /proc/sys/kernel/pid_max"
adb shell "echo 32768 > /proc/sys/kernel/pid_max"
```

要持久化就写进 `/etc/init.d/` 的启动脚本。但**先确认这真的是问题**——我曾经把它误判成
"seek 秒退"的根因，实际原因完全是另一件事（`-copyts` + `-t`）。

### `SIGCHLD` 被宿主设成 `SIG_IGN`

子进程的 `waitpid` 会恒返回 `ECHILD`。这不是失败：

```c
pid_t r = waitpid(pid, &st, WNOHANG);
if (r < 0 && errno == ECHILD) {
    /* 宿主已经替我们收尸了。用 /proc/<pid> 兜底判断存活 */
    app_logf("sigchld: ECHILD (host kept SIGCHLD=SIG_IGN) -> already reaped, rc unknown");
    return 0;   /* 当成成功 */
}
```

用 `/proc/<pid>` 存在性判断进程活着。

### 没有浏览器唤起能力

平台 JSAPI 里没有"打开 URL"的能力。要在 app 里展示 GitHub 链接之类的地址，只能用二维码：

```js
/* 自己生成 QR 点阵，用 view 行/列拼出来 */
const { grid, size } = encodeQr('https://example.com');
const rows = gridToRows(grid, size);
```

注意二维码的像素密度要匹配显示容器。我自己踩过：大码（单元 4px）和小码（单元 2px）复用了
同一套 `.qr-row { height: 4px }`，小码实际渲染高度翻倍，容器 `overflow:hidden` 只显示上半截，
扫不出来。**两种尺寸要两套 class**。

### Falcon 不能渲染 SVG

`<image src="....svg">` 不会工作，也没有 SVG 解码能力。图标必须是 PNG。

矢量素材（比如从设计工具导出的 SVG）需要自己光栅化。如果只是简单图形，写个小的
贝塞尔扁平化 + 扫描线填充就够了：

```python
# 关键：填充规则用 nonzero，不是 even-odd
# 气泡轮廓和内部圆点的绕向通常是相反的，
# nonzero 会让圆点内部净环绕数为 0（正确镂空），even-odd 会判反。
```

还有两个小坑：`k = SIZE / viewBox_size` 之后忘了乘超采样倍数，图标会缩到画布左上角
四分之一处；留白（padding）太小会顶满画布边缘。

## UI 布局约束

### 画布是固定尺寸的

逻辑画布 960×266，**没有滚动容器之说**（除了你自己放的 `scroller`）。所有布局都要在
192px 左右的高度预算内塞下，改一行 CSS 就可能挤掉别的东西。

改布局前先算一遍高度预算：

```text
8(pad-top) + 26(back) + 6 + 34(title) + 4 + 14(up) + 6 + 72(desc) + 4 + 16(note) = 190 <= 192
```

### `scroller` 里的 `text` 不能带 `lines`

```html
<!-- 错：text 自己先截断成 N 行，scroller 滚的是一个已经截断的东西 -->
<scroller class="desc"><text class="desc-text" :lines="3">{{ text }}</text></scroller>

<!-- 对：内层 text 不限制行数，只靠 scroller 容器高度裁 -->
<scroller class="desc"><text class="desc-text">{{ text }}</text></scroller>
```

### flex 尺寸要写死

这套渲染引擎对 `flex-grow/shrink/basis` 的支持不完整。需要固定高度的区域，直接写
`height` + `flex-grow: 0` + `flex-shrink: 0`，不要依赖 flex 分配。

## 触摸

### 没有 evdev 注入

`weston` 没有暴露 evdev fd，所以不能通过 `/dev/input` 注入触摸事件。真机自动化的可行路径：

1. 自己写一个 native 小工具，用平台的输入 JSAPI 或 uinput 注入（如果能拿到权限）。
2. **更好：让 app 自己驱动自己**。在 `preferences.json` 里放一个测试开关，app 启动时读取并
   自动执行一串动作。

第二种方式稳定、可复现、不需要额外权限，我在整个项目里都用它：

```json
{
  "app_autotest": { "version": 1, "enabled": true, "autotest": true, "pollEvery": 1 }
}
```

注意开关要有**双重守卫**，避免误触发：

```js
const atOn = at && (at === true || (at.enabled === true && at.autotest === true));
```

## 日志目录

- native：`/tmp/<app>.log`（需先 `touch /tmp/<app>.enable`，`/tmp` 重启会清空）
- JS / 平台：`/userdata/applog/*.log`

`/tmp` 是 tmpfs，**设备重启后全部丢失**。所有部署到 `/tmp` 的工具（采样器、注入器）重启后
都要重新 push。
