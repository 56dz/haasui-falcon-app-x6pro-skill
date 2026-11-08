# native JSAPI 模块：交叉编译、加载契约与失败语义

## 能力边界：为什么这些事只能进 native

（与 [architecture.md](architecture.md) 的「为什么必须要 native」同题但视角不同：
那边讲"为什么架构要分层"，这里讲"到底哪些能力缺失、怎么补"。）

Falcon 的高层 JSAPI 只有一个出口 `$falcon.jsapi.http.request`，够发 JSON，但有四条硬边界：

| 能力 | 平台 JS | 后果 |
| --- | --- | --- |
| 裸请求体 | 对 `data` 字符串**二次 JSON 序列化** | 发不出裸 Base64 |
| 响应头 / 非 2xx 的 body | 响应头不可见；非 2xx 时 body 拿不到 | 判错只剩状态码 |
| WebSocket | 无 `WebSocket`、无 `fetch`、无 `XMLHttpRequest` | 信令通道只能自己实现 |
| crypto | 无 WebCrypto；QuickJS `bigNum:false` 连 BigInt 都没有 | MD5/SHA256/3DES/Base64 只能下沉 |

设备自带 `libcurl` 也救不了：`curl -V` 的 `Protocols:` 里**没有 `ws`/`wss`**。WS 只能自己写：`socket` → `SSL_connect` → `GET`+`Upgrade` → 判 `101` → 手搓 RFC 6455 帧（客户端必须掩码、收帧解掩码、`ping` 回 `pong`、7/16/64 位长度与 continuation）。

代价是：**线程、生命周期、跨语言错误语义都得你自己负责**。

## 交叉编译工具链：主机上没有编译器

```text
$ for c in gcc clang cc cl make pkg-config nm readelf objdump; do command -v $c; done   # 全部 none
$ env/pyzig/ziglang/zig.exe version
0.16.0
```

**主机只有 zig**，`cc` 就是 `zig cc`，进脚本：

```powershell
$zig = $env:ZIG_EXE; if (-not $zig) { $zig = '..\env\pyzig\ziglang\zig.exe' }
if (-not (Test-Path $zig)) { throw 'zig not found: set ZIG_EXE' }
& $zig cc -target aarch64-linux-gnu.2.17 -O2 -g0 -Wl,-s -fPIC -shared -fvisibility=hidden `
  -I native/iot-miniapp-sdk/include -I native/vendor/openssl-include `
  -o libs/libjsapi_cgnative.so native/csrc/cgnative/cgnative.c `
  stubs-aarch64/libquickjs.so stubs-aarch64/libssl.so.1.1 `
  stubs-aarch64/libcrypto.so.1.1 stubs-aarch64/libcurl.so.4 `
  -lpthread -Wl,-soname,libjsapi_cgnative.so
if ($LASTEXITCODE -ne 0) { throw "build fail" }        # ← 必须查
```

- **`-target aarch64-linux-gnu.2.17`** 别省（RK3562/aarch64/Buildroot 2021.11，`2.17` 是 glibc 下限）。
- **`-Wl,-soname` 与文件名一致**；`-fvisibility=hidden` 让导出面只剩 `JQUICK_EXPORT`。
- **不许 `-static`、不许 C++**：混进 C++ 后 `DT_NEEDED` 多出 `libstdc++.so.6`，设备没这版本就加载失败。
- **`$LASTEXITCODE` 不查就拿旧 `.so` 部署**，表现是"改了没生效"。

## 链接策略：stub 动态库，符号故意留未解析

`zig cc` 的 sysroot 里没有任何第三方头/库：

```text
$ zig cc ... -lssl
error: unable to find dynamic system library 'ssl' using strategy 'paths_first'. searched paths: none
$ #include <curl/curl.h>
t3.c:1:10: fatal error: 'curl/curl.h' file not found
```

设备上更没开发环境：`pkg-config`/`gcc` → `none`，`/usr/include/{gstreamer-1.0,openssl}` 都 `No such file or directory`。**交叉编译 GStreamer/openssl 是死路。**

做法：**不链真库，链"设备库的原样副本"，`.so` 里对这些符号保持未定义**，运行时由宿主（已加载 `libquickjs/libssl/libcurl/libgstreamer`）解析。`native/stubs-aarch64/`（10 个）与 `stubs-aarch64-gst/`（5 个）就是设备 `/usr/lib/` 上的文件，**md5 逐一相同**（自己 `md5sum` 对账）。

拿原库当 stub 比手写空函数可靠 —— 签名、结构体布局、枚举天然一致，不会出现"头文件说 8 字节、真库 16 字节"的 ABI 错位。**代价是链接期必须有头文件**：自带头文件（`native/vendor/openssl-include/` 放同版本 1.1.1 的 `openssl/*.h`），或完全手写声明（libcurl 就这样）：

```c
typedef void CURL;
extern int curl_global_init(long);
extern CURL *curl_easy_init(void);
extern int curl_easy_setopt(CURL *, int, ...);
extern int curl_easy_perform(CURL *);
extern const char *curl_easy_strerror(int);
/* 常量抄一次就够：URL 10002 POSTFIELDS 10015 HTTPHEADER 10023
   CUSTOMREQUEST 10036 WRITEFUNCTION 20011 POSTFIELDSIZE 60
   SSL_VERIFYPEER 64 SSL_VERIFYHOST 81 NOSIGNAL 99 TIMEOUT_MS 155 */
