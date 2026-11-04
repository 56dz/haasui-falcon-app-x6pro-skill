---
name: haasui-falcon-app-x6pro
description: Build video/audio applications for NetEase Youdao Dictionary Pen X6 Pro (RK3562) on the Falcon mini-app runtime. Use when writing a Vue/QuickJS mini-app that plays network video, when a GStreamer + MPP + DRM/KMS pipeline must be driven from a native JSAPI, when audio routing (speaker / Bluetooth A2DP) or A/V sync is wrong, when the KMS video plane covers UI or survives after teardown, or when packaging and installing .amr builds on the device. Distilled from a real production video client; every rule here was paid for with a device-side experiment.
---

# 有道词典笔 X6 Pro · Falcon 视频应用开发

这份 skill 来自一个真实上线的词典笔视频客户端：Falcon/QuickJS 做 UI，native `.so` 驱动
GStreamer + Rockchip MPP 硬解，画面直接走 DRM/KMS 平面。文档里每一条"必须/禁止"背后都有
一次真机实测，出处写在括号里。**不要把这里的数值当通用常量**——先在目标固件上复现验证，
再写进代码。

## 适用与不适用

适用：

- Falcon / QuickJS mini-app、Vue 2 SFC 页面，需要播放网络视频或音频。
- 需要自己维护一条 `ffmpeg` 子进程 → GStreamer → MPP → KMS 的播放管线。
- 音画不同步、起播跳秒、暂停恢复卡顿、进度条回跳、画面盖住键盘。
- 扬声器与蓝牙耳机之间切换音量、耳机无声、音量被清零。
- `.amr` 打包、`miniapp_cli install`、真机验证。

不适用：

- 只有纯 UI、不涉及媒体的 app —— 直接用平台高层 JSAPI 就够，不要上 native。
- 把某个 app 的业务协议（登录、推荐流、弹幕）当平台能力。这些属于应用层，不是 Falcon。

## 先做设备画像，再写一行代码

**禁止在确认画像前写死**：逻辑/物理分辨率、旋转、平面 id、解码器名、库路径、工具链、
appid、安装目录、日志目录。

X6 Pro 的实测画像（**只能作为待验证线索，换固件必须重测**）：

```yaml
model: 有道词典笔 X6 Pro
soc: Rockchip RK3562 (armv8, 4 x A35)
runtime: Falcon / QuickJS, aiot-vue-cli + Vue 2 SFC
screen:
  logical: { width: 960, height: 266 }        # setViewPort 的设计宽
  panel:   { width: 480, height: 960, direction: 1 }   # DRM 原生竖屏，rotation=1
  # 换算方向：逻辑矩形 (x,y,w,h) 在画布上，落到面板上是 (y, 960-(x+w), h, w)
  # 例：播放区逻辑 (234,0,427,240) → 面板 dst pos[107+0,299] rect[240x427]
  # 注意面板横向有一个约 107px 的死区偏移，见 pipeline-video.md
abi: { machine: aarch64, libc: glibc, loader: /lib/ld-linux-aarch64.so.1 }
decoder: Rockchip MPP (gstreamer1.0-rockchipmpp, /dev/mpp_service)
media: gstreamer-1.0, kmssink (DRM), alsasink
```

探测用固定短命令，只读：

```sh
adb devices -l
adb shell "cat /etc/os-release; uname -m; uname -a"
adb shell "ls /usr/lib/gstreamer-1.0/ | head -50"
adb shell "gst-inspect-1.0 kmssink | head -40"
adb shell "cat /sys/kernel/debug/dri/0/summary"
adb shell "miniapp_cli --help"
```

"命令不存在"也是画像结果，记下来。不要递归扫 `/sys`，不要重启设备掩盖问题。

> **ADB 认证**：如果 `adb shell` 返回 `login with "adb shell auth" to continue`，
> 停下来请人完成认证。不要猜密码、不要试着绕过——设备上的认证是有意设置的。
> 症状是 `adb push` 报 `protocol fault: stat response has wrong message id`。

## 架构：三层，边界必须清楚

```text
┌─ Falcon UI (Vue 2 / QuickJS) ─────────────────────────────┐
│  页面、控件、触控、进度条、输入法          ← 只管状态和交互  │
└───────────────┬───────────────────────────────────────────┘
                │ JSAPI（同步 + Promise 混合）
┌───────────────▼───────────────────────────────────────────┐
│  native libjsapi_<plugin>.so                              │
│  ├─ session worker 线程（唯一的管线所有者）                │
│  ├─ cmd queue（JS 线程只投递，绝不阻塞）                   │
│  ├─ ffmpeg 子进程（拉流，写 FIFO）                         │
│  └─ GStreamer pipeline（demux → 硬解 → kmssink / alsasink）│
└───────────────┬───────────────────────────────────────────┘
                │ DRM/KMS plane + ALSA
┌───────────────▼───────────────────────────────────────────┐
│  硬件：KMS 视频平面（独立于 UI 平面，几何/层级由内核管）   │
└───────────────────────────────────────────────────────────┘
```

