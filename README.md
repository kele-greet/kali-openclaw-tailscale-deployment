# kali-openclaw-tailscale-deployment
在 Kali Linux 上安全部署 OpenClaw AI 网关，对接 DeepSeek API，并通过 Tailscale 实现远程访问的完整教程|A complete tutorial for securely deploying OpenClaw AI gateway on Kali Linux, integrating DeepSeek API, and enabling remote access via Tailscale在 Kali Linux 上安全部署 OpenClaw AI 网关，对接 DeepSeek API，并通过 Tailscale 实现远程访问的完整教程|在Kali Linux上安全部署OpenClaw AI网关、集成DeepSeek API并通过Tailscale实现远程访问的完整教程在 Kali Linux 上安全部署 OpenClaw AI 网关，对接 DeepSeek API，并通过 Tailscale 实现远程访问的完整教程|在Kali Linux上安全部署OpenClaw AI网关、集成DeepSeek API并通过Tailscale实现远程访问的完整教程在 Kali Linux 上安全部署 OpenClaw AI 网关，对接 DeepSeek API，并通过 Tailscale 实现远程访问的完整教程|在Kali Linux上安全部署OpenClaw AI网关、集成DeepSeek API并通过Tailscale实现远程访问的完整教程


以下是根据您提供的原始内容，经过重新整理、修正序号错误、补充遗漏步骤、优化逻辑顺序并增强可读性的完整教程。内容保持原有技术要点与安全原则，仅调整结构与表达方式，便于从零开始按步骤执行。

---

# 在 Kali Linux 上安全部署 OpenClaw 并通过 Tailscale 远程访问完整教程

## 📖 教程总览

本教程将指导您在 Kali Linux 上安全部署 **OpenClaw** AI 智能体网关，对接 **DeepSeek API**（成本低、兼容 OpenAI 接口），并通过 **Tailscale** 零配置 VPN 实现远程安全访问。最终您可以在任意设备的浏览器中直接操控 OpenClaw，同时使用 **Termius** SSH 客户端与 SSH 隧道进行安全远程管理。

整体架构分为五个阶段：

1. **系统安全加固** – 防火墙、SSH 加固、普通用户创建  
2. **OpenClaw 安装与配置** – 对接 DeepSeek API  
3. **Tailscale 组网** – 虚拟私有网络  
4. **Termius SSH 配置** – 远程连接管理  
5. **浏览器远程访问** – 通过 Tailscale 或 SSH 隧道操控 OpenClaw  

---

## 🛡️ 第一阶段：Kali Linux 系统安全加固

### 1.1 系统更新与基础工具安装

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install curl wget git ufw openssh-server -y
```

### 1.2 创建普通用户并授予 sudo 权限

Kali 默认使用 root 用户，需改为普通用户操作。

```bash
# 创建新用户（将 yourusername 替换为自定义名称）
sudo adduser yourusername

# 加入 sudo 组
sudo usermod -aG sudo yourusername

# 切换到新用户测试
su - yourusername
sudo whoami   # 应输出 root
```

### 1.3 配置 UFW 防火墙

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing

# 放行 SSH 端口（先放行默认 22，若后续修改端口则需调整）
sudo ufw allow 22/tcp

# 注意：不开放 18789 端口到公网，远程访问通过 Tailscale 或 SSH 隧道

sudo ufw enable
sudo ufw status verbose
```

### 1.4 SSH 服务安全加固

**⚠️ 重要**：禁用密码认证前，必须确保已将自己的 SSH 公钥添加到服务器的 `~/.ssh/authorized_keys`，否则会被锁在门外。

```bash
# 备份配置
sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak

# 编辑配置
sudo nano /etc/ssh/sshd_config
```

修改以下参数：

```ini
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes

# 可选：修改端口（例：2222）
Port 2222

# 可选：仅允许特定用户
AllowUsers yourusername
```

**生成并上传 SSH 公钥**（在本地电脑执行）：

```bash
ssh-keygen -t ed25519 -C "your_email@example.com"
ssh-copy-id -i ~/.ssh/id_ed25519.pub yourusername@kali_server_ip
```

