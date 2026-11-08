# 逆向第三方 Web 服务、发布与隐私

这一份是**接口与发布层面**的方法论，对应 [debugging.md](./debugging.md)（设备层面的取证纪律）。
这里记录的多半不是"怎么写代码"，而是"你以为学会了其实学错"的地方。
**每一个数字都是实测出来的，不是从字段名推断的。**

## 第一原则：先取证，再写代码

逆向第三方 Web 服务时顺序必须是：

```
主机侧探针（PC 上 Node/Python）
  → 登录 / 列表 / 详情 / 下单 / 状态轮询 各跑通一次
  → 把真实字段、真实单位、真实错误码记进文档
  → 才写设备端代码
```

**禁止在设备上边猜边调。** 设备端循环是分钟级的（构建 ~20s、装机 ~15s），PC 上是秒级；
更关键的是，一次错误的请求可能**消耗用户的真实额度**。

实测代价：`enqueue` 发出后 **1.66 秒**收到 `queue_status`、**2.31 秒**收到 `queue_success`
（该账号根本没排队，直接分配机位），免费时长当场 `1333 → 1298` 秒 ——
**一次 schema 探测烧掉 35 秒**，占当日额度的 2.3%。

所以：

| 想验证 | 用什么 | 成本 |
|---|---|---|
| 连接与帧解析 | `destroy_ticket`（无 ticket 时 no-op） | 0 |
| 加解密实现 | 本地 `encode` → 立即 `decode` 回环 | 0 |
| 排队成功后的字段 | **不要为了这个去排队**，等真实游玩再抓 | 真实时长 |

## ★ 参数名与类型必须实测，不能从命名推断

四个真实陷阱，全都返回 HTTP 200 或一个与真因无关的错。

### 1. 静默忽略错误参数名

```
GET api/v1/games?tag=限免&page=0&per_page=60      → total=16    ✅
GET api/v1/games?tags=限免&page=0&per_page=60     → 全量 1800+  ❌
GET api/v1/games?tag_key=限免&page=0&per_page=60  → 全量 1824  ❌
```

三条都是 **HTTP 200，无任何报错** —— 服务端把不认识的 query 参数**当成没传**。

后果：本地再按"有没有免费时长"筛，全量 935 条手游里有 421 条 `free===true`、
179 条 `daily_free_time>0`，付费游戏被混进"限免专区"，用户报「限免里全是付费游戏」。

**判据**：改参数名后 `total` 有没有变。不变 = 参数被忽略，不是数据恰好相同。

### 2. `page` 是偏移量不是页码

```
?page=1&per_page=60   → 0 条
?page=0&per_page=60   → 全部 16 条
?page=0&per_page=16   → 16 条，total=16
```

`page` 的语义是**已经取了多少条**，翻页写 `page = 已取数量`。当页码用时，
`per_page >= total` 恰好"碰对"一次、第二页永远空 —— 这种"第一页对、后面全空"
最容易误判成"服务端就只有一页"。

### 3. 校验严格的接口给的是看似无关的错

网关 WebSocket 的 `auth` 包类型不匹配时只回：

```json
{"op":"error","data":{"errcode":400,"errmsg":"format invalid"}}
```

"format invalid" 暗示编码错了，实际是 **JSON-Schema 类型不符**：

| 字段 | 必须是 | 写成 |
|---|---|---|
| `user_id` / `token` / `game_code` / `quality` | string | number → 400 |
| `w` / `h` | number | `"1280"` → 400 |
| `codecs` | array | `"h264"` → 400 |

**缺任一字段也 400**。排查：逐字段二分（先发最小集再逐个加），别盯着"format"往编码方向查。

两个"格式错"要分清触发面，别互相套用结论：

| 码 | 主要触发面 | 实测反例 |
|---|---|---|
| `400 format invalid` | **auth 包的字段类型 / 是否缺字段**（网关 schema 校验） | `w` 写成 `"1280"`、`codecs` 写成 `"h264"` |
| `10003 无效数据格式` | **帧封装层**（auth 用加密发、业务 op 用明文发、未 auth 就发加密 op） | `auth` 加密 → 10003；`enqueue` 明文 → 10003 |

但 `10003` 的触发面**比"只是帧封装"更宽**：实测把 `user_id` 写成数字（而不是字符串）
也会收到 `10003 无效数据格式`。所以排查顺序应该是——先确认**这一帧该明文还是该加密**，
再逐字段核类型；两边都做完了还不行，才去怀疑编码实现。

### 4. 单位错会污染后续状态，不只是本次失败

