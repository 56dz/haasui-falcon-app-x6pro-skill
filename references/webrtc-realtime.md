# WebRTC 实时媒体：自己实现 ICE / DTLS / SRTP / RTP

把标准 WebRTC 网关的流拉到词典笔、再送进 GStreamer + MPP + KMS 的全过程（前置：
[pipeline-video.md](pipeline-video.md)、[platform-pitfalls.md](platform-pitfalls.md)）。
**结论**：没有 `webrtcbin`/`libnice`/`libsrtp`，ICE / DTLS-SRTP / RTP 拆包必须自己写；但
`libssl 1.1.1l` 有 exporter 与 srtp profile，DTLS 那半交给它，裸 H.264 用 `appsrc` 喂
GStreamer。

## 为什么不能用 webrtcbin

```sh
adb shell "ls /lib64/gstreamer-1.0/ | grep -iE 'webrtc|nice|srtp|dtls'"
```

实测（GStreamer 1.22.0）：`webrtcbin`/`dtlssrtpdec`/`nicesrc` 全部 `no element`；
`appsrc`/`h264parse`/`mppvideodec`/`kmssink`/`opusdec` 齐备。**禁止** vendor
`libnice`：交叉编译链、库路径、插件注册三处都要动，而 ice-lite 网关只需"发 Binding +
提名 + connect"。

## 握手状态机

```text
WSS ──明文 auth──> 网关 ──推 {"op":"offer","data":{"sdp":...}}
  parseSdp() → buildAnswer() → ws.send({"op":"answer"})   ★ 不回 answer 没有流
  native.start(cfgJson):
    1. bind 0.0.0.0:0   2. STUN ×10 → 收 0x0101 = ICE connected
    3. connect(peer)    ★ 不做这步 DTLS 一个字节都发不出去
    4. DTLS 1.2 client handshake（内嵌 PEM 证书）
    5. SSL_export_keying_material(60B) → SRTP master key
    6. SRTP 解密 → RTP 拆包 → H.264 成帧 → appsrc    7. RTCP PLI 补关键帧
```

**answer 必须真的发出去**：只做 ICE 时网关回了 `0x0101`（五元组通）却全程零字节 DTLS；
补发 answer 后 924ms 收到 `op=result`（容器启动）才有媒体。网关是 **ice-lite**，不主动
发 Binding，只要一个 host candidate，**不需要 TURN**。

## SDP 解析：`slice(prefix.length)`，不要数数字

```js
const ufrag = line.slice(11);                     // 禁止：'a=ice-ufrag:' 是 12 字符
const ufrag = line.slice('a=ice-ufrag:'.length);  // 必须
```

写死 11 会让 ufrag 多一个前导冒号，STUN `USERNAME` 变成 `"<remote>:<our 带冒号>"`，
**网关校验失败后静默丢弃**：没有 `Binding Response`、没有报错、请求正常发出、对端不回。
`a=ice-pwd:`/`a=fingerprint:`/`a=candidate:`/`a=rtpmap:`/`a=setup:` 同理；
`a=fingerprint` 必须是**我方证书的真实指纹**，随机值同样静默失败。

## ICE：STUN Binding 的字节布局

ice-lite 网关是 **controlled**，我们是 **controlling**，必须主动提名：

```c
USERNAME        (0x0006) = "<remote_ufrag>:<our_ufrag>"   /* 远端在前 */
PRIORITY        (0x0024) = 1845501695
ICE-CONTROLLING (0x8029) = 0x7F000000 / 0x11223344        /* controlled 换 0x802A */
USE-CANDIDATE   (0x0025) = 空                             /* ★ 缺它网关不开 DTLS */
```

拼包顺序不能错（`MESSAGE-INTEGRITY` 覆盖它之前的所有内容，`FINGERPRINT` 再覆盖 MI）：

```c
wr16(hdr + 2, len_before_mi + 24);  hmac_sha1(remote_ice_pwd, hdr + body, &mac);
attr(0x0008, mac, 20);              /* 1) 长度先算到 MI 之后 */
wr16(hdr + 2, len_before_fp + 8);   fp = crc32(hdr + body) ^ 0x5354554e;
attr(0x8028, &fp, 4);               /* 2) 再算到 FP 之后，"STUN" 反码 */
wr16(hdr + 2, real_body_len);       /* 3) 最后写真实长度 */
port = rd16(v + 2) ^ 0x2112;                        /* XOR-MAPPED-ADDRESS 0x0020 */
for (i = 0; i < 4; i++) ip[i] = v[4+i] ^ ((0x2112A442u >> ((3-i)*8)) & 0xff);
*err = (v[2] & 0x07) * 100 + (v[3] & 0xff);         /* ERROR-CODE 0x0009 */
```

