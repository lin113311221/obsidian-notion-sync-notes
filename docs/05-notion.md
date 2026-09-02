# Notion API：五个文档里没写清的坑

基于真实实现（Notion-Version `2026-03-11`）。这几条都踩过，每一条都花了时间才定位。

---

## 坑 1：`database_id` 和 `data_source_id` 是两回事

这是新版 API 最容易卡住人的地方。你从 URL 里复制出来的 `database_id`，
**不能直接用在 `/data_sources/{id}/query` 上**，会 404。

要先换一次：

```js
const db = await api('/databases/' + databaseId, {});
const dsId = db?.data_sources?.[0]?.id || databaseId;  // 老版本可能没这个字段
// 之后所有操作用 dsId
await api('/data_sources/' + dsId + '/query', { ... });
```

创建页面时同样要用 `data_source_id`：

```js
{
  parent: { type: 'data_source_id', data_source_id: dsId },
  properties: { ... }
}
```

**注意 `|| databaseId` 这个兜底**：老版本接口没有 `data_sources` 字段。
不要因为这个字段缺失就整个崩掉。

---

## 坑 2：属性里的富文本有 2000 字符硬上限

`rich_text` 类型的单个 text 超过 2000 字符，请求直接 400。

```js
function clip(s, max) {
  const str = String(s == null ? '' : s);
  return str.length > (max || 2000) ? str.slice(0, (max || 2000) - 1) + '…' : str;
}
```

标题、作者、合集名全都过一遍。别嫌啰嗦 —— 长标题是常态。

---

## 坑 3：正文别手工切 block，用原生 markdown 字段

这条最省事，也最少人知道。

传统做法是把 Markdown 解析成 Notion block 数组，然后手工处理：
- 每个 block 的 rich_text ≤ 2000 字符
- 每次请求 ≤ 100 个 block

**这是几百行没必要的代码。** 创建页面时直接传 `markdown` 字段，服务端自己转：

```js
const payload = {
  parent: { type: 'data_source_id', data_source_id: dsId },
  properties,
  markdown: bodyMarkdown,   // 服务端负责转 block
};
```

记得把 frontmatter 去掉，Notion 不需要（它有自己的属性系统）：

```js
markdown: raw.replace(/^---\n[\s\S]*?\n---\n/, '')
```

> 备注：这个字段在我们的实现里实测可用（Notion-Version `2026-03-11`）。
> 如果你的版本报"未知字段"，说明版本太老，退回手工切 block。

---

## 坑 4：属性不存在时，缺了就跳过，别硬塞

用户库里的属性是你不可控的。**往一个不存在的属性写值 = 整个请求 400**。

```js
if (item.platform)  properties.Platform   = { select: { name: item.platform } };
if (item.author?.name) properties.Author  = { rich_text: [...] };
if (item.url)       properties.URL        = { url: item.url };
if (item.duration > 0) properties.Duration = { number: Math.round(item.duration) };
```

只有 `Name`（title）和去重用的 `SourceID` 是必需的 —— 这两个在引导用户建库时保证创建。

`select` 类型还要注意：值必须**已存在于该属性的 options 里**，否则报
`Could not find option`。要么预先建好 options，要么改用 `rich_text` / `multi_select`。

---

## 坑 5：错误码要翻译成人话

Notion 的报错对终端用户毫无意义。统一拦一层：

```js
if (/401|unauthorized/i.test(msg))
  throw new Error('Notion token 无效或已失效，请在设置里重新填写');
if (/403|restricted/i.test(msg))
  throw new Error('没有权限：请确认已在目标页面/数据库上「Add connections」添加你的 integration');
if (/404|not_found/i.test(msg))
  throw new Error('找不到目标：请确认页面已分享给 integration（父页面也要分享）');
if (/429|rate_limited/i.test(msg))
  throw new Error('Notion 限速，请稍后重试');
```

403 和 404 是最常见的两种，而且**都跟"分享给 integration"有关** ——
用户 90% 的情况下是忘了这一步。报错里直接告诉他怎么做，比让他去查文档强。

---

## 附：去重查询一定要带 filter

不要拉全量再在内存里比对，数据量一大就慢且翻页麻烦：

```js
await api('/data_sources/' + dsId + '/query', {
  method: 'POST',
  body: {
    page_size: 1,
    filter: { property: 'SourceID', rich_text: { equals: sourceId } },
  },
});
```

`page_size: 1` 是刻意的 —— 只要知道存不存在，不需要内容。

---

## 附：封面图直接外链，或走 file_upload

```js
// 简单：properties 里挂外链
properties.Cover = { url: item.cover };

// 想落进 Notion 自己的存储（外链不会过期）：
await api('/file_uploads', {
  method: 'POST',
  body: { mode: 'external_url', external_url: item.cover, filename: id + '.jpg' },
});
```

外链方案的隐患是**平台图床会有防盗链 / 过期**，几个月后图全裂。
如果用户在意长期保存，走 `file_uploads` 更稳。

---

**上一篇**：[04-obsidian.md](04-obsidian.md)　**下一篇**：[06-testing.md](06-testing.md)