带"带宽"字段的测速上报，单位是 **Mbps**：

```js
bandwidth: 60          // ✅ Mbps
bandwidth: 50000000    // ❌ 按字节/秒算
```

传错后不只是本次失败 —— **后续每一次申请机位都返回 `1288`**，因为服务端记住了这份
不合格的测速结果。恢复手段是「先释放机位（`DELETE .../games-playing/<code>`）+ 重新上报
一次正确单位的测速结果」，然后再申请；只重发 tickets 不重测速是不会恢复的。
这类"一次写错、长期中毒"的字段必须第一时间钉死单位。

⚠️ 注意 `1288` 有**两个完全不同的触发源**，别把结论互相套用：

| 触发源 | 现象 | 正确处置 |
|---|---|---|
| 测速带宽单位写错（如按字节/秒传 `50000000`） | 从那一刻起**每一次**申请都 1288，换游戏也一样 | 释放机位 + 用正确单位重新上报测速，再申请 |
| region 集合里**没有该游戏的容器** | 只对这个游戏 1288，换个游戏正常 | 不是测速问题：查 `media-servers?game_code=<code>` 取真实 region，换 region 重试 |

判断是哪一个：**换一个游戏试一次**。换了也不恢复 → 单位/测速中毒；换了就好 → region 不匹配。

## 错误码要建表，不要用文本匹配

文案会骗人。本项目最贵的一个例子：

```
1288 → "no region avail, please do speed test again"
```

字面意思是「请重新测速」，真实语义是「**这些 region 没有该游戏的容器**」。
端游只有 `<pc-region>` 一族 region，拿手机区（`<mobile-region>`）的 region 数组去申请，
连发 10 次全是 1288。改成先 `GET media-servers?game_code=<code>` 取真实可用区，一次就拿到
`gateway_url`。

**靠文案排查会走偏两天。** 正确做法是建常量表，表里写"真实语义"而非"字面文案"：

| 码 | 字面 errmsg | 真实语义 / 处置 |
|---|---|---|
| `33803` | multi instance | 多实例抢占，重试 |
| `1209` | no resource | 无空闲机位，重试 |
| `1288` | no region avail, please do speed test again | **region 集合里没有该游戏的容器**；去查 `media-servers`，不是去重测速 |
| `1269` | select region | 缺 region 参数 |
| `1506` | resume game failed | 恢复失败（通常是把 auth 包发多了字段） |
| `1811` | （无稳定字面文案） | 上一局还在退出流程中，机位没真正释放；先 `DELETE .../games-playing/<code>` 再 sleep 数秒重试 |
| `1213` | ticket required | 还没拿到 ticket 就连网关 |
| `1302` | params error | 参数缺失/为空 |
| `1006` | 登录信息失效 | token 无效，清 session，**不要重连** |
| `1100` | 发送太频繁 | 短信并发连点 |
| `1102` / `1109` | 验证码错误 / 次数过多 | 分开处理 |
| `1103` / `1104` | 区号无效 / 未注册 | 表单校验文案 |
| `1122` / `1125` | 易盾校验失败（原文 "need_yidun_validate_fail_enough"） | 风控拦截，文案统一成「请切换网络或稍后重试」 |

配套工程约定：

- 传输层必须把业务码挂到 `err.code`（不只拼进 `err.message`），否则调用方无法分类。
  本项目早期只有加密网关那条路径设 `err.code`，于是同一套 `switch` 在明文接口上永远命中 default。
- 调用方留一条**从 message 正则回捞数字**的兜底：`/^(\d{3,5})\s+/`。
- 码表放服务层而不是塞进文案，改文案时才不至于连带改判定。

## ★ 单位与语义陷阱（三个真实案例）

### 案例 1：`time_left` 是连接票的有效期，不是游玩时长

申请机位响应里有 `time_left`。连取三次：

```
请求 1: time_left=24   expires-now=25.1s
请求 2: time_left=22   expires-now=23.0s
请求 3: time_left=20   expires-now=20.8s
```

`expires` 本身**恒定**（同一时间戳），`time_left` 精确等于 `ceil(expires - now)`。

所以它是「这张 ticket 还能用来连网关多少秒」，不是「本局还能玩多少分钟」。
服务端**根本不下发单局时长限制**，玩家可以一直玩到主动退出。
把它当「本局剩余 25 分钟」显示，用户每局都以为只剩 25 分钟。

**判定方法**：同一字段连取 3 次看它跟谁同步 —— 与墙钟同步递减 = 有效期；
与实际使用量同步减少 = 配额。

### 案例 2：`free_time_left` 单位是秒不是分钟

