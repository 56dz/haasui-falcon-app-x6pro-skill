# 取证纪律、日志与测量工具

这一份是方法论。这个项目里超过一半的时间花在"证明某个东西真的坏了/真的好了"上，
而不是修它。**很多"已修复"的结论后来都被证明是探针假阳性。**

## 第一原则：只信外部可观测的事实

| 你想知道 | ❌ 不要看 | ✅ 要看 |
|---|---|---|
| 有没有画面 | native 返回 `{ok:true}` | `/sys/kernel/debug/dri/0/summary` 里视频平面有没有 `ACTIVE` |
| 首帧到了没 | 自己的 `frames` 计数器归零后重新涨 | 平面 buffer 地址变了 |
| 帧率 | JS 侧 frames 增量 / 固定 30 | 独立采样器数平面 buffer 变化，除以**片源真实 fps** |
| 声音有没有 | 软件音量值 | 音频字节数（`ab`）在不在涨 |
| 播放位置 | 上一时刻的 tick 值 | native 的 `positionMs` + `seekPending` 状态 |

JS 侧的状态是**滞后的一阶导数**，用它判断"现在发生了什么"一定会错。

## 平面几何取证

视频平面在 DRM 里是独立于 UI 的一个 plane。读它：

```sh
adb shell "grep -a 'Esmart1' -A9 /sys/kernel/debug/dri/0/summary"
```

关心这几行：

```text
Esmart1-win0: ACTIVE
  format: NV12
  src: pos[0, 0] rect[720 x 1280]
  dst: pos[107, 299] rect[240 x 427]
  buf[0]: addr 0x267e000 pitch 720
```

- `ACTIVE` = 这个平面正在送显。
- `dst` 是你唯一该信的几何值。
- `src` 是解码器原始分辨率。
- **名字 `Esmart0/Esmart1` 是这台设备的 VOP 窗口命名**，换固件可能不同，先 `ls` 一遍。

几个曾被这个工具支撑的判断：

- 暂停态下 `render-rectangle` 无效（`dst` 纹丝不动，即使日志声称成功）。
- plane 被泄漏后永远不归还（`Esmart1-win0` 整条消失 22 秒）。
- 起播"黑屏"其实平面是 ACTIVE 的（问题在帧没送到，不在平面）。

**高频采样要自己写工具**。`debugfs` 单次 read 有 576 字节上限，`while true; do cat; done`
会漏掉大部分内容。可以用 `lseek`+循环 read 读全，或用 C 写成 1ms 间隔的采样器。

```c
/* 关键：一次 read 只能拿到 576 字节，循环读直到 EOF */
static void read_summary_full(int fd, char *buf, size_t cap) {
    size_t total = 0;
    lseek(fd, 0, SEEK_SET);
    for (;;) {
        ssize_t n = read(fd, buf + total, cap - total - 1);
        if (n <= 0) break;
        total += (size_t)n;
        if (total >= cap - 1) break;
    }
    buf[total] = '\0';
}
```

## 帧率测量：数平面 buffer，不是数帧

```text
采样 20 秒，记录每次 buf[0].addr 的变化，统计间隔分布：

vid_changes=599  vid_hist <20=0 20-30=2 30-45=571 45-70=1
big_gaps(>80ms)=0  vid_fps=29.95
```

- `vid_fps` = 变化次数 / 采样时长。**这是"送到屏上的帧率"**，是用户真正看到的东西。
- `big_gaps` 比平均值重要得多。平均 30fps 但每 3 秒卡一次，用户感受是"卡"。
- `ui_changes` 单独统计——UI 平面刷新和视频平面刷新是两件事。测视频性能时 UI 最好静止。

**片源帧率必须单独确认**。有一次测得 `vid_fps=25.00` 以为是掉帧，实际片源就是 25fps。
用 `ffprobe` 或 `gst-discoverer-1.0` 先确认。

## 音画同步的量化

```c
/* 从帧数反推"实际播放到的内容时间" */
content_ms = frames / source_fps * 1000
avSkewMs   = pos - baseMs - content_ms
```

**`source_fps` 必须是真实值**。写死 30 会让 25fps 片源算出 20 秒的假偏移——我自己就被
这个数骗了很久，以为存在严重的音画不同步，实际完全正常。

