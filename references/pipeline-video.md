# 视频管线：ffmpeg → FIFO → GStreamer → MPP → KMS

本文是这套 skill 的核心。每一条都是真机实测，不是推测。

## 为什么不能用"更简单"的方案

我试过所有看起来更省事的路子，全部失败。省下的时间不够填坑的。

| 方案 | 结果 |
|---|---|
| `souphttpsrc` 直接吃 DASH `.m4s` | **永远无法 seek**。分片流没有文件长度快照，`gst_element_query_duration` 恒 0 |
| `ffmpeg` 软解后直接输出 raw 到 pipe | **0.76x 实时**。RK3562 的 4×A35 扛不住 720p60 软解，画面持续落后 |
| 下载成完整 MP4 再 `filesrc` | 长度快照一次性固化，**中途重连/seek 后永久卡死** |
| `ffmpeg -f mpegts` + `tsdemux` | 起始 PTS 巨大时 **preroll 卡死**，起播要十几秒甚至不出帧 |
| `ffmpeg -f mp4`（fMP4）| 帧间隔超标，`mp4mux` 在小 buffer 下产出不合法时序 |
| **`ffmpeg -f matroska` → FIFO → `matroskademux`** | ✅ 唯一可用 |

Matroska 赢的原因：它允许流式写入且 demuxer 容忍不完整的簇；`ffmpeg -c copy` 能保持原编码
不变（硬解要求），同时把两路音视频**多路复用进同一条字节流**——这一点是修音画同步的关键。

## 正确的管线

```text
ffmpeg -ss T -i <video-url> -ss T -i <audio-url>
       -map 0:v -map 1:a? -c copy -copyts -fflags +genpts
       -t <durationMs+5000>
       -f matroska  →  /tmp/<app>/<app>-<pid>-<n>.fifo
                            │
                     （单 FIFO，单写端）
                            ▼
          filesrc location=<fifo> blocksize=65536
                            ▼
                     matroskademux name=vd
              ┌─────────────┴──────────────┐
        (pad-added: video)          (pad-added: audio)
              ▼                            ▼
   queue → h264parse → mppvideodec   queue → aacparse → faad
              ▼                            ▼
        videorate → kmssink         audioconvert → audioresample
        name=vsink                       ▼
                                   volume name=vvol
                                        ▼
                                     alsasink
```

三个必须记住的点：

1. **`-map 1:a?` 的 `?` 是双刃剑**。它让没有音轨的视频能播，但也让"音频流中途消失"
   不产生任何错误。后面要靠看门狗兜（见 [audio-route.md](audio-route.md)）。
2. **`blocksize=65536` 不能省**。默认 block size 读 FIFO 会一次只拿几字节，逐个 buffer
   拷贝，帧率掉一半。
3. **`videorate` 是必需的**。片源有 24/25/30/60 各种帧率，`kmssink` 需要稳定帧率才不抖。
   但 `videorate` 的 `fpsCap` 不要随便设 30，否则 60fps 片源会被抽帧。

## 坑 1：单 FIFO、单 demuxer

这是修"拖动进度条后画面落后于音频"的关键，值得单独讲。

两条 FIFO、两个 demuxer 时，每个 demuxer 各自计算 `segment.start`：

```text
视频路：ffmpeg -ss T → 实际从关键帧 K 开始（K <= T）
        视频 demuxer segment.start = K
音频路：ffmpeg -ss T → 音频从 T 开始
        音频 demuxer segment.start = T

结果：两路 running_time 都从 0 起算
      音频内容 T 与视频内容 K 同时播放
      → 音频领先 (T - K)，画面永远落后
```

`T - K` 就是关键帧间隔，实际 1~2 秒。用户看到的就是"画面落后于音频"。

单 FIFO 单 demuxer 时，两路共享同一个 `segment.start = min(K, T) = K`：

```text
视频内容 K → running_time 0
音频内容 T → running_time (T - K) → 音频要等 (T-K) 才播
此时视频已经播到内容 T → 对齐
```

配套要加 `-copyts` 保留原始 PTS（而不是让 ffmpeg 重基），再用 `-fflags +genpts` 补齐。

## 坑 2：`-copyts` 与 `-t` 是致死组合

`-t` 在 `copyts` 模式下按**原始时间轴**截断。如果你写：

```sh
# 错：base 一大（比如 seek 到 50 分钟），tlimit 就算成负数，所有包被滤光
-t $((durationMs - baseMs + 5000))ms
```

ffmpeg 会 `rc=0`、输出 **0 个包**、stderr **完全空白**。表现是"seek 后视频源秒退"，
你会花很久在别处找原因。

