# 小红书：怎么绕开前端签名，而不是硬啃它

## 问题

小红书的接口带前端签名（`x-s` / `x-t` 等）。主流做法有两种：

1. **复现签名算法** —— 从混淆过的 JS 里逆向出算法，本地算出 `x-s`
2. **用无头浏览器跑真实页面** —— Puppeteer / Playwright，让平台自己发请求

我们选了 2 的变体：**内嵌平台自己的页面（Electron webview），注入脚本拦截它自己发出的请求响应。**

## 为什么不做方案 1

不是因为逆向不出来，是因为**维护成本是持续的**：

- 签名算法一改，全线崩溃，用户今天能用明天不能用
- 风控升级（验证码 / 设备指纹）时，复现算法这条路直接断
- 你在跟一个专职团队对抗，他们改代码的速度比你逆向快

我读过同类插件 `rednote2obsidian` 的 `main.js`（296 KB）：
全文 `signature` 出现 **0 次**，`window._webmsxyw`（签名注入点）出现 **0 次**，
请求的外部域名只有 `edith.xiaohongshu.com`。

**这说明一件事：登录只靠用户自己的 Cookie 就够了，根本不需要签名。**

这个反直觉的事实值得单独强调 —— 网上大量教程在教怎么复现 `x-s`，
但如果你只是要用户**自己的**收藏夹，走 Cookie 登录是更简单也更稳的路。

## 方案：webview + 响应拦截

```
┌─ Obsidian 插件 ────────────────────────────┐
│  <webview src="小红书" partition="persist"> │
│      │                                     │
│      ├─ 用户正常登录（Cookie 存进 partition）│
│      ├─ 注入脚本：劫持 XHR/fetch 响应        │
│      └─ 拉回 window.__clipin_captured       │
└────────────────────────────────────────────┘
```

核心是三个函数：

```js
// 1) 导航
webview.src = url;
await waitDomReady(45000);

// 2) 注入拦截脚本（把响应塞进 window.__clipin_captured）
await webview.executeJavaScript(INJECT_SCRIPT);

// 3) 取回
const captured = await webview.executeJavaScript('window.__clipin_captured || []');
```

拦截脚本要做的事很简单：包装 `XMLHttpRequest.prototype` 和 `window.fetch`，
把命中的响应体 push 到 `window.__clipin_captured`。

**签名的活儿由平台自己的页面完成** —— 我们只是在旁边"听"。
所以算法怎么改，都不影响我们。

## 四个踩过的坑

### 1. 每次导航后必须重新注入

`dom-ready` 会重新创建 JS 上下文，之前注入的脚本全没了：

```js
webview.addEventListener('dom-ready', () => inject(pattern), { once: true });
```

漏了这行，表现是"第一次能抓到，跳转后就没数据了" —— 很容易误判成登录失效。

### 2. partition 要持久化，否则每次都要重新登录

```js
webview.setAttribute('partition', `persist:clipin-${providerId}`);
```

不加 `persist:` 前缀，Cookie 关掉窗口就丢，用户每次同步都要扫码。

### 3. 按文字点击，不要按 class

小红书的「收藏」标签没有稳定的 class（构建产物会变）：

```js
// 遍历元素比对 innerText，而不是 querySelector('.some-hash')
const els = document.querySelectorAll(selector);
for (const el of els) {
  if ((el.innerText || '').trim() === text) { el.click(); return true; }
}
```

### 4. 链接必须带 `xsec_token`，否则过一阵就打不开

小红书的笔记链接：

```js
`https://www.xiaohongshu.com/explore/${noteId}${xsec_token ? '?xsec_token=' + xsec_token : ''}`
```

`xsec_token` 是一次性的访问凭据，跟随列表接口一起下发。
**存笔记时把它一起存进 URL**，否则几周后点开就是「内容不存在」。

这个坑很隐蔽：同步当天一切正常，等用户回头翻笔记时才发现链接全废了。

## 这条路什么时候会失效

诚实说明边界：

- 平台改版 DOM（比如「收藏」改成别的入口）→ 需要更新选择器
- 加入更强的风控（比如检测 Electron 环境）→ 目前没有，但不能排除
- **登录态过期** → 用户需要重新登录，这是设计上接受的代价

对比方案 1 的"算法一改全线崩"，这些失效模式的修复成本都低得多。

---

**下一篇**：[03-bilibili.md](03-bilibili.md) —— B站相反，接口直连更稳，但 AI 字幕要登录态。
