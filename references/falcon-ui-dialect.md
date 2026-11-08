# Falcon/Weex UI 方言：样式、布局、页面栈、输入法

最贵的一课：**构建器一路绿灯，真机照样错**。每条都带真机现象、错误串或 grep 计数。

前置阅读 [platform-pitfalls.md](platform-pitfalls.md) 的「UI 布局约束」「触摸」两节（画布尺寸、
`scroller` 里 `text` 不能带 `lines`、flex 尺寸写死、没有 evdev 注入）；本文只补充深化。

## 样式管道：三条"构建过、真机错"

### 每个 `.vue` 必须自带 `style` 块，`@import` 无前缀

```html
<style lang="less" scoped>@import "base.less";</style>
```

- 漏掉整个 `<style>` 块 → **黑屏，无任何报错**。产物里没记录，运行时也不报。
- 写 `@import "../../styles/base.less";` → LessError 构建失败。`app.json` 已配
  `"options": { "style": { "lessPaths": ["styles"] } }`，`base.less` 相对 `src/styles/` 解析；
  9 个 `.vue` 一律无前缀（`grep -rn '@import' --include=*.vue` 核对过）。

复查 `for f in src/pages/*/*.vue src/components/*.vue; do printf "%-34s " "$f"; grep -c '<style' "$f"; done`
每行末尾都必须是 `1`。

### `<style scoped>` 实际是全局合并的

`scoped` 在这运行时**不产生隔离**：所有 `.vue` 的样式合并进同一张表，后加载的覆盖前面的，
两个页面写同名类就互相污染。

真机证据：`index.vue` 删掉右上角用户名后搜索框文字仍居中 —— `search.vue:492` 的 `.body`、
`detail.vue:497` 的 `.topbar` 与 `index.vue` 同名类一起生效。补 `flex-direction: row` 后像素实测
从 x=514\~613 左移到 x=180\~279。

现状统计（本项目 9 个文件）：

```text
distinct class : 344 | defined in >1 file : 26
.body .retry-btn .retry-text .state-row .state-text .topbar ×3 文件
.back-btn .back-t .sec-head .sec-sub .grid .lcard …         ×2 文件
```

**规则：每个页面/组件用自己的前缀，覆盖率是 code review 必查项。**

覆盖率实例：`play.vue` = `p2-`(71/73)、`mine-body.vue` = `mb-`(68/68)、`index.vue` = `ih-`(53/60)；
**反例** `queue.vue` 有 15 个裸类名。复查：对每个 `<style>` 块取 `\.([a-zA-Z0-9_-]+)` 输出不带 `-`
的名单，凡有名字就该有前缀。`base.less` 的公共类（`.page` `.bar` `.card` `.btn` `.chip*` `.title`
`.hr`）**不要**加前缀 —— 它们本来就是共享的，前缀化会让公共组件失效。

### 只支持单类名选择器

```less
.a .b { }   .a > .b { }   .a.b { }   .a:hover .b { }   /* 全部构建失败 */
.a { }   .a:active { }                                 /* 唯一合法形态 */
```

构建期报 `Selector .x .y is not supported`，**整包不产出**；9 文件 compound = 0。要复合语义就拆
两个类名叠加 `:class="tab === t.id ? 'tab tab-on' : 'tab'"`。`:active` 可用（`.gcard:active` 真机有
按压反馈），带后代的绝对不行。

## 布局

### `flex-direction` 默认是 `column`

Weex 盒子默认纵向主轴。不写 `flex-direction: row` 时 `justify-content` 管**纵向**、`align-items`
管**横向**，和 CSS 直觉相反。

典型症状：**"删掉一行东西之后，文字变竖排了"**（真机在 `search-entry` 上复现）：

```less
.search-entry { height: 26px; justify-content: center; }   /* 错：文字被横向居中，像"没生效" */
.search-entry { flex-direction: row; justify-content: flex-start; align-items: center; }  /* 对 */
```

`<text>` 本身也是 flex 容器，单行文字水平居中要 `text-align: center` 而非 `justify-content`。

### `flex: 1` 无效，要写三连

```less
.body { flex: 1; }                                    /* 无效：简写被丢弃，高度退化为内容高 */
.body { flex-grow: 1; flex-shrink: 1; flex-basis: 0%; }  /* 有效 */
```

