Here is the complete English translation of your tutorial, ready to be used as `README.md` for your GitHub repository.

---

# Complete Tutorial: Securely Deploy OpenClaw on Kali Linux and Access Remotely via Tailscale

## 📖 Tutorial Overview

This tutorial will guide you through securely deploying the **OpenClaw** AI agent gateway on Kali Linux, integrating **DeepSeek API** (low-cost, OpenAI-compatible interface), and enabling secure remote access via **Tailscale** zero-config VPN. Ultimately, you will be able to directly control OpenClaw from any device's browser, while using **Termius** SSH client and SSH tunneling for secure remote management.

The architecture is divided into five phases:

1. **System Hardening** – Firewall, SSH hardening, creating a regular user  
2. **OpenClaw Installation & Configuration** – Integrating DeepSeek API  
3. **Tailscale Networking** – Virtual private network  
4. **Termius SSH Configuration** – Remote connection management  
5. **Browser Remote Access** – Controlling OpenClaw via Tailscale or SSH tunnel  

---

## 🛡️ Phase 1: Kali Linux System Hardening

### 1.1 System Update & Basic Tools

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install curl wget git ufw openssh-server -y
```

### 1.2 Create a Regular User and Grant sudo Privileges

Kali uses root by default – you should switch to a regular user.

```bash
# Create a new user (replace 'yourusername' with your chosen name)
sudo adduser yourusername

# Add user to sudo group
sudo usermod -aG sudo yourusername

# Switch to the new user and test sudo
su - yourusername
sudo whoami   # Should output "root"
```

### 1.3 Configure UFW Firewall

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Allow SSH port (default 22, adjust later if you change the port)
sudo ufw allow 22/tcp

# Note: Port 18789 is NOT exposed to the public internet; remote access will be via Tailscale or SSH tunnel

sudo ufw enable
sudo ufw status verbose
```

### 1.4 Harden SSH Service

**⚠️ Important**: Before disabling password authentication, make sure you have added your SSH public key to the server's `~/.ssh/authorized_keys` – otherwise you will lock yourself out.

```bash
# Backup original SSH config
sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak

# Edit configuration
sudo nano /etc/ssh/sshd_config
```

Modify the following parameters:

```ini
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes

# Optional: change the port (e.g., 2222)
Port 2222

# Optional: allow only specific users
AllowUsers yourusername
```

**Generate and upload your SSH public key** (run on your local machine):

```bash
ssh-keygen -t ed25519 -C "your_email@example.com"
ssh-copy-id -i ~/.ssh/id_ed25519.pub yourusername@kali_server_ip
```

**Restart SSH service**:

```bash
sudo sshd -t                # Check syntax
sudo systemctl restart sshd
```

> 💡 If you change the SSH port, remember to update UFW rules: `sudo ufw allow 2222/tcp` and remove the old rule for port 22.

---

## 📦 Phase 2: Install OpenClaw and Integrate DeepSeek API

### 2.1 Obtain DeepSeek API Key

1. Visit [DeepSeek Platform](https://platform.deepseek.com/) and register/login  
2. Go to **API Keys** → **Create API Key** → give it a name (e.g., `OpenClaw-Kali`)  
3. Copy the generated key (format `sk-xxxxxxxx`) and store it securely  

> DeepSeek offers two models:  
> - `deepseek-chat`: general conversation (DeepSeek-V3.2)  
> - `deepseek-reasoner`: deep reasoning (suitable for complex logic)

### 2.2 Install Node.js (≥22)

```bash
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt install -y nodejs
node --version   # Should show v22.x.x
```

### 2.3 Install OpenClaw

```bash
sudo npm install -g openclaw@latest
openclaw --version
```

### 2.4 Run the Initialization Wizard (Integrate DeepSeek)

```bash
openclaw onboard
```

Answer the following key questions as prompted:

| Configuration          | Recommended value                                           |
| ---------------------- | ----------------------------------------------------------- |
| **Gateway mode**       | `local gateway`                                             |
| **Authentication**     | `deepseek-api-key`                                          |
| **API Key**            | paste `sk-xxxxxxxx`                                         |
| **Default model**      | `deepseek/deepseek-chat` or `deepseek/deepseek-reasoner`    |
| **Gateway token**      | auto-generated – save it for later                          |
| **Messaging platforms**| skip for now (Web UI is sufficient)                         |
| **Skills installation**| check basic skills as needed                                |

> If the wizard does not show DeepSeek directly, choose **OpenAI** or **Custom (OpenAI Compatible)** and fill in manually:  
> - API Key: `sk-xxxxxxxx`  
> - Base URL: `https://api.deepseek.com` (do **not** add `/v1` at the end)  
> - Model: `deepseek-chat` or `deepseek-reasoner`

### 2.5 Non‑interactive Configuration (Optional)

```bash
export DEEPSEEK_API_KEY="your_api_key_here"
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice deepseek-api-key \
  --deepseek-api-key "$DEEPSEEK_API_KEY" \
  --skip-health --accept-risk
```

### 2.6 Review and Edit Configuration File (Optional)

The configuration file is located at `~/.openclaw/openclaw.json`. You can manually adjust model parameters, tokens, etc. Below is an example structure (key parts):

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
            "input": ["text"],
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
            "input": ["text"],
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
      "token": "environment_variable_or_hardcoded"
    }
  }
}
```

### 2.7 Start the Gateway and Verify

```bash
# Start the gateway (foreground, for testing)
openclaw gateway run
```

Open your browser to `http://127.0.0.1:18789` and enter the Gateway Token to log in.

