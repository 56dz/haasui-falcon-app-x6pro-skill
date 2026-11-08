---
name: haasui-falcon-app-x6pro
description: Build video, audio and realtime-streaming applications for NetEase Youdao Dictionary Pen X6 Pro (RK3562) on the Falcon mini-app runtime. Use when writing a Vue/QuickJS mini-app that plays network video, when a GStreamer + MPP + DRM/KMS pipeline must be driven from a native JSAPI, when the device has no webrtcbin/libsrtp/libnice and a WebRTC-style realtime stream (cloud gaming, remote desktop) must be decoded by hand-writing ICE + DTLS-SRTP + RTP, when touch/text input must be sent back to a remote session, when audio routing (speaker / Bluetooth A2DP) or A/V sync is wrong, when the KMS video plane covers UI or survives after teardown, when the Weex/Falcon UI dialect renders differently on the device than in the build, or when packaging and installing .amr builds. Distilled from three shipped device clients; every rule here was paid for with a device-side experiment.
---

# 有道词典笔 X6 Pro · Falcon 音视频 / 实时流应用开发

这份 skill 来自三个真实上线的词典笔应用：本地/网络视频播放器、直播客户端，以及一个把
第三方云游戏服务搬上笔的串流客户端（自写 ICE / DTLS-SRTP / RTP，硬解上 KMS，含触摸与
文本输入回传）。Falcon/QuickJS 做 UI，native `.so` 驱动 GStreamer + Rockchip MPP 硬解，
画面直接走 DRM/KMS 平面。文档里每一条"必须/禁止"背后都有一次真机实测，出处写在括号里。
**不要把这里的数值当通用常量**——先在目标固件上复现验证，再写进代码。

## 适用与不适用

适用：

- Falcon / QuickJS mini-app、Vue 2 SFC 页面，需要播放网络视频或音频。
- 需要自己维护一条 `ffmpeg` 子进程 → GStreamer → MPP → KMS 的播放管线。
- 需要播放**实时流**（WebRTC / RTP / 云游戏），设备上没有 `webrtcbin`/`libnice`/`libsrtp`，
  必须自己写 ICE + DTLS-SRTP，再用 `appsrc` 喂裸 H.264。
- 音画不同步、起播跳秒、暂停恢复卡顿、进度条回跳、画面盖住键盘。
- 扬声器与蓝牙耳机之间切换音量、耳机无声、音量被清零、蓝牙延迟。
- `.amr` 打包、`miniapp_cli install`、真机验证。
- 需要 native `.so` 把新能力（裸 HTTP / WebSocket / 加密 / 媒体）导出给 JS。
- 逆向一个第三方 Web 服务并做成客户端：接口取证、错误码、发布与隐私自检。

不适用：

- 只有纯 UI、不涉及媒体的 app —— 直接用平台高层 JSAPI 就够，不要上 native。
- 把某个 app 的业务协议（登录、推荐流、弹幕）当平台能力。这些属于应用层，不是 Falcon；
  只有"怎么安全地逆向并维护这类协议"是平台无关的工程纪律，见
  [references/service-reverse-ops.md](references/service-reverse-ops.md)。

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

## 第三条路线：实时流（WebRTC 类），设备上没有 webrtcbin

点播和直播之外还有一类：对端是一个实时媒体网关，要通过 ICE 打洞、DTLS 握手、SRTP 解密，
再把裸 H.264 / Opus 交给 GStreamer。**这条路上所有现成组件都不存在** —— 实测
`ls /usr/lib/gstreamer-1.0/` 里没有 `libgstwebrtc`、没有 `libnice`、没有 `libsrtp`；
唯一能用的是 `libssl 1.1.1l`（导出 DTLS-SRTP 的 keying material 符号）。

可用的架构只有一条：

```text
UDP socket -> STUN / DTLS / SRTP 自实现 -> RTP 拆包 -> appsrc(video/x-h264,byte-stream,au)
                                                     -> appsrc(audio/x-opus) -> 音频链路
```

五个最贵的坑（每一条都曾导致"握手成功但没画面"）：