三条铁律：

1. **一个 worker 线程独占整条管线**。JS 线程只 `enqueue(cmd)`；任何 `gst_element_set_state`、
   任何 `fork`、任何 `waitpid` 都不许在 JS 线程上跑。JS 线程被冻住，整个 app 就没响应了。
2. **JS 只信「可观测的物理事实」**，不信 native 上报的"我以为"。判首帧要看平面真的变了；
   判音画同步要看音频字节真的在涨。详见 [references/debugging.md](references/debugging.md)。
3. **状态机单向**：`IDLE → OPENING → PLAYING ⇄ PAUSED → TEARDOWN → IDLE`。每个转换都必须
   幂等、可重入、可被 `STOP` 打断。重建管线时旧 worker 必须真的退出（`pthread_join`，
   或 detach 后等 `worker_done`），否则新管线会和僵尸线程抢同一个 KMS plane。

代码分层与文件职责见 [references/architecture.md](references/architecture.md)。

## 播放管线的核心决策

**不要用 `souphttpsrc` 直接吃 DASH 分片流。** 实测 `souphttpsrc` + m4s 永远无法 seek，
`tsdemux` 在巨大起始 PTS 上 preroll 卡死，fMP4 帧间隔超标。可用的组合只有一条：

```text
ffmpeg（拉流 + 封装转换） → 单 FIFO → matroskademux → h264parse → mppvideodec → kmssink
                                        └→ aacparse → faad → audioconvert → volume → alsasink
```

关键理由和全部实测数据在 [references/pipeline-video.md](references/pipeline-video.md)。必须先读它再动媒体代码。
其中最容易踩死的四条：

1. **单 FIFO、单 demuxer**。音视频分成两条 FIFO 两条 demuxer，会让两路各自 `segment.start`
   重基，音频领先 `T-K`（K = seek 目标 T 之前最近的关键帧，间隔 1~2 秒），画面永远落后。
2. **`-copyts` 与 `-t` 是致死组合**。`-t` 在 `copyts` 下按**原始**时间轴截断，`base` 一大
   就把所有包滤光——ffmpeg `rc=0`、0 包输出、stderr 全空，表现为"seek 后源秒退"。
   截断点要写 `durationMs + 5000`。
3. **`render-rectangle` 是只写属性**，`kmssink` 只在 **reconfigure（分配 plane）** 那一刻读它。
   PAUSED 下没有新帧、不会 reconfigure，设什么值都无效。要改就先设属性、再临时切 PLAYING
   脉冲 200ms、最后切回 PAUSED。而且它是**单向棘轮**：缩到 1x1 后无法再放大，只能重建管线。
4. **起播要两段式**：先 `set_state(PAUSED)`（时钟不走、没有帧会迟到），轮询到视频有帧且音频
   分支就绪，再 `set_state(PLAYING)`。直接用 `get_state` 判 preroll 是错的——分支 sink 是
   `pad-added` 动态建的，desc 里没有 sink，`set_state(PAUSED)` 会立刻返回 SUCCESS。

## 音频：音量权威源是这套 SDK 里最大的坑

一句话总结：**扬声器看 `card1 Master`，蓝牙看耳机自己的 AVRCP，两者互不相干；配置文件里的
`spk_volume` 永远不能当权威**（真机实测它晚 8~12 秒才落盘）。

完整的实测时序表、三次纠错史和"为什么软件增益在蓝牙路由下必须固定 1.0"见
[references/audio-route.md](references/audio-route.md)。这里只列不许犯的错：

- 不许把 `card1 Master` 的百分比当成耳机音量——它是扬声器功放的寄存器，切蓝牙后可以长期停在 0。
- 不许在起播时用扬声器的值**覆盖**耳机音量。用户设的 50% 凭什么被你的 0% 改掉。
- 不许让 "AVRCP 音量" 和 "GStreamer `volume` 元素" 同时生效。它们是串联的，两级各 76% 就是 58%。
- 不许给 `alsasink` 加 `ts-offset` 来修音画偏移。它是管线时钟提供者，平移它等于平移整条管线，
  画面跟着走，相对关系不变，还会把恢复后帧率推到 90+fps。偏移加在 `kmssink` 上。
- 不许为了"抗抖动"放大 `alsasink` 的 `buffer-time`。它同时决定 sink 上报给管线的 latency，
  实测从 30ms 放到 200ms，`vid_fps` 从 29.95 掉到 19.70，帧间隔恒定 47ms。