```

发**裸 body**（本模块存在的核心理由）就靠这三行，多线程另加 `CURLOPT_NOSIGNAL 1`、`curl_global_init` 用 `pthread_once`：

```c
curl_easy_setopt(h, CURLOPT_POSTFIELDS, body);
curl_easy_setopt(h, CURLOPT_POSTFIELDSIZE, (long)bodyLen);   /* 二进制安全，必须给长度 */
if (strcmp(method, "POST")) curl_easy_setopt(h, CURLOPT_CUSTOMREQUEST, method);
```

## ★ 未解析符号会让整个 `.so` 静默加载失败

**只要有一个符号在宿主符号空间里找不到，整个模块就加载失败**，JS 侧只看到"未找到模块"。探针实测（自编 aarch64 `dlopen` 程序）：

```text
$ dlprobe /userdisk/miniapp/data/probe3.so now
dlopen FAIL: /userdisk/miniapp/data/probe3.so: undefined symbol: g_zzz_not_exist_anywhere
$ dlprobe /userdisk/miniapp/data/probe3.so lazy
dlopen FAIL: /userdisk/miniapp/data/probe3.so: undefined symbol: g_zzz_not_exist_anywhere
```

**`RTLD_LAZY` 也救不了** —— zig 产物带 `DF_BIND_NOW`（实测 `.dynamic` 里 `FLAGS=0x8`、`FLAGS_1=0x1`），重定位前全部绑定完。

处置顺序：① `.dynsym` 列出全部未解析符号 → ② 逐个确认在设备 `libc/libssl/libcrypto/libcurl/libglib/libgstreamer/libfalcon` 上能找到 → ③ 找不到的要么链接期补 stub、要么运行期 `dlsym` 取址 → ④ **绝不直接调用"可能不存在"的符号**。

主机没 `nm` 就自己读 `.dynsym`（`SHT_DYNSYM`，type=11）：24 字节一条 `<IBBHQQ`，取 `st_name` 拼 `.dynstr`，筛 `st_info>>4 == 1`（GLOBAL）且 `st_shndx == 0`（未定义）。

反例（别学）：代码里曾为"`g_free` 在本设备 libglib 查不到"而**故意不 free** 几十字节。`dlsym` 复测 `g_free … -> FOUND`、`g_zzz_nope -> MISSING` —— `g_free` **是**存在的（当初只 `strings` 了一眼）。规则成立但这个例子不成立：引用符号前按上面 4 步实测，别凭印象。

> 段表：能被 `dlopen` 加载的库一定有完整 `.dynsym`（来自 `PT_LOAD`），**与 section header 无关**；做符号检查认 `.dynsym`，不要依赖 `readelf -S`。只有静态产物（`.a`/`.o`）或 `strip` 过的库才可能段表缺失。本机量过 `libc/libssl/libcurl/libglib/libgstreamer/libfalcon` 的 `shnum` 全部 ≥20、`shstrndx` 有效 —— `docs/PLATFORM.md:47` 那条"section header 被裁剪"是早期推测，非实测。

## ★ `-Werror` 与 `__DATE__`

`zig cc` 默认把部分警告当错误 —— `const char *d(void){ return __DATE__; }` 直接失败：
`t.c:2:29: error: expansion of date or time macro is not reproducible [-Werror,-Wdate-time]`。

**禁止 `__DATE__`/`__TIME__` 打版本号**（`-Wno-error=date-time` 能降级，但等于放宽全局错误门槛），用字面量宏：

```c
#define CG_VERSION    "cgnative 1.0.0"
#define CG_BUILD_DATE "2026-11-06"   /* 字面量；__DATE__ 触 -Wdate-time 直接编译失败 */
snprintf(tmp, sizeof(tmp), "%s %s", CG_VERSION, CG_BUILD_DATE);
```

## 导出表与 C ABI

模板是 `PenBili/native/csrc/httpjson.c`（纯 C）。骨架 = 一张导出表 + 两个模块钩子：

```c
#include <jquick_config.h>
#include <jsmodules/JSCModuleExtension.h>
#include <quickjs/quickjs.h>

