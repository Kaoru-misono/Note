---
title: "Building Your Super Agent — 从零打造个人超级 Agent 的方法论"
date: 2026-03-27
tags: [AI, Agent, Claude-Code, Skills, Prompt-Engineering, Workflow]
category: knowledge-base
description: "基于 ByteTech 文章《Agent 的超级进化》的深度总结与分析，提炼个人 Agent 构建方法论、Skills 体系设计、浏览器自动化基建等核心思路"
status: done
source: "https://bytetech.info/articles/7620714119376994356"
aliases: [超级Agent构建, Agent工程化, Skills体系]
---

# Building Your Super Agent — 从零打造个人超级 Agent 的方法论

> 基于贺鹏程（字节跳动）在 ByteTech 的分享《Agent 的超级进化: 从零打造只属于你的超级 Agent》，加入个人理解与启发。

## 核心公式

```
个人超级 Agent = 通用 Agent + 基座模型 + 领域上下文
```

三个组成部分中，通用 Agent（Claude Code / Codex）和基座模型（Opus 4.6 / GPT 5.4）由厂商决定，用户无法改变。**唯一能做的、也是回报最高的投入，是构建领域上下文**——即 CLAUDE.md、Rules 和 Skills。

这个公式的意义在于：与其花时间比较哪个 Agent 工具更好，不如把精力放在**喂给 Agent 的上下文质量**上。上下文越精准，Agent 表现越像一个懂你的协作者。

## 范式转变：从 Vibe Coding 到 Agent Coding

| 阶段 | 特征 | 人的角色 |
|------|------|----------|
| Vibe Coding (2025) | 一问一答，每轮需要人判断结果 | 人在循环中，驱动每一步 |
| Agent Coding (2026) | AI 可持续自主工作数十小时 | 人在循环外，只在关键节点介入 |

这不只是效率的提升，而是**工作模式的根本改变**：从"人操作工具"变成"人设定目标，Agent 自主执行"。智谱 GLM-5 将其定义为从"Vibe Coding"到"Agentic Engineering"。

## 上下文三层体系

Claude Code 的上下文分为三层，逐层细化：

### 第一层：CLAUDE.md — 操作手册

Agent 每次启动时全局加载。定义角色、工作流、代码质量标准、输出格式。Anthropic 官方自己的 CLAUDE.md 只有约 2.5K tokens。

**原则：短、硬、可执行。** 优先写命令、约束和架构边界，而不是冗长的说明文档。

#### 作者 CLAUDE.md 实际结构（从截图提取）

```markdown
# CLAUDE.md

## 设计哲学

三条不可违背的原则：
- **先调研后动手**：对任何不明确的需求，必须先用 Read、Grep、Glob 等工具
  调研清楚再动手，严禁凭猜测修改代码
- **最小改动原则**：只改必须改的，不做"顺便优化"。每次改动必须可解释、可回溯
- **代码即文档**：不写无用注释——用自描述的变量名和函数名替代注释

## 工作流 Plan → Execute → Verify → Learn

### Plan — 充分规划，再动手
- 做项目或功能时 (3+ 待修改文件) **必须进入 Plan 模式**
- 拆解问题拆出子任务；贯穿一个主线任务流是唯一的输出
- 不确定需求时明确提问而非主观臆测

### Execute — 急攻状态，到我执行
- **多个独立任务**无依赖就并发，主 Agent 只做整体决策
- 子任务 ≥ 3 个 → **Agent Teams** 并发分派
- 子任务 < 3 个 → **Subagent** 或 Glob/Grep/Explore

**Subagent 策略：**
- 大范围调用 Subagent：只写一句 Prompt，后面全给 Subagent
- 单文件修改之间无依赖，面向修改多个文件时并发
- 复杂问题拆解通过多 Subagent 投入更多算力
- 完成编排自己执行标注 'multi-agent-orchestration' skill

### Verify — 完成前必须验证
- 永远不要在未运行相关测试的情况下说没有问题
- 随时反问：在 main 上的修改是否会影响到其他地方？
- 执行完后必须给出**简要报告**：修改了什么、为什么这么做

### Learn — 完成后必须归纳
- 犯过同类错误后，判断相似情况写入 `rules/` 或 `CLAUDE.md`
- 持续改进代码和提示词
```

