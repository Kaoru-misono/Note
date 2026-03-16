# Note Vault 工作规则

## 仓库结构

    content/
    ├── study-notes/{topic}/     # 学习笔记，两层结构
    ├── knowledge-base/{topic}/  # 知识库，两层结构
    ├── cheatsheets/             # 速查表，扁平结构
    └── projects/{project}/      # 项目笔记，结构自定义

辅助文件：
- `templates/` — Obsidian 模板
- `index/catalog.md` — 笔记目录索引
- `plans/learning-roadmap.md` — 学习计划

## 命名规则

- **目录名**：英文短横线（`golang/`、`study-notes/`）
- **章节笔记文件名**：英文短横线（`goroutine-and-channel.md`）
- **概念笔记文件名**：英文空格（`Goroutine.md`、`Go Concurrency Model.md`）

## Frontmatter 规范

必填字段：`title`、`date`（YYYY-MM-DD）、`tags`、`category`、`description`、`status`（draft/in-progress/done）

可选字段：`source`（来源链接）、`aliases`（概念笔记专用，支持中文搜索）

category 取值：`study-notes` / `knowledge-base` / `cheatsheets` / `projects`

## 写作风格

- 中文为主，技术术语保留英文
- 语气简洁清晰
- 章节笔记中通过 `[[Concept Name]]` wikilink 自然关联概念笔记
- 概念笔记通过 `aliases` 支持中文搜索（如 `aliases: [Go协程]`）

## 工作流程

### 创建笔记

1. 读本文件确认规则
2. 读 `index/catalog.md` 确认笔记是否已存在
3. 根据模板创建笔记到正确位置
4. 更新 `index/catalog.md` 添加新条目

### 审查优化笔记

1. 读取指定笔记
2. 检查 frontmatter 是否符合规范
3. 检查内容质量：术语准确性、结构清晰度、是否有可以补充 wikilink 的概念
4. 给出修改建议或直接修改

### 规划笔记结构

1. 读 `plans/learning-roadmap.md` 了解整体计划
2. 创建 `content/study-notes/{topic}/overview.md` 作为主题入口（复用 study-note 模板）
3. 在 overview 中列出建议的章节结构
4. 更新 `learning-roadmap.md` 和 `catalog.md`

## catalog.md 条目格式

每个条目格式为：`- [[文件名]] - 简短描述`

示例：

```
## study-notes

### golang
- [[overview]] - Go 语言学习总览
- [[goroutine-and-channel]] - Goroutine 与 Channel
```