#define CG_EXPORTS(F)                    /* 25 个导出 */
    F("postSync", js_post_sync, 4) F("wsOpen", js_ws_open, 3) F("wsState", js_ws_state, 1)

static int mod_init(JSContext *ctx, JSModuleDef *m) {
    JSValue def = JS_NewObject(ctx);
#define CG_DEF(n, fn, l) do {                                            \
        JS_SetPropertyStr(ctx, def, n, JS_NewCFunction(ctx, fn, n, l));   \
        JS_SetModuleExport(ctx, m, n, JS_NewCFunction(ctx, fn, n, l)); } while (0);
    CG_EXPORTS(CG_DEF)
    JS_SetModuleExport(ctx, m, "default", def);   /* 少了它 mod.default 是 undefined */
    return 0;
}
static JSModuleDef *mod_load(JSContext *ctx, const char *name) {
    if (strcmp(name, "cgnative")) return NULL;
    JSModuleDef *m = JS_NewCModule(ctx, name, mod_init);
#define CG_ADD(n, f, l) JS_AddModuleExport(ctx, m, n);
    CG_EXPORTS(CG_ADD); JS_AddModuleExport(ctx, m, "default");
    return m;
}
extern void registerCModuleLoader(const char *, LoadCModuleFunction);
JQUICK_EXPORT void custom_init_jsapis(void) {
    registerCModuleLoader("cgnative", &mod_load);
}
```

- **每个名字注册两次**：进 `def` 对象 + `JS_SetModuleExport`。只做前者，`import { md5Hex } from 'cgnative'` 拿不到。
- **`"default"` 必须导出**，否则 `mod.default.xxx` 是 undefined。
- **`JQUICK_EXPORT` 是唯一导出面**（`.dynsym` 里只有 `custom_init_jsapis` 一个导出符号，其余 25 个只是注册用的字符串）。纯 C，不写 `extern "C"`。
- **`registerCModuleLoader` 由宿主提供**：本模块里它是未定义符号，所以拿干净进程 `dlopen` 会报 `undefined symbol: registerCModuleLoader`，那是**正常**的。

## 所有返回值都是 JSON 字符串

跨语言只走一种 ABI：返回 `JS_NewString` 出来的 JSON 文本，不返回对象字面量（native 造对象要连续 `JS_SetPropertyStr` + 到处 `JS_FreeValue`，错误路径难统一）。

```text
postSync/requestSync/takeResult
  {"status":200,"curlCode":0,"ms":198,"error":null,"body":"..."}
wsState(id)
  {"state":"open","msg":"open","recv":0,"sent":552,"phase":"open","stageCode":6,"httpStatus":101,
   "closeCode":0,"closeReason":"","errno":0,"errstr":"","frames":1}