1. **`SSL_export_keying_material` 的 label 长度**：`"EXTRACTOR-dtls_srtp"` 是 **19** 字节。
   传 21 会把长度带进 PRF seed，导出的密钥完全不同 -> 每包 `auth_fail`。
   最阴的是自测服务端若也写 21，**两端同错时永远测试通过**。
2. **STUN 与 DTLS 共用同一个 UDP 5 元组**。不按首字节分流的话，STUN Binding Request
   会被当成 DTLS 记录喂给 `SSL_read`，握手随机失败。用 `recv(MSG_PEEK)` 分类。
3. **connected UDP socket 上的 ICMP 错误是瞬态的**。把 `ECONNREFUSED` 当致命错误会在
   首帧之后立刻退出媒体循环 —— 表现为"出画一瞬间然后黑屏"。
4. **`GST_STATE_NULL` 是 1，不是 0**。传 0 是非法目标态，`set_state` 静默失败，
   退出后 KMS 平面把 UI 永久盖住。
5. **MPP 不做 KMS 旋转**，面板是竖屏（480x960）、逻辑画布是横屏（960x266），
   必须在解码器里转（`mppvideodec rotation=270`），且 `render-rectangle` 要用
   **面板物理坐标**换算。

完整的状态机、SDP 解析细节、487 Role Conflict、PLI 的 SSRC 规则、SPS/PPS 解析，见
[references/webrtc-realtime.md](references/webrtc-realtime.md)。

> 这一层的调试要按"阶段"隔离：先把 ICE 打通（看有没有 Binding Response），
> 再单独验 DTLS（看 `SSL_do_handshake` 是否返回 1），再单独验 SRTP（看 `auth_fail` 是否为 0），
> 最后才接 GStreamer。四个阶段混在一起调，任何一处错都表现为"没画面"。

## UI 层：Falcon/Weex 方言与页面栈

UI 层的坑不比媒体层少，而且更隐蔽 —— **大部分是"构建通过、真机走样"或"静默不生效"**：

- 每个 `.vue` 必须有 `<style lang="less" scoped>` 且首行 `@import "base.less"`（无前缀）。漏了会黑屏且无报错。
- `<style scoped>` **实际是全局合并**，跨页撞类名会互相污染 -> 每页用自己的类名前缀。
- Weex **只支持单类名选择器**，`.a .b` / `.a.b` 一律构建失败。
- Weex **默认 `flex-direction: column`**，不写 `row` 时 `justify-content` 管的是纵向。
- `flex: 1` 无效，必须写 `flex-grow/shrink/basis` 三连。
- 没有可靠的 `position: absolute` 与 `z-index`，浮层只能用 `v-if` 整块替换。
- **把展示文案写成 method 会渲染出 `function bound xxx() { [native code] }`** —— 必须是 `data()` 里的字符串。
- **`$falcon.navBack` 在某些固件里根本不存在**（框架 bundle 里 `grep -c navBack` = 0），
  "返回"必须用 `closePageByName('<页名>')`；用 `navTo` 兜底会变成 push，页面栈越堆越深。
- **页面级 `onHide`/`onUnload` 可能根本不回调**，但 **App 级 `onHide` 会**。
  所有"切后台要做的收尾"（停管线、还平面）必须写在 `app.js`。

完整清单见 [references/falcon-ui-dialect.md](references/falcon-ui-dialect.md)。

**把触屏操作送到远端会话**（云游戏 / 远程桌面这类）是另一套独立规则：移动端触摸事件
（`down/move/up/tap`）的坐标是**视频像素**，PC 鼠标事件（`mousedown/up/move`）的坐标是
**65535 归一化百分比** —— 两者不能互换，换错就是"点了没反应"且无任何报错。此外 `pageX/pageY`
已经是**视频子矩形内的局部坐标**，再减一次原点会整体偏移。见
[references/live-input.md](references/live-input.md)。

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

媒体：