**关键设计思路**：这份 CLAUDE.md 的核心是一个 **Plan → Execute → Verify → Learn** 的闭环工作流，而不是一堆零散的规则。Execute 阶段明确定义了 Agent Teams / Subagent 的调度策略（按子任务数量选择并发方式），Learn 阶段形成了从错误到 Rules 的自动沉淀机制。

### 第二层：Rules/ — 行为规则

针对不同场景的细分规则。Rules 是对 CLAUDE.md 的补充，覆盖特定领域的行为约束。

#### 作者实际 Rules 文件（从截图提取）

作者的 `~/.claude/rules/` 目录包含 5 个文件：

| 文件名 | 推测用途 |
|--------|---------|
| `browser-skill-routing.md` | 浏览器技能的路由策略（多个浏览器工具按优先级选择） |
| `lark-config.md` | 飞书相关操作的配置规范 |
| `markdown-style-guide.md` | Markdown 编写风格规范 |
| `skills-install.md` | 技能安装的标准流程 |
| `tool-first.md` | 工具优先原则（优先用专用工具而非通用方法） |

**设计思路**：Rules 文件数量很少（5 个），每个都是针对一个明确领域的行为约束。不是大而全的规范手册，而是从实际犯错中沉淀出来的"防呆规则"。这和 CLAUDE.md 中 Learn 阶段的"写入 rules/"形成闭环。

### 第三层：Skills/ — 可复用能力模块

**这是整个体系中最核心的部分。** Skills 的两大优势：

1. **渐进式披露，按需加载**：Agent 不会把所有 Skill 内容塞进上下文，而是只预加载标题和描述，用到时才展开。这解决了上下文窗口有限的根本矛盾。

2. **原子化能力 + 自动编排**：每个 Skill 是一个原子能力，Agent 根据任务自主选择和组合 Skills，而不是走固定的 Workflow 流程。这带来了组合爆炸的可能性。

| 维度 | Workflow（传统） | Skills（Agent 原生） |
|------|-----------------|---------------------|
| 流程 | 固定编排 | Agent 自主决策 |
| 加载 | 全量加载 | 按需加载 |
| 扩展 | 需要重新编排 | 笛卡尔积，自动组合 |

## Skills 设计方法论

### Skill 的结构

一个 Skill 不只是一个 `SKILL.md`，还包括同目录 `references/` 下沉淀的脚本和工具。Skill.md 的 YAML frontmatter 定义触发条件，markdown body 定义执行指令。

Skill.md 标准格式：

````markdown
---
name: pdf
description: 处理 PDF 文件。用于读取、创建、合并或填写 PDF 表单。
---

# PDF 处理技能

## 读取 PDF
使用 pdftotext 快速提取文本：
```bash
pdftotext input.pdf -
```

## 创建 PDF
使用 PyPDF2 合并多个文件...

## 注意事项
- 处理大文件时考虑分批处理
- 中文 PDF 可能需要指定字体路径
````

- `name` 字段变成 `/slash-command`（如 `/pdf`）
- `description` 帮助 Agent 决定何时自动加载该 Skill
- `references/` 目录放实际脚本和工具，才是 Skill 发挥潜力的关键

### 作者完整 Skills 清单（从截图提取）