`crc32` 用反射多项式 `0xEDB88320`、初值终值 `^0xFFFFFFFF`。**`ERROR-CODE` 的 class 只有
低 3 位**：写成 `(v[2] & 0x7f)` 会把 `487 Role Conflict` 读成 `787`，于是没人想到做角色
切换——实测某台网关因此**永远不回 Binding Response**。切换就一行
`if (err == 487 && !controlled) controlled = 1;`（改发 `ICE-CONTROLLED(0x802A)`，且不再发
`USE-CANDIDATE`）。收到 `0x0101` 后**再补发 2 次带 `USE-CANDIDATE` 的 Binding（间隔
100ms）**：成功响应只说明地址能通，不代表已 nominated。**回包源地址与 candidate 不一致
时必须把 peer 换成回包源地址**（NAT 映射）。

## DTLS：五条必须 + 一条禁止

```c
SSL_CTX *ctx = SSL_CTX_new(DTLS_client_method());
SSL_CTX_set_tlsext_use_srtp(ctx, "SRTP_AES128_CM_SHA1_80");
SSL_CTX_set_verify(ctx, SSL_VERIFY_NONE, NULL);
SSL_CTX_set_read_ahead(ctx, 1);                        /* ★1 同 datagram 多记录一起读 */
BIO *bio = BIO_new_dgram(sock, BIO_NOCLOSE);  BIO_set_nbio(bio, 1);
BIO_ctrl(bio, BIO_CTRL_DGRAM_SET_PEER, 0, &peer);      /* ★2 不设就 -1 / errno=22 */
BIO_ctrl(bio, BIO_CTRL_DGRAM_SET_CONNECTED, 0, &peer); /* ★3 防 BIO 走向空地址 */
BIO_ctrl(bio, BIO_CTRL_DGRAM_SET_MTU, 1200, NULL);     /* ★4 关掉 PMTU 探测 */
SSL_set_bio(ssl, bio, bio);  SSL_set_connect_state(ssl);
SSL_set_options(ssl, SSL_OP_NO_QUERY_MTU);  SSL_set_mtu(ssl, 1200);  /* ★4b */
```

**★4b 最贵**：设了 `SSL_OP_NO_QUERY_MTU` 却不给 MTU，`s->d1->mtu` 保持 0，写第一条记录就
失败 —— `handshake fail e=5 r=-1 errno=0(Success)`，像网络问题，实为本地参数。

**这几个 API 都是宏，按符号名查会误判成"设备不支持"**（`SSL_ctrl`/`SSL_set_options` 才是
真符号）：

```c
#define SSL_set_mtu(s,m)         SSL_ctrl((s), SSL_CTRL_SET_MTU, (m), NULL)          /* 17 */
#define DTLSv1_get_timeout(s,a)  SSL_ctrl((s), DTLS_CTRL_GET_TIMEOUT, 0, (void*)(a))  /* 73 */
#define DTLSv1_handle_timeout(s) SSL_ctrl((s), DTLS_CTRL_HANDLE_TIMEOUT, 0, NULL)     /* 74 */
#define SSL_OP_NO_QUERY_MTU      0x00001000U
```

**★5 DTLS 不会自己重传**——`WANT_READ` 不触发任何定时器，必须自己驱动：

```c
while (now < deadline && (r = SSL_do_handshake(ssl)) != 1) {
    int e = SSL_get_error(ssl, r);
    if (e != SSL_ERROR_WANT_READ && e != SSL_ERROR_WANT_WRITE) break;
    struct timeval tv; int wait_ms = 50;
    if (DTLSv1_get_timeout(ssl, &tv)) wait_ms = tv.tv_sec*1000 + tv.tv_usec/1000;
    poll_one(sock, clamp(wait_ms, 1, 200));     /* 别超 200ms，要能响应退出 */
    if (DTLSv1_handle_timeout(ssl) > 0) retx++; /* 重发 ClientHello */
}
```

想看报文挂 `SSL_CTX_set_msg_callback`：握手记录回调里 `content_type` 传的是 **0**（真实
类型在 `buf[0]`，握手类型在 `buf[13]`）；另有 `content_type == 0x100`
（`SSL3_RT_HEADER`）的钩子，**len 恒为 13**，是"刚读完记录头"。它反复出现而握手回调一
次都没有 → 见下一节。