```sh
# 对：永远给足余量，让 GStreamer 侧的 EOS 来结束
-t $((durationMs + 5000))ms
```

## 坑 3：`render-rectangle` 只写，且是单向棘轮

这是"暂停后打开输入法，视频画面盖住键盘"的根因。

```c
/* kmssink 的 render-rectangle 是 write-only 属性：
   只能 gst_util_set_object_arg 写，g_object_get 会报 "not readable"。
   而且它只在 sink reconfigure（分配 plane）那一刻被读取。 */
```

推论：

- **PLAYING 下**设它有效，因为每帧都在 reconfigure。
- **PAUSED 下**设它无效——没有新帧、不会 reconfigure，设什么值都读不到。日志里
  `SET_RECT took 0ms` 显示成功，实际平面纹丝不动。
- **必须先设属性、再脉冲**。我第一版写成了先脉冲再设属性，reconfigure 读到的是旧值：

```c
/* 对：属性先设，再临时 PLAYING 触发 reconfigure，然后冻回 PAUSED */
if (want_hide) {
    gst_util_set_object_arg(g_w.vsink, "render-rectangle", rect);   /* 先设 */
    g_w.rect = rect;
    if (g_w.paused) {
        gst_element_set_state(g_w.pipe, GST_STATE_PLAYING);         /* 再脉冲 */
        app_sleep_ms(200);   /* 实测 mppvideodec 从 PAUSED 出帧要 100~180ms */
        rect_pulsed = 1;
    }
}
/* ……switch 结束后统一收尾，不要在每个分支里 break */
if (rect_pulsed && !rect_rebuilt && g_w.paused) {
    gst_element_set_state(g_w.pipe, GST_STATE_PAUSED);
}
```

脉冲时长 200ms 不是随便写的：60ms 时 mppvideodec 经常还没走到 configure 就切回去了。
脉冲期间最多多渲染 1~2 帧，用户无感。

**单向棘轮**：把矩形缩到 `1x1` 之后，无法再用 `render-rectangle` 放大回来。
`kmssink` 只接受"缩小"，放大要走重建管线。所以"隐藏视频"和"恢复视频"是两条完全不同的路径：

```c
if (want_hide) {
    /* 设 1x1 —— 这一半可以直接改属性 */
} else if (g_w.rect_hidden) {
    /* 恢复：投递一次原地 seek 重建管线，新管线直接带上正确矩形 */
    g_w.seek_target = g_w.last_pos_ms;
    g_w.seek_pending = 1;
    g_w.seek_last_ms = app_now_ms() - PB2_SEEK_DEBOUNCE_MS - 1;
}
```

还有一个细节：`w = 0` 或 `h = 0` 会被 kmssink 回退成**全屏**。要隐藏必须用 `1x1`，
并且 `x/y` 不要加任何面板偏移。

## 坑 4：起播要两段式，不能用 `get_state` 判 preroll

"点播起播时画面先黑屏，音频从第 0.5 秒开始，画面帧率很低，两秒后才正常"——这是迟到帧
被成批丢弃造成的。

根因：管线 `PLAYING` 后 sink 立刻完成 preroll、时钟起跑，而视频分支（`mppvideodec` 首帧
+ queue 排空）要晚 900~4800ms 才出第一帧。这批帧的 PTS 相对已经跑起来的时钟全部
"迟到 > 40ms"，被 `max-lateness` + `qos` 成批丢掉。实测 `buffer late, dropping` 出现
**279 次，全部在 `<vsink>`**，集中在 94ms 内丢光一整个 GOP。

正确做法是**先缓冲，再同时起播**：

```c
/* 1) 先 PAUSED —— 时钟不走，没有任何帧会迟到 */
sr = gst_element_set_state(g_w.pipe, GST_STATE_PAUSED);
if (sr == GST_STATE_CHANGE_FAILURE) { /* OPEN 失败 */ }

/* 2) 轮询到两个分支都就绪 —— 注意不能用 get_state 判 preroll！
      分支 sink 是 pad-added 动态建的，desc 里没有 sink，
      set_state(PAUSED) 会立刻返回 SUCCESS，那时一个分支都还没建。 */
long long t_gate = app_now_ms();
int ready = 0;
while (app_now_ms() - t_gate < gate_ms) {
    int v_ok = g_w.vbranch &&
               atomic_load_explicit(&g_w.probe.frames, memory_order_relaxed) > 0;
    int a_ok = want_audio ? g_w.abranch : 1;
    if (v_ok && a_ok) { ready = 1; break; }
    app_sleep_ms(20);
}
app_logf("start gate: %s in %lldms (v=%d/%lld frames=%lld a=%d/%d want_audio=%d)", ...);

/* 3) 超时也要放行 —— 绝不能让用户永久等 */
if (!ready) { /* 打 timeout 日志，继续走 PLAYING */ }

/* 4) 再 PLAYING */
sr = gst_element_set_state(g_w.pipe, GST_STATE_PLAYING);
```

