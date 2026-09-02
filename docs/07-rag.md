# 给笔记库做检索：为什么我们选了 BM25 而不是 embedding

先说结论，再说理由 —— 这个决定当时是反直觉的。

---

## 结论

个人笔记库（几百 ~ 几千条）做检索，**BM25 + 结构化过滤** 比 embedding 更合适，
尤其当你的用户可能用 DeepSeek 时。

---

## 理由 1：DeepSeek 根本没有 embedding 端点

这是决定性的一条。

DeepSeek 官方只提供 **`/chat/completions`**，**没有 `/embeddings`**。
（两个独立来源分别核对过：2026-04 与 2026-08 的文档/资料。）

而 DeepSeek 是国内 BYOK 场景里最常见的选择。**强依赖 embedding = 让这批用户直接跑不起来。**

你可以说"那就要求用户配 OpenAI / 其他有 embedding 的模型"，但这意味着：
用户为了用你的检索功能，得额外再申请一个他本来不需要的服务。

## 理由 2：换 embedding 模型 = 索引全量重建

这不是"效果变差一点"，是**向量空间不兼容，旧索引全部作废**。

用户今天用 A 模型建了索引，明天换成 B 模型，几万条内容要全部重新嵌入：

- 要花时间（几分钟到几小时）
- 要花钱（每千条都是 token）
- 要处理中途失败（重建到一半崩了怎么办）

BM25 的索引是纯文本的倒排表，换什么都不影响。

## 理由 3：个人库这个量级，BM25 的效果并不差

BM25 弱在"语义召回" —— 「汽车」搜不到「轿车」这类同义。
但在个人收藏库这个场景：

- 用户记得大概的用词（他当初就是这么收藏的）
- 数据量小（几百~几千条），关键词命中率很高
- **结构化过滤带来的收益，远大于语义召回的收益**

---

## 真正的差异化：元数据感知，不是向量质量

这是最容易被忽略的一点。

通用 RAG 插件（Obsidian Copilot / Smart Connections）把 Markdown **当纯文本切块**，
它们**看不见 frontmatter 里的结构化字段**。

而我们的笔记有完整 frontmatter：

```yaml
source_id: xiaohongshu:65f1a2b3000000001203b8d7
platform: xiaohongshu
title: 红烧肉的家常做法
author: 老饭骨
url: https://www.xiaohongshu.com/explore/...?xsec_token=...
collection: 菜谱
published: 2026-08-15
saved: 2026-08-20
duration: 0
tags: [xiaohongshu, 菜谱, 家常菜]
```

于是能回答这类问题：

> 「我上个月收藏的 B站视频里，哪几个讲了 Rust ownership？」

通用 RAG：整库语义召回。平台和收藏时间在嵌入向量里几乎不体现，只能碰运气。
我们：**先按 `platform=bilibili` 且收藏日期落在上个月过滤，再在子集里检索。**

**这个能力不依赖我们的 embedding 比人家好** —— 它依赖的是"能不能看见结构化字段"。
换任何向量模型都补不上。

---

## 三个实现要点

### 1. 过滤必须**先于**打分

不只是"结果更少所以更快"，而是**排序质量更好**：

```js
// 第一步：过滤出候选集
const cand = [];
for (let i = 0; i < idx.N; i++) {
  if (matchFilter(idx.entries[i], filter)) cand.push(i);
}

// 第二步：只在候选集上算 df / IDF
const df = new Map();
for (const t of qTerms) {
  const arr = idx.postings.get(t);
  if (!arr) continue;
  let n = 0;
  for (const d of arr.keys()) if (inCand.has(d)) n++;   // ← 只数候选集内的
  if (n) df.set(t, n);
}
```

如果先在全库上算 df，全库的词频分布会**稀释掉子集里的区分度**。
比如全库 3000 条里 2000 条含"视频"，这个词在全库 IDF 很低；
但锁定 B站 后，子集 800 条里只有 50 条含"视频"，区分度完全不一样。

### 2. 中文分词用 bigram，零依赖

```
「知识桥梁」→ 知识 / 识桥 / 桥梁
```

```js
const segs = s.match(/[\u4e00-\u9fff]+|[A-Za-z0-9_]+/g) || [];
for (const seg of segs) {
  if (CJK.test(seg)) {
    if (seg.length === 1) { out.push(seg); continue; }
    for (let i = 0; i < seg.length - 1; i++) out.push(seg.slice(i, i + 2));
  } else {
    const w = seg.toLowerCase();
    if (w.length >= 2 || /^\d+$/.test(w)) out.push(w);
  }
}
```

零词典、零依赖、零下载。召回略噪，但配合结构化过滤足够用。
单字要单独处理（否则「了」这种字会被完全丢掉）。

### 3. 标题加权用一个 hack：repeat(3)

```js
const title = String(e.title || '').repeat(3);
const toks = tokenize(title + ' ' + meta + ' ' + body);
```

比引入多字段加权（title_weight / body_weight）简单得多，
而且 BM25 的**词频饱和特性会自然压住重复带来的偏差** —— 正好是我们想要的效果。

### 4. 正文要截断，否则逐字稿会撑爆索引

```js
const body = String(e.body || '').slice(0, o.maxBodyChars || 6000);
```

一个小时的播客逐字稿几万字。不截断的话，单条文档的 `docLen` 会严重拉偏 `avgdl`，
导致其他所有文档的分数都被压低。

---

## 什么时候该上 embedding

诚实说明边界，BM25 不是万能的：

- 用户想搜**意思**而不是**词**（「那个讲省钱技巧的」）
- 库大到万条以上，关键词召回噪声明显
- 需要跨语言检索

我们的做法：把 embedding 做成**可选增强**，与 BM25 结果融合，默认关闭。
**不让它成为任何一个用户的必要条件。**

---

**上一篇**：[06-testing.md](06-testing.md)　**返回**：[README](../README.md)
