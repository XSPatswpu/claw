# Open Claw (Claude Code) 入门使用文档

## 目录

- [1. 什么是 Open Claw](#1-什么是-open-claw)
- [2. 安装](#2-安装)
- [3. 认证与登录](#3-认证与登录)
- [4. 快速开始](#4-快速开始)
- [5. 基础命令与斜杠命令](#5-基础命令与斜杠命令)
- [6. 权限模式](#6-权限模式)
- [7. 项目配置 (CLAUDE.md)](#7-项目配置-claudemd)
- [8. 键盘快捷键](#8-键盘快捷键)
- [9. IDE 集成](#9-ide-集成)
- [10. MCP 服务器基础](#10-mcp-服务器基础)
- [11. 常用工作流](#11-常用工作流)
- [12. 参考资源](#12-参考资源)

---

## 1. 什么是 Open Claw

Open Claw（即 Claude Code）是 Anthropic 推出的 **AI 驱动的智能编程助手**。它以 Agent（智能体）形式运行在你的终端、IDE 或桌面应用中，能够：

- **理解整个代码库**：自动读取项目文件、理解架构和上下文
- **编辑文件**：根据你的指令直接创建、修改代码
- **执行命令**：运行 shell 命令、测试、构建等
- **Git 操作**：创建 commit、管理分支、发起 Pull Request
- **连接外部工具**：通过 MCP 协议接入 GitHub、Slack、数据库等
- **并行协作**：生成多个子 Agent 同时处理不同任务

简单来说，你描述想要什么，Open Claw 负责规划并执行。

---

## 2. 安装

### 2.1 原生安装（推荐，支持自动更新）

**macOS / Linux / WSL：**

```bash
curl -fsSL https://claude.ai/install.sh | bash
```

**Windows PowerShell：**

```powershell
irm https://claude.ai/install.ps1 | iex
```

**Windows CMD：**

```cmd
curl -fsSL https://claude.ai/install.cmd -o install.cmd && install.cmd && del install.cmd
```

> 注意：Windows 需要先安装 [Git for Windows](https://git-scm.com/download/win)。

### 2.2 Homebrew（macOS，需手动更新）

```bash
brew install --cask claude-code
# 更新：brew upgrade claude-code
```

### 2.3 WinGet（Windows，需手动更新）

```powershell
winget install Anthropic.ClaudeCode
# 更新：winget upgrade Anthropic.ClaudeCode
```

### 2.4 npm（仍可用，但已不推荐）

```bash
npm install -g @anthropic-ai/claude-code
```

### 2.5 系统要求

| 项目 | 要求 |
|------|------|
| 操作系统 | macOS 13.0+、Windows 10 1809+、Ubuntu 20.04+、Debian 10+、Alpine 3.19+ |
| 内存 | 4GB+ |
| 网络 | 需要联网 |
| Shell | Bash、Zsh、PowerShell 或 CMD |

---

## 3. 认证与登录

### 3.1 支持的账户类型

- **Claude Pro / Max 订阅**（推荐个人用户）
- **Claude for Teams / Enterprise**（团队用户）
- **Claude Console**（API 预付费，开发者）
- **Amazon Bedrock / Google Vertex AI / Microsoft Foundry**（云平台用户）

### 3.2 首次登录

```bash
cd /path/to/your/project
claude
```

首次启动会自动打开浏览器进行登录。如果浏览器未自动打开，按 `c` 复制登录链接到剪贴板，手动在浏览器中打开。

### 3.3 凭证存储

| 平台 | 存储位置 |
|------|----------|
| macOS | 加密 Keychain |
| Linux / Windows | `~/.claude/.credentials.json`（Linux 权限 0600） |

### 3.4 账户切换与登出

```bash
/login     # 切换账户
/logout    # 登出
```

---

## 4. 快速开始

### 4.1 启动方式

```bash
# 交互模式（最常用）
claude

# 一次性任务
claude "给这个函数添加单元测试"

# 非交互查询（输出后退出）
claude -p "这个项目的技术栈是什么？"

# 继续上次对话
claude -c

# 恢复历史对话
claude -r
```

### 4.2 新项目推荐的第一步

进入你的项目目录后，可以先问 Claude 这些问题来建立上下文：

```
这个项目是做什么的？
项目使用了哪些技术？
项目的入口文件在哪里？
解释一下目录结构
```

### 4.3 初始化项目配置

```bash
/init
```

Claude 会分析你的代码库，自动创建 `CLAUDE.md` 文件，包含构建命令、代码规范等项目信息。

### 4.4 基本工作流

```
1. cd /your/project       # 进入项目目录
2. claude                  # 启动 Open Claw
3. /init                   # （可选）初始化项目配置
4. 描述你想做的事情         # 开始工作
5. 审查 Claude 的改动       # Claude 会请求你批准关键操作
6. Shift+Tab              # 需要时切换权限模式
```

---

## 5. 基础命令与斜杠命令

### 5.1 CLI 启动命令

| 命令 | 说明 |
|------|------|
| `claude` | 启动交互模式 |
| `claude "任务描述"` | 执行一次性任务 |
| `claude -p "查询"` | 非交互查询后退出 |
| `claude -c` | 继续最近一次对话 |
| `claude -r` | 选择并恢复历史对话 |
| `claude commit` | 创建 Git commit |

### 5.2 交互模式中的斜杠命令

在交互模式中输入 `/` 可看到所有可用命令：

| 命令 | 说明 |
|------|------|
| `/help` | 显示帮助信息 |
| `/clear` 或 `/reset` | 清除对话历史，释放上下文 |
| `/compact` | 压缩对话，可附带聚焦主题 |
| `/config` | 打开设置界面 |
| `/cost` | 显示 token 使用统计 |
| `/memory` | 编辑 CLAUDE.md、查看自动记忆 |
| `/permissions` | 查看或更新权限 |
| `/model [模型名]` | 切换 AI 模型 |
| `/plan` | 进入规划模式 |
| `/diff` | 查看交互式差异 |
| `/rewind` | 回退对话/代码到之前的状态 |
| `/init` | 初始化项目生成 CLAUDE.md |
| `/status` | 显示版本、模型、账户、连接状态 |
| `/vim` | 切换 Vim 编辑模式 |
| `/tasks` | 查看和管理后台任务 |
| `/mcp` | 管理 MCP 服务器连接 |
| `/effort` | 设置模型推理强度（low/medium/high/max） |
| `/theme` | 更换颜色主题 |
| `/doctor` | 诊断安装状态 |
| `/feedback` | 提交反馈 |

### 5.3 特殊输入前缀

| 前缀 | 功能 |
|------|------|
| `/` | 斜杠命令和 Skill |
| `!` | Bash 模式，直接运行 shell 命令 |
| `@` | 文件路径提及（触发自动补全） |
| `?` | 显示当前环境下的可用快捷键 |

---

## 6. 权限模式

Open Claw 提供 5 种权限模式，控制 Claude 在执行操作前是否需要你的批准。按 `Shift+Tab` 可在模式间切换。

### 6.1 模式对比

| 模式 | 行为 | 适用场景 |
|------|------|----------|
| **default** | 读取文件无需询问；编辑文件、shell 命令、网络请求需要批准 | 日常工作、敏感项目 |
| **acceptEdits** | 读取和编辑文件无需询问；shell 命令和网络请求需要批准 | 信任 Claude 的代码改动时 |
| **plan** | 只读模式，Claude 只分析和规划，不做任何修改 | 代码探索、方案设计 |
| **auto** | Claude 自主执行，后台分类器审查每个操作（仅 Team 计划可用） | 长时间运行的任务 |
| **bypassPermissions** | 跳过所有权限检查（危险！仅用于隔离环境） | 隔离容器/虚拟机 |

### 6.2 设置权限模式

```bash
# 启动时指定
claude --permission-mode plan

# 非交互模式指定
claude -p "重构认证模块" --permission-mode acceptEdits

# 交互模式中切换
Shift+Tab

# 设为默认（在 settings.json 中）
# "permissions": { "defaultMode": "acceptEdits" }
```

### 6.3 工具级别的权限控制

你可以在 `settings.json` 中对具体工具设置细粒度的允许/拒绝规则：

```json
{
  "permissions": {
    "allow": [
      "Read",
      "Grep",
      "Glob",
      "Bash(npm run test)",
      "Bash(npm run build)"
    ],
    "deny": [
      "Bash(rm -rf *)",
      "Bash(git push --force *)"
    ]
  }
}
```

**规则优先级：** deny > ask > allow

---

## 7. 项目配置 (CLAUDE.md)

`CLAUDE.md` 是 Open Claw 的核心配置文件，用 Markdown 格式编写，告诉 Claude 关于你项目的重要信息。

### 7.1 创建 CLAUDE.md

方法一：自动生成

```bash
/init
```

方法二：手动创建，在项目根目录新建 `CLAUDE.md`：

```markdown
# 项目配置

## 构建与测试
- 构建：`npm run build`
- 测试：`npm test`
- 开发服务器：`npm run dev`

## 代码规范
- 使用 2 空格缩进
- 函数需要写 JSDoc 注释
- 优先使用 async/await

## 项目架构
- API 处理器在 `src/api/handlers/`
- React 组件在 `src/components/`
- 样式使用 Tailwind CSS

## 注意事项
- 提交前运行 `npm test`
- 添加依赖后运行 `npm audit`
```

### 7.2 CLAUDE.md 的作用域层级

| 位置 | 作用域 | 用途 |
|------|--------|------|
| `./CLAUDE.md` 或 `./.claude/CLAUDE.md` | 项目级 | 团队共享的项目指令 |
| `~/.claude/CLAUDE.md` | 用户级 | 个人偏好，适用于所有项目 |
| `/etc/claude-code/CLAUDE.md` (Linux) | 组织级 | 公司统一规范 |
| `./.claude/rules/*.md` | 项目级（模块化） | 按路径/模块组织的规则 |

**加载顺序：** 组织级 -> 从当前目录向上遍历 -> 用户级。子目录中的 CLAUDE.md 在 Claude 访问该目录时懒加载。

### 7.3 路径特定规则

在 `.claude/rules/` 中创建带路径过滤的规则文件：

```yaml
---
paths:
  - "src/api/**/*.ts"
---
# API 开发规则
- 所有 API 端点必须有输入验证
- 返回标准化的错误格式
- 必须编写集成测试
```

### 7.4 自动记忆

Open Claw 会自动学习并记忆项目信息：

- 存储位置：`~/.claude/projects/<项目>/memory/`
- `MEMORY.md` 的前 200 行每次会话都会加载
- 包含构建命令、调试经验、发现的模式等
- 可用 `/memory` 查看和编辑

---

## 8. 键盘快捷键

### 8.1 通用控制

| 快捷键 | 功能 |
|--------|------|
| `Ctrl+C` | 取消当前输入或生成 |
| `Ctrl+D` | 退出 Open Claw |
| `Ctrl+L` | 清屏 |
| `Ctrl+R` | 反向搜索命令历史 |
| `Ctrl+T` | 切换任务列表 |
| `Ctrl+V`（Windows: `Alt+V`） | 粘贴剪贴板图片 |
| `Shift+Tab` | 切换权限模式 |
| `Esc` + `Esc` | 回退或总结 |
| `Alt+P` | 切换模型 |
| `Alt+T` | 切换扩展思考 |

### 8.2 多行输入

| 方式 | 说明 |
|------|------|
| `Shift+Enter` | 换行但不发送 |
| `Option+Enter`（macOS） | 换行但不发送 |
| `\ + Enter` | 所有终端通用的换行方式 |

### 8.3 自定义快捷键

```bash
/keybindings
# 打开 ~/.claude/keybindings.json 进行编辑
```

示例配置：

```json
{
  "$schema": "https://www.schemastore.org/claude-code-keybindings.json",
  "bindings": [
    {
      "context": "Chat",
      "bindings": {
        "ctrl+e": "chat:externalEditor",
        "ctrl+u": null
      }
    }
  ]
}
```

---

## 9. IDE 集成

### 9.1 VS Code

**安装：**

在 VS Code 中按 `Cmd+Shift+X`（macOS）或 `Ctrl+Shift+X`（Windows/Linux），搜索 "Claude Code"，点击安装。

**使用：**

1. 点击编辑器右上角的 Spark 图标，或 Activity Bar 中的 Spark 图标
2. 输入提示词
3. 查看内联 diff 差异
4. 接受、拒绝或修改

**VS Code 快捷键：**

| 快捷键 | 功能 |
|--------|------|
| `Cmd+Esc` / `Ctrl+Esc` | 在编辑器和 Claude 间切换焦点 |
| `Cmd+Shift+Esc` / `Ctrl+Shift+Esc` | 在新标签页中打开 |
| `Cmd+N` / `Ctrl+N` | 新对话 |
| `Option+K` / `Alt+K` | 插入 @-mention 及行号 |

**特色功能：**
- 内联 diff 并排比较
- `@` 提及文件及特定行范围
- 对话历史下拉菜单
- 多对话标签页

### 9.2 JetBrains IDE

支持 IntelliJ IDEA、PyCharm、Android Studio、WebStorm、PhpStorm、GoLand 等。

**安装：**

在 JetBrains Marketplace 中搜索 "Claude Code" 安装，然后完全重启 IDE。

**使用：**

```bash
# 在 IDE 内置终端中：
claude
# 然后使用 /ide 连接到 IDE
```

**JetBrains 快捷键：**

| 快捷键 | 功能 |
|--------|------|
| `Cmd+Esc` / `Ctrl+Esc` | 打开 Claude Code |
| `Cmd+Option+K` / `Alt+Ctrl+K` | 插入文件引用 |

---

## 10. MCP 服务器基础

MCP（Model Context Protocol）是连接 AI 工具与外部数据源的开放标准，让 Claude 能访问 GitHub、Slack、数据库等外部服务。

### 10.1 添加 MCP 服务器

```bash
# 远程 HTTP 服务器（最简单）
claude mcp add --transport http <名称> <URL>

# 远程 SSE 服务器
claude mcp add --transport sse <名称> <URL>

# 本地 stdio 服务器
claude mcp add --transport stdio <名称> <命令>
```

### 10.2 管理 MCP 服务器

```bash
/mcp
# 打开交互式 MCP 管理界面
```

### 10.3 配置文件方式

在 `.claude/settings.json` 中添加：

```json
{
  "mcpServers": {
    "github": {
      "transport": "http",
      "url": "https://api.github.com/mcp/"
    },
    "postgres": {
      "transport": "stdio",
      "command": "node",
      "args": ["./postgres-mcp.js"],
      "env": {
        "DATABASE_URL": "postgresql://..."
      }
    }
  }
}
```

### 10.4 MCP 配置作用域

| 作用域 | 文件位置 | 说明 |
|--------|----------|------|
| 本地 | `~/.claude/.mcp.json` | 仅本机，所有项目 |
| 项目 | `./.mcp.json` | 与团队共享（提交到 git） |
| 用户 | `~/.claude/settings.json` | 个人配置 |

### 10.5 常用 MCP 服务器

| 服务 | 用途 |
|------|------|
| GitHub | 代码审查、PR 管理 |
| Slack | 消息检索、发送更新 |
| Google Drive | 读写文档 |
| Jira | 工单管理 |
| PostgreSQL | 数据库查询 |
| Sentry | 错误监控 |

添加 MCP 后，直接用自然语言使用即可：

```
审查 PR #456
查询生产数据库
检查 Slack 上的消息
```

---

## 11. 常用工作流

### 11.1 探索新代码库

```
这个项目是做什么的？
解释一下目录结构
主要的入口文件在哪里？
画一个架构图
```

### 11.2 修复 Bug

```
运行测试看看有没有失败的
修复 src/auth/login.ts 中的登录超时问题
```

### 11.3 添加新功能

```
在用户模块中添加密码重置功能
参考现有的注册流程来实现
```

### 11.4 代码审查

```
审查最近的提交
检查这个 PR 有没有安全问题
```

### 11.5 Git 操作

```
claude commit                    # 快速创建 commit
创建一个 PR 描述我最近的修改
```

### 11.6 重构

```
/plan 重构认证模块，使用策略模式
```

先用 `/plan` 模式规划，审查后再切换到 `acceptEdits` 模式执行。

---

## 12. 参考资源

### 官方文档

- 概览：https://code.claude.com/docs/en/overview
- 快速开始：https://code.claude.com/docs/en/quickstart
- 安装配置：https://code.claude.com/docs/en/setup
- 权限模式：https://code.claude.com/docs/en/permission-modes
- CLAUDE.md：https://code.claude.com/docs/en/memory
- 快捷键：https://code.claude.com/docs/en/keybindings
- VS Code 集成：https://code.claude.com/docs/en/vs-code
- JetBrains 集成：https://code.claude.com/docs/en/jetbrains
- MCP：https://code.claude.com/docs/en/mcp
- 命令参考：https://code.claude.com/docs/en/commands
- 最佳实践：https://code.claude.com/docs/en/best-practices

### 获取帮助

- 交互模式中输入 `/help`
- 输入 `/doctor` 诊断安装问题
- 输入 `/feedback` 提交问题反馈
- GitHub Issues：https://github.com/anthropics/claude-code/issues

### 关键文件位置速查

| 文件 | 路径 |
|------|------|
| 全局设置 | `~/.claude/settings.json` |
| 全局快捷键 | `~/.claude/keybindings.json` |
| 项目设置 | `.claude/settings.json` |
| 本地设置（不提交） | `.claude/settings.local.json` |
| 项目指令 | `./CLAUDE.md` 或 `.claude/CLAUDE.md` |
| 模块化规则 | `.claude/rules/*.md` |
| MCP 配置 | `.mcp.json` |
| 自动记忆 | `~/.claude/projects/<项目>/memory/` |
| 凭证 | `~/.claude/.credentials.json` |