实测缓冲门大约 1700~2900ms 就绪。起播从首帧起采 3 秒：

```text
vid_changes=90  vid_hist 30-45=87  big_gaps=0  vid_fps=30.00
```

对比不等首帧就开测的窗口只有 `vid_fps=7.67`。

**seek 不走这个门**。seek 时 sink 里积压的是"从 K 到 T"那几秒的帧，给宽限让它们一帧不丢
地按序翻页，等于先把 K..T 全播一遍才轮到用户要看的画面——实测 seek 落点从 412ms 恶化到
2081ms。seek 就应该让 sink 丢迟到帧直接跳到 T。

## 坑 5：迟到宽限只能给 OPEN，不能给 seek

OPEN 时需要一小段"不判迟到"的宽限窗口，因为首帧本身要 900~4800ms。但宽限不能用
"从 open 时刻算 3 秒"——首帧 4801ms 才到时宽限已经过期了，等于白改。要在等首帧期间
持续顺延：

```c
static void update_late_policy(void) {
    if (g_w.late_grace_until_ms <= 0) return;
    if (g_w.frame_epoch_ms > 0) {          /* 还在等首帧 */
        if (!g_w.late_grace_armed) { g_w.late_grace_until_ms = 0; return; }
        g_w.late_grace_until_ms = app_now_ms() + grace_ms;   /* 顺延 */
        return;
    }
    if (app_now_ms() >= g_w.late_grace_until_ms) {
        gst_util_set_object_arg(g_w.vsink, "max-lateness", "40");
        gst_util_set_object_arg(g_w.vsink, "qos", "true");
        g_w.late_grace_until_ms = 0;
        app_logf("late grace: expired -> max-lateness=40 qos=true (strict)");
    }
}
```

三个关键点缺一不可：

1. **desc 里写严格的 `max-lateness=40 qos=true`**，不要写死 `-1/false`。因为 desc 在 seek
   重建时也会用，而 seek 不能有宽限。
2. **宽限靠运行时 `gst_util_set_object_arg` 下发**，只在 `do_open` 成功末尾 arm。
3. **`late_grace_armed` 标志区分"这次是起播"还是"这次是 seek"**。只有 OPEN 置 1。
4. `RESUME` 时必须清掉宽限——暂停恢复要靠严格策略丢掉积压，否则一秒涌出 400+ 帧。

实测效果：`buffer late, dropping` 从 279 次降到 1 次。

## 直播：完全不同的路径

直播不能走 ffmpeg。FLV 的 PTS 是绝对值（实测约 2854014 秒），把这样的流落到 FIFO 上做
流式 demux，`matroskademux`/`tsdemux` 都无法处理，preroll 直接卡死、永远不出帧。

```c
if (is_live || durationMs <= 0) {
    /* 直连：souphttpsrc 吃 FLV，flvdemux 拆 */
    desc = "souphttpsrc location=%s is-live=true ! "
           "queue max-size-buffers=%d max-size-bytes=%d max-size-time=%lld ! "
           "flvdemux name=vd";
    /* 位置不能用 gst_element_query_position，用会话已播时长 */
    g_w.last_pos_ms = live_position_ms();
}
```

直播相关常量与取舍：

- 视频 queue：`2s / 3MB` 是点播的值，直播要给到 **4s / 3MB**。直播源的 GOP 通常 1~2 秒，
  加上 CDN 抖动，2s 余量一次小抖动就抽干。
- 直播**不能**设 `max-lateness=-1` + `qos=false`。看起来"不丢帧更流畅"，实际会把坏帧
  全放出来，闪屏更严重。直播要的是**丢帧**。
- 直播卡顿要能自动重连：连续 6 秒无帧判定为流断了，按 0/2/4/6/8 秒退避重试，最多 5 次。

## 硬解：`mppvideodec` 的 `ignore-error` 是最大陷阱

"直播时每 2~5 秒闪一次，部分区域闪白一帧"的根因。

`mppvideodec` 的 **`ignore-error` 出厂默认是 `true`**，`max-errors` 默认 `-1`。这意味着
解码器解坏了不报错、直接把坏帧吐出来。直播丢包后参考帧缺失，解出来的就是"部分区域错位
/发白"的坏帧，照常上屏一帧——用户看到的就是闪屏。

```c
/* 正确：让解码器丢坏帧，不要放出来 */
const char *mpperr = " ignore-error=false max-errors=16";
/* 插到 mppvideodec 元素后面 */
"... ! mppvideodec rotation=%d%s ! ..."
```