wsPoll(id)   -> ["{...}"]   纯数组，读后清空，无消息 []
wsPollEx(id) -> {"msgs":[...],"total":7,"recv":1234,"sent":552}
wsList() -> [1,2,3]     wsLastFrame(id) -> {"len":552,"op":1,"head":64,"hex":"81fe02…"}
```

**`error` 是字符串不是对象**（成功为 `null`）：`"http#404"` / `"curl#28: Timeout was reached"`。别写 `err.code`。`wsPoll` 必须保持纯数组（调用方用 `Array.isArray`），要 `total` 就用 `wsPollEx`。

JS 侧统一包一层：

```js
function callJSON(mod, fn, args) {
  let raw;
  try { raw = mod[fn].apply(mod, args || []); } catch (e) { return { __error: String(e) }; }
  if (raw == null) return null;
  if (typeof raw === 'object') return raw;
  try { return JSON.parse(raw); } catch (e) { return { __raw: raw }; }
}
```

`body` 是任意字节、可能非 UTF-8：必须 `JS_NewStringLen(ctx, p, len)`，写进 JSON 前转义、非法字节转 `\u00XX`，否则 `JSON.parse` 抛异常。

## 错误语义要稳定：`stageCode` 1–12

**不要把 `errno` 透传给 JS**（平台细节，换机型就变），用一档稳定阶段码：

| stageCode | phase | 含义 | 播放页处理 |
| --- | --- | --- | --- |
| 0 | closed | 未开始 | — |
| 1 | dns | DNS 解析失败 | 提示网络，重试 |
| 2 | tcp | TCP 连接失败（含 ECONNREFUSED） | 机位过期/被封，换机位 |
| 3 | tls | TLS 握手失败 | 重试 |
| 4 | upgrade | HTTP 非 101 | 网关故障，重试（看 `httpStatus`） |
| 5 | upgrade | 升级响应解析失败 | 协议不符，换机位 |
| 6 | open | 已建立 | 正常 |
| 7 | closed | 对端 close 帧关闭 | 看 `closeCode`/`closeReason` |
| 8 | closed | 本地 `wsClose` | 用户退出 |
| 9 | closed | I/O 错误断开 | 重连 |
| 10 | dns | URL 非法 | 代码 bug |
| 11 | — | 握手超时 | 重试 |
| **12** | **tcp** | **TCP 连接超时** | **网络不通，重试** |

**12 必须和 2 分开**：阻塞 `connect()` 下超时与被拒都只给 `-1` + `EINPROGRESS(115)`，分不出"机位没了"和"网络不通"。非阻塞 `connect`+`poll`+`SO_ERROR` 才拿得到真值：

```c
fcntl(fd, F_SETFL, fcntl(fd, F_GETFL, 0) | O_NONBLOCK);
if (connect(fd, addr, alen) < 0 && errno == EINPROGRESS) {
    struct pollfd p = { fd, POLLOUT, 0 };
    if (poll(&p, 1, timeoutMs) == 0) { *stage = 4; *errNum = ETIMEDOUT; }  /* → 12 */
    int so = 0; socklen_t sl = sizeof(so);
    getsockopt(fd, SOL_SOCKET, SO_ERROR, &so, &sl);
    if (so) { *stage = 3; *errNum = so; }                                  /* 111 → 2 */
}
```

真机复现过：`stageCode=1 errno=1`、`2 errno=111`、`12 errno=110`、`4 httpStatus=400`。同步接口同理，失败回 `{"status":N,"curlCode":N,"error":"curl#28: …","body":""}`，**不抛异常穿越 FFI 边界**。

## ★ 宿主把 `SIGCHLD` 设成了 `SIG_IGN`

```text
$ grep -E "SigBlk|SigIgn" /proc/$(pidof miniapp)/status
SigIgn: 0000000000011006        ; bit16 = SIGCHLD
$ sigchld2
waitpid=-1 errno=10(No child processes)
child artifact: child-ran
```

即：**`waitpid()` 恒返回 `-1`/`ECHILD`**、`system()`/`popen()`/`pclose()` 返回 -1，**但子进程照常执行、产物照写**。判成败**必须看产物**：

```c
/* 宿主 SIGCHLD=SIG_IGN，waitpid 只会 ECHILD；子进程其实跑完了。
 * 判据 = 产物是否成型，不是退出码。 */