`flex-basis` 写 `0` 或 `0%` 都能渲染（48 处里 39 处 `0%`），**统一写 `0%`**；复查
`grep -rn "flex: 1" src/` 必须 0 命中。三连只用于"吃掉剩余空间"，**固定高度的区域照样直接写
`height` + `flex-grow: 0`**（见 platform-pitfalls「flex 尺寸要写死」），两者不冲突。

### 没有可靠的 `position: absolute`，也没有 `z-index`

这两个属性**没有可用实现**（全项目 `grep "position: absolute\|z-index"` = 0 命中，不是没人试过，
是试过之后全删了）。浮层/遮罩/弹窗只有一条路：**`v-if` 整块替换内容区**。

```html
<!-- 未出画：整块不透明遮罩顶掉视频区，而不是"盖在上面" -->
<div v-if="!vizOn" class="mask"><text class="p2-mask-title">{{ maskTitle }}</text></div>
```

代价是底层内容被卸载，弹层状态必须提前进 `data()`。`mine-body.vue` 的确认弹层都这样写（`mb-modal` + `v-if`），全文件 `position`/`z-index`/`absolute`/`flex: 1` 零命中。

### 高度是硬预算：先算账

本项目实测账（首页内嵌「我的」栏目）：

```text
.content 266 → −.topbar 38 = .ih-mine-wrap 228 → −.mb-head 32 = .mb-body 196（组件内 scroller 滚动）
```

左侧竖导航：`pt8 + brand38 + 6×(h33+mt4) + pb8 = 276 > 266`，**第 6 个 tab 被裁 10px**；改
`h31 + mt3` 得 `258` 才全可见。`.tab` 在 `base.less` 属共享区，只能用页面新前缀类覆盖。

## 文本

### `<text>` 的内容不能换行

```html
<!-- 错：构建过，真机换行被渲染成空白或整段不渲染 -->
<text class="tip">
  这句话
  换行了
</text>
<text class="tip">对：单行书写</text>
```

9 个 `.vue` 里多行 `<text>` = 0。

### CJK 不会自动裁切

长中文**既不出省略号，也不换行**，直接顶出容器压到邻居身上。`lines: 1` 本机型不可靠（仍有 27 处
`lines: N` 残留在 `search.vue`/`login.vue`/`detail.vue`，属历史包袱，勿模仿）。

可靠组合是 **JS 硬截断 + 定高 `overflow: hidden` 容器**：

```js
clipName(name, max) {
  const s = String(name || ''), n = max || 6;
  return s.length > n ? s.slice(0, n) + '…' : s;
}
```

```html
<div class="ih-notice-box"><text class="ih-notice-text">{{ noticeText }}</text></div>
```

```less
.ih-notice-box { height: 18px; overflow: hidden; flex-grow: 1; flex-shrink: 1; flex-basis: 0%; }
.ih-notice-text { font-size: 12px; line-height: 18px; color: @text-color-dim; }
```

宽度→字数自己算：74px 窄卡 + 10px 字号 = 6 汉字；278px 列表卡 = 16 字。

### `line-height` 必须显式写

`<text>` 默认行高按 `font-size × 1.3~1.4` 算。只给 `height` 不管 `line-height` 会溢出父容器
—— `overflow: hidden` 只裁容器盒子，裁不住已画出去的字形。

```less
.gcard-line { width: 70px; height: 13px; margin-top: 3px; overflow: hidden; }
.gcard-name { font-size: 10px; line-height: 12px; text-align: center; }
```

### 把文案写成 method 会渲染出函数源码

```js
methods: { countText() { return this.total + ' 款'; } }   // 错
```

```html
<text>{{ countText }}</text>
<!-- 真机输出：function bound countText() { [native code] } -->
```

运行时对模板里方法/computed 求值不可靠 —— 拿到函数对象本身，把源码画屏上。**规则：展示文案
一律是 `data()` 的字符串字段**，数据到达时由方法写入。

```js
data() { return { countText: '', moreText: '加载更多', noticeText: '' }; },
methods: {
  setCount(n)  { this.countText = n + ' 款'; },
  setNotice(t) { this.noticeText = this.clipText(t, 58); },
}
```