**Verify DeepSeek connectivity** (in the chat interface):

```
Hello, please confirm your current model identity.
```

If the reply mentions DeepSeek, the configuration succeeded.

**Running in the background**: Use `openclaw gateway start` or set up a systemd service (you can configure it yourself).

### 2.8 Retrieve the Gateway Token (for Remote Access)

```bash
cat ~/.openclaw/openclaw.json | grep token
```

Save the output string in the format `ocw_xxxxxx`.

---

## 🌐 Phase 3: Tailscale Networking

### 3.1 Install Tailscale on Kali

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo systemctl enable --now tailscaled
```

### 3.2 Log in to Tailscale

```bash
sudo tailscale up
```

Follow the link in the terminal to complete login (using Google/Microsoft/GitHub account).

### 3.3 Verify Tailscale Status

```bash
tailscale ip        # Shows Tailscale IP (format 100.x.x.x)
tailscale status    # Shows connected devices
```

### 3.4 Install Tailscale on Other Devices

- **Windows/macOS**: Download from [official website](https://tailscale.com/download)  
- **iOS/Android**: App Store / Google Play  
- **All devices must use the same Tailscale account**

### 3.5 Configure OpenClaw to Bind to Tailscale Network

```bash
openclaw config set gateway.bind tailnet
openclaw gateway restart
```

---

## 📱 Phase 4: Termius SSH Remote Connection

### 4.1 Install Termius

- **Desktop**: [termius.com/download](https://termius.com/download)  
- **Mobile**: Search for "Termius" in your app store  

Log in with the same account to sync configurations.

### 4.2 Add SSH Host

In Termius, click **+ New Host** and fill in:

| Field              | Value                                                  |
| ------------------ | ------------------------------------------------------ |
| **Label**          | `Kali-OpenClaw`                                        |
| **Hostname**       | Kali's Tailscale IP (e.g., `100.82.45.123`)            |
| **Port**           | Your SSH port (default 22 or custom, e.g., `2222`)     |
| **Username**       | The regular user you created (e.g., `yourusername`)    |
| **Authentication** | SSH Key → select your private key file                 |

### 4.3 Test Connection

Click **Connect**. A successful login means the configuration is correct.

---

## 🔗 Phase 5: Browser Remote Access and Control of OpenClaw

### 5.1 Get Kali's Tailscale IP

```bash
tailscale ip   # e.g., 100.82.45.123
```

### 5.2 Method 1: Tailscale Serve (Recommended)

On Kali, run:

```bash
sudo tailscale serve reset
sudo tailscale serve --bg --https=443 http://127.0.0.1:18789
tailscale serve status   # Shows something like https://machine-name.ts.net/
```

Now any device connected to the same Tailnet can access OpenClaw via that HTTPS address.

**First-time pairing**:

- The remote browser may show `1008: pairing required`  
- On the Kali machine, open `http://127.0.0.1:18789` locally  
- Go to **Devices → Pairing → Pending Requests** and click **Approve**  
- Refresh the remote page – it should work normally

### 5.3 Method 2: SSH Tunnel (Alternative)