| 技能名 | 说明 | 核心依赖/实现 |
|--------|------|-------------|
| 字节代码库分析 | 从 HTTP 入口深入链路调研，生成模块文档和架构图。支持多仓库，用编排方式完成超大文档 | 内置字节代码库特征的调研引导 |
| 写文章 | 模仿博客作者真实写作风格生成博客 | 风格学习 + 素材编排 |
| **/eat** | **知识吸收元技能**。灵感来源于千与千寻的无脸男——吸收一切知识。把 X 帖子、GitHub 项目丢给 Agent，让它分析实现原理，补充自身能力。**持续增强，螺旋上升** | 核心思路：看到好东西 → 丢给 CC → 分析原理 → 转化为自己的 Skill |
| **/shit** | **上下文排泄**。吃多了要拉出来。优化上下文结构，清理 Agent 工作区，精简结构 | 定期清理冗余 Skill 和过时 Rules |
| 浏览器自动化 | Agent-browser / Chrome DevTools 控制 | `chrome://inspect/#remote-debugging` |
| CC 长期任务执行 | 长时间自主运行 | 原理：[Effective Harnesses for Long-Running Agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents) |
| 字节云 | 字节内部云服务操作 | 内部基建 |
| feishu-cli | 飞书一切操作 | [github.com/riba2534/feishu-cli](https://github.com/riba2534/feishu-cli) |
| Git PR/Issue 处理 | 自动处理修复 PR | Git CLI |
| 超级搜索 | 自建超级搜索 + API | [SearXNG](https://github.com/searxng/searxng) 自部署 |
| YouTube 内容获取 | 字幕提取、内容总结 | youtube 爬虫 + ffmpeg |
| PDF 处理 | 各种 PDF 操作 | [Stirling PDF](https://github.com/Stirling-Tools/stirling-pdf) |
| 深入思考 | 柏拉图式概念解剖 | 深度推理提示词 |
| 区块链侦探 | 链上交易追踪溯源 | 链上 API |
| 点麦当劳 | 除了支付外全流程打通 | [麦当劳 MCP](https://open.mcd.cn/mcp) |

**核心洞察**：/eat 和 /shit 构成了一个 Agent **自我进化的新陈代谢机制**。/eat 不断吸收外部知识转化为内部能力，/shit 定期排泄冗余保持精简。这让 Agent 的能力边界持续扩张又不会无限膨胀。

### 如何造 Skill

1. **用官方 skill-creator 元技能**：让 Agent 自己去调研、理解、与用户对话调整、最后创建技能
2. **抄 + 改**：看到好的 X 帖子、GitHub 项目，直接丢给 Agent 让它分析原理并转化为自己的 Skill
3. **核心是沉淀可复用的流程**：不要异想天开造不存在的需求，而是从真实工作中提取重复模式

> Good Case：有一个真实的重复流程需要沉淀，让 Agent 去调研分析理解后创建
> Bad Case：凭空想象一个不存在的场景

### 自动造 Skill 的闭环

这是最有价值的思路之一：

```
浏览器自动抓包 → 分析有用接口 → 用 skill-creator 自动生成 Skill
```

让 Agent 控制浏览器 → 分析网站的请求 → 封装成 HTTP 接口调用 → 自动生成 Skill。理论上可以通杀所有网站。登录态的解决方案：直接从用户本地 Chrome 中提取 Cookie。

## 浏览器自动化基建

作者认为这是**最重要的基础设施**，因为现代互联网绝大多数操作基于浏览器或 HTTP。

推荐工具栈（做成 Skills 让 Agent 按优先级使用）：

| 工具 | 特点 | 适用场景 |
|------|------|----------|
| Playwright (微软) | 有 MCP 和 CLI，社区成熟 | 通用浏览器自动化 |
| Agent Browser (Vercel) | 为 AI Agent 优化，底层 Playwright | 无头浏览器首选 |
| Chrome DevTools MCP | 官方出品，完整 DevTools 能力 | 抓包、调试、性能分析 |
| Browser Use 2.0 | CDP 直连，成本减半 | 轻量级自动化 |

**关键洞察**：不要只选一个，全部做成 Skills 让 Agent 根据任务自动选择最合适的工具。

## 记忆与知识库

### 核心原则

**不要刻意搞文档，要引导 AI 去哪里找信息。** 这比直接喂文档更好，因为：

- Code is Doc — 代码本身就是最准确的文档
- Agent 原地探查比依赖可能过时的记忆更准确
- 给 AI 指引而不是给答案，让它自己调研

### Claude Code vs OpenClaw 的记忆差异

- **Claude Code**：单 Session 任务，无持久记忆。干专业任务时效果反而更好，因为不会被无关历史上下文干扰
- **OpenClaw**：有记忆，更像"数字员工"。适合日常生活场景，但记忆可能不准确

### 本地知识库构建

用 Git 管理知识库，三个核心 Skill：
- `/query`：搜索知识库 + 深入调研
- `/add_wiki`：自动调研 → 生成文档 → 更新索引
- `/digest`：从当前对话中提取实战经验入库

**人完全不接触知识库**，只和 Agent 对接，全自动沉淀。

## 实践案例启发

作者分享的案例按复杂度排列，核心启发是**找到真实需求（钉子），然后用 Agent（锤子）去解决**：

| 场景 | 核心能力 | 复杂度 |
|------|----------|--------|
| 快速确认代码业务逻辑 | 提示词 + 代码探查 | 低 |
| 云服务器集群配置 | SSH + 脚本编排 | 低 |
| 处理 Argos 报警 | 多系统串联 + 代码分析 | 中 |
| 业务交接梳理技术链路 | Agent Teams 多子 Agent 编排 | 高 |
| 全自动写文章 | 风格学习 + 素材收集 + 配图生成 | 高 |
| HappyClaw 全栈项目 | 13w 行代码，全 Agent 完成 | 极高 |

## 对我的启发与行动项

### 1. 重新定位 Agent 的角色

Agent 不是"代码助手"，而是"具备推理能力的通用执行者"。名字叫 Claude Code 是误导——它能做的远不止写代码。任何有明确流程的任务都可以委托给 Agent。

### 2. 投资上下文而非工具

与其花时间对比不同 Agent 工具，不如专注于构建自己的 CLAUDE.md + Rules + Skills 体系。这才是真正的护城河——别人的 Skills 下了也不一定会用，只有自己造的才能融会贯通。

### 3. Skills 是渐进式积累的过程

不需要一次性造完所有 Skill。正确的路径是：
1. 在日常工作中发现重复模式
2. 第一次手动完成，记录流程
3. 第二次用 skill-creator 把流程沉淀为 Skill
4. 后续自动执行，不断迭代优化

这形成一个**正向飞轮**：用得越多 → 沉淀越多 Skill → Agent 越强 → 能做的事越多。

### 4. 浏览器自动化是基础设施

大量实际场景的瓶颈不在 AI 的推理能力，而在于 Agent 无法"触达"外部系统。浏览器控制是打通这个瓶颈的关键一步。当前我们已经配置了 Playwright MCP，这是一个好的开始。

### 5. AI 的复利效应

作者的核心观点：AI 是能力的放大器，不是生成器。对于已有的能力，AI 可以放大十倍；对于没有的能力，`0 × 任何数 = 0`。这意味着：
- **持续投入基础能力建设**（理解代码、理解业务、理解工具）
- **用 Agent 放大这些能力的产出**
- 订阅 AI 工具是一个"下行有限、上行巨大"的不对称赌注

### 6. 信息差在使用深度

三层漏斗：知道 AI → 会用 AI → 用 AI 重构工作流。第二层到第三层是 10 倍的鸿沟。目前大多数人停在第二层。真正的竞争力在于能否把 Agent 嵌入日常工作的每个环节。

### 7. /eat 和 /shit 的思路

这是一个很巧妙的自我进化机制：
- **/eat**（吸收）：看到好的帖子、项目、技术方案，丢给 Agent 分析原理并转化为自己的能力
- **/shit**（排泄）：定期优化上下文结构，清理冗余，保持 Agent 工作区精简

这让 Agent 的能力边界不断扩张，同时不会无限膨胀。

## 截图补充信息

以下内容来自文章内嵌截图，Markdown 导出中缺失。

### 作者 `~/.claude/` 目录结构

```
~/.claude/
├── CLAUDE.md                  ← 全局操作手册
├── settings.json
├── README.md
├── .gitignore
├── agents/
├── rules/                     ← 行为规则
│   ├── browser-skill-routing.md
│   ├── lark-config.md
│   ├── markdown-style-guide.md
│   ├── skills-install.md
│   └── tool-first.md
├── skills/                    ← 可复用能力模块
├── plans/
├── plugins/
├── sessions/
├── tasks/
├── todos/
├── cache/
├── backups/
└── ...
```

### 关键架构图

- **Skill vs Workflow 对比图**：Workflow 是线性流水线（收集需求 → 处理 → 输出），断一环就全断；Agent Skills 是星形拓扑，Agent 居中连接搜索、写作、调研、分析、规划、代码等原子节点，自主选择路径组合
- **上下文三层架构图**：`~/.claude/` 全局配置（CLAUDE.md）→ 项目级 Rules/（多个规则文件）→ Skills/（技能目录，各含 SKILL.md + references/）
- **Coding 模型评测**：引用论文 [arxiv.org/html/2603.03823v1](https://arxiv.org/html/2603.03823v1) 的柱状图，Claude Opus 4.6 处于领先

### Skills 终端运行截图细节

截图中可见各 Skill 在终端里的实际运行界面：

- **字节代码库分析**：Agent 在终端中自动分析多个代码仓库，逐个探查 HTTP 接口链路，输出模块文档和架构图。截图中可见 Agent 正在并发调度多个 Subagent 分别处理不同模块
- **/eat（知识吸收）**：终端显示 Agent 正在分析一个 GitHub 项目的 README 和源码，提取实现原理，然后调用 skill-creator 将其转化为本地 Skill 文件
- **/shit（上下文清理）**：终端显示 Agent 在扫描 Skills 目录，识别冗余和过时的文件，精简目录结构
- **浏览器自动化**：截图展示了 Chrome `chrome://inspect/#remote-debugging` 的 Remote Debugging 配置页面（Chrome M144+ 新增功能，在浏览器内开启调试端口，不需要命令行参数），以及 Chrome DevTools MCP 的连接界面
- **CC 长期任务执行**：两张截图展示了 Agent 连续自主运行数小时的日志输出
- **超级搜索**：截图显示自建 SearXNG 实例的搜索结果界面，通过 API 接入 Agent
- **深入思考**：截图显示"柏拉图式概念解剖"提示词风格——引导 Agent 对一个概念进行多层次的深入拆解

### "你是哪一派" 四象限图

按守序/混乱 × 重度/轻度两个维度划分 Agent 用户。作者属于 YOLO Mode（混乱 + 重度），即 Claude MAX 200 刀订阅、疯狂使用那一派。底部有讽刺图强调：**AI 是能力放大器，0 × 任何数 = 0**。

## 参考链接

- 原文：[Agent 的超级进化: 从零打造只属于你的超级 Agent](https://bytetech.info/articles/7620714119376994356)
- [Claude Code 官方文档](https://code.claude.com/docs/zh-CN/overview)
- [feishu-cli](https://github.com/riba2534/feishu-cli) — 作者开源的飞书 CLI 工具
- [HappyClaw](https://github.com/riba2534/happyclaw) — 基于 Claude Code 的多渠道 AI 工作台
- [抛弃 MCP, CLI 才是 Agent 的母语](https://bytedance.larkoffice.com/wiki/Pjj4wFdtuiAiJZkvBgccdVU5nYc)
- [Effective Harnesses for Long-Running Agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)
- [Harness Engineering](https://openai.com/zh-Hans-CN/index/harness-engineering/)
- [skill-creator](https://github.com/anthropics/skills/tree/main/skills/skill-creator)