为什么点播没问题、只有直播闪？因为点播不丢包，参考帧永远完整。软解也不闪，因为软解自己
会丢坏帧。**这个坑只有在"直播 + 硬解"这个组合下才暴露**，所以特别难查。

排查时值得知道：`mppvideodec` 的 GStreamer debug **category 名不叫 `mppvideodec`**。
`gst-inspect-1.0` 显示的 factory 名是它，但 debug category 是别的名字，用
`GST_DEBUG=mppvideodec:5` 抓不到任何东西。

## 音频分支的 desc

```c
/* 扬声器 */
"queue max-size-buffers=32 ! aacparse ! faad ! audioconvert ! audioresample ! "
"audio/x-raw,rate=48000,channels=2 ! volume name=vvol ! "
"alsasink name=asink buffer-time=30000 latency-time=10000"

/* 蓝牙 A2DP */
"queue max-size-buffers=32 ! aacparse ! faad ! audioconvert ! audioresample ! "
"audio/x-raw,rate=48000,channels=2 ! volume name=vvol ! "
"alsasink name=asink device=bluealsa:DEV=<MAC>,PROFILE=a2dp "
"buffer-time=30000 latency-time=10000"
```

`volume name=vvol` 必须存在——即使蓝牙路由下软件增益固定 1.0，留着一个具名元素方便
后续插桩。`buffer-time` 对蓝牙和扬声器都用 30ms/10ms，**不要因为"蓝牙要抗抖动"就放大**
（见 [audio-route.md](audio-route.md)）。

## probe 挂在哪

```c
/* 视频：挂在 sink pad，能拿到送显前的帧 */
gst_pad_add_probe(vsink_sinkpad, GST_PAD_PROBE_TYPE_BUFFER, on_video_buffer, NULL, NULL);

/* 音频：挂在 demuxer 的 src pad，不是 ghost sink pad */
gst_pad_add_probe(audio_pad, GST_PAD_PROBE_TYPE_BUFFER,      on_audio_buffer, NULL, NULL);
gst_pad_add_probe(audio_pad, GST_PAD_PROBE_TYPE_BUFFER_LIST, on_audio_list,   NULL, NULL);
```

- `GST_PAD_PROBE_TYPE_BUFFER_LIST` 的值是 **32**（不是 128，128 是 EVENT_UPSTREAM）。
- `GstPadProbeInfo` 是不透明类型，不能读 `info->type`。必须拆成两个回调。
- 音频挂在 ghost sink pad 上只能收到 1~2 个 buffer，拿不到真实统计。
- **音频 probe 只统计、绝不 DROP**。曾经写过 `frames==0 → DROP`，把几千个音频 buffer
  丢掉，导致音频分支再也 preroll 不起来、PCM 变 closed，表现是"开头没声音"。

## 转屏与坐标

面板是 480×960 竖屏，逻辑画布是 960×266 横屏。视频平面在**面板坐标系**里，需要换算。

实测的映射（逻辑 (x, y, w, h) → 面板 (x', y', w', h')，对应 rotation=1 / 顺时针 90°）：

```c
/* w 是逻辑宽度、h 是逻辑高度；输出时两者互换 */
x' = y
y' = PANEL_HEIGHT - (x + w)
w' = h
h' = w
```

注意 `PANEL_HEIGHT` 用的是**面板**高度（960），不是逻辑高度（266）。写错这一项的症状是
视频整体上下偏移几百像素。

还要注意**面板横向有一个死区偏移**：这台设备上可显示区域不是从 `x=0` 开始，而要从
`x=107` 起。所以真正下发的 `x'` 要再加一个常量（实测 107）。验证：

```text
逻辑视频区 (x=213, y=0, w=469, h=240)，16:9 片源信箱适配后
  → 逻辑 (234, 0, 427, 240)
  → 面板 (0+107, 299, 240, 427)
  → debugfs 实测 dst: pos[107, 299] rect[240 x 427]   ✅
```

**隐藏视频（1x1）时不要加这个偏移**。加了你会在屏幕左侧残留一条 107px 宽的条带。
判据就是 `w>1 || h>1` 时才加。

原则：**只写一份转换函数，JS 和 native 都用它的结果**。两处各算一遍必然不一致，症状是
"视频位置差几个像素"或"隐藏后残留一条边"。这套映射建议写成纯函数并加单元测试——
旋转变换错了极难靠肉眼排查。

验证方法只有一个：改完读 `/sys/kernel/debug/dri/0/summary`，看视频平面那行的
`dst: pos[x, y] rect[w x h]` 是不是你想要的值。读 JS 返回值没有意义。