## STUN keepalive 会污染 DTLS 记录层（必读）

ICE 打通后网关每 1~2 秒往同一五元组发 **STUN Binding Request**，首字节 `0x00` 不是合法
DTLS content type，`dtls1_get_record` 静默丢它继续下一条：

```text
#1 DTLS << ? len=13 type=-1      × 14      ← 全是记录头回调，零握手报文
#1 handshake diag: out=10 in=14 (in_bytes=182) retx=4
```

`in_bytes = 182 = 14 × 13`，每条"记录"恰好 13 字节，那就是记录头本身。**解法：收包前分
类，禁止把这些字节喂给 `SSL_read`。**

```c
/* 必须 MSG_PEEK | MSG_DONTWAIT；只 PEEK 不 DONTWAIT，在阻塞 socket 上会永久睡死 */
n = recvfrom(sock, peek, sizeof(peek), MSG_PEEK | MSG_DONTWAIT, NULL, NULL);
switch (classify(peek, n)) {
case PKT_STUN: consume(); stun_reply(peek, n); break;   /* 真读掉 + 必须应答 */
case PKT_DTLS: return 1;                               /* ★ 不消费，留给 dgram BIO */
default:       consume();                              /* 垃圾丢，计 rx_other */
}
/* STUN: b[0] 高 2 位 0 && n>=20 && offset4 == 0x2112A442
 * DTLS: b[0] ∈ 20..23 && b[1]==0xFE && b[2] ∈ {0xFD,0xFF}
 * RTP/RTCP: b[0] 高 2 位 0x80；b[1]>=192 是 RTCP */
```

**keepalive 必须应答，且 key 用我方 pwd**（RFC 8445 §7.2.2：响应与请求同 key）：不回，
网关认为 ICE 断了会停流；用对端 pwd 会静默失败。回 `0x0101` + XOR-MAPPED-ADDRESS +
MESSAGE-INTEGRITY + FINGERPRINT，txn ID 从请求 `pkt+8` 原样回填，key 取
`our_pwd[0] ? our_pwd : remote_pwd`。

**媒体阶段也要应答**：把"分类 + 应答"收敛成统一收包泵，握手与媒体循环都调它
（`1 = RTP，2 = RTCP，0 = 没拿到，<0 = 真错误`）。不改的话媒体期 STUN 被当 SRTP 解。

## connected UDP socket 上的 ICMP 是瞬态的

`connect()` 之后收到 ICMP port-unreachable，`recvfrom` 会返回 `ECONNREFUSED`。当成致命
错误 `break` 的症状是 **"出画一瞬间，然后永久黑屏"**（用户原话）。

```c
if (errno == ECONNREFUSED || errno == ECONNRESET || errno == ENETUNREACH ||
    errno == EHOSTUNREACH || errno == ENETDOWN || errno == EPROTO) {
    s->stats.rx_icmp++;  poll(NULL, 0, 5);
    continue;                       /* 继续收，下次就会来真包 */
}
```

判据：`rx_icmp` 计数 + `frames` 是不是"涨一下就停"。

## `SSL_export_keying_material` 的标签长度（最有价值的坑）

```c
SSL_export_keying_material(ssl, km, 60, "EXTRACTOR-dtls_srtp", 21, NULL, 0, 0);   // 禁止
SSL_export_keying_material(ssl, km, 60, "EXTRACTOR-dtls_srtp",
                           sizeof("EXTRACTOR-dtls_srtp") - 1, NULL, 0, 0);         // 必须
```

`strlen("EXTRACTOR-dtls_srtp") == 19`。传 21 时这个长度作为 `label_len` 直接进 TLS 1.2
PRF 的 seed（`client_random ‖ server_random ‖ [label_len ‖ label ‖ context]`），导出的 60
字节 keying material **完全不同**。症状像教科书：握手正常、KM 正常导出、**每包 SRTP
HMAC 都验不过**：

```text
rtp_rx=2917  auth_fail=2594        ← 改成 19 之后 auth_fail=0，首帧立刻出画
```

为什么藏得深：自测脚本的**服务端也写了 21**，两端同错，导出的 KM 反而一致，
`identical keying material` 照样 PASS。

> **教训比 bug 值钱**：对称的错误永远暴露不出来。自测必须有一处外部固定向量（RFC 3711
> 附录 B.3），或一条判别性断言「只把 label_len 从 19 改成 21，导出的 KM 必须不同」。