接口字段表写的是「手游免费剩余（**秒**）」，样例 `975`。按分钟算，`1333` 秒变成
「22 小时 13 分」，真实值是 **22 分 13 秒**，与该账号官网显示的「23 分钟」吻合。

**判定方法**：拿同一账号官网显示做对照，量级差 60 倍一定是分钟/秒搞反了。

### 案例 3：布尔字段被当成分钟数

`bluray` 是「是否有蓝光权限」的 **boolean**，被写成
`if (typeof b === 'number') return '蓝光 ' + b + ' 分钟';` —— 永不成立，但代码看着很合理。
同族真正带数值的是 `bluray_left`（单位秒）：先判它有没有，再退回布尔，才能覆盖两种账号。

通用验证流程：连续采样 3 次，同时打印相邻的已知字段（时间戳/配额/计数），
看它跟着谁变；再用另一种账号状态复验一次，避免单账号特例。

## 加密 / 偏移（obfuscation）实现要点

服务端把偏移量藏在**字符串模板**里，用固定规则拼出来（实测值）：

```
encrypt = "method:1,number:<n>,timestamp:<ts>,salt:<s>"

每段取冒号后的值 → 用 'd' 连接 → 按 'd' 切 → 取第 1/2/3 项
off = (((a | b) % 256) + c) % 256
```

请求体 = `base64( map(b => (b + off) % 256, utf8(JSON)) )`，
响应体 = 同样规则反向。**HTTP 与 WebSocket 共用同一个 off。**

实现上必须做到这几条：

1. **双向回环测试**：`decode(encode(x)) === x`，ASCII / 中文 / emoji 各测一遍。没有它，
   偏移写错时你看到的是"服务端返回乱码"，不会想到是自己编错了。
2. **不要一把梭**：同一平台通常并存明文（匿名 v1）和加密（登录态 v2）两条路径，全塞进
   加密层会让匿名接口失效，且错误信息很像"加密不对"。
3. **响应可能是密文也可能是明文**：先看首字节，`{`/`[` 当明文 JSON，否则才解密。
4. **UTF-8 手写编解码**：QuickJS 里 `TextEncoder` 不一定存在，而偏移是按**字节**做
   `(b + off) % 256`，按字符做会直接错。
5. **非 2xx 的 body 也要能读到**。平台自带的 HTTP jsapi 拿不到非 2xx body，
   所以加密接口必须走 native curl 路径 —— 这是"用哪条传输"的分水岭，不是性能优化。

## 登录态与凭据

**token、uid、手机号一律不进源码、不进日志、不进截图。** 三条工程措施：

1. **会话只存一个键**：持久化只写 `preferences.json` 的单个键（整份 session 快照）。
   拆成多个键散落各处，删除时一定会漏一个。
2. **日志只打长度或 hash**。
   ```js
   logInfo('[auth] token len=' + t.length + ' sha8=' + sha8(t));
   ```
   本项目早期日志里直接打了 uid 原值。uid 本身不算密钥，但它是**跨平台可关联的身份**，
   和截图里的昵称叠加就是可用的画像。
3. **发布前双编码扫描**。Weex 打包后字符串是 **UTF-16LE**，只扫 ASCII/UTF-8 会全部漏掉。
   实测：中文串「加载中」在 `.falcon_/*.js.bin` 里 UTF-8 命中 **0** 次、UTF-16LE 命中 **6** 次。
   范围覆盖源码、`package.json`、`.falcon_/*.js.bin`、`*.so`、`*.amr`、
   `profiles/evidence/**` 和 git 历史（见下节）。

## ★ 截图是最大的隐私泄露面

两个真实泄露案例，性质不同：

| 案例 | 泄露内容 | 为什么扫文本扫不到 |
|---|---|---|
| 登录页截图 | 完整手机号 + 昵称 | 在像素里 |
| 游戏内截图 | 玩家 UID（**游戏自己渲染的**） | 既不在 DOM 也不在文本，是视频平面里的字 |

第二条尤其隐蔽：grep 扫不到，只有人眼看。

工程措施：

- **发布前把 `profiles/evidence/**` 逐张看过**，不是 grep 文档。39 张图看一遍 5 分钟，
  比事后重写历史便宜几个数量级。
- 含账号的图**打码或删除**，不要"挪到别处"；打码要真的覆盖，不是缩小
  （某次"脱敏"只是把图缩到 320 宽，UID 依然可读）。