- [ ] 真机跑过一次**不静音**的完整播放，确认出声、音画同步、无破音。
- [ ] 起播、暂停→恢复、拖动进度条、seek 到片尾前、切集，五个动作的帧率都是片源帧率。
- [ ] 拔插蓝牙耳机后仍能出声，且音量等于用户设置，不是 0。
- [ ] 蓝牙路径下 `aplay -v` 的 `buffer_size` 是你设的值，且日志里 `underrun` 计数为 0。
- [ ] 暂停时打开输入法，视频平面不遮挡键盘。
- [ ] 退出播放页后 `pidof ffmpeg` 为空，`/tmp` 下无残留 FIFO，`pidof aplay` 为空。
- [ ] 连按两次"退出"、切后台、直接关 app，三条路径下 KMS 视频平面都消失（`/sys/kernel/debug/dri/0/summary` 里没有视频 plane 的 `ACTIVE`）。
- [ ] 连续两次进入/退出播放页，无 timer、订阅、线程、子进程泄漏。

native 与打包：

- [ ] `readelf -d` 的每个 `NEEDED` 在设备上都存在。
- [ ] `nm -D --undefined-only` 的每个符号都能被宿主解析（否则整个 `.so` 加载失败且只报"未找到模块"）。
- [ ] `import('<plugin>')` 在真机成功，且一次 `status()` 能拿到合理字段。
- [ ] `.amr` 装完 `miniapp_cli install` 返回 `{"ret": 0}`，解包核对 manifest/脚本/资源/`.so` 相对路径。
- [ ] 版本号在 `version.js`、`package.json`、git tag、Release 四处一致。

UI 与交互：

- [ ] 五个主要页面都真机截图看过，没有"文字竖排""渲染出函数源码""布局没撑满"。
- [ ] 每个页面用自己的类名前缀，跨页无同名类。
- [ ] 返回动画是 pop 不是 push（连拍 16 帧可验），且页面栈不会越堆越深。
- [ ] 切后台再回来，状态正确（不假死、不重复订阅）。

发布与隐私：

- [ ] 源码、构建产物、`.so`、`.amr` 用 **ASCII + UTF-16LE 双编码**扫过手机号/uid/密码/token/序列号/MAC，0 命中。
- [ ] `profiles/evidence/**` 里的每张截图**逐张看过**，含账号、手机号、游戏 UID、内网 IP 的已打码或删除。
- [ ] git 历史（不只是工作树）同样扫过；若改过历史，确认 `refs/remotes/*` 已删、`gc --prune=now` 做过、**远端 tag 已重指**。
- [ ] release 附件回下载并比对 sha256。

## 参考文件索引

| 文件 | 什么时候读 |
|---|---|
| [references/architecture.md](references/architecture.md) | 搭建项目分层、线程模型、状态机、命令队列 |
| [references/pipeline-video.md](references/pipeline-video.md) | 写/改媒体管线、seek、起播、直播、KMS 矩形 |
| [references/webrtc-realtime.md](references/webrtc-realtime.md) | 实时流：自写 ICE / DTLS-SRTP / RTP，喂 `appsrc`，KMS 上屏 |
| [references/audio-route.md](references/audio-route.md) | 音量、蓝牙、扬声器、音画偏移、音频停摆、`fdsink`+`aplay` 路线 |
| [references/native-jsapi.md](references/native-jsapi.md) | 写/改 native `.so`：交叉编译、stub 链接、JSON ABI、错误码 |
| [references/falcon-ui-dialect.md](references/falcon-ui-dialect.md) | Weex 方言、类名前缀、页面栈与返回动画、IME、渲染验收 |
| [references/live-input.md](references/live-input.md) | 把触屏操作送进远端会话：两套坐标单位、异常静默、点击验收 |
| [references/debugging.md](references/debugging.md) | 取证、日志、采样器、性能测量、常见误判 |
| [references/platform-pitfalls.md](references/platform-pitfalls.md) | 屏幕保活、输入法、打包安装、系统限制 |
| [references/service-reverse-ops.md](references/service-reverse-ops.md) | 逆向第三方服务、错误码与单位、发布流程、隐私自检 |
