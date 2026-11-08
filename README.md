# haasui-falcon-app-x6pro

写给有道词典笔 **X6 Pro**（Rockchip RK3562）Falcon 小程序运行时的开发笔记。

这份文档来自三个真实上线的词典笔应用：一个本地/网络视频播放器、一个直播客户端，
以及一个把第三方云游戏服务搬上笔的串流客户端（自写 ICE / DTLS-SRTP / RTP，
硬解上 KMS 屏，含触摸与文本输入回传）。它不是"平台 API 大全"，而是
**一份把踩过的坑按因果链整理出来的记录**——每条"必须/禁止"背后都有一次真机实测，
出处和反例都写在正文里。目标是让你在写新软件时，不必重新踩一遍。

## 里面有什么

| 文件 | 内容 |
|---|---|
| [SKILL.md](SKILL.md) | 总览：设备画像、三条技术路线、最贵的坑、四组交付检查清单 |
| [references/architecture.md](references/architecture.md) | 分层、线程模型、命令队列、状态机、teardown 顺序 |
| [references/pipeline-video.md](references/pipeline-video.md) | 播放管线、seek、起播缓冲门、KMS 矩形、直播、硬解 |
| [references/webrtc-realtime.md](references/webrtc-realtime.md) | 实时流：自写 ICE / DTLS-SRTP / RTP，喂 `appsrc`，KMS 上屏 |
| [references/audio-route.md](references/audio-route.md) | 音量权威源、蓝牙路由、音画偏移、`fdsink`+`aplay` 备选路线 |
| [references/native-jsapi.md](references/native-jsapi.md) | 写 native `.so`：交叉编译、stub 链接、JSON ABI、错误码分级 |
| [references/falcon-ui-dialect.md](references/falcon-ui-dialect.md) | Weex 方言、类名前缀、页面栈与返回动画、IME、渲染验收 |
| [references/live-input.md](references/live-input.md) | 把触屏操作送进远端会话：两套坐标单位、异常静默、点击验收 |
| [references/debugging.md](references/debugging.md) | 取证纪律、平面采样、帧率测量、三个假阳性案例 |
| [references/platform-pitfalls.md](references/platform-pitfalls.md) | 屏幕保活、输入法、打包安装、系统限制、UI 布局 |
| [references/service-reverse-ops.md](references/service-reverse-ops.md) | 逆向第三方服务、错误码与单位、发布流程、隐私自检 |

## 如果你只读一段

读这个，能省掉最多时间：

> **不要相信"操作返回成功"。** 这个平台上有太多"设了但没生效"的属性：
> `render-rectangle` 在 PAUSED 下设了不读、`hal-audio` 写配置文件要 8~12 秒才落盘、
> `alsasink` 改 `device` 在 PLAYING 下无效、`gst_element_get_state` 会在状态过渡中
> 返回 FAILURE。判断任何事都要去看**外部可观测的物理事实**：DRM 平面的
> `dst: pos[…] rect[…]`、平面 buffer 的地址变化、音频字节数有没有涨。
> 你自己的返回值、日志里的一句"成功"、JS 侧的 `{ok:true}`，都不能作为证据。

## 适用边界

- ✅ 面向词典笔的 Falcon / QuickJS 小程序，尤其是需要自己驱动媒体管线的。
- ✅ 调试音画不同步、起播跳秒、暂停恢复卡顿、进度条回弹、画面遮挡、蓝牙无声。
- ✅ 设备上没有 `webrtcbin`/`libsrtp`/`libnice`，但你需要接一路实时视频流（自写 ICE + DTLS-SRTP）。
- ✅ 需要自己写 native JSAPI 模块（`.so` + 设备动态加载）来绕开 JS 层的能力缺口。
- ✅ 需要把一个第三方 web 服务逆向成客户端（登录、机位、鉴权、加密 body、错误码），并合规地发布。
- ⚠️ 文中所有具体数值（分辨率、平面 id、音量映射、时间阈值）都是**某一份固件上的实测**。
  换机型或换固件必须重新做设备画像，不要照抄。
- ❌ 纯 UI 应用不需要看这里，直接用平台高层 JSAPI。

## 使用方式

这是给 AI 编码助手用的 skill（也可当人读的手册，`references/` 下每份文件都能独立阅读）。

```sh
git clone https://github.com/56dz/haasui-falcon-app-x6pro-skill.git \
  ~/.dsh/skills/haasui-falcon-app-x6pro
```

**注意目录名**：`SKILL.md` 的 frontmatter 是 `name: haasui-falcon-app-x6pro`，
所以克隆下来要落到**同名目录**（如上），否则部分加载器会以目录名注册它。

之后当任务涉及"词典笔 / Falcon / 小程序音视频 / 实时流 / Weex 界面"时，agent 会自动加载它。

各文件都可以单独读，不必按顺序：

| 你的处境 | 直接读 |
|---|---|
| 第一次接触这个机型 | [SKILL.md](SKILL.md) 的「先做设备画像」 |
| 有画面但没声音 / 蓝牙没声 | [references/audio-route.md](references/audio-route.md) |
| 画面出不来、位置或方向不对 | [references/pipeline-video.md](references/pipeline-video.md) |
| 要接一路实时流（云游戏 / 远程桌面） | [references/webrtc-realtime.md](references/webrtc-realtime.md) |
| 要写 native `.so` | [references/native-jsapi.md](references/native-jsapi.md) |
| 界面构建过了但真机走样 | [references/falcon-ui-dialect.md](references/falcon-ui-dialect.md) |
| 点了没反应（远端会话） | [references/live-input.md](references/live-input.md) |
| 要逆向一个第三方服务并发布 | [references/service-reverse-ops.md](references/service-reverse-ops.md) |
| 不知道从哪下手、怎么取证 | [references/debugging.md](references/debugging.md) |

## 关于

这些笔记记录的是**我们这一路怎么错的**，而不只是最后怎么对的。因为在这个平台上，
"正确答案"往往只能从"错误答案为什么错"里推出来——比如"为什么不能给 alsasink 加
ts-offset"，比"ts-offset 应该加在哪"更能防止你下次绕回去。

如果你在别的固件/机型上验证出不同的结论，欢迎改。

## 许可

GPL-3.0。见 [LICENSE](LICENSE)。

本文档不含任何个人信息：设备序列号、蓝牙 MAC、账号 ID、私有 appid、内网路径
以及第三方服务的签名参数均已移除或替换为占位符。