而且这个指标只适合看"收敛趋势"，不适合看绝对值：`frames` 包含 preroll 前的内容，
基线天然偏低（实测稳定在大约 -900ms）。真正的判据是它**是否持续增长**：

- 两路对齐 → seek 后收敛到某个稳定值，不再增长。
- 音频领先 (T-K) → 会稳定停在 (T-K) 附近不收敛，或者持续增长。

## 假阳性的三个真实案例

### 案例 1：探针的恢复动作掩盖了失败

bug：暂停时打开输入法，视频平面应缩到 1x1，实际没生效。

探针逻辑：设 1x1 → 等 2.5 秒 → 恢复原尺寸 → 报告结果。

结果每次都是"通过"。原因：**恢复动作走的是"1x1 无法放大 → 重建管线"路径，重建本身
触发了一次 configure，把播放器恢复成了正常状态**。探针看到的"正常"是恢复动作造成的，
不是修复生效。

教训：**探针必须隔着外部动作观察**。验证 A 是否生效，就不要在同一个探针里做会触发 B 的动作；
或者在中途做一次**外部**采样（读 debugfs），而不是等最后看结果。

### 案例 2：帧数增长不代表 seek 完成

bug：拖动进度条后位置回弹。

断言：`st.frames > frames0` 就认为 seek 完成。

实际：seek 命令发出后 200ms 内，**旧管线还在出帧**，帧数当然涨了。断言通过，然后 tick
用 native 的旧位置覆盖了刚写下的落点，进度条回弹——真 bug 被判定为"已修复"。

正确判据要同时满足：

```c
/* native 明确还在 seek → 一律不算完成 */
if (st.seekPending === true) return false;

/* 帧数涨了 **且** 位置已在落点附近 */
const grew = frames > frames0;
const nearTarget = isFinite(pos) && isFinite(target) &&
                   target > 0 && Math.abs(pos - target) <= tolerance;
if (target > 0 && grew && nearTarget) return true;

/* 或者：亲眼见过 pending=true 且它已落回 false */
if (sawPending && (grew || nearTarget)) return true;
```

"亲眼见过 pending" 这个补充条件是因为轮询 1 秒一次、`do_seek` 只要 600~1400ms，
整段 pending 可能完整落在两次采样之间。

### 案例 3：日志冻结 ≠ 死锁

有一次发现 `frames`、`pos`、`ab` 三个值同时冻结，判定为"单 FIFO 背压死锁"，准备加 1MB
pipe 和大 queue。

真因：**我自己的测试脚本执行了 `hal-screen off`，触发了 JS 的自动息屏暂停**
（`state=paused`）。

正确的读法必须同时看 `state=` 和 `screen=` 字段：

```text
probe t=12s screen=DIM(3) state=playing  frames=417(+60) pos=13176
probe t=14s screen=OFF(1) state=paused   frames=428(+11) pos=13540
probe t=16s screen=OFF(1) state=paused   frames=428(+0)  pos=13540
```

前 12 秒屏幕亮着时帧率稳定 +56/+63/+59/+59/+60/+60，`stall` 4~103ms，`ab` 线性增长——
**单 FIFO 完全健康**。

教训：真死锁不会连 `pos` 都停在整数上。多个指标同时冻结时，先找**共同的触发源**
（屏幕、暂停、输入法），再怀疑管线。

## 日志

### 路径

```sh
# native：需要显式开启，默认关闭
adb shell "touch /tmp/<app>.enable"
adb shell "cat /tmp/<app>.log"          # 会跨会话累积

# JS：平台自己写
adb shell "ls -t /userdata/applog/*.log | head -1"
```

### `grep -a` 是必需的

日志里混有二进制（FIFO 名、URL 片段），不加 `-a` 时 grep 会认为文件是 binary 而跳过匹配：

```sh
adb shell "grep -a 'audio stall' /tmp/<app>.log"     # 对
adb shell "grep 'audio stall' /tmp/<app>.log"        # 可能什么都不返回
```

### 跨会话累积要定位起始行

```sh
adb shell "grep -an '=== <app> log start' /tmp/<app>.log | tail -1"
# 输出 8094:=== <app> log start (pid=8541) ===
adb shell "tail -n +8094 /tmp/<app>.log"
```

### 90% 是噪声，先滤波