同源性能坑：`v-for` 每帧重跑模板表达式，24 卡 × 2 表达式在 4×A35 上肉眼可见卡顿。截断/拼接要
**预计算到字段**（`g.sName`/`g.dName`/`g.dSum`），模板只写 `{{ g.sName }}`。

## 过渡动画

### `transition` 简写不要用

`falcon-styler` 的 validator 承认 `transition`/`transform`/`animation` 属性名（`validator.js:732-749`），
`transform` 甚至透传不校验（`TRANSFORM_VALIDATOR` 只有 `// TODO return { value: v }`）。**能过构建
≠ 真机能动**：`transition` 不触发，元素直接跳终态。

### `@keyframes` + `animation` 可用

构建期 `src/steps/build.js:42-52` 的 `appendKeyFrameCode()` 检查 `share.keyframes.length > 0` 后把
`$falcon.__KEYFRAMES = [...]` 追加写进 App 入口；`falcon-jsfm` 运行时 bundle 里 `__KEYFRAMES`
出现 2 次，确实被读。真机验证通过：

```less
@keyframes ihFadeIn { from { opacity: 0; transform: translateX(8px); }
                      to   { opacity: 1; transform: translateX(0); } }
.ih-fade { animation-name: ihFadeIn; animation-duration: 220ms;
           animation-timing-function: ease-out; animation-fill-mode: forwards; }
```

`animation-timing-function` 只接受 `linear|ease|ease-in|ease-out|ease-in-out|cubic-bezier(n,n,n,n)`
（`validator.js:480`），别写 `ease-out-back`。

### 让"两种加载路径"看起来一样

动画不解决"半空/抖动"。原病根：`setTab()` 先清空四个数组再 async 拉数据，两个接口（banner+模块
vs 列表）时序不同，视觉上就成了"部分滑进、部分直接出现"。三条一起做才有效：

1. **不清空旧帧**。切栏目只置 `loading`，旧数据留到新数据到齐。
2. **一次性提交**。所有块在 `Promise.all` 之后一起赋值，中间不 `await` 渲染。
3. **统一入场**。提交完 `fadeKey++`，模板 `<scroller :key="fadeKey" class="body ih-fade">` 靠
   `:key` 变化重挂载重放动画 —— 两条路径用同一个类、同一个 key。

`loading` 只在**首屏**整页替换（`v-if="loading && !hasAny"`）；有旧帧时错误降级成顶部一条小提示
（`v-if="errText"` 的 `.ih-err`）。

## 页面栈与返回

### `$falcon.navBack` 在本固件不存在

框架 bundle（`/etc/miniapp/resources/framework/jsfm-nvue.js.bin`、`js-framework.min.bin`）里
`grep -c navBack` **都是 0**，可见 API 只有 `navTo / closeApp / closePageByName / closePageById`。

所以 `typeof $falcon.navBack === 'function'` 恒假，兜底 `navTo('index')` **不是返回，是新压一层
index**：动画从 pop 变 push（用户报的"退出也是向左划入"）、栈越堆越深、首页数据全丢。

```js
navBack(pageName) {
  if ($falcon.closePageByName) { $falcon.closePageByName(pageName); return true; }
  return false;
}
```

`$falcon.navBack` 的兼容分支（真有该 API 的固件）留着，但**不能当主路径**。

### 用连拍证明动画方向

`miniapp_cli capture` 连拍约 **75ms/帧**，3~4 张即可判方向（要留证据就连拍 16 帧）：新页从右压入、旧页不动 = **push**
（错，"exit 也向左划入"就是它）；旧页占住 0..W、当前页整体右移滑出 = **pop**（对）。留档
`nav_back_push_bug.png`（搜索页停在 0\~525，index 从 540 压入）与 `nav_back_pop_fixed.png`。

### `navTo` 的参数在部分固件会丢

`$falcon.navTo(name, options)` 的 options 不一定透传到目标页。参数要走模块级通道并带过期时间
（避免残旗很久以后误触发跳转）：

```js
export function takeTabParam() {
  if (!pendingTab) return '';
  const { tab, at } = pendingTab;
  pendingTab = null;                          // 取一次即销
  return Date.now() - at > 30000 ? '' : tab;  // 30s 过期，防残旗误触发
}
```

