# 分层架构、线程模型与状态机

## 为什么必须要 native

Falcon 的高层 JSAPI 覆盖不到"自己拉一条 GStreamer 管线"这件事。当你需要
`kmssink`（直接把画面送到 DRM 平面）、需要 `mppvideodec`（Rockchip 硬解）、需要控制
`ffmpeg` 子进程时，只能写 native `.so`，通过 `custom_init_jsapis()` 把能力导出给 JS。

代价是：**你要自己负责线程、生命周期、跨语言错误语义**。这就是本文的全部内容。

## 目录布局

```text
project/
  package.json                  appid、版本、构建脚本
  native/
    build.ps1                   （或 build.sh）交叉编译入口，锁死工具链
    iot-miniapp-sdk/            SDK 头文件
    csrc/
      jsapi_<plugin>.c          JSAPI 入口、参数校验、返回包装
      session/
        session.c               管线与 worker（本文件通常最大）
        cmd.c / cmd.h           fork/exec 执行系统命令（amixer、ffmpeg）
      source/
        src_http.c              ffmpeg argv 构造、FIFO 生命周期
      audio/
        audio_out.c             ALSA/混音器路由
      core/
        queue.c / queue.h       命令队列、统计结构（stats）
        log.c                   日志开关与落盘
      third_party/
        gst_min.h               手写的 GStreamer 最小 extern 声明
  libs/
    libjsapi_<plugin>.so        已验证的产物
  src/
    app.js / app.json           App 生命周期、viewport
    base-page.js                统一订阅与 timer 清理
    pages/
    services/                   适配层：能力检测、参数转换、错误归一
  test/
    run.js                      纯逻辑单元测试（Node 跑，不需要设备）
```

## 线程模型

```text
JS 线程 (QuickJS)                 worker 线程                  ffmpeg 进程
─────────────────                 ────────────                 ────────────
js_start()  ──enqueue(START)──►   worker loop
                                  ├─ do_open()
                                  │   ├─ fork/exec ffmpeg ──────────────► 写 FIFO
                                  │   └─ gst_parse_bin_from_description
                                  │        ↳ pad-added → 建分支
                                  ├─ tick: 刷新统计、检查看门狗
                                  ├─ 消费 cmd（SEEK/PAUSE/RESUME/SET_RECT）
                                  └─ 退出：teardown → patch IDLE → worker_done=1
js_status() ──读 stats（原子）◄──  app_stats_patch_*()
```

规则：

1. **JS 线程只做两件事**：`enqueue` 和读 `stats`。两者都是无锁或短暂持锁的。
2. **worker 是唯一能动管线的人**。任何 `gst_element_set_state` 只在 worker 里调。
3. **`stats` 用原子写**，JS 侧读到的永远是"某个时刻的快照"，不要求一致。
4. **不要用 `pthread_cancel`**。用 `quit` 命令 + worker 自己检查退出标志。

## 命令队列

队列元素是固定大小的 POD：

```c
typedef struct {
    int type;            /* PB2_CMD_OPEN / SEEK / PAUSE / RESUME / SET_RECT / STOP / QUIT */
    int i0, i1, i2, i3;  /* 整型参数 */
    long long i64;       /* seek target / duration */
    char s0[256];        /* URL、rect 字符串等 */
} app_cmd_t;
```

要点：

- **`SET_RECT` 用字符串传**（`"<x,y,w,h>"`），因为它最终要变成一个 GObject 属性值，
  在 native 里格式化一次就够了，不要在 JS 和 native 之间来回拼。
- **seek 用覆盖式 debounce**：用户拖动进度条会连发几十条 `SEEK`。不要在 JS 端限流，
  在 worker 端做——收到新 `SEEK` 就覆盖 `g_w.seek_target`，等输入安静
  `PB2_SEEK_DEBOUNCE_MS` 后才真正重建。实测这样一次拖动只重建一次管线。