for (i = 0; i < 300; i++) {                 /* 300 × 50ms = 15s 上限 */
    r = waitpid(pid, &st, WNOHANG);
    if (r < 0) reaped = 1;                  /* ECHILD = 已被回收，不是失败 */
    if (file_size_at_least(dst, MIN)) { ok = 1; break; }
    usleep(50 * 1000);
}
if (!ok) { kill(pid, SIGKILL); waitpid(pid, &st, WNOHANG); }
ok = ok || file_size_at_least(dst, MIN);    /* 兜底再看一次产物 */
```

这条害过一次：第一版把 `waitpid` 的 `-1` 当失败返回空串，而子进程早把 65752 B 的 PNG 写好了，排查方向被彻底误导。

## 调试日志：只有 warn/error 落盘

设备**只持久化 `console.warn`/`console.error`**，`console.log` 直接丢；**没有 `logcat`**（`exec: line 1: logcat: not found`）。

```bash
L=$(adb shell 'ls -t /userdata/applog/DictPen_*.log | head -1')   # 名字带时间戳，别写死
adb shell "grep -h '\[pencg\]' $L | tail -40"
# 11-08 00:02:57 ERROR 989:547775… (console().warn.0): [pencg][I] [native] cgnative 就绪
```

`ERROR` 前缀是宿主统一加的，**不代表你打了 error**。native 里的 `printf` 通常看不到，要落盘就把诊断字段 `JSON.stringify` 后由 JS 侧 `console.warn` 一次。

## 部署：`adb push` 的三个坑叠在一起

1. **目标必须是绝对路径**。相对路径 adb 报 `remote No such file or directory`（设备 shell cwd 是只读的 `/oem/YoudaoDictPen/output`）：

```text
$ adb push x.txt tmp/pushtest.txt
x.txt: 1 file pushed, 0 skipped.
adb: error: failed to copy 'x.txt' to 'tmp/pushtest.txt': remote No such file or directory
```

2. **MSYS/Git-Bash 会把 `/tmp/...` 改写成 Windows 路径**，`MSYS_NO_PATHCONV`/`MSYS2_ARG_CONV_EXCL` 不一定生效。可靠做法是 PowerShell 包一层（它不改写）：

```bash
powershell -NoProfile -Command "& '.\env\adb\adb.exe' push '<local amr>' /userdisk/miniapp/data/pencg.amr"
```

3. **带管道的复杂查询别塞进 `adb shell '...'`**（嵌套引号会挂死）：先 push 脚本，再 `adb shell 'sh /userdisk/miniapp/data/x.sh'`。

另外两条：删通配符**必须单引号** `adb shell 'rm -f /path/pc_*.png'`（双引号会被本地 shell 先展开，看着删了其实没删，"冷启动"测的其实是缓存）；**`miniapp_cli` 没有 stop/kill/restart**，全表只有 `memoryApp/trimImageCache/memoryUsage/memoryUsageGC/dumpMemory/install/uninstall/start/startService/capture/captureFB/beginMonkey/stopMonkey`，要重启只能 `kill -9 $(pidof miniapp)`，而且**进程活着时内存态会覆盖你刚改的 `preferences.json`**。`install` 对不存在的文件也只给 `{"reason":"install failed: 1","ret":-1}`，先 `ls`。

## 动态加载链路与 AMR 打包

```text
libs/libjsapi_cgnative.so                     本地产物 62200 B
 └─ AMR: libs/libjsapi_cgnative.so            原样，字节不变
     └─ /userdisk/miniapp/data/mini_app/pkg/<appid>/<slot>/libs/
          libjsapi_cgnative_<随机数>.so       安装器重命名 + 重打包
          └─ 宿主 dlopen(绝对路径) → custom_init_jsapis() → JS: import('cgnative')
