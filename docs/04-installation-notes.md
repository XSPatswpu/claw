# OpenClaw 安装与问题排查记录

## 目录

- [1. 环境信息](#1-环境信息)
- [2. 安装步骤](#2-安装步骤)
- [3. 首次配置](#3-首次配置)
- [4. 遇到的问题与解决](#4-遇到的问题与解决)
  - [4.1 Telegram 连接失败：SSL wrong version number](#41-telegram-连接失败ssl-wrong-version-number)
  - [4.2 Systemd Service 硬编码代理变量](#42-systemd-service-硬编码代理变量)
  - [4.3 Telegram 配对流程](#43-telegram-配对流程)
- [5. Clash TUN 模式与 HTTP 代理冲突详解](#5-clash-tun-模式与-http-代理冲突详解)
- [6. 修改记录](#6-修改记录)

---

## 1. 环境信息

| 项目 | 值 |
|------|-----|
| OS | Ubuntu, Linux 6.14.0-34-generic |
| Node.js | v22.22.1 |
| OpenClaw | 2026.3.23-2 (7ffe7e4) |
| 安装方式 | npm 全局安装 |
| 安装路径 | `/usr/bin/openclaw` → `/usr/lib/node_modules/openclaw/openclaw.mjs` |
| 代理工具 | Clash (mihomo) TUN 模式 |
| Clash 路径 | `/opt/clash/` (通过 [clash-for-linux-install](https://github.com/nelvko/clash-for-linux-install) 安装) |

---

## 2. 安装步骤

```bash
# 1. 全局安装 openclaw
npm install -g openclaw

# 2. 首次引导配置
openclaw onboard

# 3. 启动 gateway
openclaw gateway

# 4. 查看状态
openclaw gateway status
```

---

## 3. 首次配置

配置完成后生成的配置文件位于 `~/.openclaw/openclaw.json`，关键配置项：

- **认证**: Anthropic API Token
- **模型**: `anthropic/claude-opus-4-6`
- **Telegram**: 启用，使用 polling 模式，`dmPolicy: "pairing"`
- **Gateway**: 端口 18789，绑定 loopback

---

## 4. 遇到的问题与解决

### 4.1 Telegram 连接失败：SSL wrong version number

**现象**：Gateway 启动后，所有 Telegram API 调用失败，日志持续报错：

```
Telegram webhook cleanup failed: Network request for 'deleteWebhook' failed!; retrying in 30s.
telegram setMyCommands failed: Network request for 'setMyCommands' failed!
```

**排查过程**：

1. 用 `curl` 测试 `https://api.telegram.org`，报 SSL 错误：
   ```
   OpenSSL/3.0.13: error:0A00010B:SSL routines::wrong version number
   ```

2. 用 raw TCP 连接抓取实际返回内容，发现 443 端口返回的是 HTTP 明文：
   ```
   HTTP/1.1 407 OK
   ```
   这是代理认证失败的响应，而非 TLS 握手数据。

3. 测试 Google 等其他被墙网站，均正常。说明代理节点本身可用，但 Telegram 走的节点有问题。

**根因**：Clash TUN 模式 + HTTP 代理环境变量同时生效，导致流量被双重代理处理，加上当时 Telegram 规则走的代理节点异常。

**解决**：更换订阅配置 + 清除代理环境变量（详见 [4.2](#42-systemd-service-硬编码代理变量) 和 [第5节](#5-clash-tun-模式与-http-代理冲突详解)）。

---

### 4.2 Systemd Service 硬编码代理变量

**现象**：即使在 shell 中清除了代理环境变量，重启 gateway 后 Telegram 仍然连不上。

**根因**：OpenClaw 安装时会生成 systemd service 文件 `~/.config/systemd/user/openclaw-gateway.service`，其中硬编码了安装时环境中的代理变量：

```ini
# 这些变量在 TUN 模式下不需要，会导致双重代理
Environment=HTTP_PROXY=http://127.0.0.1:7890
Environment=HTTPS_PROXY=http://127.0.0.1:7890
Environment=ALL_PROXY=socks5h://127.0.0.1:7890
Environment=http_proxy=http://127.0.0.1:7890
Environment=https_proxy=http://127.0.0.1:7890
Environment=all_proxy=socks5h://127.0.0.1:7890
Environment=NO_PROXY=localhost,127.0.0.1,::1
Environment=no_proxy=localhost,127.0.0.1,::1
```

**解决**：删除 service 文件中的所有代理相关 `Environment` 行。TUN 模式在网络层劫持流量，无需应用层代理变量。

---

### 4.3 Telegram 配对流程

**现象**：Telegram 发送消息给 bot 后，提示 "access not configured"，需要配对。

**原因**：配置中 `dmPolicy` 设为 `"pairing"`，新用户需要管理员授权。

**解决**：Bot 会返回配对码，在终端执行：

```bash
openclaw pairing approve telegram <配对码>
```

---

## 5. Clash TUN 模式与 HTTP 代理冲突详解

### 背景

本机使用 [clash-for-linux-install](https://github.com/nelvko/clash-for-linux-install) 安装的 Clash (mihomo)，同时开启了：

- **TUN 模式**：在内核层创建虚拟网卡 `Meta`，通过 fake-ip (`198.18.0.0/16`) 劫持所有流量
- **HTTP 代理环境变量**：通过 shell 脚本 `watch_proxy` → `clashon` → `_set_system_proxy` 设置

### 冲突机制

```
正常 TUN 模式流量路径：
  应用 → DNS(fake-ip) → TUN网卡 → Clash内核 → 代理节点 → 目标

双重代理时的流量路径：
  应用 → 读取 HTTP_PROXY → Clash HTTP代理(7890) → TUN网卡再次截获 → 异常
```

当应用（如 Node.js）检测到 `HTTP_PROXY` 环境变量时，会将请求发给 Clash 的 HTTP 代理端口 (7890)，而这个出去的流量又被 TUN 截获，导致流量被处理两次，SSL 握手异常。

### 为什么 shell 脚本会设置代理变量

`~/.bashrc` 中加载了 Clash 管理脚本：

```bash
source /opt/clash/script/common.sh && source /opt/clash/script/clashctl.sh && watch_proxy
```

`watch_proxy` 在每个新交互式 shell 启动时检测是否设置了 `http_proxy`，如果没有就调用 `clashon`，而 `clashon` 会无条件调用 `_set_system_proxy` 设置代理变量。**这个脚本没有检测 TUN 模式是否开启。**

### 修复

修改 `/opt/clash/script/clashctl.sh` 中的 `clashon` 函数，在设置代理变量前检测 TUN 状态：

```bash
function clashon() {
    _get_proxy_port
    systemctl is-active "$BIN_KERNEL_NAME" >&/dev/null || {
        sudo systemctl start "$BIN_KERNEL_NAME" >/dev/null || {
            _failcat '启动失败: 执行 clashstatus 查看日志'
            return 1
        }
    }
    # TUN 模式下不需要设置代理环境变量，避免双重代理
    if _tunstatus >/dev/null 2>&1; then
        _unset_system_proxy
    else
        clashproxy status >/dev/null && _set_system_proxy
    fi
    _okcat '已开启代理环境'
}
```

---

## 6. 修改记录

| 文件 | 修改内容 |
|------|---------|
| `/opt/clash/script/clashctl.sh` | `clashon` 函数增加 TUN 检测，TUN 开启时不设置代理环境变量 |
| `~/.config/systemd/user/openclaw-gateway.service` | 删除所有 `*PROXY*` 相关的 Environment 行 |

### 注意事项

- OpenClaw 安装/重装时可能会重新生成 service 文件并写入当前环境变量，如果当时 shell 中有代理变量，需要再次清理
- 更换订阅配置使用 `clashupdate <订阅URL>`
- 如果 Telegram 不通，优先检查 Clash 节点是否正常，以及进程环境变量是否干净