60 字节布局（我们是 DTLS **client**：收流用 server_write，发 PLI 用 client_write）：
`km[0..15]` client key、`km[16..31]` server key、`km[32..45]` client salt、
`km[46..59]` server salt。

## SRTP：KDF 与 ROC

```c
srtp_init(&rx, km + 16, km + 46);   /* 收：server_write_* */
srtp_init(&tx, km +  0, km + 32);   /* 发：client_write_* */
```

KDF（AES-CM），一处都不能改：label `0x00` = cipher key(16B)、`0x02` = salt(14B)、
`0x01` = auth key(20B)。

```c
memcpy(iv, master_salt, 14);        /* salt 只 14 字节，高 2 字节留 0 */
iv[7] ^= label;                     /* ★ label 异或到 salt 的第 7 字节 */
iv[14] = ctr >> 8; iv[15] = ctr;    /* 计数器在最后 2 字节，大端 */
/* AES-128-ECB 加密该 iv 块，取前 len 字节 */
```

`iv[7]` 是穷举验证过的：挪到任何其它偏移，cipher key 仍对，但 salt 与 auth key 全错。用
RFC 3711 附录 B.3 自查（salt `0EC675AD498AFEEBB6960B3AABE6` + label `02` → `…EE9B…`）：

```text
master key  E1F97A0D3E018BE0D64FA32C06DE4139
cipher key  C61E7A93744F39EE10734AFE3FF7A087    cipher salt 30CBBC08863D8C85D49DB34A9AE1
auth key    CEBE321F6FF7716B6FD4AB49AF256A156D38BAA4
```

AES-CTR：`iv = salt(14B)`，SSRC 异或进 `iv[4..7]`，48 位包索引 `(roc<<16)|seq` 异或进
`iv[8..13]`，计数器在 `iv[14..15]`。认证标签 = `HMAC-SHA1(RTP头 ‖ 密文payload ‖ ROC 4B
大端)` 取前 10 字节，**先验 HMAC 再解密**。ROC 每 SSRC 一份：

```c
if (!inited) { roc = 0; max_seq = seq; inited = 1; }
else if (seq < max_seq && (uint16_t)(max_seq - seq) > 0x8000) roc++;
max_seq = seq;
```

**禁止**验不过时静默跳过：要计数（`auth_fail`）并打印前几个包的 `seq/ssrc/roc/hl/len`。
`auth_fail` 从 2594 变 0，是判断 exporter 修复是否生效的唯一判据。

## RTP 拆包：三种形态 + PT 分流

```c
int hlen = 12 + (pkt[0] & 0x0f) * 4;                    /* CSRC */
if (pkt[0] & 0x10) {                                    /* extension */
    int words = (pkt[hlen + 2] << 8) | pkt[hlen + 3];
    hlen += 4 + words * 4;
}
```

按 PT 分流并校验 SSRC；不匹配的 PT（rtx 99/97/108、RTCP 200-204）直接丢并计数，前 12 条
打 `pt`/`ssrc`/前 4 字节——这是确认网关实际用哪个 PT 的唯一手段。

```c
nal = payload[0] & 0x1f;
/* STAP-A (24)：跳 1 字节 NAL 头后循环 [2B 长度][NAL]
 * FU-A   (28)：fu_hdr & 0x1F = 原 NAL type，0x80 = start，0x40 = end
 *   start 补 NAL 头 (fu_ind & 0xE0) | (fu_hdr & 0x1F)；end 整帧 flush
 * 单 NAL (1/5/7/8)：整包送，marker 置位时 flush
 * 25/26/27 (STAP-B/MTAP/FU-B)：packetization-mode=1 不会发，直接丢 */
```

每个 NAL 前补 4 字节 `00 00 00 01`，整帧拼完再 push（caps 用 `alignment=au`）。**关键帧
判定**：单 NAL 看 `nal == 5`，FU-A 看 `(fu_ind & 0xE0) | typ == 5`。记账**只在一处做**
（`flush_frame(s, is_key)` 内），否则 FU-A 的 start 与 flush 分支各记一次，`kf` 翻倍。

## SPS/PPS 自己解析

```c
/* 从 STAP-A 或裸 NAL type 7 取 SPS，去 00 00 03 转义后按 Exp-Golomb 读：
 * profile_idc → high profile(100/110/122/244…) 再读 chroma_format_idc（=3 还要 scaling list）
 * level_idc → pic_width_in_mbs_minus1 → pic_height_in_map_units_minus1
 * → frame_mbs_only_flag → frame_cropping_flag（必须减掉 crop）
 * w = (width_mbs+1)*16 ; h = (height_units+1)*16*(2-frame_mbs_only_flag) - crop_t - crop_b */
```