```

逐环验证：

```bash
ls -l libs/libjsapi_*.so                                    # 1 本地产物
python -c "import zipfile;print([n for n in zipfile.ZipFile('*.amr').namelist() if n[:5]=='libs/'])"  # 2 在包里
adb shell 'ls -l /userdisk/miniapp/data/mini_app/pkg/<appid>/*/libs/'   # 3 带随机数=正常
adb shell "grep -h cgnative \$(ls -t /userdata/applog/DictPen_*.log|head -1)"   # 4 import 成功
```

**AMR 里是原样 62200 B，设备上那份被安装器改写成了 132776 B**（尾部追加 `…/libs/\0libjsapi_cgnative_<随机数>.so\0` 绝对路径串）。这是 `dlopen` 走绝对路径的产物，**不是损坏**，别拿它比 md5。

模块名是硬编码字符串，必须和 JS 侧 `import { … } from 'cgnative'`、文件名三者一致。

## 交付前必须自检的 5 件事

1. **`nm -D --undefined-only`（或读 `.dynsym`）没有"设备上也找不到"的符号。** 一个顺手调用就让整个 `.so` 加载失败，宿主只报"未找到模块"。**`RTLD_LAZY` 不是退路**（`DF_BIND_NOW`）。
2. **`readelf -d` 的每个 `NEEDED` 在设备上都存在。** 本模块实测只有 `libquickjs.so / libssl.so.1.1 / libcrypto.so.1.1 / libcurl.so.4 / libc.so.6 / libpthread.so.0`；多出 `libstdc++.so.6` 说明混进 C++。
3. **真机 `await import('<name>')` 成功且导出名对得上。** 诊断页第一行打 `Object.keys(m).filter(k=>k!=='default').length`，和 C 侧导出表条数比（本模块 25）。
4. **每个导出的返回 JSON 都能 `JSON.parse`。** `error` 是字符串或 null；`wsState/wsPoll/wsPollEx/wsList` 都是字符串；`wsPoll` 的 `Array.isArray` 必须为 true。
5. **`wsState()` 成功与失败各跑一遍。** 成功要 `phase:"open" stageCode:6 httpStatus:101`，失败要 `phase:"tcp" stageCode:12 errno:110`。**只测成功等于没测。**

## 症状 → 方向

| 症状 | 方向 |
| --- | --- |
| `import('xxx')` 报未找到模块，`.so` 在 slot 里 | `.so` 有解析不到的符号（逐条核 `.dynsym`）；或模块名字符串不一致 |
| 编译报 `expansion of date or time macro` | 用了 `__DATE__`/`__TIME__`，换字面量宏 |
| `unable to find dynamic system library 'ssl'` | 没传 stub `.so` 全路径；zig sysroot 无第三方库 |
| `'curl/curl.h' file not found` | 手写 `extern` 声明 + `CURLOPT_*` 数字常量，别 `#include` |
| `adb push` 说 pushed，`install` 却 `ret:-1` | 用了相对路径（设备 cwd 只读）；改绝对路径 |
| Bash 里 `adb push ... /tmp/x` 落到 Windows 临时目录 | MSYS 路径改写；用 `powershell -NoProfile -Command` 包一层 |
| `waitpid()==-1/ECHILD`、`system()==-1`，但产物文件在 | 宿主 `SIGCHLD=SIG_IGN`；判据改成看产物 |
| 改了代码、部署了、日志没变 | 没查 `$LASTEXITCODE`（用了旧 `.so`）；或进程内存态覆盖了配置 |
| 日志只有一半，`console.log` 全不见 | 设备只落 `console.warn`/`console.error`；没有 `logcat` |
| WS 拿到 `101` 却立刻 close | 客户端帧没掩码、`Sec-WebSocket-Key`/`Accept` 算错、长度档位错 |
| `stageCode` 2 和 12 分不开 | 用了阻塞 `connect()`；改非阻塞 + `poll` + `SO_ERROR` |
| `mod.md5Hex` 是 undefined | 只做了 `JS_SetPropertyStr(def)` 没做 `JS_SetModuleExport`；或忘导出 `"default"` |
| JS 一读返回值就抛，`err.code` 是 undefined | 返回值不是 JSON 字符串 / `error` 是字符串；走 `callJSON()` |
| `JSON.parse` 偶尔失败 | `body` 含非 UTF-8 字节没转义；用 `JS_NewStringLen` + 非法字节转 `\u00XX` |
