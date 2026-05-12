# ai-doc-styles

本仓库用于维护**对外发表的中文技术文章**（例如发布在掘金等技术社区的博文、教程稿）的写作体例，并配套 **Cursor Rules**，方便本地编辑 Markdown 时由人或 AI 遵循同一套规范。

## 用途说明

- **体例事实源**：[`.cursor/writing-guide/chinese-tech-writing.md`](.cursor/writing-guide/chinese-tech-writing.md)（条文、示例、给 Agent 的编排说明与 AI 操作注意事项均在此维护）。
- **Cursor 入口**：[`.cursor/rules/chinese-tech-writing.mdc`](.cursor/rules/chinese-tech-writing.mdc)（在编辑 `*.md` 时提示必读上述事实源；正文不重复列举，避免与事实源漂移）。
- 条文整合自开源仓库 [ruanyf/document-style-guide](https://github.com/ruanyf/document-style-guide)（公共领域），本仓库在「对外技术文章」场景下做了适用范围与本地化说明。

## 使用方式

1. **克隆或拉取本仓库**（获取已发布的最新规范）：
   `git clone https://github.com/JUST-Limbo/ai-doc-styles.git`
   或在已有克隆中执行：`git pull origin main`
2. **阅读体例全文**：打开 `.cursor/writing-guide/chinese-tech-writing.md`。
3. **在 Cursor 中使用**：将本仓库作为工作区打开（或把 `.cursor/rules` 与 `.cursor/writing-guide` 按你方策略引入目标项目），编辑文章类 Markdown 时 Agent 会按 Rule 提示先 Read 事实源。

本规范**针对对外技术文章**，不替代业务需求、接口文档、内部运维说明等工程文档的写法约定；若在其他类型 Markdown 上也需要套用，应由作者在任务中明确要求。