Use this method if you prefer not to use Tailscale Serve or want a more traditional approach.

**Configure port forwarding via Termius**:

1. Edit your Kali host configuration in Termius  
2. Find **Port Forwarding** (or **SSH Tunnels**)  
3. Add a rule:  
   - Type: `Local`  
   - Local Address: `127.0.0.1`  
   - Local Port: `18789`  
   - Remote Host: `127.0.0.1`  
   - Remote Port: `18789`  
4. Save and reconnect

**Or use the command line**:

```bash
ssh -N -L 18789:127.0.0.1:18789 yourusername@<Kali_Tailscale_IP> -p <SSH_port>
```

Keep the tunnel running, then open `http://localhost:18789` in your local browser.

### 5.4 Log in to OpenClaw Web UI

Enter the Gateway Token you saved earlier (`ocw_xxxxxx`). Once logged in, you can use the chat, configuration, device management, and other features.

### 5.5 Functional Test: Browser Control

In the chat interface, type:

```
Open a browser, go to https://www.google.com, and take a screenshot
```

OpenClaw will use the DeepSeek API to understand the command, automatically launch a browser, and return a screenshot – confirming the entire pipeline works correctly.

### 5.6 Advanced Option: Chrome Relay Extension (Control Your Local Browser)

If you want the AI to control the browser you use daily (instead of a separate browser launched by OpenClaw), you can set up the Chrome Relay extension.

On Kali, generate a control token:

```bash
openclaw browser serve
```

Then install the OpenClaw extension in your local Chrome, enter Kali's Tailscale IP and the token, and connect.

---

## 🛠️ Troubleshooting

### OpenClaw Gateway Not Running

```bash
openclaw gateway status
openclaw gateway restart
```

### DeepSeek API 401 Error (Invalid Key)

```bash
cat ~/.openclaw/openclaw.json | grep apiKey
openclaw config set deepseek.apiKey "sk-newkey"
openclaw gateway restart
openclaw doctor   # Check connectivity
```

### Insufficient DeepSeek Balance

Log in to [DeepSeek Platform](https://platform.deepseek.com/) → **Top Up** (recommended to add 10-20 CNY for testing).

### Tailscale Connectivity Issues

```bash
tailscale status
tailscale ping <other_device_tailscale_ip>
```

### SSH Tunnel Fails

```bash
sudo systemctl status sshd
sudo ufw status   # Confirm the port is allowed
```

### Web UI Shows "origin not allowed"

Edit `~/.openclaw/openclaw.json` and add:

```json
{
  "gateway": {
    "controlUi": {
      "allowedOrigins": ["http://localhost:18789", "https://your-tailscale-domain.ts.net"]
    }
  }
}
```

Restart the gateway: `openclaw gateway restart`

### Pairing Request Does Not Appear

Access `http://127.0.0.1:18789` locally on Kali, go to **Devices → Pending Requests**, and approve manually.

---

## 📋 Security Best Practices Summary

| Measure                                                | Purpose                               |
| ------------------------------------------------------ | ------------------------------------- |
| Disable root SSH login                                 | Prevent brute‑force attacks           |
| Disable SSH password authentication (key only)         | Prevent password leaks                |
| Change default SSH port                                | Reduce automated scans                |
| UFW allows only necessary ports (SSH + Tailscale)      | Minimize exposure                     |
| All remote access via Tailscale encrypted tunnel       | Traffic security                      |
| Do not expose port 18789 to the public internet        | Prevent unauthorized access           |
| Use environment variables or config files for API keys | Avoid hardcoding secrets              |
| Regularly run `sudo apt update && sudo apt upgrade`    | Patch security vulnerabilities        |

---

## 📚 Reference Resources

- [OpenClaw Official Documentation](https://docs.openclaw.ai)  
- [OpenClaw DeepSeek Provider Documentation](https://docs.openclaw.ai/providers/deepseek)  
- [DeepSeek Platform](https://platform.deepseek.com/)  
- [Tailscale Documentation](https://tailscale.com/docs)  
- [Kali Linux Documentation](https://www.kali.org/docs/)  
- [Termius Support](https://support.termius.com)

---

**Ready to use.** This tutorial preserves all the key technical steps from the original, corrects numbering errors, command order, firewall details, and enhances readability. You can follow the sections in order.

---