- 设备侧 `miniapp_cli capture` 只拍 Falcon UI 合成面（960×266），**不含视频平面** ——
  所以游戏内 UID 只可能出现在用外部工具（`kmssrc` 等）拼出来的 `*_uv.png` 一类图里。
  反过来也是排查线索：UI 截图里出现视频内容 = 你拍的不是 `capture`。

## ★ 改历史要用 plumbing，不要指望 filter-branch

Windows + MSYS 下 `git filter-branch` 会把 `GIT_DIR` 传成 `/d/...`，Windows 版 git 立刻
`fatal: not a git repository`。**不要在这件事上花时间** —— 自写脚本逐提交重放
（本项目 `git rev-list --count HEAD` = 10，跑完 20 秒）：

```python
# 每个提交：
#   1. read-tree <commit>          （带独立 GIT_INDEX_FILE）
#   2. update-index --force-remove / --cacheinfo   （删图 / 换成脱敏版 blob）
#   3. hash-object -w 重写需要脱敏的文本，再 --cacheinfo 换进去
#   4. write-tree
#   5. commit-tree 带原 author/committer 与日期，父提交换成新映射
```

关键细节：

- 每个提交用**独立的 `GIT_INDEX_FILE`**（如 `.git/tmpindex`），调用前 `os.remove`，
  否则上一个提交的索引会污染下一个。
- 调子进程前 `os.environ.pop('GIT_DIR', None)` —— MSYS 会把 `/d/...` 传下去。
- 保留原 author/committer 名字、邮箱、**原始 ISO 日期**（`%aI` / `%cI`），否则时间线全乱。

**别忘三件事**（少做任何一条，旧数据都还在线上）：

1. **删掉 `refs/remotes/origin/main`**。远端引用还指着旧提交时，旧提交仍可达，
   `git gc` 清不掉，`git log --all` 也还能翻出来。
2. `git reflog expire --expire=now --all` + `git gc --prune=now`。
3. **远端的 tag 也要重指**。release tag 指向旧提交时，旧截图在 GitHub release 页面上
   **仍然可以直接浏览** —— 这时候本地已经"干净"了，是最容易收工的时刻。

删完自证：`git rev-list --all --objects | wc -l` 前后对比，并用旧 blob hash
`git cat-file -e <old-blob>` 确认已经不存在（返回非 0 才算真删掉）。

## 发布流程（可复用清单）

### 版本号必须三处一致

| 位置 | 作用 |
|---|---|
| `src/services/version.js` 的 `APP_VERSION` | 唯一来源，界面/关于卡片/页脚全读它 |
| `package.json` 的 `"version"` | 还决定 AMR 文件名 |
| git tag / release | 用户看到的版本 |

只改 tag 不改界面 → 「release 是 v1.0.3、软件里写 1.0.0」。本项目被用户报过一次。

**不要改**那些"冒充官方客户端"的请求头（如 `X-Ver: 1.0.0`）—— 它对应官方 web 客户端的
协议版本，和发版号无关，改了可能被判成未知客户端。代码里要写注释钉死。

### release 附件上传必须按字节流

```js
// 1) upload_url 末尾带 RFC6570 模板 {?name,label}，必须剥掉
const up = d.upload_url.replace(/\{.*\}$/, '');
// 2) 必须 application/octet-stream，body 传 Buffer 不是字符串
await fetch(up + '?name=PenCG-1.0.3.amr', { method: 'POST', body: fs.readFileSync(file),
  headers: { Authorization: 'token ' + PAT, 'Content-Type': 'application/octet-stream' } });
```

用字符串 body 上传时 `size` 会比本地大几百字节（重编码）——「返回 201 但文件是坏的」的典型形态。

**校验不要用 `releases/download/...`** —— 国内常直接 `UND_ERR_CONNECT_TIMEOUT`，
会被误判成"上传失败"。走 API 回下载：
`GET /repos/<o>/<r>/releases/assets/<id>` 带 `Accept: application/octet-stream`，
取 arrayBuffer 算 sha256 与本地比对。

### 给仓库设 topics 要用专用接口

```
PUT /repos/{owner}/{repo}/topics   {names:[...]}     ✅
PATCH /repos/{owner}/{repo}        {topics:[...]}    ❌ 静默忽略
```

`PATCH` 返回 200 且其它字段都生效了，只有 topics 没变 —— 典型的"部分成功"。

### PAT 从 Windows 凭据管理器读

```powershell
CredReadW("git:https://github.com", 1, 0, [ref]$ptr)   # 类型 1 = GENERIC
$c    = Marshal::PtrToStructure($ptr, [type][CREDENTIAL])
$tok  = Marshal::PtrToStringUni($c.CredentialBlob, $c.CredentialBlobSize / 2)  # 必须 /2
```