**重启 SSH 服务**：

```bash
sudo sshd -t                # 检查语法
sudo systemctl restart sshd
```

> 💡 修改端口后，记得更新 UFW 规则：`sudo ufw allow 2222/tcp` 并删除旧的 22 规则。

---

## 📦 第二阶段：OpenClaw 安装与对接 DeepSeek API

### 2.1 获取 DeepSeek API Key

1. 访问 [DeepSeek 开放平台](https://platform.deepseek.com/) 注册/登录  
2. 进入 **API Keys** → **创建 API Key** → 填写名称（如 `OpenClaw-Kali`）  
3. 复制生成的密钥（格式 `sk-xxxxxxxx`）并妥善保存  

> DeepSeek 提供两种模型：  
>
> - `deepseek-chat`：通用对话（DeepSeek-V3.2）  
> - `deepseek-reasoner`：深度推理（适合复杂逻辑）

### 2.2 安装 Node.js（≥22）

```bash
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt install -y nodejs
node --version   # 应显示 v22.x.x
```

### 2.3 安装 OpenClaw

```bash
sudo npm install -g openclaw@latest
openclaw --version
```

### 2.4 运行初始化向导（对接 DeepSeek）

```bash
openclaw onboard
```

按提示回答以下关键问题：

| 配置项           | 推荐值                                                   |
| ---------------- | -------------------------------------------------------- |
| **网关模式**     | `local gateway`                                          |
| **身份验证方式** | `deepseek-api-key`                                       |
| **API Key**      | 粘贴 `sk-xxxxxxxx`                                       |
| **默认模型**     | `deepseek/deepseek-chat` 或 `deepseek/deepseek-reasoner` |
| **网关令牌**     | 自动生成，记下备用                                       |
| **消息平台**     | 暂时跳过（Web UI 足够）                                  |
| **Skills 安装**  | 按需勾选基础技能                                         |

> 若向导未直接显示 DeepSeek，选择 **OpenAI** 或 **Custom (OpenAI Compatible)** 并手动填写：  
>
> - API Key：`sk-xxxxxxxx`  
> - Base URL：`https://api.deepseek.com`（末尾不加 `/v1`）  
> - Model：`deepseek-chat` 或 `deepseek-reasoner`

### 2.5 非交互式配置（可选）

```bash
export DEEPSEEK_API_KEY="你的API密钥"
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice deepseek-api-key \
  --deepseek-api-key "$DEEPSEEK_API_KEY" \
  --skip-health --accept-risk
```

### 2.6 检查并编辑配置文件（可选）

配置文件位于 `~/.openclaw/openclaw.json`，可手动调整模型参数、令牌等。示例结构如下（关键部分）：

```json
{
  "meta": {
    "lastTouchedVersion": "2026.3.13",
    "lastTouchedAt": "2026-03-24T08:24:21.981Z"
  },
  "wizard": {
    "lastRunAt": "2026-03-23T09:37:08.490Z",
    "lastRunVersion": "2026.3.13",
    "lastRunCommand": "doctor",
    "lastRunMode": "local"
  },
  "models": {
    "mode": "merge",
    "providers": {
      "deepseek": {
        "baseUrl": "https://api.deepseek.com",
        "apiKey": "${DEEPSEEK_API_KEY}",
        "api": "openai-completions",
        "models": [
          {
            "id": "deepseek-chat",
            "name": "DeepSeek Chat (V3.2)",
            "reasoning": false,
            "input": [
              "text"
            ],
            "cost": {
              "input": 2.8e-7,
              "output": 4.2e-7,
              "cacheRead": 2.8e-8,
              "cacheWrite": 2.8e-7
            },
            "contextWindow": 128000,
            "maxTokens": 8192
          },
          {
            "id": "deepseek-reasoner",
            "name": "DeepSeek Reasoner (V3.2)",
            "reasoning": true,
            "input": [
              "text"
            ],
            "cost": {
              "input": 2.8e-7,
              "output": 4.2e-7,
              "cacheRead": 2.8e-8,
              "cacheWrite": 2.8e-7
            },
            "contextWindow": 128000,
            "maxTokens": 65536
          }
        ]
      }
    }
  },
  "agents": {
    "defaults": {
      "models": {
        "deepseek/deepseek-chat": {},
        "deepseek/deepseek-reasoner": {}
      }
    }
  },
  "commands": {
    "native": "auto",
    "nativeSkills": "auto",
    "restart": true,
    "ownerDisplay": "raw"
  },
  "gateway": {
    "mode": "local",
    "auth": {
      "mode": "token",
      "token": "环境变量或者硬编码都行"
    }
  }
}

```

### 2.7 启动网关并验证

```bash
# 启动网关（前台运行，测试用）
openclaw gateway run
```

打开浏览器访问 `http://127.0.0.1:18789`，输入 Gateway Token 登录。

**验证 DeepSeek 连通性**（在聊天界面输入）：

```
你好，请确认你当前的模型身份。
```

若回复中提到 DeepSeek，则配置成功。

**后台运行**：使用 `openclaw gateway start` 或 systemd 方式（可自行配置）。

### 2.8 获取 Gateway Token（用于远程访问）

```bash
cat ~/.openclaw/openclaw.json | grep token
```

保存输出的 `ocw_xxxxxx` 格式字符串。

---

## 🌐 第三阶段：Tailscale 组网配置

### 3.1 在 Kali 上安装 Tailscale

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo systemctl enable --now tailscaled
```

### 3.2 登录 Tailscale

```bash
sudo tailscale up
```

按提示打开浏览器完成登录（使用 Google/Microsoft/GitHub 账号）。

### 3.3 验证 Tailscale 状态

```bash
tailscale ip        # 显示 100.x.x.x 格式的 Tailscale IP
tailscale status    # 查看连接设备
```

### 3.4 在其他设备上安装 Tailscale

- **Windows/macOS**：从 [官网](https://tailscale.com/download) 下载  
- **iOS/Android**：App Store / Google Play  
- **所有设备必须使用同一个 Tailscale 账号登录**

### 3.5 配置 OpenClaw 绑定 Tailscale 网络

```bash
openclaw config set gateway.bind tailnet
openclaw gateway restart
```

---

## 📱 第四阶段：Termius SSH 远程连接配置

### 4.1 安装 Termius

- 桌面版：[termius.com/download](https://termius.com/download)  
- 移动版：应用商店搜索 “Termius”  

使用同一账号登录以同步配置。

### 4.2 添加 SSH 主机

在 Termius 中点击 **+ New Host**，填写：

| 字段               | 值                                             |
| ------------------ | ---------------------------------------------- |
| **Label**          | `Kali-OpenClaw`                                |
| **Hostname**       | Kali 的 Tailscale IP（如 `100.82.45.123`）     |
| **Port**           | 你设置的 SSH 端口（默认 22 或自定义如 `2222`） |
| **Username**       | 你创建的普通用户名（如 `yourusername`）        |
| **Authentication** | SSH Key → 选择你的私钥文件                     |

### 4.3 测试连接

点击 **Connect**，成功登录即代表配置正确。

---

## 🔗 第五阶段：浏览器远程访问与操控 OpenClaw

### 5.1 获取 Kali 的 Tailscale IP

```bash
tailscale ip   # 例如 100.82.45.123
```

### 5.2 方法一：Tailscale Serve（推荐）

在 Kali 上执行：

```bash
sudo tailscale serve reset
sudo tailscale serve --bg --https=443 http://127.0.0.1:18789
tailscale serve status   # 显示 https://机器名.ts.net/ 等地址
```

现在，任何连接同一 Tailnet 的设备均可通过该 HTTPS 地址访问 OpenClaw。

**首次访问配对**：

- 远程浏览器访问时可能提示 `1008: pairing required`  
- 此时在 Kali 本地打开 `http://127.0.0.1:18789`  
- 进入 **Devices → Pairing → Pending Requests**，点击 **Approve**  
- 刷新远程页面即可正常使用

### 5.3 方法二：SSH 隧道（备选）

此方法适合不想用 Tailscale Serve 或需要更传统方式的场景。

**通过 Termius 配置端口转发**：

1. 编辑 Kali 主机配置  
2. 找到 **Port Forwarding**（或 **SSH Tunnels**）  
3. 添加规则：  
   - Type: `Local`  
   - Local Address: `127.0.0.1`  
   - Local Port: `18789`  
   - Remote Host: `127.0.0.1`  
   - Remote Port: `18789`  
4. 保存并重新连接

**或使用命令行**：

```bash
ssh -N -L 18789:127.0.0.1:18789 yourusername@<Kali的TailscaleIP> -p <SSH端口>
```

保持隧道运行，然后在本地浏览器访问 `http://localhost:18789`。

### 5.4 登录 OpenClaw Web UI

输入之前保存的 Gateway Token（`ocw_xxxxxx`），成功登录后即可使用聊天、配置、设备管理等功能。

### 5.5 功能测试：浏览器控制

在聊天界面输入：

```
打开浏览器访问 https://www.baidu.com 并截图
```

OpenClaw 将调用 DeepSeek API 理解指令，自动打开浏览器并返回截图，表示全链路正常工作。

### 5.6 高级选项：Chrome Relay 扩展（操控本地浏览器）

若希望 AI 操控你日常使用的浏览器，可在 Kali 上生成控制 token：

```bash
openclaw browser serve
```

然后在本地 Chrome 中安装 OpenClaw 扩展，填入 Kali 的 Tailscale IP 和 token 即可。

---

## 🛠️ 故障排查

### OpenClaw 网关未运行

```bash
openclaw gateway status
openclaw gateway restart
```

### DeepSeek API 401 错误（密钥无效）

```bash
cat ~/.openclaw/openclaw.json | grep apiKey
openclaw config set deepseek.apiKey "sk-新密钥"
openclaw gateway restart
openclaw doctor   # 检查连通性
```

### DeepSeek 余额不足

登录 [DeepSeek 平台](https://platform.deepseek.com/) → **充值**（建议首次 10-20 元测试）。

### Tailscale 连接问题

```bash
tailscale status
tailscale ping <其他设备 Tailscale IP>
```

### SSH 隧道失败

```bash
sudo systemctl status sshd
sudo ufw status   # 确认端口已放行
```

### Web UI 显示“origin not allowed”

编辑 `~/.openclaw/openclaw.json`，添加：

```json
{
  "gateway": {
    "controlUi": {
      "allowedOrigins": ["http://localhost:18789", "https://你的tailscale域名.ts.net"]
    }
  }
}
```

重启网关：`openclaw gateway restart`

### 配对请求未出现

在 Kali 本地访问 `http://127.0.0.1:18789` → **Devices → Pending Requests** 手动批准。

---

## 📋 安全最佳实践总结

| 措施                                           | 目的           |
| ---------------------------------------------- | -------------- |
| 禁用 root SSH 登录                             | 防止暴力破解   |
| 禁用 SSH 密码认证（仅用密钥）                  | 防止密码泄露   |
| 修改 SSH 默认端口                              | 减少自动化扫描 |
| UFW 仅放行必要端口（SSH + Tailscale）          | 最小化暴露面   |
| 所有远程访问经 Tailscale 加密隧道              | 流量安全       |
| 不将 18789 端口暴露到公网                      | 防止未授权访问 |
| DeepSeek API Key 使用环境变量或配置文件        | 避免硬编码泄露 |
| 定期执行 `sudo apt update && sudo apt upgrade` | 修补安全漏洞   |

---

## 📚 参考资源

- [OpenClaw 官方文档](https://docs.openclaw.ai)  
- [OpenClaw DeepSeek 提供方文档](https://docs.openclaw.ai/providers/deepseek)  
- [DeepSeek 开放平台](https://platform.deepseek.com/)  
- [Tailscale 文档](https://tailscale.com/docs)  
- [Kali Linux 文档](https://www.kali.org/docs/)  
- [Termius 支持](https://support.termius.com)

---

**整理完毕**。本教程保留了原技术方案的所有关键步骤，修正了序号重复、命令顺序和防火墙规则等细节，并增强了可读性。您可以直接按章节顺序执行。
