# OpenClaw 深度使用文档：Skill 系统与扩展

## 目录

- [1. 什么是 Skill](#1-什么是-skill)
- [2. 内置 Skill 与安装 Skill](#2-内置-skill-与安装-skill)
- [3. 如何使用 Skill](#3-如何使用-skill)
- [4. 如何创建自定义 Skill](#4-如何创建自定义-skill)
- [5. SKILL.md 格式详解](#5-skillmd-格式详解)
- [6. Skill 存储位置与优先级](#6-skill-存储位置与优先级)
- [7. Skill 的门控机制 (Gating)](#7-skill-的门控机制-gating)
- [8. Skill 配置与环境变量注入](#8-skill-配置与环境变量注入)
- [9. 探索新 Skill](#9-探索新-skill)
- [10. ClawHub — Skill 注册中心](#10-clawhub--skill-注册中心)
- [11. 社区生态](#11-社区生态)
- [12. 自己写 Skill 实战](#12-自己写-skill-实战)
- [13. Skill 安全注意事项](#13-skill-安全注意事项)
- [14. Plugin 系统](#14-plugin-系统)
- [15. 高级模式](#15-高级模式)
- [16. 最佳实践](#16-最佳实践)
- [17. 参考资源](#17-参考资源)

---

## 1. 什么是 Skill

Skill（技能）是 OpenClaw 的**核心扩展机制**。它是一个目录，包含一个 `SKILL.md` 文件，用 YAML frontmatter + Markdown 指令告诉 Agent 如何完成特定任务。

### 核心特征

- **纯文本**：不需要 SDK、不需要编译、不需要特殊运行时
- **就是 Markdown**：YAML frontmatter 定义元数据，Markdown 正文定义行为
- **兼容 AgentSkills 标准**：跨工具兼容的开放标准
- **社区驱动**：ClawHub 注册中心有 13,000+ 社区贡献的 Skill
- **自适应**：Agent 可以自己编写和更新 Skill

### Skill 能做什么

Skill 能让 Agent 学会任何可用工具组合完成的任务，例如：
- 调用特定 API（GitHub、Spotify、智能家居）
- 执行自动化工作流（部署、CI/CD、数据处理）
- 操作特定软件（Obsidian、Gmail、数据库）
- 浏览器自动化（网页抓取、表单填写）
- 设备控制（摄像头、通知、文件管理）

---

## 2. 内置 Skill 与安装 Skill

### 2.1 内置 Skill（Bundled）

OpenClaw 安装包自带一批基础 Skill，开箱即用。你可以在配置中启用/禁用它们：

```json5
{
  skills: {
    entries: {
      "bundled-skill-name": {
        enabled: false  // 禁用某个内置 Skill
      }
    },
    // 或者用白名单模式，只启用指定的内置 Skill
    allowBundled: ["skill-a", "skill-b"]
  }
}
```

### 2.2 从 ClawHub 安装 Skill

```bash
# 安装 Skill
openclaw skills install <skill-slug>

# 或使用 npx（无需预先安装 clawhub CLI）
npx clawhub@latest install <skill-slug>

# 更新所有已安装 Skill
openclaw skills update --all
```

### 2.3 手动安装 Skill

将 Skill 目录复制到以下位置之一：

```bash
# 全局（所有 Agent 可用）
~/.openclaw/skills/<skill-name>/SKILL.md

# 工作区（仅该 Agent 可用）
<workspace>/skills/<skill-name>/SKILL.md
```

---

## 3. 如何使用 Skill

### 3.1 自动使用（模型调用）

大多数 Skill 默认会被注入系统提示词，Agent **自动判断何时使用**。你只需正常对话：

```
帮我在 GitHub 上创建一个 issue
```

如果安装了 GitHub Skill，Agent 会自动使用它。

### 3.2 手动触发（斜杠命令）

`user-invocable: true`（默认）的 Skill 可以作为斜杠命令使用：

```
/skill-name
/skill-name 参数
```

### 3.3 禁止自动调用

某些有副作用的 Skill 可以设置为仅手动触发：

```yaml
disable-model-invocation: true  # Agent 不会自动使用，必须用 /skill-name
```

### 3.4 隐藏 Skill

仅作为背景知识注入，不暴露为斜杠命令：

```yaml
user-invocable: false  # 不出现在 / 菜单中
```

---

## 4. 如何创建自定义 Skill

### 4.1 最简 Skill（60 秒完成）

```bash
mkdir -p ~/.openclaw/skills/hello
```

创建 `~/.openclaw/skills/hello/SKILL.md`：

```markdown
---
name: hello
description: 用中文打招呼并报告当前系统状态
---

当用户触发此 Skill 时：

1. 用中文友好地打招呼
2. 运行 `uname -a` 查看系统信息
3. 运行 `uptime` 查看运行时间
4. 运行 `df -h /` 查看磁盘使用
5. 用简洁的格式汇报以上信息
```

现在在聊天中发送 `/hello` 即可使用。

### 4.2 带 API 调用的 Skill

```markdown
---
name: weather
description: 查询指定城市的天气预报
---

当用户询问天气时：

1. 从用户消息中提取城市名
2. 使用 curl 调用天气 API：
   ```
   curl -s "https://wttr.in/<城市>?format=j1"
   ```
3. 解析 JSON 响应，提取：
   - 当前温度
   - 天气状况
   - 未来 3 天预报
4. 用清晰的格式回复用户
```

### 4.3 带工具调用的 Skill

```markdown
---
name: screenshot-report
description: 截取网页截图并生成分析报告
---

当用户提供一个 URL 时：

1. 使用 browser 工具打开该 URL
2. 等待页面加载完成
3. 截取全屏截图
4. 分析截图内容：
   - 页面布局
   - 主要内容
   - 可能的问题（加载错误、样式异常）
5. 生成报告发送给用户
```

---

## 5. SKILL.md 格式详解

### 5.1 基本结构

```markdown
---
name: skill-identifier
description: 简短描述这个 Skill 做什么以及何时使用
---

# Skill 标题

Markdown 格式的指令正文...
```

### 5.2 完整 Frontmatter 字段

| 字段 | 类型 | 默认 | 说明 |
|------|------|------|------|
| `name` | string | 目录名 | Skill 标识符 |
| `description` | string | - | 功能描述（Agent 据此判断是否使用） |
| `homepage` | string | - | 在 UI 中显示为"Website"链接 |
| `user-invocable` | boolean | `true` | 是否作为斜杠命令暴露 |
| `disable-model-invocation` | boolean | `false` | 是否禁止 Agent 自动使用 |
| `command-dispatch` | string | - | 设为 `tool` 跳过模型直接调用工具 |
| `command-tool` | string | - | 与 `command-dispatch: tool` 配合，指定目标工具 |
| `command-arg-mode` | string | - | 设为 `raw` 直接传递未解析的参数 |
| `metadata` | object | - | 门控条件、平台过滤、安装配置等 |

### 5.3 metadata 字段详解

```yaml
metadata:
  openclaw:
    # 门控条件（不满足则不加载）
    requires:
      bins: ["ffmpeg"]              # 必须存在的二进制
      anyBins: ["brew", "apt"]      # 任一存在即可
      env: ["GITHUB_TOKEN"]         # 必须设置的环境变量
      config: ["channels.telegram.botToken"]  # 必须有的配置

    # 平台过滤
    os: "darwin"                     # darwin | linux | win32

    # 始终加载（跳过所有门控）
    always: true

    # UI 图标（macOS）
    emoji: "🌤️"

    # API Key 关联
    primaryEnv: "WEATHER_API_KEY"

    # 安装方式声明
    install:
      - id: "brew"
        kind: "brew"
        formula: "ffmpeg"
        bins: ["ffmpeg"]
        label: "通过 Homebrew 安装"
      - id: "node"
        kind: "node"
        package: "some-tool"
        bins: ["some-tool"]
```

### 5.4 目录结构

```
my-skill/
├── SKILL.md           ← 必需，主文件
├── README.md          ← 可选，给人看的说明
├── examples/          ← 可选，示例文件
├── templates/         ← 可选，模板文件
└── scripts/           ← 可选，辅助脚本
```

---

## 6. Skill 存储位置与优先级

### 6.1 三级存储

| 位置 | 优先级 | 说明 |
|------|--------|------|
| `<workspace>/skills/` | 最高 | 工作区级，每个 Agent 独立 |
| `~/.openclaw/skills/` | 中 | 本地/管理级，所有 Agent 共享 |
| 安装包内置 | 最低 | Bundled Skill |

**同名时高优先级覆盖低优先级。**

### 6.2 额外目录

```json5
{
  skills: {
    load: {
      extraDirs: ["/path/to/extra/skills"]
    }
  }
}
```

### 6.3 多 Agent 场景

```
~/.openclaw/skills/          ← 所有 Agent 共享
~/openclaw-work/skills/      ← 仅 work Agent 可用
~/openclaw-personal/skills/  ← 仅 personal Agent 可用
```

---

## 7. Skill 的门控机制 (Gating)

门控是 OpenClaw 的智能 Skill 管理机制——不满足前提条件的 Skill 不会加载，不会浪费上下文空间。

### 7.1 二进制依赖

```yaml
metadata:
  openclaw:
    requires:
      bins: ["docker", "git"]     # 必须全部存在
      anyBins: ["nvim", "vim"]    # 任一存在即可
```

### 7.2 环境变量依赖

```yaml
metadata:
  openclaw:
    requires:
      env: ["OPENAI_API_KEY"]     # 必须设置该环境变量
```

### 7.3 配置依赖

```yaml
metadata:
  openclaw:
    requires:
      config: ["channels.slack.botToken"]  # 必须有 Slack 配置
```

### 7.4 平台过滤

```yaml
metadata:
  openclaw:
    os: "darwin"    # 仅 macOS 加载
```

### 7.5 跳过门控

```yaml
metadata:
  openclaw:
    always: true    # 无条件加载
```

---

## 8. Skill 配置与环境变量注入

### 8.1 为 Skill 配置 API Key

在 `~/.openclaw/openclaw.json` 中：

```json5
{
  skills: {
    entries: {
      "weather": {
        enabled: true,
        apiKey: {
          source: "env",
          provider: "default",
          id: "WEATHER_API_KEY"
        }
      }
    }
  }
}
```

### 8.2 为 Skill 注入环境变量

```json5
{
  skills: {
    entries: {
      "my-skill": {
        env: {
          DATABASE_URL: "postgres://...",
          API_SECRET: "..."
        },
        config: {
          customKey: "customValue"
        }
      }
    }
  }
}
```

### 8.3 环境变量作用域

OpenClaw 的环境注入是**安全隔离**的：

```
Agent 运行前：
  1. 读取 Skill metadata
  2. 将配置的 env 和 apiKey 注入 process.env
  3. 构建系统提示词

Agent 运行后：
  4. 恢复原始环境变量
```

这意味着 Skill 的环境变量**不会泄露到其他进程**。

---

## 9. 探索新 Skill

### 9.1 ClawHub 在线浏览

访问 https://clawhub.ai 浏览和搜索 Skill。

### 9.2 awesome-openclaw-skills 精选列表

GitHub 上的社区精选列表，从 13,729 个 Skill 中精选了 5,211 个高质量 Skill：

https://github.com/VoltAgent/awesome-openclaw-skills

### 9.3 按分类浏览

社区精选 Skill 按 25 个类别组织：

| 类别 | Skill 数量 | 示例 |
|------|-----------|------|
| Coding Agents & IDEs | 1,184 | 代码生成、IDE 集成、Lint |
| Web & Frontend | 919 | React、Vue、CSS 工具 |
| DevOps & Cloud | 393 | Docker、K8s、AWS、CI/CD |
| Search & Research | 345 | 搜索引擎、学术搜索 |
| Browser & Automation | 322 | 网页抓取、表单填写 |
| Productivity & Tasks | 205 | 日程管理、TODO |
| CLI Utilities | 180 | 命令行工具增强 |
| AI & LLMs | 176 | 模型集成、Prompt 工程 |
| Image & Video | 170 | 图像生成、视频处理 |
| Git & GitHub | 167 | PR 管理、Issue 追踪 |
| Communication | 146 | 邮件、消息平台 |
| PDF & Documents | 105 | 文档处理、OCR |
| Marketing & Sales | 102 | SEO、社媒管理 |
| Health & Fitness | 87 | WHOOP、健康追踪 |
| Media & Streaming | 85 | Spotify、播客 |
| Notes & PKM | 70 | Obsidian、笔记管理 |
| Calendar & Scheduling | 65 | 日历、会议管理 |
| Security & Passwords | 53 | 安全审计、密码管理 |
| Shopping & E-commerce | 51 | 购物、价格追踪 |
| Speech & Transcription | 45 | 语音识别、转写 |
| Apple Apps & Services | 44 | Apple 生态集成 |
| Smart Home & IoT | 41 | Hue、智能家居控制 |
| Self-Hosted | 33 | 自建服务集成 |
| iOS & macOS Dev | 29 | Swift、Xcode |
| Data & Analytics | 28 | 数据分析、BI |

### 9.4 CLI 搜索

```bash
# 列出已安装的 Skill
openclaw skills list

# 从 ClawHub 安装
openclaw skills install <slug>
```

---

## 10. ClawHub — Skill 注册中心

### 10.1 什么是 ClawHub

ClawHub（https://clawhub.ai）是 OpenClaw 的官方 Skill 注册中心，类似 npm 之于 Node.js：

- **发布**：上传你的 Skill 让所有人使用
- **版本管理**：像 npm 一样管理版本
- **向量搜索**：智能搜索找到合适的 Skill
- **开放**：无审核门槛（"No gatekeeping, just signal"）

### 10.2 安装 Skill

```bash
# 通过 OpenClaw CLI
openclaw skills install <slug>

# 通过 npx
npx clawhub@latest install <slug>
```

### 10.3 发布 Skill

```bash
# 发布本地 Skill 到 ClawHub
clawhub publish <path-to-skill>

# 同步更新
clawhub sync
```

### 10.4 管理已安装的 Skill

```bash
clawhub install <slug>       # 安装
clawhub uninstall <slug>     # 卸载
clawhub list                 # 列出已安装
clawhub update --all         # 更新全部
```

### 10.5 注册中心规模

截至 2026 年 2 月底，ClawHub 上有 **13,729** 个社区贡献的 Skill。

---

## 11. 社区生态

### 11.1 GitHub 仓库

| 仓库 | 说明 |
|------|------|
| [openclaw/openclaw](https://github.com/openclaw/openclaw) | 主仓库（100,000+ stars） |
| [openclaw/clawhub](https://github.com/openclaw/clawhub) | ClawHub 注册中心源码 |
| [VoltAgent/awesome-openclaw-skills](https://github.com/VoltAgent/awesome-openclaw-skills) | 社区精选 5,200+ Skill |

### 11.2 社区渠道

| 渠道 | 用途 |
|------|------|
| Discord | 官方社区，讨论和支持 |
| GitHub Discussions | 功能请求、技术讨论 |
| GitHub Issues | Bug 报告 |

### 11.3 贡献 Skill 到 awesome 列表

贡献步骤：
1. Skill 必须已发布到 `github.com/openclaw/skills` 仓库
2. PR 中需包含 ClawHub 链接和 GitHub 仓库链接
3. 不接受个人仓库或外部来源的 Skill

### 11.4 项目历史

```
2025-11  Clawdbot 发布（最初名为"WhatsApp Relay"周末项目）
         → 72 小时内获得 60,000+ GitHub stars
2025-12  因 Anthropic 商标投诉更名为 Moltbot
2026-01  更名为 OpenClaw
2026-02  ClawHub 上 13,729 个 Skill
2026-03  持续快速发展中
```

---

## 12. 自己写 Skill 实战

### 12.1 实战一：GitHub Issue 管理

```markdown
---
name: gh-issues
description: 管理 GitHub Issue：创建、列表、搜索、关闭
---

# GitHub Issue 管理

## 创建 Issue
当用户要求创建 Issue 时：
1. 从消息中提取仓库名、标题、描述
2. 运行：`gh issue create --repo <repo> --title "<title>" --body "<body>"`
3. 返回创建的 Issue URL

## 列出 Issue
当用户要求查看 Issue 时：
1. 运行：`gh issue list --repo <repo> --state open --limit 20`
2. 格式化为清晰的列表返回

## 搜索 Issue
当用户搜索 Issue 时：
1. 运行：`gh issue list --repo <repo> --search "<query>"`
2. 返回匹配结果

## 关闭 Issue
当用户要求关闭 Issue 时：
1. 确认 Issue 编号
2. 运行：`gh issue close <number> --repo <repo>`
3. 确认已关闭
```

门控条件：

```yaml
metadata:
  openclaw:
    requires:
      bins: ["gh"]
      env: ["GITHUB_TOKEN"]
```

### 12.2 实战二：Obsidian 笔记

```markdown
---
name: obsidian-notes
description: 在 Obsidian vault 中创建和搜索笔记
---

# Obsidian 笔记管理

Obsidian vault 位于 ~/Documents/Obsidian/

## 创建笔记
1. 确定笔记标题和内容
2. 生成 Markdown 文件，包含 YAML frontmatter：
   ```
   ---
   created: <当前日期>
   tags: [<相关标签>]
   ---
   ```
3. 写入 ~/Documents/Obsidian/<标题>.md

## 搜索笔记
1. 使用 grep 在 vault 目录中搜索关键词
2. 返回匹配的笔记文件名和相关上下文

## 每日笔记
1. 创建 ~/Documents/Obsidian/Daily/<YYYY-MM-DD>.md
2. 包含日期模板和用户提供的内容
```

### 12.3 实战三：智能家居控制（Hue）

```markdown
---
name: hue-control
description: 控制 Philips Hue 智能灯光
---

# Philips Hue 控制

使用 Hue Bridge API 控制灯光。

## 查看所有灯
```
curl -s http://<bridge-ip>/api/<key>/lights | python3 -m json.tool
```

## 开/关灯
```
curl -s -X PUT http://<bridge-ip>/api/<key>/lights/<id>/state \
  -d '{"on": true}'
```

## 调节亮度
```
curl -s -X PUT http://<bridge-ip>/api/<key>/lights/<id>/state \
  -d '{"on": true, "bri": <0-254>}'
```

## 变色
```
curl -s -X PUT http://<bridge-ip>/api/<key>/lights/<id>/state \
  -d '{"on": true, "hue": <0-65535>, "sat": <0-254>}'
```
```

配合环境变量注入：

```json5
{
  skills: {
    entries: {
      "hue-control": {
        env: {
          HUE_BRIDGE_IP: "192.168.1.100",
          HUE_API_KEY: "your-api-key"
        }
      }
    }
  }
}
```

### 12.4 实战四：数据库查询

```markdown
---
name: db-query
description: 查询 PostgreSQL 数据库
disable-model-invocation: true
---

# 数据库查询工具

仅在用户明确使用 /db-query 时触发。

## 安全规则
- 只允许 SELECT 查询
- 禁止 DROP、DELETE、UPDATE、INSERT（除非用户明确要求）
- 查询前先确认 SQL 语句

## 执行查询
```bash
psql "$DATABASE_URL" -c "<SQL>"
```

## 查看表结构
```bash
psql "$DATABASE_URL" -c "\dt"           # 列出所有表
psql "$DATABASE_URL" -c "\d <table>"    # 查看表结构
```
```

---

## 13. Skill 安全注意事项

### 13.1 核心原则

> **"Treat third-party skills as untrusted code. Read them before enabling."**
>
> 将第三方 Skill 视为不受信的代码。启用前先阅读。

### 13.2 安全防护

| 防护 | 说明 |
|------|------|
| **路径校验** | Workspace/extraDirs 的发现机制会验证 realpath 包含关系 |
| **环境隔离** | `skills.entries.*.env` 和 `apiKey` 只注入主机进程，不进入沙箱 |
| **沙箱执行** | 对不受信的输入优先使用沙箱运行 |
| **Session 快照** | Skill 在会话开始时快照，运行中修改不立即生效 |

### 13.3 审查建议

安装第三方 Skill 前：
1. 阅读 SKILL.md 的全部内容
2. 检查是否有异常的 Shell 命令
3. 检查是否访问了不必要的文件/网络
4. 在沙箱模式下先测试
5. 检查 awesome-openclaw-skills 中的过滤说明（已移除 373 个恶意 Skill）

---

## 14. Plugin 系统

Plugin 是比 Skill 更大的扩展包，可以同时包含 Skill、渠道和配置。

### 14.1 Plugin 结构

```
my-plugin/
├── openclaw.plugin.json       ← 插件描述文件
├── skills/
│   ├── skill-a/SKILL.md
│   └── skill-b/SKILL.md
└── channels/                   ← 可选，自定义渠道
```

### 14.2 Plugin 中的 Skill

Plugin 的 `openclaw.plugin.json` 中声明 skills 目录：

```json
{
  "name": "my-plugin",
  "version": "1.0.0",
  "skills": ["skills"]
}
```

Plugin 中的 Skill 遵循标准的优先级规则，可以通过 `metadata.openclaw.requires.config` 进行门控。

---

## 15. 高级模式

### 15.1 直接工具分发 (Command Dispatch)

跳过 AI 模型推理，直接调用工具：

```yaml
---
name: quick-shell
description: 直接执行 Shell 命令
command-dispatch: tool
command-tool: exec
command-arg-mode: raw
---
```

使用 `/quick-shell ls -la` 直接执行，不经过模型处理。

### 15.2 Agent 自我编写 Skill

OpenClaw 的 Agent 可以自己创建和更新 Skill：

```
你：帮我写一个管理我的 Todoist 任务的 Skill

Agent：好的，我来创建一个 Todoist Skill...
（Agent 在 workspace/skills/ 中创建 SKILL.md）
```

Skill 观察器检测到新文件后自动加载，下次对话即可使用。

### 15.3 Skill 热重载

```json5
{
  skills: {
    // 默认开启，文件变更后自动刷新
    // 观察器检测 SKILL.md 变化 → 刷新快照
  }
}
```

开发 Skill 时无需重启 Gateway，保存文件即可。

### 15.4 跨节点 Skill

当 Linux Gateway 连接到 macOS 节点时：

```
Linux Gateway ←→ macOS Node (允许 system.run)
                     │
                     ▼
              macOS-only Skill
              (如需要 Homebrew 的 Skill)
```

macOS 专属 Skill 通过 `nodes` 工具在远程节点上执行。

### 15.5 安装配置声明

Skill 可以声明安装方式，让 Gateway 自动安装依赖：

```yaml
metadata:
  openclaw:
    install:
      - id: "brew"
        kind: "brew"
        formula: "ripgrep"
        bins: ["rg"]
        label: "通过 Homebrew 安装 ripgrep"
      - id: "node"
        kind: "node"
        package: "@anthropic-ai/sdk"
        bins: ["anthropic"]
      - id: "go"
        kind: "go"
        module: "github.com/user/tool@latest"
        bins: ["tool"]
      - id: "uv"
        kind: "uv"
        package: "my-python-tool"
        bins: ["my-tool"]
```

支持的安装方式：`brew`、`node`、`go`、`uv`、`download`。

---

## 16. 最佳实践

### 16.1 写好 description

description 是 Agent 判断何时使用 Skill 的核心依据：

```yaml
# 不好 - 太模糊
description: 工具

# 好 - 明确说明做什么、何时用
description: 查询 PostgreSQL 数据库，当用户需要查看数据、运行 SQL 或检查表结构时使用
```

### 16.2 SKILL.md 保持聚焦

- 一个 Skill 专注一件事
- 指令清晰、步骤明确
- 不超过 500 行（减少 token 开销）

### 16.3 善用门控

不必要的 Skill 不应加载：

```yaml
metadata:
  openclaw:
    requires:
      bins: ["docker"]    # 没有 docker 就不加载
    os: "linux"           # 只在 Linux 上加载
```

### 16.4 有副作用的操作禁止自动调用

```yaml
disable-model-invocation: true  # 部署、删除等危险操作
```

### 16.5 先测试再发布

1. 放到 `<workspace>/skills/` 目录
2. 在对话中测试各种使用场景
3. 确认无误后再发布到 ClawHub

### 16.6 安全第一

- 不在 SKILL.md 中硬编码密钥
- 使用 `skills.entries.*.env` 或 `apiKey` 注入敏感信息
- 危险操作加上确认步骤

---

## 17. 参考资源

### 官方文档

| 资源 | 链接 |
|------|------|
| Skill 官方文档 | https://docs.openclaw.ai/tools/skills |
| GitHub 主仓库 | https://github.com/openclaw/openclaw |
| Skill 源码 (docs) | https://github.com/openclaw/openclaw/blob/main/docs/tools/skills.md |
| ClawHub 注册中心 | https://clawhub.ai |
| ClawHub 源码 | https://github.com/openclaw/clawhub |

### 社区资源

| 资源 | 链接 |
|------|------|
| awesome-openclaw-skills | https://github.com/VoltAgent/awesome-openclaw-skills |
| DigitalOcean Skill 指南 | https://www.digitalocean.com/resources/articles/what-are-openclaw-skills |
| DataCamp ClawHub 最佳 Skill | https://www.datacamp.com/blog/best-clawhub-skills |
| FindSkill.ai Skill 完整指南 | https://findskill.ai/blog/openclaw-skills-guide/ |
| LumaDock 自定义 Skill 教程 | https://lumadock.com/tutorials/build-custom-openclaw-skills |

### AgentSkills 开放标准

OpenClaw 的 Skill 格式兼容 AgentSkills 开放标准，确保跨工具可移植性。

### Skill 模板

```yaml
---
name: my-skill
description: 简短描述功能和使用场景
user-invocable: true
disable-model-invocation: false
metadata:
  openclaw:
    requires:
      bins: []
      env: []
    os: null
    emoji: null
    primaryEnv: null
---

# Skill 名称

## 功能说明
描述 Skill 的功能...

## 使用步骤
1. 步骤一
2. 步骤二
3. 步骤三

## 注意事项
- 安全约束
- 边界条件
```
