# OpenClaw + DingTalk Setup on Windows (via WSL2)

Setup guide for running a self-hosted OpenClaw agent connected to DingTalk, for field-visit logging and HR reporting. Written for a Windows machine using WSL2 as the Linux host.

---

## 0. Prerequisites

- Windows 10/11 with WSL2 support
- A DingTalk enterprise account with admin access to create an internal app
- An LLM API key (Claude, OpenAI, etc.)
- Admin access to a machine that stays on 24/7 for production use (see note at the end — WSL2 on a laptop is fine for testing, not for production)

---

## 1. Install WSL2

In **Windows PowerShell** (run as Administrator):

```powershell
wsl --install
```

This installs Ubuntu by default. Restart if prompted, then open the **Ubuntu** app from the Start menu and finish setup by creating a Linux username and password.

> **Note:** Your WSL Linux username/password is separate from your Windows login. Don't mix them up later when `sudo` asks for a password.

To find your Linux username at any time:
```bash
ls /home
```

To reset a forgotten Linux password, from **PowerShell** (not inside WSL):
```powershell
wsl -u root
```
Then inside that root shell:
```bash
passwd your-actual-username
exit
```

---

## 2. Fix a broken dpkg state (if you hit this)

If `sudo` fails a few times mid-install, `apt`/`dpkg` can get left in an interrupted state, causing every future install to fail with:

```
E: dpkg was interrupted, you must manually run 'sudo dpkg --configure -a' to correct the problem.
```

Fix it with, in order:
```bash
sudo dpkg --configure -a
sudo apt update
sudo apt --fix-broken install
sudo apt install -f
```

All four should complete with no errors before continuing.

---

## 3. Install OpenClaw

Inside your WSL2 Ubuntu shell:

```bash
curl -fsSL https://openclaw.ai/install.sh | bash
```

This detects your OS, installs Node.js if missing, installs OpenClaw, and launches the onboarding wizard (choose a model provider, paste an API key).

Manual alternative, if you prefer to control each step:
```bash
npm install -g openclaw@latest
openclaw onboard --install-daemon
```

**Verify it's running:**
```bash
openclaw --version
openclaw gateway status   # should show it listening on port 18789
openclaw dashboard        # opens the Control UI in your browser
```

---

## 4. Create the DingTalk app (robot)

Done in the **DingTalk Open Platform** developer console (browser, any OS):

1. **应用开发 → 企业内部开发 (Enterprise Internal Development) → 创建应用 (Create Application)**
2. **应用能力 (Application Capabilities) → 机器人 (Robot) → 开启机器人配置 (Enable Robot Configuration)**
3. Set message reception mode to **Stream 模式 (Stream Mode)** — no public IP or webhook server needed
4. **基础信息 (Basic Information) → 应用信息** — copy the **AppKey (clientId)** and **AppSecret (clientSecret)**
5. **Publish** the app (at least the test version) — an unpublished bot won't respond
6. Note the **CorpId** and **AgentId** from the same page

> ⚠️ **Quota check:** DingTalk's free unlimited API/Webhook/Stream quota for AI agent integrations is, by default, valid only until **2026-03-31** (extendable to 2026-04-30 with an approved application). Check **钉钉开发者后台 → 资源管理** before relying on this in production.

---

## 5. Install and configure the DingTalk plugin

```bash
openclaw plugins install @soimy/dingtalk
```
Requires OpenClaw 2026.3.24 or newer — update OpenClaw first if needed.

Configure interactively:
```bash
openclaw configure --section channels
```

...or edit the config file directly:
```json
{
  "plugins": {
    "enabled": true,
    "allow": ["dingtalk"]
  },
  "channels": {
    "dingtalk": {
      "enabled": true,
      "clientId": "dingxxxxxx",
      "clientSecret": "your-app-secret",
      "robotCode": "dingxxxxxx",
      "corpId": "dingxxxxxx",
      "agentId": "123456789",
      "dmPolicy": "open",
      "groupPolicy": "open",
      "messageType": "markdown"
    }
  }
}
```

- `dmPolicy` / `groupPolicy` — restrict who can message the bot once past testing (e.g. limit to the sales team's DingTalk group instead of `"open"`)
- `messageType: "markdown"` — enables formatted replies

Restart the gateway to apply:
```bash
openclaw gateway restart
```

---

## 6. Verify the connection

DM the bot in DingTalk, or @-mention it in a group it's added to. You should get an AI reply.

If nothing happens:
```bash
openclaw doctor          # checks for config issues
openclaw gateway status  # confirms the daemon is alive
```

Common causes: app not published, Stream mode not enabled, mismatched clientId/clientSecret.

---

## 7. What's still needed for the field-visit-tracking use case

Getting the bot talking is only step one. For the actual goal — HR/management verifying real client visits — still need to build:

- A tool/skill that writes each reported visit (employee, client, timestamp, notes, ideally location) to a real structured database, not just chat history
- A report-generation tool HR can trigger to query that database and produce Excel/PDF output
- Consider pulling from DingTalk's native **外勤打卡 (field clock-in)** API for GPS + photo verification, rather than relying solely on free-text self-reports which are easy to fake

---

## 8. Production note

WSL2 shuts down background processes when all WSL windows close or the machine sleeps — this kills the OpenClaw Gateway daemon along with it. Fine for testing on a Windows laptop; **not reliable for a bot field staff depend on daily**. Once the setup works locally, move the finished config to an always-on Linux VPS (DigitalOcean, Hetzner, etc. — see OpenClaw's Linux server docs) for real deployment.

---

## Alternative plugin

Official DingTalk-maintained connector instead of the community one:
`DingTalk-Real-AI/dingtalk-openclaw-connector` on GitHub — same Stream-mode approach, but DingTalk's own docs lean toward personal/individual use rather than direct enterprise production deployment without your own security review.