- **不要给队列设很小的深度**。用户连点三个视频、每点一次先发 `STOP` 再发 `OPEN`，
  队列浅了会丢命令，表现是"点了没反应"。

## 状态机

```text
                    ┌──────────────── STOP / QUIT ────────────────┐
                    ▼                                             │
IDLE ──OPEN──► OPENING ──首帧──► PLAYING ⇄ PAUSED ──SEEK──► OPENING
  ▲                │                │         │                 │
  └──── 失败 ──────┴────────────────┴─────────┴─────────────────┘
                    teardown_pipeline()
```

必须满足：

- 每次进入 `OPENING` 都自增一个 epoch（`frame_epoch_ms = now`）。首帧到达时如果 epoch
  已经变了，说明这是**上一条管线**的迟到帧，必须丢弃。
- `teardown_pipeline()` 幂等：`g_w.pipe` 为空时直接返回。
- `PAUSE` / `RESUME` 不重建管线，只 `set_state`。`RESUME` 时要 `t_start += 暂停时长`，
  否则位置统计会漂。
- `SET_RECT` 在 `PAUSED` 下的处理特殊，见 [pipeline-video.md](pipeline-video.md)。

## teardown 顺序（这条错了会泄漏 KMS plane）

```c
static void teardown_pipeline_ex(void) {
    if (!g_w.pipe) return;

    /* 1) 先转 NULL 并**等到真的到 NULL**。set_state 是异步的。 */
    app_logf("teardown: set_state(NULL)");
    gst_element_set_state(g_w.pipe, GST_STATE_NULL);
    GstState cur = 0, pend = 0;
    GstStateChangeReturn gr = gst_element_get_state(g_w.pipe, &cur, &pend,
                                                    (GstClockTime)3000 * GST_MSECOND);
    if (gr == GST_STATE_CHANGE_FAILURE || cur != GST_STATE_NULL) {
        /* 重试一次 —— 实测偶尔第一次会得到 gr=0 而 cur 还是 READY */
        app_logf("teardown: WARN state did not reach NULL (gr=%d cur=%d) -> retry", gr, cur);
        gst_element_set_state(g_w.pipe, GST_STATE_NULL);
        gst_element_get_state(g_w.pipe, &cur, &pend, (GstClockTime)2000 * GST_MSECOND);
    }

    /* 2) unref 之后还要留一段空窗 */
    gst_object_unref(g_w.pipe);
    g_w.pipe = NULL;
    app_sleep_ms(120);

    /* 3) 清句柄 */
    g_w.vsink = g_w.asink = g_w.vbranch = g_w.abranch = NULL;
    g_w.last_pos_ms = 0;
}
```

**绝对不要在这里先 `set_state(PAUSED)`**。这是我踩过的坑：加了"先 PAUSED 等排空"之后，
`set_state(NULL)` 被取消掉，`get_state` 返回 `gr=0 cur=2(READY) pend=3`，管线永远停在
READY，`kmssink` 的 dispose 不执行，**DRM plane 永远不还给内核**。之后的管线申请不到
plane，表现是"seek 后黑屏、日志显示帧在涨但屏幕不动"。

`unref` 之后的 120ms 空窗也不能省：`kmssink` 申请不到 plane 时**不会重试**，
必须等内核的 atomic commit 完成。

## 别 unref 你可能不拥有的引用

```c
/* 错：gst_parse_bin_from_description 返回的是 floating ref，
   gst_bin_add 会 sink 掉它。你 unref 就是双重释放。 */
gst_object_unref(g_w.vsink);

/* 对：如果你需要长期持有，就 ref 一份并成对 unref：
   gst_object_ref(bin) 之后才归你 unref。 */
```

误加 unref 的后果我实测过：分支 bin 提前析构，plane 76 连续 22 秒从 `debugfs` 里消失，
系统里没有任何报错，只是画面不动。

## 进程管理

用 `posix_spawn` 而不是 `system()`，参数走数组不要拼 shell 字符串：

