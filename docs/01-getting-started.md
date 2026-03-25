# OpenClaw 入门使用文档

## 目录

- [1. 什么是 OpenClaw](#1-什么是-openclaw)
- [2. 系统要求](#2-系统要求)
- [3. 安装](#3-安装)
- [4. 首次配置与启动](#4-首次配置与启动)
- [5. 连接聊天渠道](#5-连接聊天渠道)
- [6. 基本使用](#6-基本使用)
- [7. Control UI 控制面板](#7-control-ui-控制面板)
- [8. 聊天命令](#8-聊天命令)
- [9. DM 安全与配对机制](#9-dm-安全与配对机制)
- [10. 模型配置](#10-模型配置)
- [11. 桌面与移动端](#11-桌面与移动端)
- [12. 常见问题排查](#12-常见问题排查)
- [13. 参考资源](#13-参考资源)

---

## 1. 什么是 OpenClaw

OpenClaw 是一个 **开源的个人 AI 助手平台**，由奥地利开发者 Peter Steinberger 于 2025 年 11 月创建（最初名为 Clawdbot，后更名为 OpenClaw）。

它的核心理念是：**"Your assistant. Your machine. Your rules."（你的助手，你的机器，你的规则。）**

### 与传统 AI 聊天机器人的区别

| 特性 | 传统 AI 聊天机器人 | OpenClaw |
|------|-------------------|----------|
| 运行位置 | 云端服务器 | **你自己的机器**（笔记本、HomeLab、VPS） |
| 交互方式 | 专用网页/App | **你日常使用的聊天应用**（WhatsApp、Telegram、Discord 等） |
| 能力范围 | 只能对话 | **真正执行任务**（文件操作、Shell 命令、浏览器控制、智能家居等） |
| 数据控制 | 数据在第三方 | **你掌控所有数据和 API Key** |
| 可扩展性 | 封闭 | **开源 + 社区 Skill 生态** |

### 核心能力

- **多渠道接入**：通过 WhatsApp、Telegram、Discord、Slack、iMessage 等 21+ 聊天平台与 AI 交互
- **本地执行**：在你的机器上执行 Shell 命令、读写文件、运行脚本
- **浏览器控制**：自动化网页操作、表单填写、数据提取
- **持久记忆**：跨会话保持上下文，24/7 记住你的偏好
- **多模型支持**：支持 Claude (Anthropic)、GPT (OpenAI)、Gemini (Google)、本地模型 (Ollama) 等 35+ 模型提供商
- **Skill 扩展**：社区构建的 13,000+ Skill，覆盖从 GitHub 到智能家居的各种场景
- **多 Agent**：在一个 Gateway 上运行多个隔离的 AI Agent
- **沙箱安全**：支持 Docker 沙箱隔离执行

---

## 2. 系统要求

| 项目 | 要求 |
|------|------|
| Node.js | **v24**（推荐）或 v22.16+ |
| 操作系统 | macOS、Linux、Windows（WSL2） |
| 网络 | 需要联网（调用 AI 模型 API） |
| API Key | 来自 Anthropic / OpenAI / Google 等模型提供商 |

验证 Node 版本：

```bash
node --version
```

> Windows 用户建议使用 WSL2 以获得最佳稳定性。

---

## 3. 安装

### 3.1 一键安装（推荐）

**macOS / Linux：**

```bash
curl -fsSL https://openclaw.ai/install.sh | bash
```

**Windows (PowerShell)：**

```powershell
iwr -useb https://openclaw.ai/install.ps1 | iex
```

### 3.2 npm 安装

```bash
npm install -g openclaw@latest
```

### 3.3 从源码安装（适合开发者）

```bash
git clone https://github.com/openclaw/openclaw.git
cd openclaw
pnpm install
pnpm ui:build
pnpm build
pnpm openclaw onboard --install-daemon
```

### 3.4 其他方式

Docker 和 Nix 安装方式也可用，详见官方安装文档。

---

## 4. 首次配置与启动

### 4.1 运行引导向导（约 2 分钟）

```bash
openclaw onboard --install-daemon
```

引导向导会帮你完成：
- 选择 AI 模型提供商（Anthropic / OpenAI / Google 等）
- 配置 API Key
- 安装 Gateway 守护进程（launchd/systemd 服务）
- 创建工作区
- 连接聊天渠道

### 4.2 验证 Gateway 状态

```bash
openclaw gateway status
```

Gateway 应该在端口 `18789` 上监听。

### 4.3 打开控制面板

```bash
openclaw dashboard
```

这会在浏览器中打开 Control UI，你可以在里面直接与 AI 对话。

### 4.4 发送第一条消息

在 Control UI 的聊天界面中输入任意消息，比如：

```
你好！你能做什么？
```

恭喜，OpenClaw 已经运行起来了！

---

## 5. 连接聊天渠道

OpenClaw 支持 21+ 聊天渠道，以下是最常用的几个：

### 5.1 Telegram（最快上手）

1. 在 Telegram 中找 @BotFather 创建一个 Bot，获取 Bot Token
2. 在 `~/.openclaw/openclaw.json` 中配置：

```json5
{
  channels: {
    telegram: {
      enabled: true,
      botToken: "你的Bot Token",
      dmPolicy: "pairing",     // 需要配对才能使用
      allowFrom: []             // 或填入允许的用户 ID
    }
  }
}
```

3. 重启 Gateway 或等待热重载

### 5.2 WhatsApp

使用 Baileys 协议连接，通过二维码扫描配对。按引导向导步骤操作即可。

### 5.3 Discord

1. 在 Discord Developer Portal 创建 Bot 并获取 Token
2. 配置 `channels.discord.botToken`
3. 邀请 Bot 到你的服务器

### 5.4 其他支持的渠道

Slack、Signal、iMessage (BlueBubbles)、Google Chat、Microsoft Teams、Matrix、IRC、Mattermost、Feishu（飞书）、LINE、Nostr、Twitch 等。

> 同一个 Gateway 可以**同时连接多个渠道**，消息互通。

---

## 6. 基本使用

### 6.1 通过聊天应用使用

连接渠道后，直接在你的聊天应用中给 Bot 发消息即可：

```
帮我写一个 Python 脚本，从 CSV 文件中提取邮箱地址
```

```
查看一下 ~/projects 目录下有什么文件
```

```
打开浏览器搜索最新的 Node.js 文档
```

OpenClaw 不仅会告诉你怎么做，它会**直接在你的机器上执行**。

### 6.2 通过 Control UI 使用

```bash
openclaw dashboard
```

浏览器打开的 Control UI 提供：
- 聊天界面
- 会话管理
- 配置编辑
- 日志查看

### 6.3 通过 CLI 使用

```bash
openclaw chat "你好"      # 发送消息
openclaw status            # 查看状态
openclaw doctor            # 诊断问题
```

### 6.4 群聊中使用

在群聊中，默认需要 @提及 Bot 才会触发回复：

```
@openclaw 帮我总结一下上面的讨论
```

可以通过配置改为 `activation: always` 让 Bot 监听所有消息。

---

## 7. Control UI 控制面板

Control UI 是 OpenClaw 的浏览器管理界面，由 Gateway 直接提供服务。

### 启动方式

```bash
openclaw dashboard
```

### 功能

- **聊天界面**：直接与 AI 对话，支持图片上传
- **会话管理**：查看和切换不同聊天会话
- **配置管理**：可视化编辑 `openclaw.json` 配置
- **Skill 管理**：浏览、安装、启用/禁用 Skill
- **日志监控**：实时查看 Gateway 日志
- **渠道状态**：监控各聊天渠道的连接状态

### 远程访问

默认只绑定 localhost。如需远程访问，可通过以下方式：

- **Tailscale**：`gateway.tailscale.mode` 配置 tailnet 或 public 访问
- **SSH 隧道**：`ssh -L 18789:localhost:18789 your-server`

---

## 8. 聊天命令

在 WhatsApp、Telegram、Slack、Discord 等聊天应用中可直接使用以下命令：

| 命令 | 说明 |
|------|------|
| `/status` | 查看会话状态（token 用量、费用） |
| `/new` 或 `/reset` | 重置会话，开始新对话 |
| `/compact` | 压缩上下文（保留摘要，释放空间） |
| `/think <级别>` | 控制推理深度（off / low / medium / high / xhigh） |
| `/verbose on\|off` | 切换详细输出模式 |
| `/usage off\|tokens\|full` | 控制用量信息显示 |
| `/model <模型名>` | 切换 AI 模型 |
| `/restart` | 重启 Gateway（仅群主可用） |
| `/activation mention\|always` | 群聊激活模式 |

---

## 9. DM 安全与配对机制

OpenClaw 默认**不允许陌生人直接使用你的 Bot**，通过 DM Policy 控制访问：

### 9.1 配对模式（默认，推荐）

```
dmPolicy: "pairing"
```

工作流程：
1. 陌生人给 Bot 发消息
2. Bot 返回一个配对码
3. 你在本机批准配对：

```bash
openclaw pairing approve <渠道> <配对码>
```

4. 该用户被加入白名单，之后可以正常使用

### 9.2 白名单模式

```
dmPolicy: "allowlist"
```

只有 `allowFrom` 列表中的用户可以使用。

### 9.3 开放模式（慎用）

```
dmPolicy: "open"
```

任何人都可以使用。需要在 `allowFrom` 中显式添加 `"*"`。

### 9.4 安全诊断

```bash
openclaw doctor              # 基本检查
openclaw security audit      # 安全审计
openclaw security audit --deep  # 深度审计（检测实时 Gateway）
```

---

## 10. 模型配置

### 10.1 模型格式

模型使用 `提供商/模型名` 格式：

```
anthropic/claude-sonnet-4-6
openai/gpt-4.1
google/gemini-2.5-pro
```

### 10.2 配置主模型和备用模型

在 `~/.openclaw/openclaw.json` 中：

```json5
{
  agents: {
    defaults: {
      model: {
        primary: "anthropic/claude-sonnet-4-6",
        fallbacks: ["openai/gpt-4.1"]
      }
    }
  }
}
```

### 10.3 支持的提供商

包括 Anthropic、OpenAI、Google、以及本地模型方案（Ollama、vLLM、SGLang）等 35+ 提供商。

### 10.4 切换模型

在聊天中使用 `/model` 命令，或在配置中修改 `agents.defaults.models` 对象。

---

## 11. 桌面与移动端

### 11.1 macOS 菜单栏应用（Beta）

提供：
- 菜单栏快速访问
- 语音唤醒（Voice Wake）
- Push-to-Talk 悬浮窗
- WebChat 内嵌
- 调试工具

### 11.2 iOS Node

通过设备配对连接：
- Canvas 画布
- 语音唤醒和对话模式
- 摄像头和屏幕录制
- 推送通知

### 11.3 Android Node

支持：
- Connect / Chat / Voice 标签页
- Canvas 画布
- 摄像头和屏幕截图
- Android 设备命令（通知、定位、短信、相册、联系人、日历等）

---

## 12. 常见问题排查

### 12.1 诊断工具

```bash
openclaw doctor           # 诊断配置问题
openclaw doctor --fix     # 自动修复
openclaw gateway status   # 检查 Gateway 状态
```

### 12.2 常见问题

**Gateway 启动失败：**
- 检查端口 18789 是否被占用
- 检查 Node.js 版本是否满足要求
- 运行 `openclaw doctor` 查看具体错误

**聊天渠道连接不上：**
- 确认 Bot Token / 凭证是否正确
- 检查网络连接
- 查看 Gateway 日志：`/tmp/openclaw/openclaw-YYYY-MM-DD.log`

**Bot 不回复消息：**
- 检查 DM Policy 是否允许该用户
- 群聊中是否需要 @提及
- 检查 API Key 是否有效

### 12.3 更新版本

```bash
# 稳定版
openclaw update --channel stable

# 测试版
openclaw update --channel beta

# 开发版
openclaw update --channel dev
```

---

## 13. 参考资源

### 官方资源

| 资源 | 链接 |
|------|------|
| 官网 | https://openclaw.ai |
| 官方文档 | https://docs.openclaw.ai |
| GitHub 仓库 | https://github.com/openclaw/openclaw |
| ClawHub（Skill 市场） | https://clawhub.ai |
| 博客 | https://openclaw.ai/blog |

### 文档导航

| 主题 | 链接 |
|------|------|
| 入门指南 | https://docs.openclaw.ai/start/getting-started |
| 引导向导 | https://docs.openclaw.ai/start/wizard |
| Gateway 配置 | https://docs.openclaw.ai/gateway/configuration |
| 安全文档 | https://docs.openclaw.ai/gateway/security |
| 远程访问 | https://docs.openclaw.ai/gateway/remote |
| Skill 系统 | https://docs.openclaw.ai/tools/skills |
| 多 Agent | https://docs.openclaw.ai/concepts/multi-agent |
| 功能列表 | https://docs.openclaw.ai/concepts/features |
| 故障排查 | https://docs.openclaw.ai/gateway/troubleshooting |

### 社区

| 资源 | 说明 |
|------|------|
| Discord | OpenClaw 官方社区 |
| GitHub Discussions | 问题讨论和功能请求 |
| awesome-openclaw-skills | 社区精选 5,200+ Skill 列表 |
