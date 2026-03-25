# Open Claw (Claude Code) 工作原理文档

## 目录

- [1. 总体架构](#1-总体架构)
- [2. Agentic Loop（智能体循环）](#2-agentic-loop智能体循环)
- [3. 工具系统 (Tool System)](#3-工具系统-tool-system)
- [4. 上下文管理 (Context Management)](#4-上下文管理-context-management)
- [5. 权限系统 (Permission System)](#5-权限系统-permission-system)
- [6. CLAUDE.md 系统](#6-claudemd-系统)
- [7. Hook 系统](#7-hook-系统)
- [8. MCP 集成 (Model Context Protocol)](#8-mcp-集成-model-context-protocol)
- [9. Agent / Subagent 系统](#9-agent--subagent-系统)
- [10. 记忆系统 (Memory System)](#10-记忆系统-memory-system)
- [11. 设置系统 (Settings System)](#11-设置系统-settings-system)
- [12. Git 集成](#12-git-集成)
- [13. SDK 与自定义 Agent](#13-sdk-与自定义-agent)
- [14. 架构总览图](#14-架构总览图)

---

## 1. 总体架构

Open Claw 的核心是一个 **"Agentic Harness"（智能体外壳）**，它将 Claude 语言模型包装在一个可执行环境中。整个架构由以下核心组件构成：

```
┌─────────────────────────────────────────────────────────────┐
│                   Claude 语言模型                             │
│                (Sonnet / Opus / Haiku)                       │
└──────────────────────┬──────────────────────────────────────┘
                       │
              ┌────────▼────────┐
              │  Agentic Loop   │
              │ 收集上下文→执行→验证│
              └───┬─────────┬───┘
                  │         │
      ┌───────────▼──┐  ┌──▼──────────────┐
      │   工具系统    │  │   上下文管理     │
      │ File/Bash/   │  │ 会话状态/        │
      │ Search/Web/  │  │ CLAUDE.md/       │
      │ Agent        │  │ 记忆/压缩        │
      └──────┬───────┘  └────────┬─────────┘
             │                   │
      ┌──────▼───────┐   ┌──────▼─────────┐
      │  权限系统     │   │  执行环境       │
      │ Deny/Ask/    │   │ 本地/云端/      │
      │ Allow/Hook   │   │ 远程控制        │
      └──────────────┘   └──────┬──────────┘
                                │
                  ┌─────────────┼─────────────┐
                  │             │             │
           ┌──────▼───┐  ┌────▼────┐  ┌────▼─────┐
           │ MCP 服务器│  │ 子Agent │  │ Hook/   │
           │ stdio/   │  │ Explore/│  │ Skill/  │
           │ HTTP/SSE │  │ Plan/   │  │ Plugin  │
           └──────────┘  │ Custom  │  └──────────┘
                         └─────────┘
```

**各层职责：**

| 层级 | 职责 |
|------|------|
| **语言模型** | 推理、决策、生成代码和计划 |
| **Agentic Loop** | 协调"收集→执行→验证"循环 |
| **工具系统** | 提供文件操作、命令执行、搜索等能力 |
| **上下文管理** | 管理会话状态、对话历史、自动压缩 |
| **权限系统** | 细粒度的访问控制和安全防护 |
| **执行环境** | 本地机器、云端 VM、远程控制等运行环境 |
| **扩展层** | MCP 服务器、子 Agent、Hook、Skill、Plugin |

---

## 2. Agentic Loop（智能体循环）

Agentic Loop 是 Open Claw 的核心运行机制，每次接收到用户请求后，Claude 进入一个三阶段循环，直到任务完成：

### 2.1 阶段一：收集上下文 (Gather Context)

Claude 首先理解当前项目状态：

```
用户请求
   │
   ▼
┌─────────────────────┐
│ 1. 读取相关文件      │ ← Read / Glob / Grep 工具
│ 2. 执行探索命令      │ ← Bash 工具（如 git status, ls）
│ 3. 加载 CLAUDE.md   │ ← 项目配置和规则
│ 4. 加载自动记忆      │ ← MEMORY.md
│ 5. 使用 Explore Agent│ ← 深度代码探索
└─────────┬───────────┘
          │
          ▼
     已理解上下文
```

### 2.2 阶段二：执行操作 (Take Action)

基于收集到的上下文，Claude 选择并执行操作：

```
     已理解上下文
          │
          ▼
┌─────────────────────┐
│ 1. 选择合适的工具    │ ← 模型推理
│ 2. 权限检查         │ ← 权限系统评估
│ 3. PreToolUse Hook  │ ← 可拦截或修改
│ 4. 执行工具         │ ← 实际操作
│ 5. PostToolUse Hook │ ← 后处理
│ 6. 收集工具输出      │ ← 结果反馈
└─────────┬───────────┘
          │
          ▼
     操作已执行
```

### 2.3 阶段三：验证结果 (Verify Results)

Claude 评估操作结果，决定下一步：

```
     操作已执行
          │
          ▼
┌──────────────────────┐
│ 1. 分析工具输出       │
│ 2. 运行测试/检查      │
│ 3. 对比预期结果       │
│ 4. 判断是否完成       │
└─────┬──────────┬─────┘
      │          │
   完成？      未完成
      │          │
      ▼          ▼
  返回结果    回到阶段一
```

### 2.4 循环中断

用户可以在任何时刻按 `Ctrl+C` 中断循环，提供新的上下文或修改方向。

---

## 3. 工具系统 (Tool System)

工具是 Open Claw 的"手和脚"，让 Claude 能够在真实环境中执行操作。

### 3.1 工具分类

| 类别 | 工具 | 说明 |
|------|------|------|
| **文件操作** | `Read` | 读取文件内容（支持代码、图片、PDF、Jupyter Notebook） |
| | `Write` | 创建新文件或完全重写文件 |
| | `Edit` | 编辑现有文件（只发送 diff） |
| | `NotebookEdit` | 编辑 Jupyter Notebook 单元格 |
| **搜索** | `Glob` | 按文件名模式匹配（如 `**/*.ts`） |
| | `Grep` | 按内容搜索文件（底层使用 ripgrep） |
| **执行** | `Bash` | 执行 shell 命令，工作目录在命令间保持 |
| **网络** | `WebFetch` | 获取网页内容 |
| | `WebSearch` | 搜索互联网 |
| **编排** | `Agent` | 启动子 Agent 处理复杂任务 |
| | `Skill` | 调用 Skill（技能） |
| | `TaskCreate/List/Update` | 创建和管理任务列表 |
| **用户交互** | `AskUserQuestion` | 向用户提问 |
| **调度** | `CronCreate/Delete/List` | 创建定时任务 |
| **模式切换** | `EnterPlanMode/ExitPlanMode` | 进入/退出规划模式 |
| **工作树** | `EnterWorktree/ExitWorktree` | 进入/退出 git 工作树隔离环境 |

### 3.2 工具执行流程

每次工具调用都经历以下完整流程：

```
Claude 生成工具调用请求
         │
         ▼
┌──────────────────┐
│  1. 权限检查      │ ← deny > ask > allow 优先级
│     ↓ 通过       │
│  2. PreToolUse   │ ← Hook 可以拦截（exit 2）或允许（exit 0）
│     Hook 执行    │
│     ↓ 通过       │
│  3. 工具实际执行  │ ← 在对应环境中运行
│     ↓            │
│  4. PostToolUse  │ ← Hook 后处理
│     Hook 执行    │
│     ↓            │
│  5. 结果返回     │ ← 输出注入到 Claude 的上下文中
└──────────────────┘
```

### 3.3 各工具详解

#### Read 工具

```
功能：读取文件内容
输入：file_path（绝对路径）、offset（起始行）、limit（行数）、pages（PDF 页范围）
特点：
  - 使用 cat -n 格式输出（带行号）
  - 支持图片（PNG/JPG 等，多模态解析）
  - 支持 PDF（大文件需指定页范围，每次最多 20 页）
  - 支持 Jupyter Notebook（返回所有单元格及输出）
  - 默认读取前 2000 行
```

#### Write 工具

```
功能：创建新文件或完全重写现有文件
输入：file_path（绝对路径）、content（文件内容）
安全约束：
  - 写入现有文件前必须先用 Read 读取
  - 覆盖前会确认
```

#### Edit 工具

```
功能：编辑现有文件（增量修改）
输入：file_path、old_string（要替换的文本）、new_string（替换后的文本）
特点：
  - 只发送差异部分，效率高
  - 适合小范围修改
  - 比 Write 更精确，更安全
```

#### Bash 工具

```
功能：执行 shell 命令
输入：command（命令字符串）、timeout（超时，最大 600000ms）、run_in_background（后台运行）
特点：
  - 工作目录在命令间保持
  - 环境变量不跨命令保持
  - 每个命令在独立进程中运行
  - 支持后台运行（run_in_background: true）
  - 支持沙箱模式限制文件系统和网络访问
```

#### Glob 工具

```
功能：按文件名模式快速搜索
输入：pattern（glob 模式，如 "**/*.ts"）、path（搜索目录）
特点：
  - 支持任意大小的代码库
  - 返回按修改时间排序的文件路径列表
```

#### Grep 工具

```
功能：按内容搜索文件
输入：pattern（搜索模式）、path（搜索目录）、include（文件过滤）
底层：使用 ripgrep 引擎，速度极快
```

#### Agent 工具

```
功能：启动子 Agent 处理复杂任务
输入：prompt（任务描述）、subagent_type（Agent 类型）、isolation（隔离模式）
类型：
  - Explore：快速代码库搜索（使用 Haiku 模型）
  - Plan：方案设计和架构规划
  - general-purpose：通用多步骤任务
  - 自定义 Agent
隔离：
  - 默认：共享文件系统
  - worktree：独立 git 工作树
```

#### WebFetch 工具

```
功能：获取网页内容
输入：url（目标 URL）
限制：受权限系统中的域名白名单控制
```

#### WebSearch 工具

```
功能：搜索互联网
输入：query（搜索查询）
用途：查找最新文档、解决方案、API 参考等
```

---

## 4. 上下文管理 (Context Management)

### 4.1 上下文窗口结构

Claude 的上下文窗口是有限的（目前最大 1M token），Open Claw 按以下层级组织内容：

```
上下文窗口 (Context Window)
├── 系统提示词 (System Prompt)        ← 固定，定义 Claude 的行为
├── CLAUDE.md 文件内容                ← 会话开始时完整加载
├── 自动记忆 MEMORY.md               ← 加载前 200 行
├── .claude/rules/ 规则文件           ← 按条件加载
├── 活跃 MCP 服务器的工具定义         ← 占用上下文空间
├── 对话历史                         ← 最新的优先保留
├── 工具调用结果                      ← 保留到压缩时清理
└── Skill 内容                       ← 按需加载
```

### 4.2 上下文压缩策略

当上下文使用量达到约 95% 时，Open Claw 自动触发压缩：

```
压缩流程：
1. 优先清理最旧的工具输出（测试结果、命令输出等）
2. 如果仍然超限，对对话历史进行摘要
3. 保留用户请求和关键代码片段
4. 重新注入 CLAUDE.md（压缩后重新加载）
5. 重新注入自动记忆 MEMORY.md 前 200 行
6. 从 CLAUDE_ENV_FILE 恢复环境变量
```

**手动触发压缩：**

```bash
/compact                      # 基本压缩
/compact 聚焦在认证模块       # 带聚焦主题的压缩
```

### 4.3 管理上下文的最佳实践

| 策略 | 说明 |
|------|------|
| 使用 `/compact` | 主动释放空间 |
| 使用子 Agent | 隔离大量输出（如测试结果）到独立上下文 |
| 拆分 CLAUDE.md | 将大文件拆分到 `.claude/rules/` 按路径加载 |
| Skill 按需加载 | 将大量参考内容放入 Skill，仅在需要时加载 |
| `/clear` 重置 | 任务完成后清理上下文开始新任务 |

---

## 5. 权限系统 (Permission System)

### 5.1 分层评估架构

权限系统采用三级评估，按优先级从高到低：

```
┌──────────────────────────────────┐
│  Deny 规则（最高优先级）           │  ← 无条件拒绝
├──────────────────────────────────┤
│  Ask 规则                        │  ← 弹出提示让用户决定
├──────────────────────────────────┤
│  Allow 规则（最低优先级）          │  ← 自动批准
└──────────────────────────────────┘
```

### 5.2 配置作用域层级

```
优先级从高到低：
1. Managed Settings（组织管理员设置，不可覆盖）
2. CLI 参数（如 --allowedTools，会话级）
3. Local Settings (.claude/settings.local.json，本机专用)
4. Project Settings (.claude/settings.json，团队共享)
5. User Settings (~/.claude/settings.json，个人默认)
```

### 5.3 工具级权限规则语法

**Bash 工具：** 通配符模式，支持词边界

```json
{
  "allow": [
    "Bash(npm run test)",       // 精确匹配
    "Bash(npm run *)",          // * 前有空格 → 词边界匹配
    "Bash(npm*)"                // * 前无空格 → npm 后接任意字符
  ],
  "deny": [
    "Bash(rm -rf *)",
    "Bash(git push --force *)",
    "Bash(git * main)"          // 拒绝所有针对 main 分支的 git 操作
  ]
}
```

**Read/Edit/Write 工具：** Gitignore 风格的路径模式

```json
{
  "allow": [
    "Read",                      // 允许读取所有文件
    "Edit(src/**/*.ts)",         // 允许编辑 src 下的 TS 文件
    "Write(~/docs/**)"           // 允许写入 home 下的 docs 目录
  ],
  "deny": [
    "Read(.env*)",               // 禁止读取环境变量文件
    "Edit(//etc/**)"             // 禁止编辑 /etc 下的系统文件
  ]
}
```

**WebFetch 工具：** 域名规则

```json
{
  "allow": ["WebFetch(domain:docs.example.com)"],
  "deny": ["WebFetch(domain:internal.corp.com)"]
}
```

**MCP 工具：** 服务器和工具名匹配

```json
{
  "allow": [
    "mcp__github__search_repositories",  // 允许特定工具
    "mcp__slack__*"                       // 允许 Slack 服务器的所有工具
  ]
}
```

### 5.4 权限模式详解

| 模式 | 读文件 | 写文件 | Shell 命令 | 网络请求 | 安全机制 |
|------|--------|--------|-----------|---------|---------|
| `default` | 自动 | 询问 | 询问 | 询问 | 标准 |
| `acceptEdits` | 自动 | 自动 | 询问 | 询问 | 信任文件编辑 |
| `plan` | 自动 | 禁止 | 只读命令 | 禁止 | 完全只读 |
| `auto` | 自动 | 自动 | 自动 | 自动 | 后台分类器审查 |
| `bypassPermissions` | 自动 | 自动 | 自动 | 自动 | 无（仅 .git/.claude 保护） |

### 5.5 Hook 参与权限评估

PreToolUse Hook 可以参与权限决策：

```
Hook 退出码：
  exit 0  → 允许（仍受 deny 规则约束）
  exit 2  → 拦截（优先于 allow 规则）

结构化输出（JSON）：
  { "permissionDecision": "allow" }  → 允许
  { "permissionDecision": "deny" }   → 拒绝
  { "permissionDecision": "ask" }    → 交给用户决定
```

---

## 6. CLAUDE.md 系统

### 6.1 加载顺序

```
会话启动时的加载流程：

1. 组织级 CLAUDE.md（不可排除）
   ├── macOS: /Library/Application Support/ClaudeCode/CLAUDE.md
   ├── Linux/WSL: /etc/claude-code/CLAUDE.md
   └── Windows: C:\Program Files\ClaudeCode\CLAUDE.md

2. 从当前目录向上遍历
   ├── ./CLAUDE.md 或 ./.claude/CLAUDE.md    ← 完整加载
   ├── ../CLAUDE.md                           ← 完整加载
   └── 继续向上直到项目根目录

3. 嵌套 CLAUDE.md（懒加载）
   └── 当 Claude 读取子目录中的文件时才加载

4. 用户级 CLAUDE.md
   └── ~/.claude/CLAUDE.md

5. 项目规则
   └── .claude/rules/*.md
       ├── 无路径限制 → 无条件加载
       └── 有 paths 字段 → 匹配时才加载
```

### 6.2 CLAUDE.md 中的特殊语法

**文件导入：**

```markdown
@path/to/other-file.md
```

引用其他文件的内容，适合将大型配置拆分管理。

**路径条件规则（.claude/rules/ 中）：**

```yaml
---
paths:
  - "src/api/**/*.ts"
  - "src/**/*.{ts,tsx}"
---
# 此规则仅在 Claude 处理匹配路径的文件时加载
```

### 6.3 CLAUDE.md 在压缩中的行为

- CLAUDE.md 内容在上下文压缩后会**重新加载**
- 这确保了项目指令始终可用
- 因此 CLAUDE.md 应保持简洁（建议 < 200 行/文件）

---

## 7. Hook 系统

Hook 是 Open Claw 的生命周期自动化机制，允许你在特定事件发生时执行自定义逻辑。

### 7.1 Hook 事件全览

| 事件类别 | 事件名 | 触发时机 |
|---------|--------|---------|
| **会话事件** | `SessionStart` | 启动、恢复、清除、压缩时 |
| | `InstructionsLoaded` | CLAUDE.md/rules 加载时 |
| | `SessionEnd` | 会话结束时 |
| **工具事件** | `PreToolUse` | 工具执行前（可拦截/修改） |
| | `PostToolUse` | 工具执行后（可拦截并反馈） |
| | `PostToolUseFailure` | 工具执行失败后（仅通知） |
| **权限事件** | `PermissionRequest` | 权限请求时（可自动决策） |
| | `Notification` | 权限提示、空闲提示等 |
| **Agent 事件** | `SubagentStart` | 子 Agent 启动时 |
| | `SubagentStop` | 子 Agent 停止时 |
| | `Stop` | Agent 完成一轮时 |
| **上下文事件** | `PreCompact` | 上下文压缩前 |
| | `PostCompact` | 上下文压缩后 |
| **协作事件** | `TeammateIdle` | Agent 团队中有成员空闲 |
| | `TaskCompleted` | 任务完成时 |
| **基础设施** | `WorktreeCreate` | 工作树创建时 |
| | `WorktreeRemove` | 工作树移除时 |

### 7.2 Hook 输入输出格式

**输入（通过 stdin 传入 JSON）：**

```json
{
  "session_id": "abc123",
  "transcript_path": "/path/to/transcript.jsonl",
  "cwd": "/current/dir",
  "hook_event_name": "PreToolUse",
  "tool_name": "Bash",
  "tool_input": { "command": "npm test" },
  "permission_mode": "default"
}
```

**输出（退出码含义）：**

| 退出码 | 含义 |
|--------|------|
| `0` | 成功（可通过 stdout 返回 JSON 进行结构化控制） |
| `2` | 拦截操作（stderr 的内容作为反馈传给 Claude） |
| 其他 | 非阻塞错误（仅在 verbose 模式下显示） |

### 7.3 Hook 类型

| 类型 | 说明 |
|------|------|
| `command` | 执行 shell 脚本（stdio 通信） |
| `http` | 调用远程 HTTP 端点（POST JSON） |
| `prompt` | 单轮 LLM 评估（默认使用 Haiku） |
| `agent` | 多轮子 Agent（60 秒超时） |

### 7.4 配置示例

在 `.claude/settings.json` 中配置：

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "./scripts/validate-command.sh"
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "prettier --write $FILE_PATH"
          }
        ]
      }
    ],
    "SessionStart": [
      {
        "matcher": "startup",
        "hooks": [
          {
            "type": "command",
            "command": "echo '欢迎回来！' >&2"
          }
        ]
      }
    ]
  }
}
```

### 7.5 Hook 配置位置

| 位置 | 作用域 |
|------|--------|
| `~/.claude/settings.json` | 用户级（所有项目） |
| `.claude/settings.json` | 项目级（团队共享） |
| `.claude/settings.local.json` | 本机专用（不提交到 git） |
| Skill/Agent 的 frontmatter | 仅在该 Skill/Agent 激活时 |
| Plugin 的 `hooks/hooks.json` | 仅在该 Plugin 启用时 |

---

## 8. MCP 集成 (Model Context Protocol)

### 8.1 MCP 架构

MCP 是一个开放标准协议，让 AI 工具能够连接外部数据源和服务。

```
Open Claw
├── MCP 连接管理器
│   ├── Stdio 服务器（本地进程）
│   ├── HTTP 服务器（远程端点）
│   └── SSE 服务器（已废弃，流式传输）
└── 工具适配器
    └── 将 MCP 工具转换为 Open Claw 工具
        格式：mcp__<服务器名>__<工具名>
```

### 8.2 服务器生命周期

```
1. 配置      → 存储在 .mcp.json / settings.json / managed-mcp.json
2. 启动      → 会话开始时连接（或 Plugin 启用时）
3. 工具发现  → 通过 list_tools JSON-RPC 调用发现可用工具
4. 动态更新  → list_changed 通知触发工具刷新
5. 关闭      → 会话结束时断开连接
```

### 8.3 传输类型

| 类型 | 说明 | 适用场景 |
|------|------|----------|
| **stdio** | 本地进程通信 | 本地脚本、npx 命令、Python 脚本 |
| **HTTP** | JSON-RPC over HTTP | 远程服务端点 |
| **SSE** | Server-Sent Events（已废弃） | 旧版流式服务 |

### 8.4 工具命名规则

MCP 工具在 Open Claw 中以统一格式出现：

```
mcp__<服务器名>__<工具名>

示例：
  mcp__github__search_repositories
  mcp__slack__send_message
  mcp__postgres__query
```

权限匹配支持通配符：`mcp__github__*`

### 8.5 上下文开销

每个 MCP 服务器的工具定义都会加载到每次请求中，多个服务器可能在工作开始前就消耗大量上下文空间。使用 `/mcp` 命令查看每个服务器的上下文开销。

---

## 9. Agent / Subagent 系统

### 9.1 子 Agent 架构

```
主会话 (Main Session)
├── 子 Agent 实例 1 (Explore)
│   ├── 独立上下文窗口
│   ├── 受限工具集（只读）
│   └── 使用 Haiku 模型（快速）
│
├── 子 Agent 实例 2 (自定义 reviewer)
│   ├── 自定义系统提示
│   ├── 受限工具集
│   └── 可选持久化记忆
│
└── 工作树子 Agent (isolation: worktree)
    ├── 独立 git 工作树
    └── 隔离的文件系统状态
```

### 9.2 内置子 Agent 类型

| 子 Agent | 模型 | 工具权限 | 用途 |
|---------|------|---------|------|
| **Explore** | Haiku | 只读（Read/Glob/Grep 等） | 快速代码库搜索和分析 |
| **Plan** | 继承主会话 | 只读 | 方案设计和架构规划 |
| **general-purpose** | 继承主会话 | 全部工具 | 复杂多步骤任务 |

### 9.3 自定义子 Agent

在 `.claude/agents/` 目录下创建 Agent 定义文件：

```yaml
# .claude/agents/code-reviewer.md
---
name: code-reviewer
description: 审查代码质量问题
tools: Read, Grep, Glob, Bash
model: sonnet
memory: project
permissionMode: acceptEdits
maxTurns: 20
isolation: default
---
你是一个代码审查专家。分析代码并提出改进建议。

## 审查重点
1. 代码可读性
2. 潜在 Bug
3. 性能问题
4. 安全漏洞
```

### 9.4 子 Agent 生命周期

```
1. 调用     → Claude 自动委派或用户手动触发
2. 启动     → 子 Agent 以独立上下文窗口启动
3. 隔离     → 输出不会膨胀主会话的上下文
4. 执行     → 使用配置的工具集独立工作
5. 摘要     → 将结果摘要返回主会话
6. 清理     → 30 天后自动清理 Transcript
```

### 9.5 上下文隔离的好处

| 优势 | 说明 |
|------|------|
| 输出隔离 | 测试结果等大量输出留在子 Agent 内 |
| 探索分离 | 代码搜索与实现逻辑分离 |
| 独立容量 | 每个子 Agent 获得全新的上下文空间 |
| 主会话专注 | 主对话保持简洁，只包含摘要 |

### 9.6 工作树隔离模式

设置 `isolation: worktree` 可以：

- 自动创建临时 git 工作树
- 给子 Agent 提供独立的文件系统
- 如果无更改则自动清理
- 防止文件冲突（多个 Agent 同时修改代码时）

### 9.7 Agent 持久化与恢复

- Transcript 存储在：`~/.claude/projects/<项目>/<sessionId>/subagents/`
- 可通过 `SendMessage` 工具发送消息恢复已有 Agent
- Agent 停止后收到消息可自动恢复
- 完整对话历史保留

---

## 10. 记忆系统 (Memory System)

### 10.1 双重记忆架构

Open Claw 有两套互补的记忆机制：

| 方面 | CLAUDE.md（手动记忆） | Auto Memory（自动记忆） |
|------|----------------------|----------------------|
| **写入者** | 用户 | Claude |
| **存储位置** | 项目代码库（提交到 git） | `~/.claude/projects/<项目>/memory/` |
| **作用域** | 项目/用户/组织 | 项目特定（按 git 仓库） |
| **加载方式** | 会话开始时完整加载 | MEMORY.md 前 200 行 |
| **格式** | Markdown 指令 | Markdown 笔记 + 主题文件 |
| **生命周期** | 手动维护 | 自动积累 |

### 10.2 自动记忆的文件结构

```
~/.claude/projects/<项目>/memory/
├── MEMORY.md           ← 索引文件，前 200 行每次加载
├── debugging.md        ← 主题文件，按需懒加载
├── api-conventions.md  ← 主题文件
└── architecture.md     ← 主题文件
```

每个主题文件使用 frontmatter 格式：

```yaml
---
name: 记忆名称
description: 一行描述（用于判断是否相关）
type: user|feedback|project|reference
---

记忆内容...
```

### 10.3 记忆类型

| 类型 | 用途 | 示例 |
|------|------|------|
| `user` | 用户角色、偏好、技能 | "用户是高级后端工程师，熟悉 Go" |
| `feedback` | 用户的工作方式偏好 | "不要在回复末尾总结刚做的事情" |
| `project` | 项目动态、决策 | "3月5日起冻结非关键 PR" |
| `reference` | 外部资源指针 | "Pipeline Bug 在 Linear INGEST 项目中跟踪" |

### 10.4 子 Agent 持久记忆

子 Agent 可以维护独立的记忆目录：

| 记忆作用域 | 存储位置 |
|-----------|----------|
| `memory: user` | `~/.claude/agent-memory/<agent-name>/` |
| `memory: project` | `.claude/agent-memory/<agent-name>/` |
| `memory: local` | `.claude/agent-memory-local/<agent-name>/` |

---

## 11. 设置系统 (Settings System)

### 11.1 设置优先级层级

```
优先级从高到低：

1. Managed Settings（企业管理设置）
   ├── 服务器管理（Anthropic 服务器下发）
   ├── 文件管理（managed-settings.json / managed-mcp.json）
   └── OS 级（macOS plist / Windows 注册表）

2. CLI 参数
   └── --model, --allowedTools 等（仅当前会话）

3. Local Project Settings
   └── .claude/settings.local.json（本机专用，gitignored）

4. Project Settings
   └── .claude/settings.json（团队共享，提交到 git）

5. User Settings
   └── ~/.claude/settings.json（个人默认，所有项目）
```

### 11.2 设置文件结构

```json
{
  "model": "sonnet",
  "defaultMode": "default",
  "autoMemoryEnabled": true,

  "permissions": {
    "defaultMode": "acceptEdits",
    "allow": ["Bash(npm run test)", "Read", "Grep"],
    "deny": ["Bash(rm -rf *)"],
    "ask": ["Bash(*)"]
  },

  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          { "type": "command", "command": "./validate.sh" }
        ]
      }
    ]
  },

  "env": {
    "NODE_ENV": "production"
  },

  "mcpServers": {
    "github": { "type": "http", "url": "..." }
  },

  "sandbox": {
    "enabled": true,
    "network": {
      "allowedDomains": ["github.com", "*.internal.example.com"]
    }
  }
}
```

### 11.3 关键配置字段

| 字段 | 说明 |
|------|------|
| `model` | 模型别名（sonnet/opus/haiku）或完整 ID |
| `defaultMode` | 默认权限模式 |
| `autoMemoryEnabled` | 是否启用自动记忆 |
| `agent` | 默认子 Agent |
| `effort` | 推理强度（low/medium/high/max） |
| `claudeMdExcludes` | 排除特定 CLAUDE.md 的 glob 模式 |
| `sandbox.enabled` | 是否启用 OS 级文件系统/网络沙箱 |
| `additionalDirectories` | 扩展文件访问范围 |
| `disableAllHooks` | 一键禁用所有 Hook |

---

## 12. Git 集成

### 12.1 自动读取的 Git 状态

Open Claw 在会话开始时自动读取：

- 当前分支名
- 未提交的更改（已跟踪和未跟踪文件）
- 最近的提交历史
- 主分支信息（用于 PR）
- Git 配置

### 12.2 Git 工作树集成

工作树用于并行会话和隔离执行：

```bash
# 手动创建工作树
git worktree add ../branch-name branch-name
cd ../branch-name
claude

# 子 Agent 自动工作树（isolation: worktree）
# 在 Agent 定义中设置，自动创建和清理
```

**工作树清理：**
- `WorktreeRemove` Hook 在会话退出时触发
- 子 Agent 无更改时自动清理
- 手动清理：`git worktree remove <path>`

### 12.3 安全约束

Open Claw 在 Git 操作中遵循严格的安全准则：

- 不自动修改 git 配置
- 不执行破坏性 git 命令（除非用户明确要求）
- 不跳过 Hook（--no-verify）
- 不强制推送到 main/master
- 优先创建新 commit 而非 amend
- 暂存时指定文件名，避免 `git add -A`

---

## 13. SDK 与自定义 Agent

### 13.1 可用 SDK

| SDK | 语言 | 状态 |
|-----|------|------|
| Python SDK | Python | 稳定 |
| TypeScript SDK | Node.js/TS | 稳定 |
| TypeScript V2 | Node.js/TS | 预览 |

### 13.2 SDK 核心能力

- 非交互自动化（CI/CD 集成）
- 程序化工具调用（Agent 自主决策）
- 多 Agent 协调
- 成本跟踪和监控
- 自定义权限评估
- 流式响应
- 扩展思考
- 结构化输出

### 13.3 使用场景

| 场景 | 说明 |
|------|------|
| CI/CD 流水线 | 在 GitHub Actions 等中运行 Claude 自动化任务 |
| 代码审查机器人 | 自动审查 PR 并提供反馈 |
| 文档生成 | 批量处理代码库生成文档 |
| 测试生成 | 自动为代码生成测试用例 |
| 自定义工作流 | 结合多个 Agent 处理复杂业务流程 |

---

## 14. 架构总览图

```
┌─────────────────────────────────────────────────────────────────┐
│                         用户输入                                 │
│              (终端 / VS Code / JetBrains / 桌面应用)              │
└─────────────────────────┬───────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Claude 语言模型层                              │
│                 (Opus / Sonnet / Haiku)                          │
│              推理 · 规划 · 代码生成 · 决策                        │
└─────────────────────────┬───────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Agentic Loop 控制器                            │
│              收集上下文 → 选择工具 → 执行 → 验证                   │
│                                                                  │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐  │
│  │  上下文管理  │  │   权限系统   │  │      Hook 系统          │  │
│  │ · 会话状态   │  │ · Deny/Ask  │  │ · PreToolUse           │  │
│  │ · CLAUDE.md  │  │   /Allow    │  │ · PostToolUse          │  │
│  │ · 自动记忆   │  │ · 工具级控制 │  │ · SessionStart/End    │  │
│  │ · 上下文压缩 │  │ · 沙箱模式  │  │ · 自定义 Shell/HTTP   │  │
│  └─────────────┘  └─────────────┘  └─────────────────────────┘  │
└─────────────────────────┬───────────────────────────────────────┘
                          │
            ┌─────────────┼─────────────────┐
            │             │                 │
            ▼             ▼                 ▼
┌───────────────┐ ┌──────────────┐ ┌───────────────────┐
│   内置工具     │ │   MCP 服务器  │ │  Agent / Subagent │
│ · Read/Write  │ │ · stdio      │ │ · Explore         │
│ · Edit        │ │ · HTTP       │ │ · Plan            │
│ · Bash        │ │ · SSE        │ │ · 自定义 Agent     │
│ · Glob/Grep   │ │              │ │ · 工作树隔离       │
│ · WebFetch    │ │ 工具格式：    │ │                   │
│ · WebSearch   │ │ mcp__srv__fn │ │ Skill / Plugin    │
│ · Agent       │ │              │ │ · 内置 Skill       │
│ · Task*       │ │ 服务：        │ │ · 自定义 Skill     │
│ · Cron*       │ │ GitHub/Slack │ │ · Plugin 市场      │
│ · Notebook    │ │ DB/Jira/...  │ │                   │
└───────────────┘ └──────────────┘ └───────────────────┘
            │             │                 │
            └─────────────┼─────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────────┐
│                       执行环境                                    │
│          本地文件系统 · Shell · Git · 网络 · 外部服务              │
└─────────────────────────────────────────────────────────────────┘
```

---

## 参考资源

| 主题 | 链接 |
|------|------|
| 官方文档首页 | https://code.claude.com/docs/en/ |
| 权限模式 | https://code.claude.com/docs/en/permission-modes |
| CLAUDE.md | https://code.claude.com/docs/en/memory |
| Hook 指南 | https://code.claude.com/docs/en/hooks-guide |
| MCP 集成 | https://code.claude.com/docs/en/mcp |
| 子 Agent | https://code.claude.com/docs/en/sub-agents |
| Skill 系统 | https://code.claude.com/docs/en/skills |
| 最佳实践 | https://code.claude.com/docs/en/best-practices |
| Agent SDK | https://platform.claude.com/docs/agents |
