# Obsidian：frontmatter 规范与双链

同步进 Obsidian 最容易做成「把内容倒进去，然后变成一片孤立的文档」。
这几条决定决定了笔记是**进入你的知识图谱**，还是**只是躺在一个文件夹里**。

---

## frontmatter 字段规范

完整字段（顺序刻意固定）：

```yaml
---
source_id: xiaohongshu:65f1a2b3000000001203b8d7
platform: xiaohongshu
title: 红烧肉的家常做法
author: 老饭骨
url: https://www.xiaohongshu.com/explore/...?xsec_token=ABC...
collection: 菜谱
published: 2026-08-15
saved: 2026-08-20
duration: 0
cover: 红烧肉的家常做法-cover.jpg
tags: [xiaohongshu, 菜谱, 家常菜]
ai_summary: true
transcript: true
synced_at: 2026-08-20T10:30:00.000Z
---
```

### 为什么 `source_id` 必须放第一个

它是去重键（`${platform}:${id}`），放最前面便于**人肉排错** ——
用户打开笔记一眼就能看到这条内容的唯一标识，查重复问题时不用翻。

顺序本身不影响解析，但影响**调试效率**。

### `published` vs `saved` 必须分开

| 字段 | 含义 |
|---|---|
| `published` | 内容发布时间 |
| `saved` | **你收藏的时间** |

「我上个月收藏的」用 `saved`。混用会让按时间筛选的功能答非所问 ——
用户关心的是自己什么时候存进去的，不是作者什么时候发的。

### 日期格式

统一用 `YYYY-MM-DD`（`isoDate()` 处理），**不要带时间戳**。

原因：筛选时按字符串比较，`2026-08-20` 和 `2026-08-20T10:30:00Z` 的字典序行为不一样：

```js
'2026-08-20' > '2026-08-20'           // false，符合预期
'2026-08-20T10:30:00Z' > '2026-08-20' // true！同一天被判成"之后"
```

如果字段里混入带时间戳的值，边界会被错杀。保险做法是**比较时一律截前 10 位**：

```js
const day = (v) => String(v || '').slice(0, 10);
```

---

## 双链：让笔记进图谱

这是**同步工具最容易被忽略、但价值最高的一个细节**。

普通做法：

```markdown
作者：老饭骨
```

我们的做法：

```markdown
作者：[[老饭骨]]
```

差别是：前者是一行死文本，后者在 Obsidian 里会**自动创建一个可点击的节点**，
进入全局图谱。收藏了同一个作者的 10 篇笔记，图谱里就会有一个"老饭骨"的枢纽。

```js
infoBits.push(o.linkAuthor === false
  ? `作者：${item.author.name}`
  : `作者：[[${item.author.name}]]`);
```

保持一个开关（`linkAuthor`）—— 有些用户的库里已经有同名笔记，双链会误连。

### 标签同理

```js
tags.push(item.platform);                        // xiaohongshu
if (item.collection !== '未分类') tags.push(sanitizeTag(item.collection));
(item.tags || []).forEach(t => tags.push(sanitizeTag(t)));
const uniq = [...new Set(tags.filter(Boolean))];  // 记得去重
```

`sanitizeTag` 必须做：Obsidian 标签不能含空格和部分特殊字符，
平台标签（比如「科技, 数码」）直接塞进去会破坏 YAML。

---

## 封面本地化

默认把封面下载到本地，用相对路径引用：

```yaml
cover: 红烧肉的家常做法-cover.jpg
```

理由：**平台图床会挂**。视频下架、图片防盗链、CDN 过期 —— 几个月后图全裂。
本地化的笔记，十年后打开还是完整的。

文件名要做清理（去除 `/` `:` 等非法字符），并和笔记放在同一目录，
用相对路径而非绝对路径（否则移动 vault 就全断）。

---

## 一个容易漏的点：正文里也会有 `---`

解析 frontmatter 时，**只认文件最开头的那个 `---`**：

```js
// 错误：全文找第一个 --- 和第二个 ---
const m = md.match(/^---\n([\s\S]*?)\n---/);   // 实际上这个正则是对的（^ 锚定开头）

// 危险：没锚定开头
const m = md.match(/---\n([\s\S]*?)\n---/);    // 正文里的分隔线会被误认
```

Markdown 正文里用 `---` 做水平分割线非常常见。不锚定开头，
解析会把正文第一段当成 frontmatter，字段全乱。

---

**上一篇**：[03-bilibili.md](03-bilibili.md)　**下一篇**：[05-notion.md](05-notion.md)
