# 音频：路由、音量、音画偏移、停摆

这套平台上音频的复杂度被严重低估。核心结论先说：

> **扬声器看 `card1 Master`，蓝牙看耳机自己的 AVRCP，两者互不相干。
> 配置文件里的 `spk_volume` 永远不能当权威。**

## 设备上的音频体系

```text
/data/cfg/sys_config.conf
    spk_volume=NN          ← 扬声器音量的"记录"
    ahs_volume=NN
    dhs_volume=NN
    bt_volume=NN           ← 蓝牙音量的"记录"

card 0 (rockchip-rk817)
    "Playback Path"        ← enum 控件：SPK / OFF
    "Master"               ← 这是输出通路的开关，不是音量

card 1 (rockchip-aw883xx)  ← 真正的扬声器功放
    numid=54 "Master"      ← 系统音量滑杆写的就是它

bluealsa（不占 ALSA card）
    "DEV=<MAC> - A2DP"     ← 耳机硬件音量（AVRCP absolute volume）
```

系统音量入口是 `hal-audio -s <0-100>`。真机实测它写什么、什么时候写：

| 目标 | 写入时机 | 是否可靠 |
|---|---|---|
| `card1 Master`（numid=54） | **立刻** | ✅ 唯一可信的"用户意图" |
| bluealsa AVRCP | **从不** | — |
| `sys_config.conf` 的 `spk_volume` | 约 **8~12 秒**后 | ❌ 不能当权威 |

`hal-audio -s N` 到 `card1 Master` 的映射是非线性的：`-s 50` → `176/255 = 69%`，
`-s 100` → `255 = 100%`。

> **重要**：一度以为 `hal-audio` 也会写 AVRCP，因为观察到 `-s 100` 之后耳机音量跟着变了。
> 后来在干净环境（先 `killall -9 <app>`）重测才发现：那是**播放器自己写的**，因果完全颠倒。
> 做这类因果判断前一定要先杀掉自己，确认观测对象是系统而不是自己。

## 三条会踩的坑

### 坑 1：把 `card1 Master` 当耳机音量

`card1 Master` 是**扬声器功放**的寄存器。切到蓝牙后它不再影响耳机输出，而且可以长期停在
任意值——比如上次用扬声器静音时的 0%。

实测到过的状态：

```text
card1 Master      : Playback 0   [0%]
bluealsa AVRCP    : Playback 56  [44%]
```

用户明明设了 44%，但代码读 Master 得到 0，于是把软件增益设成 0 → **完全没声音**，
而耳机硬件音量还是 44%。用户看到的现象是"连上耳机就没声"。

### 坑 2：起播时用扬声器的值覆盖耳机音量

修完坑 1 之后我改成"两条路由都读 Master 并镜像给耳机"，结果引出更糟的问题：

```text
用户操作：Master 保持 0（上次外放静音留下的），耳机单独调到 50%，点开视频
结果    ：起播读到 vol_pct=0 → apply_bt_avrcp_volume() → 耳机被写成 0%
```

用户的 50% 凭什么被你的 0% 改掉。而且系统不会帮你纠正——实测把 AVRCP 强制设成 0% 之后
等 15 秒，`spk_volume` 和 `Master` 纹丝不动。

### 坑 3：AVRCP 与 GStreamer `volume` 元素双重衰减

这是两个**串联**的增益级。两级都设 76% 实际是 58%；两级都设 20% 是 4%，几乎听不到。

而 `alsasink` 本身在蓝牙下已经把音量交给 AVRCP 了（`a2dp-volume` 协议），软件增益再乘一遍
就是纯损失。

## 正确的模型

```c
/* 起播：分路由取权威值，绝不反向覆盖 */
if (g_w.bt_present && g_w.adev[0]) {
    int av = app_mixer_volume_read(g_w.bt_ctl_name);   /* 读 AVRCP */
    if (av >= 0) {
        g_w.vol_pct = av;
        g_w.bt_vol_applied = av;                       /* 对齐，后续不会重复下发 */
        app_logf("audio: volume authority = AVRCP %d%% (slider baseline %d%%)", av, master);
    } else {
        g_w.bt_vol_applied = -1;
        app_logf("audio: AVRCP unreadable -> fallback to slider");
    }
    g_w.master_last = app_mixer_volume_read(NULL);     /* 记下滑杆基线 */
} else {
    g_w.vol_pct = app_mixer_volume_read(NULL);         /* 扬声器 = Master */
    g_w.bt_vol_applied = -1;
}
```

播放中的刷新（每秒一次）：

