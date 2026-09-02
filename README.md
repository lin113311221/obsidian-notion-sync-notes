# obsidian-notion-sync-notes

把小红书 / B站 的收藏同步进 **Obsidian** 和 **Notion** —— 一个真实上线产品在集成过程中踩过的坑。

这不是教程，也不是 SDK。是**工程笔记**：哪些做法跑通了、哪些是死路、哪些坑藏得很深。

代码来自 [知识桥梁 Knowledge Bridge](https://github.com/lin113311221/bili2obsidian-plugin)（Obsidian 插件，2 平台 × 2 终点）。
笔记刻意抽掉了业务代码，只留**可复用的判断和边界条件**。

---

## 目录

| 文档 | 解决什么问题 |
|---|---|
| [01-architecture.md](docs/01-architecture.md) | 4 平台 × 2 终点怎么共用一份引擎，加平台不改核心 |
| [02-xiaohongshu.md](docs/02-xiaohongshu.md) | 小红书有前端签名风控，怎么绕开而不是硬啃 |
| [03-bilibili.md](docs/03-bilibili.md) | B站 SESSDATA、收藏夹接口、AI 字幕要登录态 |
| [04-obsidian.md](docs/04-obsidian.md) | frontmatter 规范、双链、让笔记进图谱而不是变孤点 |
| [05-notion.md](docs/05-notion.md) | Notion API 的 `database_id` ≠ `data_source_id`、2000 字符陷阱 |
| [06-testing.md](docs/06-testing.md) | Obsidian 插件怎么测（`require('obsidian')` 进不了 Node 测试套件） |
| [07-rag.md](docs/07-rag.md) | 给笔记库做检索：为什么选 BM25 而不是 embedding |

---

## 几条贯穿全文的判断

这几条比具体代码值钱：

1. **不要跟平台的前端签名硬碰硬。** 算法一改，全线崩溃。改用内嵌平台自己的页面、监听它自己的请求响应——签名怎么变都不影响你（[02](docs/02-xiaohongshu.md)）。
2. **依赖缺失就当场炸掉，不要写兜底三元。** 我们有一个 bug 藏了很久：`normalizePath` 从 `model` 导入但 `model` 从未导出，防御性三元 `normalizePath ? ... : 兜底` 永远走兜底，功能实际从未生效，而测试全绿（[06](docs/06-testing.md)）。
3. **接线（wiring）写在不可测的层里 = 永远测不到。** 「提取过滤条件 → 传给检索」这行写在 `plugin/main.js`，因为 `require('obsidian')` 进不了测试，写漏了没人发现。**解法不是给不可测的层补测试，是把接线收进可测的层**（[06](docs/06-testing.md)）。
4. **测试全绿不等于没坏。** 用变异测试验证：故意注入真实回归，测试仍全绿 = 有盲区（[06](docs/06-testing.md)）。
5. **embedding 不总是更好的选择。** 最常用的 DeepSeek **根本没有 `/embeddings` 端点**。强依赖它等于让这批用户跑不起来（[07](docs/07-rag.md)）。

---

## 谁该读

- 在做「把外部内容搬进笔记软件」的开发者
- 被小红书 / B站 接口签名卡住的人
- 想给 Obsidian 插件写自动化测试的人
- 在纠结 RAG 用 BM25 还是向量检索的人

## License

MIT。文档可以自由使用，能帮到你就够了 —— 如果省了你几天时间，去给
[插件仓库](https://github.com/lin113311221/bili2obsidian-plugin) 点个 star 就行。
