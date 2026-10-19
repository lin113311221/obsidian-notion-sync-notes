# Obsidian / Notion 同步工程笔记

整理自 Savault 开发过程的历史工程笔记，记录收藏同步、适配层、笔记落盘、检索和测试的设计取舍。本仓是文档资料，不是可安装插件、SDK 或运行时源码。

产品与安装入口见 [Savault 插件发布仓](https://github.com/lin113311221/savault-plugin) 和 [Savault Skill](https://github.com/lin113311221/savault-skills)。协作接手见 [agentconversation.md](agentconversation.md)。

## 阅读顺序

| 文档 | 关注点 |
|---|---|
| [01-architecture.md](docs/01-architecture.md) | 多平台、多目标的共享引擎与适配器 |
| [02-xiaohongshu.md](docs/02-xiaohongshu.md) | 小红书集成的会话、页面响应与变化边界 |
| [03-bilibili.md](docs/03-bilibili.md) | B站收藏接口、字幕与登录态 |
| [04-obsidian.md](docs/04-obsidian.md) | Markdown、frontmatter、双链与图谱 |
| [05-notion.md](docs/05-notion.md) | Notion 标识、结构及字段限制 |
| [06-testing.md](docs/06-testing.md) | 可测试边界、真实适配器接线与回归 |
| [07-rag.md](docs/07-rag.md) | 本地笔记检索与 BM25 / 向量检索的取舍 |

先读架构和测试，再按任务读平台或目标章节。文中的平台数量、API 限制、模型能力和测试数字对应写作时的实现；实际采用前要对照当前代码与官方资料核验，不作为今天的产品承诺。

## 如何使用与贡献

- 不需要安装依赖。本仓没有 package.json、构建或 npm 测试入口。
- 优先复用设计方法：共享引擎与适配器分离、把关键接线放入可测试层、针对真实回归检查测试盲区。
- 不把一种会话或页面集成方法描述成永久不受平台变化影响；接口、权限、结构和风控都可能变化。
- 检索方案应按用户已有服务、数据规模与实际检索结果选择；旧服务商能力描述需要重新确认。
- 修改笔记时补充适用版本、来源、边界及可复现示例；示例使用合成数据，不含凭据或私人笔记。
- 提交前运行 `git diff --check`，检查相对链接、代码块与引用；默认分支 `master` 的变更通过 PR 审核。

## 许可

[MIT](LICENSE) 覆盖本仓文档。它不改变 Savault 私有实现、专有运行包或第三方服务的授权条件。