实测：**切换画质档位时 SPS 报的分辨率恒为 1280x720**，只有 `profile`（66 → 100）和
`level` 会变。所以判断"码流参数变了"**只能看 profile/level**，看分辨率永远是假。一变就
`have_kf = 0` + 重发 PLI 突发，逼网关给带新 SPS 的 IDR；否则解码器仍按旧 SPS 初始化，
画面花屏或纯绿。

## RTCP PLI：sender SSRC 不能用 media SSRC

```c
pkt[0] = 0x81;  pkt[1] = 206;  wr16(pkt + 2, 2);   /* RFC 4585 PSFB，FMT=1，12 字节 */
wr32(pkt + 4, tx_ssrc);                            /* ★ 我方 SSRC，禁止等于 media ssrc */
wr32(pkt + 8, video_ssrc);
/* 有 SRTP 密钥就走 SRTCP：前 8 字节不加密，index 置最高位 E=1，尾部 10B tag → 26 字节 */
```

`tx_ssrc` 用任意固定值（例如 ASCII `'PENC'`），**但不能等于 `video_ssrc`**：网关做 RTCP
环路检测，看到"自己发的包"直接丢。这一个改动把"连发 78 次 PLI 拿不到 IDR"变成"第一次
PLI 就来 IDR"。节奏：首流连发 3 次（≥200ms 间隔），之后每 2 秒检查，`have_kf` 且上次关
键帧不到 2 秒就跳过。

## 喂给 GStreamer：caps 与"push 返回 OK 不代表成功"

```c
"appsrc name=vsrc is-live=true format=time do-timestamp=true "
"caps=video/x-h264,stream-format=byte-stream,alignment=au "
"! h264parse ! mppvideodec rotation=270 ! videoconvert ! kmssink ..."

"appsrc name=asrc is-live=true format=time do-timestamp=true caps=audio/x-opus "
"! queue ! opusdec ! audioconvert ! audioresample "
"! audio/x-raw,format=S16LE,layout=interleaved,rate=48000,channels=2 "
"! alsasink name=asink sync=true"
```

**必须 1**：`is-live=true` + `format=time` + `do-timestamp=true`；RTP 没有 PTS，
`do-timestamp` 按到达时间打戳。

**必须 2**：opus 的 caps 在 `gst_app_src_set_caps()` 里必须写全
`audio/x-opus,rate=48000,channels=2,channel-mapping-family=0`。**禁止**只写
`audio/x-opus`：本机 `opusdec` 的 sink 模板有两支（`channel-mapping-family: 0` 与
`[1,255]` + `rate`），只写 `audio/x-opus` 会同时命中、协商不出唯一结果，appsrc 以
`GST_FLOW_NOT_NEGOTIATED` 收场 —— 而 **`gst_app_src_push_buffer()` 照样返回 OK**。不挂
bus 监视器就完全静默：包在涨、管线 PLAYING、PCM 停 `state: OPEN`、`hw_ptr=0`。

**必须 3**：每个统计周期抽一次 bus
`gst_bus_timed_pop_filtered(bus, 0, GST_MSG_ERROR | GST_MSG_WARNING | GST_MSG_EOS)`——
`kmssink`/`opusdec`/`alsasink` 的致命错误只出现在这里。

## KMS 上屏：几个几何与生命周期坑

详见 [pipeline-video.md](pipeline-video.md)，这里记实时流最容易错的几条。

**1. `render-rectangle` 是面板物理坐标**（面板 480x960 竖屏、逻辑 960x266 横屏）：
`px = 107 + y`（107 是左侧死区）、`py = 960 - x - w`（**不是 y**）、`pw = h`、`ph = w`。
把 `py` 写成 `y` 的症状：**画面跑到最右侧并被拉伸**。逻辑 `(0,0,473,266)` →
`pos[107,487] rect[266x473]`。

**2. 旋转必须在解码器里做**。`modetest -M rockchip -p` 实测 plane 76 的 `rotation` 只有
`rotate-0`/`reflect-y`，**没有 90/270**；KMS 平面绕过 weston，故交给解码器
`mppvideodec rotation=270`（真机 A/B：90 整体倒置），软解等价
`avdec_h264 ! videoflip method=counterclockwise`。