同文件还有 `takePending(name)`（普通参数）与 `requestHome(hops)`（"一路退回首页"接力旗：每页在
自己 `onShow` 取一次，有剩余就接着关自己 —— 关自己时 JS 上下文连同定时器一起没了，所以"先关自己
再 setTimeout 关别人"必然停在中间页）。

### 页面级 `onShow` 可靠，`onHide`/`onUnload` 不可靠

真机实测（`miniapp_cli start <别的 appid>` 后翻日志）：

| 钩子 | 回调 |
|---|---|
| 页面 `onShow` | ✅ 可靠，"回到这页"的唯一可靠钩子 |
| 页面 `onHide` / `onUnload` | ❌ 实测都不触发（play 页） |
| **App `onHide`** | ✅ **会触发** |

所以"切后台/直接退出时的收尾"（停管线、还 KMS 平面、退机位）**必须写在 `app.js` 的 `onHide`**；
写在页面里等于没写 —— 退出后视频最后一帧会一直盖在系统界面上。该钩子还要带输入法守卫：系统
键盘是另一个全屏 miniapp，弹出/收起也触发 App `onHide`，此时拆管线会把整局游戏拆掉。

```js
onHide() {
  super.onHide();
  if (inputBusy()) return;     // 键盘使用中，不是真退后台
  stopPlay();
  if (streaming && code) stopGame(code);
}
```

`base-page.js` 另补 `onStart`/`onStop`（`libfalcon.so` 有这两条日志），谁先到用谁。

### 组件引用三级回退 + 重试

`v-if` 挂载的组件首次 `onShow` 时**可能还没 ready**，`this.$refs.body` 在 Weex 渲染树上常取不到
（渲染树和 Vue 实例树不是一一对应）：

```js
const pick = () => (this.$refs && this.$refs.body)
  || (this.$root && this.$root.__mineBody) || this.__mineBody;
if (run()) return;
setTimeout(run, 120); setTimeout(run, 260);   // 两轮总约 260ms
```

组件侧 `mounted` 里 `this.$root.__mineBody = this`，`beforeDestroy` 清掉。

## 系统输入法

没有 `<input>`/`<textarea>`，真接口是 `global` 模块：

```js
import globalModule from 'global';
const m = new globalModule.Global();
const uuid = m.startTextEdit(JSON.stringify({ text, placeholder, maxlength, autofocus: true }));
m.textEditFinished.on(handler);                  // 模块信号
$falcon.on('textEditFinished', handler);         // 总线信号，两条都要订阅
setTimeout(() => m.closeTextEdit(uuid), 60);     // 关闭必须延迟，同步会重入卡死
```

**竞态（真机日志实证）**：`open()` 开头先 `_closeCurrent()` 关掉上一个仍挂着的面板，这个关闭动作
异步抛一个**空结果** `textEditFinished`，把新会话 pending 解析成 `{confirmed:false, text:''}`。日志
顺序是 `edit ok=false` 出现在 `startTextEdit 返回 uuid=...` **之前**；用户不可能 800ms 内打完
确认，所以判据是"极快返回的空结果"：

```js
for (let attempt = 0, t0; attempt < 3; attempt++) {
  t0 = Date.now(); r = await im.open(opts);
  const empty = !r || (!r.confirmed && !r.text);
  if (!empty || (r && r.error) || Date.now() - t0 > 800 || r.canceled) break;
  await sleep(250);
}
```

其他：回调值字段不稳定（`value`/`text`/`editText`/`contents`/`content`）要归一化；第二参数可能是
**字符串化 JSON** 要递归 `parse`；`requestIMAppClose` 只记日志、**不能**当完成回调（会误触发
`closeTextEdit` 关键盘）；页面销毁只解除订阅、**不关面板**；真起不来才退化为自绘屏幕键盘（整块
`v-if` 替换内容区）。

## 触摸事件与坐标

> 本节只讲**本地事件对象长什么样**。要把这些坐标再发给远端会话（云游戏 / 远程桌面），
> 单位的换算规则完全不同，见 [live-input.md](live-input.md)。