`CredentialBlobSize` 是**字节数**而 `PtrToStringUni` 要**字符数**；忘了 `/2` 会读到乱码
加尾部垃圾字符，表现为"token 无效 401"。用完立刻删临时文件。

## 装不上先证明包没问题

AMR 装不上时第一反应往往是"包坏了"，然后去查构建、签名、目录结构。

**先做一次 6 字节推送**：

```sh
printf 'probe' > /tmp/probe.txt
adb push /tmp/probe.txt /tmp/probe.txt
```

若报的是同样一类错（如 `protocol fault: stat response has wrong message id`），
问题在**设备 ADB 认证**，不在包 —— 判据是错误发生在 transfer 之前，包内容根本没被读过。

同类假阳性：

| 现象 | 实际常是 |
|---|---|
| `adb push` 报 `remote Read-only file system` | MSYS 把 `/tmp/x` 转成 `C:\...\Temp\x`，push 的是不存在的本地路径 |
| 装机 `ret:0` 但界面没变 | `start <appid> <page>` 用了双横线 `--page`，ret=0 但不切页 |
| 装完就白屏 | `.vue` 少了 `<style>` 块 / 少了 `@import "base.less";` |

MSYS 下 `export MSYS2_ARG_CONV_EXCL='*'`，或把设备路径写成 `//tmp/x`（双斜杠）。

## 接口排查 → 方向

| 症状 | 先查 |
|---|---|
| 改了筛选参数但结果不变 | 参数名是否被静默忽略：对比 `total` 有没有变 |
| 第一页对、第二页空 | `page` 是偏移量不是页码；`page = 已取数量` |
| 明明有数据却返回空数组 | 是否少传分页参数（缺 `page`/`per_page` 会返回全量或空，不是报错） |
| WS 回 `400 format invalid` | 逐字段二分：string/number/array 类型 + 是否缺字段 |
| WS 回 `10003 无效数据格式` | 帧封装：auth 必须明文，其它 op 必须加密 |
| 申请机位永远 `1288` | 不是测速问题，是 region 集合里没有该游戏的容器；查 `media-servers` |
| 带宽相关接口一直失败 | 单位是 Mbps 还是 byte/s；错一次会污染后续所有请求 |
| 时长显示大 60 倍 | 秒被当分钟 |
| 时长显示恒为 0 | 初始化/恢复会话时把快照字段覆盖成了 0（下划线 vs 驼峰） |
| 布尔字段显示出「N 分钟」 | 字段本身是权限标志；找同族真正带数值的 `_left` 字段 |
| 密文接口报文乱码 | 先跑 `decode(encode(x))` 回环，再怀疑服务端 |
| 登录后立刻 401 | token 是否从会话快照里被清掉；`credentialBlobSize / 2` 之类 |
| 重启后会员态丢失 | 快照字段名与响应对不齐（`is_vip` vs `isVip`），恢复时把布尔冲成 false |

## 发布前隐私自检清单

- [ ] `profiles/evidence/**` **逐张人眼看过**（不是 grep），确认无手机号/昵称/UID/token/二维码
- [ ] 含账号的截图已删除或**真正覆盖打码**（不是缩小分辨率）
- [ ] 游戏内截图（`*_uv.png` 这类外部合成图）单独复看：UID 是游戏渲染的，扫文本扫不到
- [ ] 源码扫描：ASCII **和** UTF-16LE 双编码（Weex 打包后字符串是 UTF-16）
- [ ] 扫描覆盖 `.falcon_/*.js.bin`、`*.so`、`*.amr`，不只扫 `src/`
- [ ] 日志里没有 token / uid / 手机号原值（只有长度或 hash）
- [ ] 会话只落一个 storage 键，`logout` 能一次清干净（含内存单例）
- [ ] git 历史已重放：`git rev-list --all` 里旧 blob 不可达
- [ ] `refs/remotes/origin/*` 已删，`reflog expire --expire=now --all` + `gc --prune=now` 已跑
- [ ] **远端 tag 已重指**，GitHub release 页面上的旧截图已不可浏览
- [ ] 版本号三处一致：`version.js` / `package.json` / tag
- [ ] `X-Ver` 之类冒充官方客户端的请求头**没有**跟着版本号改
- [ ] release 附件 sha256 用 **API 回下载**校验过（不是 `releases/download` 那个 URL）
- [ ] 调试开关（`/tmp/*.enable`、`GST_DEBUG` 文件触发）在最终包里不存在
- [ ] 私有 appid / 内网地址在文档里已用占位符替换