## 直播与点播是不同的东西

- 点播：`ffmpeg -ss T` 精确 seek，一个文件。
- 直播：**不能走 ffmpeg**。FLV 的 PTS 是绝对值（实测约 2854014 秒），落到 FIFO 上的流式 demux
  无法处理巨大起始 PTS，preroll 直接卡死。直播走 `souphttpsrc ! flvdemux` 直连，
  `durationMs=0`，位置用"本次会话已播时长"而不是绝对 PTS。

## 构建、打包、安装

用**项目锁定的** Node 和 `aiot-vue-cli`，不要捡 PATH 上第一个。

```sh
npm run native                       # 交叉编译 libjsapi_<plugin>.so（aarch64)
node <aiot-vue-cli>/src/cli.js -q -p # QuickJS AMR
```

native 库进包前必须验：

```sh
file libjsapi_<plugin>.so              # aarch64、ELF64、shared object
readelf -h libjsapi_<plugin>.so        # Machine / Class / Type
readelf -d libjsapi_<plugin>.so        # NEEDED 依赖是否都在设备上
nm -D libjsapi_<plugin>.so | grep custom_init_jsapis   # 导出符号必须存在
```

`.amr` 只能由项目打包器生成，**不要用普通 zip 替代**（它不是 zip）。装完要解包核对 manifest、
脚本、资源和每个 `.so` 的相对路径。

```sh
adb push app.amr /tmp/app.amr
adb shell miniapp_cli install /tmp/app.amr    # 必须返回 {"ret": 0}
adb shell miniapp_cli start <appid> index     # 页面名不带 -- 前缀
```

**安装会自动在 slot a / slot b 之间轮换**，装完看 `pkg/<appid>/<slot>/` 确认实际生效的是哪个。
换 appid 后旧数据目录不会自动迁移，登录态要手动搬 `data/sharedpreferences/`。

更多平台坑（屏幕保活、输入法、`hal-screen`、烧屏、pid 上限）见
[references/platform-pitfalls.md](references/platform-pitfalls.md)。

## 调试与取证纪律

这是最容易做错的部分——**很多"bug 已修复"的结论最后都被证明是探针假阳性**。

- 判首帧：读 `/sys/kernel/debug/dri/0/summary` 里视频平面的 `dst: pos[…] rect[…]`，不是读 JS 的 `{ok:true}`。
- 判帧率：用独立采样器数平面 buffer 变化，不是数 JS 计数器的增量。
- 判音画同步：`frames / 真实片源 fps` 对 `pos`，**不要用写死的 30fps**——25fps 的片源会算出
  20 秒的假偏移。
- 探针必须**隔着外部动作**观察。曾经有个 bug：暂停后设矩形没生效，但探针在 2.5 秒后做恢复动作，
  恢复本身触发了 reconfigure，把失败掩盖成了成功。
- 开着 `GST_DEBUG` 测帧率会得到 2.27fps，这不是 bug。测性能先 `rm -f` 掉调试开关文件。

日志路径、`grep -a` 的必要性、`GST_DEBUG` category 名找不到的绕法、平面采样脚本、
滤波词清单，全部在 [references/debugging.md](references/debugging.md)。

## 交付前检查

- [ ] 真机跑过一次**不静音**的完整播放，确认出声、音画同步、无破音。
- [ ] 起播、暂停→恢复、拖动进度条、seek 到片尾前、切集，五个动作的帧率都是片源帧率。
- [ ] 拔插蓝牙耳机后仍能出声，且音量等于用户设置，不是 0。
- [ ] 暂停时打开输入法，视频平面不遮挡键盘。
- [ ] 退出播放页后 `pidof ffmpeg` 为空，`/tmp` 下无残留 FIFO。
- [ ] 连续两次进入/退出播放页，无 timer、订阅、线程、子进程泄漏。
- [ ] `readelf -d` 的每个 `NEEDED` 在设备上都存在。
- [ ] 版本号、manifest、README、Release 三处一致。

## 参考文件索引

| 文件 | 什么时候读 |
|---|---|
| [references/architecture.md](references/architecture.md) | 搭建项目分层、线程模型、状态机、命令队列 |
| [references/pipeline-video.md](references/pipeline-video.md) | 写/改媒体管线、seek、起播、直播、KMS 矩形 |
| [references/audio-route.md](references/audio-route.md) | 音量、蓝牙、扬声器、音画偏移、音频停摆 |
| [references/debugging.md](references/debugging.md) | 取证、日志、采样器、性能测量、常见误判 |
| [references/platform-pitfalls.md](references/platform-pitfalls.md) | 屏幕保活、输入法、打包安装、系统限制 |