```c
int master = app_mixer_volume_read(NULL);

if (g_w.bt_present && g_w.adev[0]) {
    /* "用户动了滑杆"的唯一判据：Master 相对上次采样发生了变化 */
    int slider_moved = (master >= 0 && master != g_w.master_last);
    if (master >= 0) g_w.master_last = master;

    int av = g_w.bt_ctl_name[0] ? app_mixer_volume_read(g_w.bt_ctl_name) : -1;

    if (slider_moved) {
        /* 用户动滑杆 → 推给耳机。这是唯一会写耳机的路径。 */
        g_w.vol_pct = master;
        apply_volume_now();
        apply_bt_avrcp_volume();
    } else if (av >= 0 && av != g_w.vol_pct) {
        /* 用户按了耳机上的键 → 采纳，不要把你的值覆盖回去 */
        g_w.vol_pct = av;
        g_w.bt_vol_applied = av;
        apply_volume_now();
        app_logf("audio: bt volume adopted from AVRCP (headset key) %d%%", av);
    }
} else {
    int pct = (master >= 0) ? master
                            : read_sys_config_int("spk_volume", 100);
    g_w.master_last = master;
    if (pct >= 0 && pct != g_w.vol_pct) { g_w.vol_pct = pct; apply_volume_now(); }
}
```

软件增益：

```c
static void apply_volume_now(void) {
    if (!g_w.vvol) return;

    if (g_w.bt_present && g_w.adev[0]) {
        /* 蓝牙：增益固定 1.0，音量全交给 AVRCP，避免串联双衰减 */
        if (g_w.vol_applied == 100) return;
        g_object_set(g_w.vvol, "volume", 1.0, NULL);
        g_w.vol_applied = 100;
        app_logf("audio: volume slider %d%% -> bt: software gain kept 1.000 "
                 "(AVRCP handles it)", g_w.vol_pct);
        return;
    }
    if (g_w.vol_applied == g_w.vol_pct) return;
    g_object_set(g_w.vvol, "volume", (double)g_w.vol_pct / 100.0, NULL);
    g_w.vol_applied = g_w.vol_pct;
}
```

真机验证的四项（都应该过）：

| 测试 | 期望 |
|---|---|
| 耳机 50%、Master 0%，起播 | 耳机仍是 50%，日志 `adopted from AVRCP 50% (not overwriting)` |
| 播放中滑杆拉满 | 耳机跟着到 100% |
| 播放中按耳机按键调到 30% | 不被覆盖回去，代码采纳 30% |
| 播放中滑杆拉小 | 耳机跟着降 |

## 关于"只升不降"

我一度给 `apply_bt_avrcp_volume()` 加过保护：只在目标值比当前值**高**时才写。

初衷是防"按读错的值（滞后 10 秒的 conf）把耳机压静音"。但源头换成 Master 之后这个前提
就不存在了，而它带来一个明显的坏结果：**系统滑杆向下调完全失效**。

要防的不是"写入更低的值"，而是"读到错的值"。这两件事应该分别解决，不要用一个副作用很
大的保护去绕。

## 音画偏移：`ts-offset` 只能加在 `kmssink`

"连蓝牙后音频快于视频约 150ms"。

### 为什么不能加在 `alsasink`

`alsasink` 是管线的**时钟提供者**（`provide-clock` 默认 true）。实测把它关掉，视频会飙到
88~119fps——整条管线失去时间基准。给它加 `ts-offset` 等于把整条管线平移，视频跟着时钟走，
相对关系完全不变。

实测三档 `ts-offset`（0/150/300ms 加在 alsasink）：`avSkewMs` 毫无变化
（-1481/-1461/-1464），而且 `pauseProbe` 恢复后帧率飙到 99.30fps。

### 正确做法

```c
/* 视频 desc 追加 ts-offset，值取负 */
char tsoff[48];
if (g_w.adev[0] && g_w.bt_ts_offset_ns != 0) {
    snprintf(tsoff, sizeof(tsoff), "ts-offset=%lld", -g_w.bt_ts_offset_ns);
} else {
    tsoff[0] = '\0';
}
/* desc 里用 "%s %s %s " 拼接，分隔符由格式串负责 */
```

**踩过的坑**：`snprintf` 写成 `" ts-offset=%lld"`（自带前导空格），desc 用 `%s%s%s` 拼接，
结果 `ts-offset=-150000000` 和 `skip-vsync=true` 粘成一个值：

```text
parse_bin FAIL: could not set property "ts-offset" in element "kmssink"
                to "-150000000skip-vsync=true"
```

kmssink 属性解析失败 → **整个视频分支 parse_bin 失败** → 视频分支根本建不起来 → 只有蓝牙
路径会写 tsoff，所以现象是"插蓝牙就黑屏，拔了就正常"。这类"只在某条路由下坏"的症状，
优先怀疑字符串拼接。

