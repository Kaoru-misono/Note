---
name: new-section
description: 为学习笔记新建章节小节的文件骨架、下载图片、更新 catalog
user-invocable: true
---

# 新建章节小节

用户提供原书 URL 和章节编号（如 `2.3`），为任意书籍的学习笔记创建标准文件结构。

## 输入

- 章节编号（如 `2.3`）
- 原书在线页面 URL（用于下载图片和推断标题）
- 所属 topic 目录（如 `pbrt4`、`real-time-rendering` 等）

## 步骤

1. **读取规范**：读 `CLAUDE.md` 和 `content/study-notes/CLAUDE.md` 确认命名规则和格式要求

2. **确认文件名**：根据章节号和 URL 中的标题确定文件名，格式 `C{章}.{节}-{英文短横线描述}`
   - 笔记：`C2.3-sampling-inversion-method.md`
   - 翻译：`C2.3-sampling-inversion-method-translation.md`（放在 `translations/` 子目录）

3. **下载图片**（如有 URL）：
   - 从原书 URL 的 HTML 中提取所有 `<img>` 标签的 `src` 属性
   - 排除 logo/favicon 等非内容图片
   - 下载到 `chapter{N}/assets/` 目录
   - 用 `ls -la` 确认下载成功

4. **创建翻译文件骨架**：在 `translations/` 下创建 `-translation.md`，包含：
   - 完整 frontmatter（title、date、tags、category、description、status: in-progress、source）
   - 一级标题
   - 图片占位（用 HTML 居中格式 + `^fig-` 锚点预留）

5. **创建笔记文件骨架**：在 `chapter{N}/` 下创建笔记文件，包含：
   - 完整 frontmatter（status: draft）
   - 一级标题
   - `> 原文翻译：[[C{章}.{节}-xxx-translation]]` 链接
   - `> 前置知识：` 链接到前序章节
   - 各子节标题占位

6. **更新 catalog.md**：在对应 chapter 分组下添加新条目

7. **报告**：告知用户创建了哪些文件，下载了几张图片，等待用户提供翻译内容

## 注意事项

- 不输出笔记正文内容（等用户提供）
- 翻译文件也只是骨架（等用户粘贴翻译）
- 文件名中的描述部分从 URL 路径推断，用英文短横线
- 如果 `assets/` 或 `translations/` 目录不存在则创建
- 如果没有提供 URL，跳过图片下载步骤
