# Note-Taking Workflow Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Scaffold the Obsidian note-taking vault with directory structure, templates, index files, and CLAUDE.md so the workflow is ready to use.

**Architecture:** Create all directories and files defined in the spec. Configure Obsidian's Templates plugin to point to `templates/`. Write CLAUDE.md as the single source of rules for Claude's behavior in this repo.

**Tech Stack:** Markdown, YAML frontmatter, Obsidian core Templates plugin, Git

---

## Chunk 1: Scaffold and Configuration

### Task 1: Create content directory structure

**Files:**
- Create: `content/study-notes/.gitkeep`
- Create: `content/knowledge-base/.gitkeep`
- Create: `content/cheatsheets/.gitkeep`
- Create: `content/projects/.gitkeep`

- [ ] **Step 1: Create the four content directories with .gitkeep files**

```bash
mkdir -p content/study-notes content/knowledge-base content/cheatsheets content/projects
touch content/study-notes/.gitkeep content/knowledge-base/.gitkeep content/cheatsheets/.gitkeep content/projects/.gitkeep
```

- [ ] **Step 2: Verify directory structure**

```bash
find content -type f
```

Expected:
```
content/cheatsheets/.gitkeep
content/knowledge-base/.gitkeep
content/projects/.gitkeep
content/study-notes/.gitkeep
```

- [ ] **Step 3: Commit**

```bash
git add content/
git commit -m "scaffold: create content directory structure"
```

---

### Task 2: Create templates

**Files:**
- Create: `templates/study-note.md`
- Create: `templates/concept-note.md`

- [ ] **Step 1: Create study-note template**

Write `templates/study-note.md`:

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

- [ ] **Step 2: Create concept-note template**

Write `templates/concept-note.md`:

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

- [ ] **Step 3: Commit**

```bash
git add templates/
git commit -m "scaffold: add study-note and concept-note templates"
```

---

### Task 3: Configure Obsidian Templates plugin

**Files:**
- Modify: `.obsidian/templates.json` (create if not exists)

- [ ] **Step 1: Set Templates plugin folder to `templates/`**

Write `.obsidian/templates.json`:

```json
{
  "folder": "templates"
}
```

- [ ] **Step 2: Verify Templates plugin is enabled**

```bash
grep '"templates"' .obsidian/core-plugins.json
```

Expected: `"templates": true`. If not present or false, update `.obsidian/core-plugins.json` to set `"templates": true`.

- [ ] **Step 3: Commit**

```bash
git add .obsidian/templates.json
git commit -m "config: point Obsidian Templates plugin to templates/"
```

---

### Task 4: Create index and plans files

**Files:**
- Create: `index/catalog.md`
- Create: `plans/learning-roadmap.md`

- [ ] **Step 1: Create catalog.md with initial skeleton**

Write `index/catalog.md`:

```markdown
# 笔记目录

## study-notes

## knowledge-base

## cheatsheets

## projects
```

- [ ] **Step 2: Create learning-roadmap.md with initial skeleton**

Write `plans/learning-roadmap.md`:

```markdown
# 学习路线

## 当前

## 计划

## 已完成
```

- [ ] **Step 3: Commit**

```bash
git add index/ plans/
git commit -m "scaffold: add catalog index and learning roadmap"
```

---

### Task 5: Write CLAUDE.md

**Files:**
- Create: `CLAUDE.md`

- [ ] **Step 1: Write CLAUDE.md with all workflow rules**

Write `CLAUDE.md` with the following content (use a heredoc or Write tool to avoid fenced code block conflicts):

```
# Note Vault 工作规则
```

The file should contain the following sections (exact content provided here as plain text):

**Section: 仓库结构**

    content/
    ├── study-notes/{topic}/     # 学习笔记，两层结构
    ├── knowledge-base/{topic}/  # 知识库，两层结构
    ├── cheatsheets/             # 速查表，扁平结构
    └── projects/{project}/      # 项目笔记，结构自定义

    辅助文件：
    - templates/ — Obsidian 模板
    - index/catalog.md — 笔记目录索引
    - plans/learning-roadmap.md — 学习计划

**Section: 命名规则**

    - 目录名：英文短横线（golang/、study-notes/）
    - 章节笔记文件名：英文短横线（goroutine-and-channel.md）
    - 概念笔记文件名：英文空格（Goroutine.md、Go Concurrency Model.md）

**Section: Frontmatter 规范**

    必填字段：title、date（YYYY-MM-DD）、tags、category、description、status（draft/in-progress/done）
    可选字段：source（来源链接）、aliases（概念笔记专用，支持中文搜索）
    category 取值：study-notes / knowledge-base / cheatsheets / projects

**Section: 写作风格**

    - 中文为主，技术术语保留英文
    - 语气简洁清晰
    - 章节笔记中通过 [[Concept Name]] wikilink 自然关联概念笔记
    - 概念笔记通过 aliases 支持中文搜索（如 aliases: [Go协程]）

**Section: 工作流程**

    创建笔记：
    1. 读本文件确认规则
    2. 读 index/catalog.md 确认笔记是否已存在
    3. 根据模板创建笔记到正确位置
    4. 更新 index/catalog.md 添加新条目

    审查优化笔记：
    1. 读取指定笔记
    2. 检查 frontmatter 是否符合规范
    3. 检查内容质量：术语准确性、结构清晰度、是否有可以补充 wikilink 的概念
    4. 给出修改建议或直接修改

    规划笔记结构：
    1. 读 plans/learning-roadmap.md 了解整体计划
    2. 创建 content/study-notes/{topic}/overview.md 作为主题入口（复用 study-note 模板）
    3. 在 overview 中列出建议的章节结构
    4. 更新 learning-roadmap.md 和 catalog.md

**Section: catalog.md 条目格式**

    每个条目格式为：- [[文件名]] - 简短描述

    示例：
    ## study-notes
    ### golang
    - [[overview]] - Go 语言学习总览
    - [[goroutine-and-channel]] - Goroutine 与 Channel

- [ ] **Step 2: Commit**

```bash
git add CLAUDE.md
git commit -m "scaffold: add CLAUDE.md with workflow rules"
```

---

### Task 6: Clean up and final commit

**Files:**
- Delete: `欢迎.md`（Obsidian 默认欢迎文件，不再需要）

- [ ] **Step 1: Remove default welcome file**

```bash
git rm 欢迎.md
```

- [ ] **Step 2: Verify final structure**

```bash
find . -not -path './.git/*' -not -path './.obsidian/*' -not -path './docs/*' -not -name '.git' -not -name '.obsidian' -not -name 'docs' | sort
```

Expected:
```
.
./CLAUDE.md
./content
./content/cheatsheets
./content/cheatsheets/.gitkeep
./content/knowledge-base
./content/knowledge-base/.gitkeep
./content/projects
./content/projects/.gitkeep
./content/study-notes
./content/study-notes/.gitkeep
./index
./index/catalog.md
./plans
./plans/learning-roadmap.md
./templates
./templates/concept-note.md
./templates/study-note.md
```

- [ ] **Step 3: Commit**

(`git rm` already staged the deletion, no additional `git add` needed.)

```bash
git commit -m "scaffold: remove default welcome file, finalize vault structure"
```

- [ ] **Step 4: Push to remote**

```bash
git push origin master
```