另一个坑：别写 `" ts-offset=-%lld"` 然后把正数传进去。负数标定时会输出
`ts-offset=--2000000000`，gst 解析失败、属性保持 0，但日志看起来"设了"。

## 音量控件读写的实现

```c
/* 解析第一个 [NN%]，返回 0~100 或 -1 */
static int parse_pct(const char *buf) {
    const char *p = strchr(buf, '[');
    if (!p) return -1;
    int v = atoi(p + 1);
    return (v >= 0 && v <= 100) ? v : -1;
}

/* 走 bluealsa 或指定 card 读音量 */
int app_mixer_volume_read(const char *bt_ctl_name) {
    if (bt_ctl_name && bt_ctl_name[0]) {
        return mixer_get_pct_bluealsa(bt_ctl_name);   /* 读不到返回 -1 */
    }
    return mixer_get_pct(1, "Master");
}
```

- **读不到就返回 -1，不要 fallback 到别的路由**。让调用方决定要不要兜底，否则蓝牙读不到
  时会静默返回扬声器的值，又回到坑 1。
- enum 控件（`Playback Path`）解析 `Item0: 'SPK'` 里的单引号内容。
- pvolume 控件（`Master`）解析 `[NN%]`。
- **别用 `grep -o "'[A-Z_]*'"` 读 enum**。它会先匹配到 Capabilities 的 Items 列表，
  第一项通常是 `'OFF'`，你会误判成"通路是关的"。用 `grep 'Item0'`。

## 不要动不属于你的音量

播放器不拥有系统音量。这条听起来理所当然，但很容易破坏：

```c
/* 错：为了"确保出声"，把 Master 拉满 */
amixer -c 1 sset Master 100%

/* 错：为了"静音测试"，把 Master 压 0 —— 这会销毁用户的设置且无法还原 */
amixer -c 1 sset Master 0%
```

静音应该只关输出通路（`Playback Path = OFF`），不要动 Master。`Master` 是用户的音量，
压 0 之后你自己都还原不回去（原来的值没有地方记）。

检查一个用户是否静音，也**不能只看 `Playback Path`**——实测 `hal-audio -s 0` 之后
`Master` 是 0 而 `Playback Path` 仍然是 `SPK`。要看 Master 百分比：

```c
if (mixer_get_pct(1, "Master") == 0) {
    app_logf("mixer: master=0%% (user muted via system) -> leave path untouched");
    return 0;
}
```

## 音频停摆：必须靠看门狗，不能指望 ffmpeg 报错

"外放播放到一半声音没了，视频还在播"。

根因：点播的音视频来自**两台不同 CDN 主机**（实测视频和音频的 host 完全不同）。音频那路
TCP 中途断流，而：

- ffmpeg 默认既不报错也不重连；
- `-map 1:a?` 里的 `?` 让音频是**可选流**，连"流没了"都不会让 ffmpeg 失败；
- GStreamer 侧只是再也收不到音频包，管线状态一切正常。

证据是音频字节数冻结而视频帧数继续涨：

```text
17:13:03  frames=996   pos=24736  ab=173187    ← ab 最后一次增长
17:13:44  frames=2206  pos=65025  ab=173187    ← 41 秒没变
```

而 `err-*.log` 是 0 字节（ffmpeg 自己认为一切正常）。

### 治本一半：让 ffmpeg 重连

```c
static void add_reconnect_args(char **argv, int *ip) {
    argv[(*ip)++] = "-reconnect";
    argv[(*ip)++] = "1";
    argv[(*ip)++] = "-reconnect_streamed";
    argv[(*ip)++] = "1";
    argv[(*ip)++] = "-reconnect_delay_max";
    argv[(*ip)++] = "5";
    argv[(*ip)++] = "-reconnect_on_http_error";
    argv[(*ip)++] = "4xx,5xx";
}
```

**两个 `-i` 前面都要加**。`-reconnect` 等是 per-input 选项，只作用于紧跟其后的那个 `-i`。
只给第一路加，第二路就是裸奔——实测表现为第二路 CDN 直接 403。

**不要加 `-reconnect_at_eof`**。点播播完之后会从头重播，用户会看到视频循环。

### 治本另一半：看门狗

重连不能覆盖所有情况（比如服务端一直不给数据）。必须有一个能发现"音频不动了"的看门狗：