`app_run`（每次 amixer 调用都打）、`sigchld: ECHILD`、`argv[N]=`、`avSkewMs`、
`bt: no connected`——这些会把真事件埋掉。分析前先过滤：

```sh
adb shell "grep -av 'app_run\|sigchld\|argv' /tmp/<app>.log"
```

### 解析日志一律 pull 到 PC

`adb shell` 里的 `awk -F'[][]'` 和各种正则会被 shell 引号吃掉，踩过多次。
统一做法：

```sh
adb pull /tmp/<app>.log .tmp/app_now.log
python -c "..."   # 按 [<秒>.<毫秒>] 解析
```

## 性能测量的干扰因素

- **`GST_DEBUG` 会严重拖慢**。开着 `basesink:7` 抓 15 秒只得到 **2.27fps**，日志 11MB；
  关掉后同场景 30.00fps。**测性能前必须 `rm -f` 调试开关文件**。
- **stderr 落盘也会慢**。调试期间把 fifo 的 err 写到 `/tmp` 是可以的，但不要在热路径上
  每帧写日志。
- **每秒多打一行 `logf` 就有 4 次 fork**。如果每帧都调 `amixer` 或 `bluetoothctl`，
  4 核 A35 上会直接掉帧。

## `GST_DEBUG` 的正确用法

```sh
# 开关文件在 gst_init 之前被读取
adb shell "echo 'basesink:7,kmssink:6' > /tmp/<app>_gstdebug"
adb shell "echo 'matroskademux:6,queue:6,h264parse:5' > /tmp/<app>_gstdebug"
```

在 `gst_init(NULL, NULL)` 之前：

```c
static void gst_debug_bootstrap(void) {
    FILE *f = fopen("/tmp/<app>_gstdebug", "r");
    if (!f) return;
    char line[256] = {0};
    if (fgets(line, sizeof(line), f)) {
        line[strcspn(line, "\r\n")] = '\0';
        if (line[0]) {
            setenv("GST_DEBUG", line, 1);
            setenv("GST_DEBUG_FILE", "/tmp/<app>_gst.log", 1);
            setenv("GST_DEBUG_NO_COLOR", "1", 1);
        }
    }
    fclose(f);
}
```

**发布前必须删掉这个开关文件，并评估这段代码是否保留。** 留着一个能被文件触发的调试
开关是风险。

已知的坑：`mppvideodec` 的 debug category 名**不叫 `mppvideodec`**。所以
`GST_DEBUG=mppvideodec:5` 抓不到任何东西，即使 `gst-inspect-1.0 mppvideodec` 显示
factory 名就是它。

## 有用的调试问题清单

按这个顺序问，比乱翻日志快：

1. 平面 `ACTIVE` 吗？`dst` 对不对？（区分"没上屏"和"位置错"）
2. 帧在涨吗？涨的速率对不对？`big_gaps` 多少？（区分"没帧"和"掉帧"）
3. `state=` 是什么？`screen=` 是什么？（排除暂停/息屏）
4. 音频字节在涨吗？（区分"静音"和"没数据"）
5. ffmpeg 进程还活着吗？`err-*.log` 有内容吗？
6. 上次 `worker start` 到 `worker end` 之间发生了什么？

## 一些具体的症状 → 方向对照

| 症状 | 先查 |
|---|---|
| 日志显示帧在涨但屏幕不动 | 平面 `dst` 是不是被缩成 1x1 没恢复；plane 是不是被泄漏 |
| 起播从第 1~3 秒开始 | 迟到丢帧；`max-lateness` 与缓冲门 |
| 暂停恢复后掉到 10fps | 是否用了 SIGSTOP 冻进程（不要那么做） |
| 拖进度条后位置回弹 | tick 用旧值覆盖了落点；`seekPending` 判据 |
| 只在蓝牙下黑屏 | `ts-offset` 字符串拼接；`vbranch` 是否建起来 |
| 只在蓝牙下没声音 | 音量权威源读错（Master vs AVRCP） |
| 直播每几秒闪一次 | `mppvideodec` 的 `ignore-error` 默认 true |
| 播到一半没声音、视频正常 | 音频分支断流，需要 ffmpeg reconnect + 看门狗 |
| seek 后秒退 | `-copyts` + `-t` 组合 |
| seek 后音频领先 (T-K) | 双 demuxer，各自 segment.start |