**3. `GST_STATE_NULL` 是 1，不是 0**（`VOID_PENDING=0, NULL=1, READY=2, PAUSED=3,
PLAYING=4`）。传 0 是非法目标态，`set_state` 直接失败返回，pipeline 从没进 NULL，kmssink
一直占着 KMS 平面 → **退出后最后一帧永远留在屏上**。必须 `set_state(p, 1)` 再
`get_state(p, &got, &pending, 1500000000ULL)` 等落地，最后才 unref。

**4. `setRect` 是单向棘轮**：缩小可热改（PAUSED→PLAYING 脉冲），**放大必须过 READY**
（`1x1 --直接 set--> 473x266` 无效；`1x1 --READY → apply_rect → PLAYING` 有效）。**禁止**
用"销毁重建"来放大：新管线等 IDR，重建期 PLI 也发不出去，画面永远回不来。

**5. 不要给视频分支加 queue**。bus 会报
`Pipeline construction is invalid, please add queues.` —— **但视频那条不要加**：实测加了
之后 `mppvideodec` 输出全零缓冲，画面纯绿 `RGB(0,135,0)`。**音频那条 queue 必须保留。**

## 会话生命周期与看门狗

媒体线程靠 JS 的 `poll()` 心跳判断小程序还活着，否则 KMS 平面永久盖住 UI。比较**必须防
无符号下溢**：

```c
if (now - s->last_poll_ms > WATCHDOG_MS) { teardown(); }                     /* 禁止 */
if (s->last_poll_ms && now > s->last_poll_ms &&
    (now - s->last_poll_ms) > WATCHDOG_MS) { teardown(); }                   /* 必须 */
```

`last_poll_ms` 由 JS 线程写、`now` 在媒体线程读；取完 `now` 后 JS 正好写了个更大的值，
`now - last_poll_ms` 就下溢成 `2^64-1`：`JS heartbeat lost (18446744073709551615 ms)`。
**`volatile` 挡不住这种语义竞态**——它只防编译器优化掉读取。另外进媒体循环时先自己刷一
次 `last_poll_ms`，否则首个 `poll()` 前会误杀。

## 症状 → 方向

| 症状 | 先查 |
|---|---|
| 有 STUN 响应，DTLS 一个字节都发不出去 | `SSL_OP_NO_QUERY_MTU` 后漏了 `SSL_set_mtu()`（`e=5 errno=0`） |
| DTLS 有记录头回调、零握手报文 | 网关 STUN keepalive 被喂进记录层，做首字节分类 |
| `handshake fail e=5 errno=89(EDESTADDRREQ)` | UDP 没 `connect()`，`SET_PEER` 不等价 |
| DTLS 正常、每包 `auth_fail` | exporter `label_len` 写成 21（应 19）；或收流用了 client_write |
| 没有 Binding Response、也没有任何报错 | ufrag 前缀写死 11（应 12）多冒号；或 `ERROR-CODE` 掩码写成 `0x7f`（487 读成 787） |
| 出画一瞬间然后永久黑屏 | `recvfrom` 的 `ECONNREFUSED`/`ENETUNREACH` 被当致命错误 |
| 画面跑到最右侧并被拉伸 | `py` 写成逻辑 y，应为 `960 - x - w` |
| 画面顺时针转 90° / 纯绿 `RGB(0,135,0)` | 缺 `mppvideodec rotation=270`；或视频分支加了 `! queue` |
| 管线 PLAYING、包在涨、没画面 | `KMS_SINK_PROPS` 漏前导 `!`（`state=3 pending=4`） |
| 退出后画面残留 | `GST_STATE_NULL` 传了 0（应 1），且要等 `get_state` 落地 |
| 有包没声音、`hw_ptr=0` | opus caps 缺 `channel-mapping-family=0` |
| 连发几十次 PLI 拿不到 IDR | PLI 的 sender SSRC 等于 media SSRC，被环路检测丢 |
| 切画质后花屏/纯绿 | 分辨率恒 1280x720，须看 SPS 的 profile/level |
| 中途被 `JS heartbeat lost (18446744073709551615 ms)` 强拆 | 看门狗缺 `now > last_poll_ms` |

## 未在此机型验证

- **端到端出声**：音频只验到 `audio pipeline PLAYING` 与 caps 协商通过。
- **TURN / DTLS 1.3**：allocate 401 → realm → nonce 通了但没用上；DTLS 只用 1.2。
- **SCTP/DataChannel**：`m=application` 只回落一段 `a=recvonly`。
