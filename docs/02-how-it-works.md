# OpenClaw 工作原理文档

## 目录

- [1. 总体架构](#1-总体架构)
- [2. Gateway（网关）](#2-gateway网关)
- [3. Agent 运行时（Pi）](#3-agent-运行时pi)
- [4. Channel 渠道系统](#4-channel-渠道系统)
- [5. Session 会话管理](#5-session-会话管理)
- [6. Tool 工具系统](#6-tool-工具系统)
- [7. Skill 技能系统](#7-skill-技能系统)
- [8. 多 Agent 架构](#8-多-agent-架构)
- [9. 安全与沙箱](#9-安全与沙箱)
- [10. 配置系统](#10-配置系统)
- [11. 媒体处理管道](#11-媒体处理管道)
- [12. 设备节点 (Node)](#12-设备节点-node)
- [13. 定时任务与 Webhook](#13-定时任务与-webhook)
- [14. 架构总览图](#14-架构总览图)

---

## 1. 总体架构

OpenClaw 采用 **Hub-and-Spoke（中心辐射）** 架构。一个中央 Gateway 进程连接所有组件：

```
                        聊天渠道
                  ┌── WhatsApp (Baileys)
                  ├── Telegram (grammY)
                  ├── Discord (discord.js)
                  ├── Slack (Bolt)
                  ├── iMessage (BlueBubbles)
                  ├── ... 21+ 渠道
                  │
用户 ──消息──→ [Gateway]  ←──→  [Pi Agent 运行时]
                  │                    │
                  ├── Control UI       ├── 工具（Shell/浏览器/文件）
                  ├── CLI              ├── Skill 系统
                  ├── macOS App        ├── 模型 API 调用
                  └── 移动 Node        └── 会话 & 记忆
```

**核心原则：一切都在本地运行。** Gateway 进程是唯一的协调中心，AI 模型调用通过 API 发往云端，但所有数据和执行都在你的机器上。

---

## 2. Gateway（网关）

Gateway 是 OpenClaw 的核心进程，负责协调所有组件。

### 2.1 职责

| 职责 | 说明 |
|------|------|
| **WebSocket 控制面板** | 绑定 `ws://127.0.0.1:18789`，作为所有客户端的连接入口 |
| **渠道管理** | 启动/停止/监控各聊天渠道的连接 |
| **消息路由** | 将入站消息路由到正确的 Agent |
| **会话管理** | 创建、隔离、回收会话 |
| **健康监控** | 定期检查渠道状态，自动重启异常渠道 |
| **配置热重载** | 大部分配置修改无需重启即可生效 |
| **Control UI 服务** | 直接提供浏览器管理界面 |

### 2.2 生命周期

```
启动流程：
1. 读取 ~/.openclaw/openclaw.json 配置
2. 校验配置（严格模式，未知字段会阻止启动）
3. 启动 WebSocket 服务器 (端口 18789)
4. 初始化 Agent 运行时
5. 连接各聊天渠道
6. 启动健康监控
7. 启动 Skill 观察器

运行时：
- 接收消息 → 路由 → Agent 处理 → 回复
- 热重载配置变更
- 健康检查 & 自动恢复
```

### 2.3 热重载

Gateway 支持 `hybrid` 模式（默认），区分需要/不需要重启的配置：

**无需重启（热应用）：**
- 渠道配置（channels）
- Agent 配置（agents）
- 模型配置（models）
- Hook 和 Cron
- 会话、工具、UI、日志

**需要重启：**
- `gateway.*`（端口、绑定、认证、TLS）
- 基础设施设置
- Plugin 配置

```
配置项：
gateway.reload.mode: hybrid | hot | restart | off
gateway.reload.debounceMs: 防抖时间
```

### 2.4 健康监控

```
gateway.channelHealthCheckMinutes: 检查间隔（0 禁用）
gateway.channelStaleEventThresholdMinutes: 陈旧阈值
gateway.channelMaxRestartsPerHour: 每小时最大重启次数
channels.<provider>.healthMonitor.enabled: 按渠道覆盖
```

当渠道长时间未收到事件时，Gateway 自动重启该渠道连接。

---

## 3. Agent 运行时（Pi）

Agent 运行时是 OpenClaw 的"大脑"，负责 AI 推理和任务执行。

### 3.1 工作流程

```
消息到达
    │
    ▼
┌─────────────────────┐
│  1. 解析消息         │ ← 提取文本、媒体、上下文
│  2. 加载会话上下文    │ ← 恢复历史对话
│  3. 注入 Skill 定义  │ ← 构建系统提示词
│  4. 调用 AI 模型     │ ← 通过 API 发送请求
│  5. 解析模型响应      │ ← 文本 / 工具调用
│  6. 执行工具调用      │ ← Shell / 文件 / 浏览器
│  7. 循环直到完成      │ ← 多轮工具调用
│  8. 返回最终回复      │ ← 通过渠道发回用户
└─────────────────────┘
```

### 3.2 RPC 模式

Agent 运行时通过 RPC（远程过程调用）与 Gateway 通信，支持工具调用和流式响应。这种设计使 Agent 可以独立升级和替换。

### 3.3 模型调用

```
模型格式：提供商/模型名
示例：anthropic/claude-sonnet-4-6

支持 35+ 提供商：
- 云端：Anthropic、OpenAI、Google、Azure、AWS Bedrock
- 本地：Ollama、vLLM、SGLang
- 认证：OAuth 订阅或 API Key
- 降级：主模型不可用时自动切换到 fallback 模型
```

### 3.4 流式响应与分块

对于长回复，Agent 支持流式输出和分块发送，避免用户等待过长时间。每个渠道可配置不同的分块策略。

---

## 4. Channel 渠道系统

渠道是 OpenClaw 与用户之间的通信桥梁。

### 4.1 支持的渠道

| 渠道 | 底层库 | 说明 |
|------|--------|------|
| WhatsApp | Baileys | 扫码配对，最流行的渠道 |
| Telegram | grammY | 通过 BotFather 创建 Bot |
| Discord | discord.js | 支持服务器、线程、多账户 |
| Slack | Bolt | 工作区集成 |
| iMessage | BlueBubbles | 需要 macOS + BlueBubbles 服务 |
| Signal | - | 加密通信 |
| Google Chat | - | 企业集成 |
| Microsoft Teams | - | 企业集成 |
| Matrix | - | 去中心化协议 |
| IRC | - | 传统 IRC 协议 |
| Mattermost | Plugin | 自建团队通信 |
| Feishu（飞书） | - | 国内企业通信 |
| LINE | - | 亚洲地区流行 |
| Twitch | Plugin | 直播聊天 |
| Nostr | - | 去中心化社交 |
| WebChat | 内置 | Control UI 中的聊天界面 |

### 4.2 渠道架构

```
用户消息
    │
    ▼
┌─────────────────────┐
│  Channel Adapter     │  ← 每个渠道一个适配器
│  - 认证 & 连接管理   │
│  - 消息格式转换      │
│  - 媒体处理          │
│  - DM / 群聊区分     │
│  - 提及检测          │
└─────────┬───────────┘
          │ 标准化消息
          ▼
┌─────────────────────┐
│  Message Router      │  ← Gateway 核心
│  - 路由到正确 Agent   │
│  - 会话查找/创建     │
│  - 安全策略检查      │
└─────────────────────┘
```

### 4.3 多账户支持

同一渠道可以绑定多个账户（如多个 WhatsApp 号码、多个 Telegram Bot），通过 `accountId` 区分，路由到不同的 Agent。

### 4.4 群聊提及门控

```json5
{
  agents: {
    "my-agent": {
      groupChat: {
        mentionPatterns: ["@openclaw", "openclaw"]
      }
    }
  },
  channels: {
    telegram: {
      groups: {
        "*": { requireMention: true }
      }
    }
  }
}
```

只有被 @提及时，Bot 才会在群聊中响应。

---

## 5. Session 会话管理

会话是 Agent 维护对话上下文的基本单位。

### 5.1 会话模型

```
会话存储：~/.openclaw/agents/<agentId>/sessions/*.jsonl

会话类型：
- main：DM 的默认会话（所有 DM 共享）
- per-peer：每个发送者独立会话
- per-channel-peer：每个发送者+渠道组合独立会话
- 群聊会话：每个群/频道独立
```

### 5.2 会话隔离策略

通过 `session.dmScope` 控制 DM 会话的隔离程度：

| 模式 | 说明 |
|------|------|
| `main` | 所有 DM 共享一个会话（默认，保持连续性） |
| `per-peer` | 每个发送者独立会话 |
| `per-channel-peer` | 每个发送者+渠道组合独立 |
| `per-account-channel-peer` | 多账户完全隔离 |

### 5.3 会话重置

```json5
{
  session: {
    reset: {
      mode: "daily",         // 每日重置
      atHour: 3,             // 凌晨 3 点
      // 或者
      mode: "idle-based",
      idleMinutes: 60        // 空闲 60 分钟后重置
    }
  }
}
```

用户也可以在聊天中手动 `/reset` 或 `/new`。

### 5.4 会话间通信

Agent 可以跨会话交互：

```
tools:
- sessions_list    → 发现活跃会话
- sessions_history → 获取其他会话的历史记录
- sessions_send    → 向其他会话发送消息
```

---

## 6. Tool 工具系统

工具是 Agent 与外部世界交互的能力。

### 6.1 内置工具

| 工具类别 | 工具 | 说明 |
|---------|------|------|
| **Shell 执行** | `exec` | 执行 Shell 命令（bash/zsh/PowerShell） |
| **浏览器** | `browser` | 通过 CDP 控制 Chrome/Chromium，网页自动化 |
| **Canvas** | `canvas` | A2UI 推送/重置、eval 执行、截图 |
| **文件操作** | 文件读写 | 读取、创建、编辑文件 |
| **Web 搜索** | `web_search` | 支持 Brave、Perplexity、Gemini、Grok、Kimi、Firecrawl |
| **定时任务** | `cron` | 创建和管理 Cron Job |
| **Webhook** | `hooks` | 接收外部系统的回调 |
| **会话** | `sessions_*` | 跨会话通信 |
| **设备** | `nodes` | 控制远程设备节点（摄像头、屏幕、通知等） |
| **Gmail** | `gmail` | Pub/Sub 集成，邮件通知 |
| **Discord/Slack** | 平台 action | 平台特定的高级操作 |

### 6.2 工具执行安全控制

```json5
{
  tools: {
    exec: {
      security: "ask"     // "deny" | "ask" | "full"
    },
    elevated: {
      enabled: false       // 高风险操作（摄像头、联系人、短信）
    }
  }
}
```

| 安全级别 | 行为 |
|---------|------|
| `deny` | 禁止 Shell 执行 |
| `ask` | 每次执行前需要用户批准 |
| `full` | 自动执行（谨慎使用） |

### 6.3 工具执行流程

```
Agent 决定调用工具
       │
       ▼
┌──────────────────┐
│ 1. 安全策略检查    │ ← deny / ask / full
│ 2. 沙箱决策       │ ← 是否在 Docker 中执行
│ 3. 权限审批       │ ← ask 模式下等待用户确认
│ 4. 执行工具       │ ← 在主机或沙箱中运行
│ 5. 收集输出       │ ← stdout / stderr / 文件
│ 6. 返回给 Agent   │ ← Agent 继续推理
└──────────────────┘
```

---

## 7. Skill 技能系统

Skill 是 OpenClaw 的扩展机制，通过 Markdown 文件教会 Agent 新的能力。

### 7.1 Skill 的本质

Skill = 一个目录 + 一个 `SKILL.md` 文件。没有 SDK、不需要编译、不需要特殊运行时。SKILL.md 中的 YAML frontmatter + Markdown 指令告诉 Agent：

- 这个 Skill 叫什么
- 什么时候使用它
- 怎么使用它（工具调用步骤）

### 7.2 加载顺序与优先级

```
优先级从高到低：
1. 工作区 Skill   → <workspace>/skills/
2. 本地 Skill     → ~/.openclaw/skills/
3. 内置 Skill     → 安装包自带

同名时高优先级覆盖低优先级。
额外目录通过 skills.load.extraDirs 配置。
```

### 7.3 Skill 加载时的门控（Gating）

Skill 在加载时会检查依赖条件：

```yaml
metadata:
  openclaw:
    requires:
      bins: ["ffmpeg"]          # 必须安装 ffmpeg
      anyBins: ["brew", "apt"]  # brew 或 apt 任一存在
      env: ["GITHUB_TOKEN"]     # 必须设置环境变量
      config: ["channels.telegram.botToken"]  # 必须配置
    os: "darwin"                # 仅 macOS
```

条件不满足的 Skill 不会加载，不消耗上下文空间。

### 7.4 Token 开销

每个 Skill 在系统提示词中的开销：

```
基础开销（>=1 个 Skill）：195 字符
每个 Skill：97 字符 + name + description + location 的长度
粗略估算：每个 Skill 约 24 token
```

### 7.5 Skill 运行时行为

```
会话开始
    │
    ▼
快照合格 Skill → 注入到系统提示词
    │
    ▼
Agent 运行中...
    │
    ├── SKILL.md 文件变更 → 观察器检测 → 刷新快照
    │
    └── 会话结束 → 快照丢弃
```

（详细的 Skill 创建和使用请参见深度使用文档。）

---

## 8. 多 Agent 架构

OpenClaw 支持在一个 Gateway 上运行多个完全隔离的 Agent。

### 8.1 Agent 隔离三维度

每个 Agent 是一个"完全独立的大脑"：

| 维度 | 说明 |
|------|------|
| **Workspace** | 独立的工作目录、SOUL.md 人设、AGENTS.md、本地笔记 |
| **State Directory (agentDir)** | `~/.openclaw/agents/<agentId>/`，存储认证、模型注册、配置 |
| **Session Store** | `~/.openclaw/agents/<agentId>/sessions/`，独立的聊天历史 |

> 重要：不要在不同 Agent 之间复用 agentDir，会导致认证/会话冲突。

### 8.2 消息路由机制

入站消息通过 **Bindings（绑定规则）** 路由到正确的 Agent：

```
路由优先级（从高到低）：
1. 精确 peer 匹配（DM 或群 ID）
2. 父级 peer 匹配（线程继承）
3. Guild/Role 组合（Discord）
4. Guild 或 Team ID
5. Account ID 匹配
6. 渠道级 fallback
7. 默认 Agent
```

采用 **"最具体的优先"** 原则。绑定规则中多个字段同时指定时，必须全部匹配（AND 语义）。

### 8.3 配置示例

```json5
{
  agents: {
    list: [
      {
        id: "work-agent",
        workspace: "~/openclaw-work",
        model: { primary: "anthropic/claude-sonnet-4-6" },
        bindings: [
          { channel: "slack", accountId: "work-slack" }
        ]
      },
      {
        id: "personal-agent",
        workspace: "~/openclaw-personal",
        model: { primary: "openai/gpt-4.1" },
        bindings: [
          { channel: "whatsapp" },
          { channel: "telegram" }
        ]
      }
    ]
  }
}
```

### 8.4 应用场景

| 场景 | 说明 |
|------|------|
| 工作/生活分离 | 不同渠道路由到不同 Agent，数据完全隔离 |
| 多人共用 | 一个 Gateway 多人使用，每人一个 Agent |
| 多语言/人设 | 不同 Agent 配置不同人设和语言偏好 |
| 不同模型 | 不同任务使用不同模型（如代码用 Claude、日常用 GPT） |

---

## 9. 安全与沙箱

### 9.1 安全模型

OpenClaw 采用 **个人助手安全模型**（非多租户隔离）：

```
威胁模型：
- 一个可信运营者（你）控制一个 Gateway 实例
- 攻击者可能通过消息尝试 Prompt Injection
- 不受信的内容（网页、邮件、附件）可能包含恶意指令
- 被泄露的凭证可能导致工具被滥用
```

### 9.2 多层安全控制

```
┌──────────────────────────────────────────┐
│ 第 1 层：访问控制                          │
│  - Gateway 认证 (Token / Password)        │
│  - DM Policy (pairing / allowlist / open) │
│  - 群聊白名单                              │
│  - 提及门控                                │
├──────────────────────────────────────────┤
│ 第 2 层：工具授权                          │
│  - exec security: deny / ask / full       │
│  - elevated tools 开关                    │
│  - 解释器白名单 (strictInlineEval)        │
├──────────────────────────────────────────┤
│ 第 3 层：执行沙箱                          │
│  - Docker 容器隔离                        │
│  - 工作区访问控制 (none / ro / rw)        │
│  - 网络出口控制                            │
├──────────────────────────────────────────┤
│ 第 4 层：凭证保护                          │
│  - 文件权限 (700/600)                     │
│  - 环境变量作用域隔离                      │
│  - Secret 不进入 Agent 工作区             │
└──────────────────────────────────────────┘
```

### 9.3 Docker 沙箱

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "non-main",    // off | non-main | all
        scope: "session"     // session | agent | shared
      }
    }
  }
}
```

| 模式 | 说明 |
|------|------|
| `off` | 不使用沙箱 |
| `non-main` | 非 main session 使用沙箱 |
| `all` | 所有 session 都使用沙箱 |

| 范围 | 说明 |
|------|------|
| `session` | 每个会话一个容器 |
| `agent` | 每个 Agent 一个容器 |
| `shared` | 所有 Agent 共享容器 |

### 9.4 Prompt Injection 防御

官方承认 Prompt Injection "尚未被解决"，采用多层缓解策略：

1. **收紧访问**：通过 DM Policy 和白名单限制谁能发消息
2. **缩减工具范围**：对公开 Agent 禁用危险工具（exec、browser）
3. **选择强模型**：弱/小模型的抗注入能力差
4. **沙箱隔离**：即使注入成功，也限制可造成的损害
5. **凭证隔离**：敏感信息不放在 Agent 可访问的工作区

### 9.5 安全审计

```bash
openclaw security audit          # 基本检查
openclaw security audit --deep   # 探测运行中的 Gateway
openclaw security audit --fix    # 自动修复权限和配置问题
```

---

## 10. 配置系统

### 10.1 配置文件

```
文件：~/.openclaw/openclaw.json
格式：JSON5（支持注释和尾逗号）
缺失时：使用安全默认值
```

### 10.2 配置结构

```json5
{
  // Agent 配置
  agents: {
    defaults: {
      workspace: "~/openclaw",
      model: { primary: "anthropic/claude-sonnet-4-6" },
      sandbox: { mode: "off" }
    },
    list: [/* 多 Agent 列表 */]
  },

  // 渠道配置
  channels: {
    telegram: { enabled: true, botToken: "..." },
    whatsapp: { enabled: true, dmPolicy: "pairing" }
  },

  // 会话配置
  session: {
    dmScope: "main",
    reset: { mode: "daily", atHour: 3 }
  },

  // 工具配置
  tools: {
    exec: { security: "ask" },
    elevated: { enabled: false }
  },

  // Skill 配置
  skills: {
    entries: { /* 按名称配置 Skill */ },
    load: { extraDirs: [] }
  },

  // 定时任务
  cron: { enabled: true },

  // Webhook
  hooks: { enabled: false, token: "..." },

  // Gateway 配置
  gateway: {
    bind: "loopback",
    port: 18789,
    auth: { token: "..." },
    reload: { mode: "hybrid" }
  }
}
```

### 10.3 环境变量

加载优先级：

```
1. 父进程环境变量
2. .env（当前目录）
3. ~/.openclaw/.env（全局）
```

配置文件中支持变量替换：

```json5
{
  channels: {
    telegram: {
      botToken: "${TELEGRAM_BOT_TOKEN}"
    }
  }
}
```

### 10.4 配置文件拆分 ($include)

```json5
{
  $include: ["channels.json5", "agents.json5"],
  gateway: { /* ... */ }
}
```

- 相对路径解析到包含文件所在目录
- 最多 10 层嵌套
- 同级字段覆盖 include 的值
- 支持单文件或数组（数组时深度合并）

### 10.5 CLI 配置工具

```bash
openclaw onboard              # 交互式引导
openclaw configure            # 配置向导
openclaw config get <key>     # 获取配置值
openclaw config set <key> <value>  # 设置配置值
openclaw config unset <key>   # 删除配置项
openclaw doctor               # 诊断验证
openclaw doctor --fix         # 自动修复
```

### 10.6 严格校验

OpenClaw 对配置进行**严格 Schema 校验**——未知字段、类型错误或无效值会**阻止启动**。只有 `$schema` 作为根级别例外允许用于编辑器 JSON Schema 提示。

---

## 11. 媒体处理管道

OpenClaw 支持图片、音频、视频和文档的收发处理。

### 11.1 处理流程

```
入站媒体：
  用户发送图片/语音/文件
      │
      ▼
  渠道适配器接收
      │
      ▼
  保存到临时文件
      │
      ▼
  类型处理：
  ├── 图片 → 调整尺寸（imageMaxDimensionPx，默认 1200px）→ 发给模型（Vision）
  ├── 音频 → 转写为文本（Whisper 等）→ 注入对话
  ├── 文档 → 提取文本 → 注入对话
  └── 视频 → 提取帧/音轨 → 分别处理

出站媒体：
  Agent 生成 → 渠道适配器 → 发送给用户
```

### 11.2 音频详细配置

语音消息可以配置转写提供商、TTS（文本转语音）输出等。

---

## 12. 设备节点 (Node)

设备节点让远程设备的能力（摄像头、屏幕、传感器等）可被 Agent 使用。

### 12.1 节点类型

| 节点 | 平台 | 能力 |
|------|------|------|
| **macOS App** | macOS | 菜单栏、语音唤醒、Push-to-Talk、WebChat、调试、远程 Gateway 控制 |
| **macOS Node** | macOS | system.run、system.notify、Canvas、摄像头 |
| **iOS Node** | iOS | Canvas、语音唤醒、Talk Mode、摄像头、屏幕录制、推送通知 |
| **Android Node** | Android | Canvas、摄像头、屏幕截图、通知、定位、短信、相册、联系人、日历、运动数据、App 更新 |

### 12.2 节点通信

```
设备节点 ←─ 配对 ─→ Gateway
              │
              ▼
     Agent 调用 `nodes` 工具
              │
              ▼
     节点执行操作并返回结果
```

### 12.3 跨平台 Skill

当 Linux Gateway 连接到 macOS 节点且允许 `system.run` 时，需要 macOS 二进制的 Skill 可以通过节点执行。

---

## 13. 定时任务与 Webhook

### 13.1 Cron 定时任务

Agent 可以创建和管理定时任务：

```json5
{
  cron: {
    enabled: true,
    maxConcurrentRuns: 3,
    sessionRetention: "24h",    // 保留历史运行记录
    runLog: {
      maxBytes: "2mb",
      keepLines: 2000
    }
  }
}
```

使用场景：
- 定时检查邮件
- 定期生成报告
- 定时健康检查
- 定时数据同步

### 13.2 Heartbeat 心跳

Agent 可以定时主动联系用户：

```json5
{
  agents: {
    defaults: {
      heartbeat: {
        every: "2h",            // 每 2 小时
        target: "whatsapp",     // 通过 WhatsApp 发送
        directPolicy: "allow"   // 允许直接发消息
      }
    }
  }
}
```

### 13.3 Webhook

外部系统可以通过 Webhook 触发 Agent 动作：

```json5
{
  hooks: {
    enabled: true,
    token: "your-secret-token",  // 验证密钥
    path: "/hooks",
    defaultSessionKey: "webhook",
    mappings: [/* 路由规则 */]
  }
}
```

使用场景：
- GitHub PR 通知 → Agent 自动审查
- 监控告警 → Agent 分析并通知
- 外部系统事件 → Agent 自动处理

---

## 14. 架构总览图

```
┌─────────────────────────────────────────────────────────────────────┐
│                            用户                                      │
│    WhatsApp / Telegram / Discord / Slack / iMessage / WebChat ...   │
└────────────────────────────────┬────────────────────────────────────┘
                                 │ 消息
                                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│                         Gateway (端口 18789)                         │
│                                                                      │
│  ┌─────────────┐  ┌──────────────┐  ┌────────────┐  ┌───────────┐  │
│  │ Channel      │  │  Message     │  │  Session   │  │  Health   │  │
│  │ Adapters     │  │  Router      │  │  Manager   │  │  Monitor  │  │
│  │ (21+ 渠道)   │  │  (Bindings)  │  │  (隔离策略) │  │  (自动恢复)│  │
│  └──────┬───────┘  └──────┬───────┘  └─────┬──────┘  └───────────┘  │
│         │                 │                │                         │
│         └─────────────────┼────────────────┘                         │
│                           │                                          │
│  ┌────────────────────────▼─────────────────────────────────────┐   │
│  │                    Agent Runtime (Pi)                         │   │
│  │                                                               │   │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌─────────────┐  │   │
│  │  │  Model    │  │  Tool    │  │  Skill   │  │  Memory &   │  │   │
│  │  │  Provider │  │  System  │  │  Engine  │  │  Context    │  │   │
│  │  │  (35+)   │  │          │  │          │  │             │  │   │
│  │  │ Claude   │  │ - exec   │  │ Bundled  │  │ Sessions    │  │   │
│  │  │ GPT      │  │ - browser│  │ Managed  │  │ SOUL.md     │  │   │
│  │  │ Gemini   │  │ - files  │  │ Workspace│  │ Persistence │  │   │
│  │  │ Ollama   │  │ - search │  │ ClawHub  │  │             │  │   │
│  │  │ ...      │  │ - nodes  │  │          │  │             │  │   │
│  │  └──────────┘  │ - cron   │  └──────────┘  └─────────────┘  │   │
│  │                │ - hooks  │                                   │   │
│  │                └──────────┘                                   │   │
│  └───────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                     安全层                                    │   │
│  │  DM Policy · 工具授权 · Docker 沙箱 · 凭证隔离 · 审计工具    │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  ┌────────────┐  ┌────────────┐  ┌──────────────────────────────┐   │
│  │ Control UI │  │ CLI        │  │ Device Nodes                 │   │
│  │ (浏览器)    │  │ 命令行工具  │  │ macOS · iOS · Android       │   │
│  └────────────┘  └────────────┘  └──────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        本地执行环境                                   │
│          文件系统 · Shell · 浏览器 (CDP) · 智能家居 · 网络           │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 参考资源

| 主题 | 链接 |
|------|------|
| 官方文档 | https://docs.openclaw.ai |
| GitHub 仓库 | https://github.com/openclaw/openclaw |
| Gateway 配置 | https://docs.openclaw.ai/gateway/configuration |
| 安全文档 | https://docs.openclaw.ai/gateway/security |
| 多 Agent | https://docs.openclaw.ai/concepts/multi-agent |
| Skill 系统 | https://docs.openclaw.ai/tools/skills |
| 功能列表 | https://docs.openclaw.ai/concepts/features |
| 故障排查 | https://docs.openclaw.ai/gateway/troubleshooting |