坐标只用 `pageX/pageY`。`libfalcon.so` 只暴露 `pageX/pageY/screenX/screenY/identifier/
changedTouches`，**没有** `clientX`/`offsetX`（全项目 `grep clientX|offsetX` = 0 命中）。

关键是 `pageX/pageY` 是**相对接收事件的元素**的局部坐标：

```text
三栏播放页 VIDEO.x = 243
点 (111,480) → 逻辑 (480,4) → 回调 page = (237,3)
237 = 480 − 243  ← 已是 stage 内坐标，不要再减 VIDEO.x
```

`pageX ∈ [0,473]` 对应 stage 水平、`pageY ∈ [0,266]` 垂直。**"再减子矩形原点"都是错的**（历史改
过一次，点击整体偏移，已撤回）。事件对象只认 `e.changedTouches`：
`onTS(e) { const ts = (e && e.changedTouches) || []; ... }`

`e.touches` 在 `touchmove` 上常为空，快滑时 `touchmove` 甚至被丢弃 —— **落点必须从 `touchend` 的
`changedTouches` 重取**，拿不到才退回 `start + 累计位移`：

```js
onBarEnd(e) {
  const ts = (e && e.changedTouches) || [];
  let end = null;
  if (ts.length) { const y = Number(ts[0].pageY); if (isFinite(y)) end = y; }
  if (end === null) end = this.barY + this.barMoved;
  const dy = end - this.barY;
  if (Math.abs(dy) < 40) return;          // 小于阈值当点击
  this.stepTab(dy < 0 ? 1 : -1);          // 上滑 = 下一个
  this.swipeLock = true;                  // 先切后上锁
  setTimeout(() => { this.swipeLock = false; }, 400);
}
```

顺序关键：**先 `stepTab` 再上锁**，反过来会吞掉自己这次的 `setTab`；锁只吞滑动后系统补发的 `click`。

## 渲染验收

`miniapp_cli capture` 只拍 **UI 合成面**（不含 KMS 视频平面）。首屏等 **12\~15 秒**（native 加载慢）：

| 截图字节数 | 含义 |
|---|---|
| 几十~几百 KB | 正常 |
| **1086 字节** | 纯白：背光关着，`ret:0` 但没内容（`hal-screen` 见 platform-pitfalls） |
| **1667 字节** | 纯黑：没有合成面，页面被切走或没启动 |

## UI 症状 → 排查方向

| 症状 | 方向 |
|---|---|
| 整页黑屏、无报错 | 缺 `<style lang="less" scoped>` 块 |
| 构建报 LessError | `@import` 路径带了 `../../styles/` 前缀 |
| 构建报 `Selector ... is not supported` | 用了 `.a .b` / `.a > .b` / `.a.b` |
| 改了 A 页 B 页也变 | `<style scoped>` 全局合并，撞了类名 |
| 文字变竖排 / 居中失效 | 忘写 `flex-direction: row` |
| 布局不撑满/塌成内容高 | `flex: 1` 没写三连 |
| 弹层盖不住/透出底页 | 无 `absolute`/`z-index`，必须 `v-if` 整块替换 |
| 长中文压到邻居 | 没 JS 截断 + 没定高 `overflow: hidden` |
| 文字溢出卡片 | 只给 `height`，没写 `line-height` |
| 屏幕出现 `function bound xxx()` | 文案写成了 method/computed，要放 `data()` |
| 切栏目"部分滑进、部分直接出现" | 先清空数组再异步拉数，两路径入场不一致 |
| 入场动画不生效 | 用了 `transition`，本机型不触发，改 `@keyframes` + `animation` |
| 返回动画方向反了、要按两次 | 用了 `navTo` 兜底压深了栈，改 `closePageByName` |
| 退出后画面还在 | 收尾写在页面 `onHide`/`onUnload`（不回调），要写 App `onHide` |
| 拿不到子组件实例 | `$refs` 在 Weex 不可靠，要三级回退 + 重试 |
| 输入框拿到空串 | IME 空结果竞态，`<800ms && 空 && 无 error` 时重试最多 3 次 |
| 点没反应 | 用了 `e.touches`/`offsetX`，只认 `e.changedTouches[0].pageX/pageY` |
| 点击位置整体偏移 | 对已是局部坐标的 `pageX/pageY` 又减了一次矩形原点 |
