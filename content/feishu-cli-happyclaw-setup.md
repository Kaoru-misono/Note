# feishu-cli + HappyClaw 环境配置记录

> 日期：2026-03-26

## 一、feishu-cli 构建

1. **安装 Go** — `winget install GoLang.Go`（安装了 Go 1.26.1）
2. **安装 Make** — `winget install GnuWin32.Make`（GNU Make 3.81）
3. **构建** — `go build -o bin/feishu-cli.exe .`

## 二、飞书应用配置

1. 在[飞书开放平台](https://open.feishu.cn/app)创建**企业自建应用**
2. 获取 App ID 和 App Secret
3. 写入 `~/.bashrc` 环境变量：
   - `FEISHU_APP_ID`
   - `FEISHU_APP_SECRET`
   - `FEISHU_OWNER_EMAIL`（用于自动授权）
   - `FEISHU_TRANSFER_OWNERSHIP=true`（文档所有权自动转移）
4. 验证 — `feishu-cli doc create --title "Hello Feishu"` 创建成功
5. 给自己添加文档权限 — `feishu-cli perm add` 授予 `full_access`

## 三、HappyClaw 部署

1. **Clone** — `git clone https://github.com/riba2534/happyclaw.git` 到 `E:\github\happyclaw`
2. **安装依赖** — 主项目 + agent-runner + web 三个 `npm install`
3. **编译** — 主项目 + agent-runner + web 三个 build
4. **修复 Windows 兼容性 bug** — `container/agent-runner/src/index.ts` 中 `new URL(import.meta.url).pathname` 改为 `fileURLToPath(import.meta.url)`，解决路径重复问题（`E:\E:\...`）
5. **配置飞书** — Web 向导中填入 App ID 和 App Secret
6. **配置 Claude** — 使用 Claude Max 订阅

### 启动命令

```bash
# 带飞书环境变量启动
export FEISHU_APP_ID="cli_xxx"
export FEISHU_APP_SECRET="xxx"
export FEISHU_OWNER_EMAIL="user@example.com"
export FEISHU_TRANSFER_OWNERSHIP=true
cd /e/github/happyclaw && nohup node dist/index.js > happyclaw.log 2>&1 &
```

Web 界面：http://localhost:3000

## 四、飞书机器人配置

1. 应用中开启**机器人能力**
2. 事件配置选择**长连接（WebSocket）**模式
3. 订阅 **`im.message.receive_v1`** 事件（关键，否则收不到消息）
4. 添加权限：`im:message`、`im:message:send_as_bot` 等
5. **发布应用**，等管理员审批

## 五、feishu-cli 接入 HappyClaw

1. feishu-cli 复制到系统 PATH — `C:\Users\Admin\bin\feishu-cli.exe`
2. HappyClaw 启动时带飞书环境变量（Agent 进程会继承 `process.env`）
3. feishu-cli 的 13 个技能复制到 `~/.claude/skills/`，通过 HappyClaw 同步给 Agent

## 架构

```
飞书对话 → HappyClaw（WebSocket 长连接）→ Claude Agent SDK → Bash → feishu-cli → 飞书 API
```

| 项目 | 角色 |
|------|------|
| HappyClaw | 大脑 + 嘴巴 — AI Agent 运行时 + 飞书聊天入口 |
| feishu-cli | 手 — 操作飞书的 CLI 工具 |

## 踩过的坑

| 问题 | 原因 | 解决 |
|------|------|------|
| 创建的文档自己打不开 | Bot 身份创建，用户无权限 | `perm add` 授权 + 配置 `FEISHU_OWNER_EMAIL` |
| Agent 路径报错 `E:\E:\...` | Windows 下 `URL.pathname` 多了盘符前缀 | 改用 `fileURLToPath()` |
| 飞书机器人收不到消息 | 没订阅 `im.message.receive_v1` 事件 | 开放平台添加事件订阅 |
| Agent 找不到 feishu-cli | 启动时没加载 `.bashrc` 的 PATH | 复制到系统默认 PATH 目录 |
| 重启 HappyClaw 时 Claude 被杀 | 误杀了 node 进程 | 先确认 PID 归属再 kill |
