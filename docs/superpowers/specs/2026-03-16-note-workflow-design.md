# Note-Taking Workflow Design Spec

## 概述

为 Obsidian 笔记仓库设计一套结构化的笔记工作流，支持学习笔记和个人知识库两种场景，由 Claude 辅助创建、审查和规划笔记。

## 目录结构

```
Note/
├── content/
│   ├── study-notes/
│   │   └── {topic}/
│   │       ├── overview.md
│   │       └── chapter-name.md
│   ├── knowledge-base/
│   │   └── {topic}/
│   │       └── Concept Name.md
│   ├── cheatsheets/
│   │   └── topic-name.md
│   └── projects/
│       └── {project-name}/
│           └── ...
├── templates/
│   ├── study-note.md
│   └── concept-note.md
├── index/
│   └── catalog.md
├── plans/
│   └── learning-roadmap.md
├── CLAUDE.md
└── .obsidian/
```

## 命名规则

- **目录名**：英文短横线（如 `study-notes/`、`golang/`）
- **章节笔记文件名**：英文短横线（如 `goroutine-and-channel.md`）
- **概念笔记文件名**：英文空格（如 `Goroutine.md`、`Go Concurrency Model.md`）

## Frontmatter 规范

### 章节笔记（study-note）

```yaml
---
title: "Goroutine 与 Channel"
date: 2026-03-16
tags:
  - golang
  - concurrency
category: study-notes
description: "Go 并发编程基础：goroutine 的创建与 channel 的使用"
status: draft
source: "https://go.dev/doc/effective_go"
---
```

### 概念笔记（concept-note）

```yaml
---
title: "Goroutine"
date: 2026-03-16
tags:
  - golang
  - concurrency
category: knowledge-base
description: "Go 语言中的轻量级线程"
status: done
aliases:
  - Go协程
---
```

### 字段说明

| 字段 | 必填 | 说明 |
|------|------|------|
| title | 是 | 笔记标题 |
| date | 是 | 创建日期，格式 YYYY-MM-DD |
| tags | 是 | 标签列表，扁平结构 |
| category | 是 | 笔记分类：study-notes / knowledge-base / cheatsheets / projects |
| description | 是 | 简短描述 |
| status | 是 | 状态：draft / in-progress / done |
| source | 否 | 来源链接 |
| aliases | 否 | 概念笔记专用，便于中文搜索和 wikilink |

## 模板

### `templates/study-note.md`

```markdown
---
title: "{{title}}"
date: {{date:YYYY-MM-DD}}
tags: []
category: study-notes
description: ""
status: draft
source: ""
---

# {{title}}
```

### `templates/concept-note.md`

```markdown
---
title: "{{title}}"
date: {{date:YYYY-MM-DD}}
tags: []
category: knowledge-base
description: ""
status: draft
aliases: []
---

# {{title}}
```

## 根目录辅助文件

### `CLAUDE.md`

Claude 的行为规则与写作风格指南：

- **仓库结构说明**：目录约定、命名规则、frontmatter 规范
- **写作风格**：中文为主，技术术语保留英文；语气简洁清晰
- **笔记创建规则**：章节笔记放哪、概念笔记放哪、文件怎么命名
- **维护规则**：创建笔记后更新 `catalog.md`，按需更新 `learning-roadmap.md`

### `index/catalog.md`

全局笔记目录索引，按类型和主题分组，使用 wikilink 链接到各笔记。Claude 每次创建笔记后负责更新。

### `plans/learning-roadmap.md`

学习计划与路线，记录当前学习内容、计划学习内容和已完成内容。Claude 参考此文件来建议笔记结构。

## Claude 工作流程

### 1. 创建笔记

1. 读 `CLAUDE.md` 确认规则
2. 读 `index/catalog.md` 确认是否已存在
3. 根据模板创建笔记到正确位置
4. 更新 `index/catalog.md` 添加新条目

### 2. 审查优化笔记

1. 读取指定笔记
2. 检查 frontmatter 是否符合规范
3. 检查内容质量：术语准确性、结构清晰度、是否有可以补充 wikilink 的概念
4. 给出修改建议或直接修改

### 3. 规划笔记结构

1. 读 `plans/learning-roadmap.md` 了解整体计划
2. 创建 `content/study-notes/{topic}/overview.md` 作为主题入口
3. 在 overview 中列出建议的章节结构
4. 更新 `learning-roadmap.md` 和 `catalog.md`

## 语言与风格

- 中文为主，技术术语保留英文
- 章节笔记中通过 `[[Concept Name]]` wikilink 自然关联概念笔记
- 概念笔记通过 `aliases` 支持中文搜索
