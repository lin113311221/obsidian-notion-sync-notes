# 4 平台 × 2 终点，怎么共用一份引擎

一个同步工具最容易长歪的地方：每加一个平台，就复制一份流程代码。
加到第 3 个平台时，你会发现在维护 3 份几乎一样、细节各异的代码。

---

## 三层结构

```
core/
  ├─ providers/        平台差异：怎么拿数据、怎么登录
  │   ├─ bilibili.js
  │   ├─ xiaohongshu.js
  │   └─ ...
  ├─ targets/          写入差异：写进谁、什么格式
  │   ├─ obsidian.js
  │   └─ notion.js
  ├─ model.js          归一化数据结构
  ├─ render.js         → Markdown
  └─ engine.js         唯一流水线
```

**关键点：`engine.js` 是唯一的流程代码。** provider 和 target 都不允许有自己的"主流程"。

---

## 契约：NormalizedItem

provider 的输出、target 的输入、render 的输入，都是同一个结构。
平台差异在进入流水线之前就已经被抹平了。

```js
{
  id,              // 平台原生 ID
  sourceId,        // `${platform}:${id}` —— 全局去重的唯一键
  platform,        // 'bilibili' | 'xiaohongshu' | ...
  title,
  url,
  author:   { name },
  collection,      // 收藏夹 / 合集名
  publishedAt,     // ISO
  collectedAt,     // 收藏时间（≠ 发布时间，别混）
  duration,        // 秒，图文为 0
  cover,
  tags: [],
  transcript,      // 逐字稿
  xsec_token,      // 小红书专用，见 02
}
```

### 两个容易搞混的时间字段

`publishedAt`（内容发布）和 `collectedAt`（你收藏的时间）**必须分开存**。

「我上个月收藏的」用的是 `collectedAt`，不是 `publishedAt` ——
用户关心的是自己什么时候存进去的。混用的话，按时间筛选的功能会答非所问。

### 去重键为什么带平台前缀

```js
sourceId: `${platform}:${id}`
```

不同平台的原生 ID 可能撞车（都是自增 ID 的平台尤其）。
不带前缀会在跨平台同步时误判"已存在"，直接跳过。

---

## provider 要实现什么

```js
module.exports = {
  id: 'bilibili',
  name: 'B站',
  capabilities: { loginUrl, needCookie, supportTranscript },

  validate(creds),      // 凭证是否有效
  listItems(creds, opts),   // 拉收藏列表（分页）
  fetchDetail(creds, id),   // 拉详情（正文 / 逐字稿）
};
```

就这四个。**不要往 provider 里塞渲染逻辑、去重逻辑、配额逻辑** —— 那些属于 engine。

---

## target 要实现什么

```js
module.exports.createObsidianTarget = (deps) => ({
  id: 'obsidian',
  exists(item),              // 去重：这条同步过吗
  write(item, markdown, meta),  // 写入
});
```

`markdown` 由 `render.js` 统一产出，target **不负责拼字符串** ——
否则每加一个终点就要重写一遍 Markdown 模板。

---

## 加一个新平台的真实成本

加第 3、4 个平台时，我们实际只做了两件事：

1. 写 `core/providers/xxx.js`（实现上面四个方法）
2. 在 `tests/run.js` 的 `MATRIX` 里加一行

```js
const MATRIX = [
  { provider: 'bilibili',    target: 'obsidian' },
  { provider: 'bilibili',    target: 'notion'   },
  { provider: 'xiaohongshu', target: 'obsidian' },
  { provider: 'xiaohongshu', target: 'notion'   },
  // 加平台 = 加两行
];
```

引擎、渲染、去重、配额、错误处理**一行都没动**。

> **检验分层是否合理的标准：加一个平台，需要改几处核心代码？**
> 答案是 0。如果答案是 3，说明 provider 泄漏进了 engine。

---

## 一个反例：别让 target 依赖平台

`targets/notion.js` 里有这段：

```js
const platformName = {
  bilibili: 'B站', xiaohongshu: '小红书', /* ... */
}[item.platform] || item.platform;
const title = `[${platformName}] ${item.title}`;
```

这是**唯一**一处 target 里出现平台相关知识的地方，而且只是个显示用的映射表。

它违反了分层原则吗？严格说是的。但实践上：
把它做成 provider 的一个 `displayName` 字段会更干净 —— 这是我们的技术债之一。

**记录在这里是因为：分层不是教条，但每次违反都要明确知道代价。**
这个例子的代价是"加平台时要记得来这里补一行"，而它没有任何测试能提醒你。

---

**下一篇**：[02-xiaohongshu.md](02-xiaohongshu.md)