```c
pid_t pid;
posix_spawn_file_actions_t fa;
posix_spawn_file_actions_init(&fa);
/* stdout 直接指到 FIFO 写端，不要 dup2 到临时 fd */
posix_spawn_file_actions_addopen(&fa, STDOUT_FILENO, fifo_path, O_WRONLY, 0);
posix_spawn_file_actions_addopen(&fa, STDERR_FILENO, errlog_path, O_WRONLY | O_CREAT | O_TRUNC, 0600);
posix_spawn(&pid, "/usr/bin/ffmpeg", &fa, NULL, argv, environ);
```

- **stderr 一定要落盘**。默认丢 `/dev/null` 会让你在"ffmpeg 秒退但不知道为什么"上耗掉一天。
  实测有效路径：`/tmp/<app>/err-<fifo名>`，每次 spawn 覆盖写。
- **`SIGCHLD` 可能被宿主设成 `SIG_IGN`**，此时 `waitpid` 恒返回 `ECHILD`。
  不要把它当失败——用 `/proc/<pid>` 兜底判断进程是否还在。日志里会看到
  `sigchld: ECHILD (host kept SIGCHLD=SIG_IGN) -> already reaped, rc unknown`，这是正常的。
- 父进程必须持有一个 FIFO 的 `O_RDWR` 句柄（`hold_fd`），否则写端一关读端立刻 EOF，
  demuxer 会在播完前收到 EOS。播放结束前不要关它。

## 跨语言错误语义

JS 侧的 API 一律返回 `{ ok: bool, code: int, msg: string, ... }`，不要抛异常穿越 FFI 边界。

```c
static JSValue js_seek(JSContext *ctx, JSValueConst this_val, int argc, JSValueConst *argv) {
    int ms = 0;
    if (JS_ToInt32(ctx, &ms, argv[0])) return mk_err(ctx, PB2_E_BADARG, "seek: bad ms");
    /* 只投递，不等结果 */
    app_cmd_t c = { .type = PB2_CMD_SEEK, .i64 = ms };
    app_cmdq_push(&g_w.q, &c);
    return mk_ok(ctx);
}
```

约定：

- **投递类 API 立即返回 `{ok:true}`**，不代表操作已完成。完成与否去 `status()` 里看。
- **`status()` 是唯一真相源**，返回原子快照：`state / frames / positionMs / audioBytes /
  seekPending / vbranch / abranch / error`。
- 错误码要稳定、要能在 JS 里 switch。不要返回 `errno` 或 GStreamer 的枚举值。

## 自愈要看门狗，不要看计数器

任何"我以为在播"的判断都可能错。看门狗必须同时满足两个条件才动作，否则会把正常缓冲误判成故障：

```c
/* 音频停摆：音频字节连续 8 秒不动，**同时**视频涨了 >=30 帧。
   只看音频不动是不够的 —— 整体暂停时它本来就不动。 */
if (ab == g_w.audio_stall_ab) {
    if (now - g_w.audio_stall_ms >= PB2_AUDIO_STALL_MS &&
        frames - g_w.audio_stall_frames0 >= PB2_AUDIO_STALL_FRAMES) {
        /* 原地 seek 把它拉回来，每会话最多 2 次 */
    }
} else { 刷新基准 }
```

配额很重要。无上限自愈会在源本身坏掉时变成"无限重建管线"，比不修还糟。

## 测试策略

- **纯逻辑放 Node 跑**：坐标转换、状态机转移、错误码映射、配置归一化、同步判据。
  这些不需要设备，跑得又快，应该占测试的绝大多数。
- **真机只验"外部可观测的行为"**：平面几何、帧率、音频字节、进程存活。
- **不要给探针写"看着对就行"的断言**。我曾写过一条 `frames > frames0` 就认为 seek 完成，
  结果被旧管线的出帧骗过，把"进度条回弹"这个真 bug 判成了通过。
