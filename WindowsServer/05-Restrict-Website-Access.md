# 05 — Restrict Website Access During Working Hours

> Prerequisite: server must be a Domain Controller (also running DNS by default). See `03-Active-Directory-Setup.md`.

## Important caveat
Group Policy has **no native "block this site between these hours"** setting. To get real time-based restriction, this guide combines the server's built-in **DNS role** with **Scheduled Tasks** that toggle the block on and off. This is free and works well for typical office browsing, but has limits — see "Limitations" below.

## Method: DNS Blackhole Zones + Scheduled Tasks

### Step 1 — Confirm DNS role is active
Since the server is a Domain Controller, DNS is already running and all domain-joined PCs use it automatically.
- Server Manager → **Tools** → **DNS** to confirm.

### Step 2 — Create blackhole zones for restricted sites
1. In DNS Manager, right-click **Forward Lookup Zones** → **New Zone**.
2. Choose **Primary Zone**.
3. Name it exactly after the site to block, e.g. `facebook.com`.
4. Finish the wizard, leaving the zone with **no records**.
   - An empty zone for that domain name causes DNS resolution to fail for it, which blocks access.
5. Repeat for each site to restrict (e.g. `youtube.com`, `instagram.com`, etc.).

### Step 3 — Schedule the block/unblock times
On the server, open **Task Scheduler**:

**Task A — Block (runs at start of restricted hours, e.g. 8:00 AM):**
```powershell
Enable-DnsServerZone -Name "facebook.com"
Enable-DnsServerZone -Name "youtube.com"
# repeat per site
```

**Task B — Unblock (runs at end of restricted hours, e.g. 6:00 PM):**
```powershell
Disable-DnsServerZone -Name "facebook.com"
Disable-DnsServerZone -Name "youtube.com"
# repeat per site
```

Set each as a Scheduled Task with a **Daily** trigger at the appropriate time, running as SYSTEM or an admin account, action = **Start a Program** → `powershell.exe` → arguments = `-Command "<the commands above>"`.

### Step 4 — Test
1. On a domain-joined test PC, during blocked hours, try visiting a restricted site — it should fail to load (DNS error).
2. Manually run the `Disable-DnsServerZone` commands (or wait until after "block end" time) and confirm the site loads again.

## Limitations
- This only blocks access **by domain name**. A user who knows the site's raw IP address can bypass it — this covers the vast majority of casual use but isn't airtight.
- It blocks the domain **entirely** for its scheduled hours — there's no per-user exception without additional configuration (e.g., a separate GPO/OU for exempted staff).
- For proper **category-based filtering** (social media, gambling, adult content, etc.) without maintaining a manual site list, consider pointing your DNS forwarder to a filtered DNS service such as **OpenDNS / Cisco Umbrella (free tier)** instead of maintaining blackhole zones by hand.