```c
#define PB2_AUDIO_STALL_MS      8000
#define PB2_AUDIO_STALL_FRAMES  30
#define PB2_AUDIO_STALL_HEALS   2

static void check_audio_stall(void) {
    /* ……各种早退：无管线、无音频分支、暂停、EOS、直播、还在等首帧、正在 seek、配额用尽…… */
    if (g_w.audio_stall_heals >= PB2_AUDIO_STALL_HEALS) return;

    unsigned long long ab = atomic_load_explicit(&g_w.probe.abytes, memory_order_relaxed);
    unsigned long long fr = atomic_load_explicit(&g_w.probe.frames, memory_order_relaxed);
    if (ab == 0) return;

    long long now = app_now_ms();
    if (ab != g_w.audio_stall_ab) {
        /* 动了 —— 只在冻结超过 1 秒时才打日志，否则每秒几十行 */
        long long frozen = g_w.audio_stall_ms ? (now - g_w.audio_stall_ms) : 0;
        if (frozen >= 1000) {
            app_logf("audio stall: resumed after %lldms (ab=%llu)", frozen, ab);
        }
        g_w.audio_stall_ab = ab;
        g_w.audio_stall_ms = now;
        g_w.audio_stall_frames0 = fr;
        return;
    }

    /* 双条件：音频冻住 8 秒，**并且**视频涨了 >=30 帧 */
    if (now - g_w.audio_stall_ms >= PB2_AUDIO_STALL_MS &&
        fr - g_w.audio_stall_frames0 >= PB2_AUDIO_STALL_FRAMES) {
        g_w.audio_stall_heals++;
        app_logf("audio stall: abytes frozen %llu for %lldms while video advanced "
                 "%llu frames -> in-place seek to re-sync audio (heal %d/%d, target=%lldms)",
                 ab, now - g_w.audio_stall_ms, fr - g_w.audio_stall_frames0,
                 g_w.audio_stall_heals, PB2_AUDIO_STALL_HEALS, g_w.last_pos_ms);
        g_w.seek_target = g_w.last_pos_ms;
        g_w.seek_pending = 1;
        g_w.seek_last_ms = app_now_ms() - PB2_SEEK_DEBOUNCE_MS - 1;   /* 立刻执行，不等 debounce */
    }
}
```

三个设计要点：

1. **双条件不可省**。只看"音频不动"会在用户暂停、缓冲、起播等场景误触发。
2. **配额不可省**。源本身坏掉时，无上限自愈会变成无限重建管线，比不修还糟。
3. **`do_seek` 里只复位前三个字段，不要清 `heals`**。重建期间音频字节本来就不动，
   清零会让看门狗把同一次故障算成两次，白白耗尽配额。

日志频率也要注意：第一版每个 tick（几十毫秒）都打一行 `resumed`，一分钟 5000+ 行，
把真事件淹没了。

## 路由切换：拔插蓝牙

蓝牙消失时重建管线换到扬声器：

```c
g_w.bt_present = 0;
g_w.adev[0] = '\0';
g_w.bt_ctl_name[0] = '\0';
g_w.bt_vol_applied = -1;
app_logf("audio: BT sink gone -> rebuild pipeline to use speaker");

if (g_w.paused) {
    g_w.route_pending = 1;       /* 暂停态延迟到恢复时再做 */
    app_logf("audio: route change deferred (paused) -> pending");
    return;
}
/* 点播：投递原地 seek 重建；直播：走 live_rebuild */
g_w.seek_target = g_w.last_pos_ms;
g_w.seek_pending = 1;
g_w.seek_last_ms = app_now_ms() - PB2_SEEK_DEBOUNCE_MS - 1;
```

**已知未解的问题**：这条路要走"重新拉流 → 重建管线"，实测黑屏 5~7 秒
（`SEEK first frame on screen +6994ms SLOW`）。根治方向是"只换 `alsasink` 的 device、
不重建整条管线"，但 `PLAYING` 下改 `device` 属性不生效，需要 pause→改→resume 并处理
pause 期间的状态，属于中等改动。如果你的场景对拔插耳机敏感，从这里开始做。

## 蓝牙探测的开销

`bluetoothctl devices` + `bluetoothctl info <MAC>` 每次约 50~60ms（两次 fork）。在 4 核
A35 上每秒探测一次会和 A2DP 编码抢 CPU，表现为播放卡顿。

分档节流：

```c
long long period = g_w.bt_present ? 5000 : 1000;   /* 已连上就 5 秒探一次 */
if (now - g_w.bt_check_ms < period) return;
```

另外注意 `bluealsa-aplay -L` 在这种固件上会**直接崩**（`Aborted`），不能用来枚举 PCM。
可靠的路径是 `bluetoothctl devices` 找已连设备，再拼 `bluealsa:DEV=<MAC>,PROFILE=a2dp`。
