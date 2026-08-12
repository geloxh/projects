# OpenClaw + DingTalk: automated client visit reports

## Overview

This project connects **OpenClaw** (a self-hosted AI agent) to **DingTalk** so that
field employees can log client visits by chatting with a bot, and the agent
automatically formats and distributes a clean visit report to a group chat.

Two deployment options were covered:

| Option | Where it runs | Best for |
|---|---|---|
| A. Managed | Alibaba Cloud Simple Application Server (pre-built OpenClaw image) | Fastest setup, no local machine dependency, always-on |
| B. Self-hosted | Your own machine (e.g. Windows 11 via WSL2) or VPS | Full control, no cloud server cost, but machine must stay on |

---

## Part 1 — Managed setup (Alibaba Cloud)

1. **Buy the server**: use Alibaba Cloud's OpenClaw pre-configured purchase link (Simple
   Application Server, ≥2 GiB memory).
2. **Initialize**: Application Details tab → Initialize → configure a model
   provider (Alibaba Cloud Model Studio, DeepSeek, Kimi, GLM, Zhipu, etc.) with an
   API key → optionally enable the Web UI to chat with the agent in-browser.
3. **Create the DingTalk bot**: go to the [DingTalk Open Platform](https://open-dev.dingtalk.com/),
   use the one-click "auto-create OpenClaw bot" option, then copy the generated
   **Client ID** and **Client Secret**.
4. **Connect the channel**: Application Details → Channels → Add Channel → DingTalk →
   paste in Client ID/Secret → Apply.
5. **Test**: add the bot to a DingTalk group (`@botname`) or message it directly.

---

## Part 2 — Self-hosted setup (Windows 11 via WSL2)

### 2.1 Install WSL2 + Ubuntu

```powershell
wsl --install -d Ubuntu
```
Create a Linux username and password when prompted. Confirm you're in the right
distro (not `docker-desktop`, which is a Docker-internal utility VM, not a usable
Linux environment):

```bash
cat /etc/os-release   # should show Ubuntu, not "Docker Desktop"
```

If you have multiple distros installed, always launch explicitly:
```powershell
wsl -d Ubuntu
# or set it as default:
wsl --set-default Ubuntu
```

### 2.2 Enable systemd

```bash
sudo tee /etc/wsl.conf > /dev/null << 'EOF'
[boot]
systemd=true
[interop]
enabled=true
appendWindowsPath=true
EOF
```
Then from PowerShell: `wsl --shutdown`, and reopen Ubuntu.

### 2.3 Install Node.js 22+

```bash
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt-get install -y nodejs
node --version   # v22.x or higher
```

### 2.4 Install OpenClaw

```bash
curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash
```
Or manually: `npm install -g openclaw@latest && openclaw onboard --install-daemon`.

Verify:
```bash
openclaw --version
openclaw doctor
openclaw gateway status
```

### 2.5 Configure a model provider

During onboarding (or via the `Crestodian` setup assistant), connect an LLM
provider — Anthropic, OpenAI, DeepSeek, or a local model via Ollama for a
no-API-cost option. This determines the report-writing quality and running cost.

### 2.6 Install the DingTalk plugin

```bash
openclaw plugins install @adongguo/openclaw-dingtalk
```

### 2.7 Create the DingTalk app/bot

1. [DingTalk Open Platform](https://open-dev.dingtalk.com/) → Application
   Development → Enterprise Internal Development → **Create Application**.
2. Application Capabilities → Robot → Enable Robot Configuration → **Stream mode**.
3. Copy **AppKey** and **AppSecret** from Basic Information.
4. Publish the app (at least the test version).

### 2.8 Wire the plugin config

```bash
openclaw config set channels.dingtalk.appKey "dingXXXXXXXX"
openclaw config set channels.dingtalk.appSecret "your_app_secret"
openclaw config set channels.dingtalk.enabled true
```
Stream mode runs over WebSocket — no public IP, domain, or SSL cert required.

### 2.9 Keep it running

A local Windows machine only serves the bot while WSL2/Ubuntu is running. For
reliability, keep the machine on and consider registering OpenClaw's gateway to
start automatically (Windows Scheduled Task, or systemd auto-restart inside WSL).

---

## Cost note

Hosted model providers (Anthropic, OpenAI, etc.) charge per token used — for a
short, conversational task like formatting a visit report, this is typically low
cost (cents to a few dollars per month at moderate volume). A fully free
alternative is running a local model via **Ollama**, at the cost of lower output
quality and needing a capable machine.

---

## Runtime workflow

```
Employee at client site
        |
   sends message in DingTalk
        v
   DingTalk bot (Stream mode, WebSocket)
        v
   OpenClaw gateway (WSL2 / cloud)
        v
   Agent captures visit details
   (client, purpose, outcome, next steps)
        v
   LLM formats report
   (adds employee name + timestamp)
        v
   Posted via DingTalk group webhook
        v
   Manager / sales group sees report
```

*(See the interactive diagram shared earlier in this conversation for the visual
version of this flow.)*

## Setting up the automation rule

Once the bot is connected, describe the desired behavior to OpenClaw in plain
language — no code required. Example instruction:

> "Whenever someone messages you 'visit report' followed by client name, visit
> purpose, and outcome, format it into a clean report with employee name and
> timestamp, then post it to this group webhook: `[webhook URL]`."

Get the webhook from the target DingTalk group:
**Group Settings → Group Management → Bot → click bot avatar → copy webhook address.**

Optional extensions:
- A **daily digest**: "Every day at 6pm, ask the sales group who visited a client
  today, then post a consolidated summary to [webhook]."
- **Logging**: route captured visits to a shared spreadsheet or CRM in addition to
  the group post, if a connector/API is available for that system.

---

## Troubleshooting notes from this session

| Symptom | Cause | Fix |
|---|---|---|
| `sudo: not found` | Wrong shell / missing sudo | Check `whoami` and `/etc/os-release`; ensure real Ubuntu, not a minimal shell |
| `PRETTY_NAME="Docker Desktop"` | Inside Docker's internal WSL utility VM, not a real Linux distro | Launch `wsl -d Ubuntu` explicitly, or set Ubuntu as default distro |
| `-sh: wsl: not found` | Ran `wsl` from inside a Linux shell instead of Windows PowerShell | `wsl` is a Windows-side command; open actual PowerShell first |
| Account creation interrupted (Ctrl+C) | Partial user setup | Reopen the Ubuntu app; if broken, `wsl --unregister Ubuntu` then reinstall |
| Forgot WSL user password | — | `wsl -d Ubuntu -u root` then `passwd <username>` to reset |