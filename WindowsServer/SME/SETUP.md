# SETUP.md — Office Server Build (Hyper-V + Windows Server 2022)

This document records the actual setup completed for the office: a Windows Server 2022 VM running on Hyper-V, providing Active Directory, Group Policy management, USB restriction, time-based website blocking, and printer activity tracking.

---

## 1. Environment Overview

| Item | Value |
|---|---|
| Hypervisor | Hyper-V (Windows 11 Pro host) |
| Guest OS | Windows Server 2022 Standard (Desktop Experience), evaluation edition |
| Server hostname | `WIN-T8F9UOBC7VV` |
| Domain name | `officecorp.local` |
| Server static IP | `192.168.100.230` |
| Subnet mask | `255.255.255.0` |
| Default gateway | `192.168.100.1` |
| Preferred DNS (server) | `127.0.0.1` (self, once promoted to DC) |
| Network | Office Wi-Fi network, `192.168.100.0/24` |

---

## 2. Hyper-V Networking Setup

1. Enabled Hyper-V on the Windows 11 Pro host via **Turn Windows features on or off**.
2. Created a VM (`PrintServer`/domain controller VM) with 4 GB RAM and a 60–80 GB dynamically expanding virtual hard disk.
3. **Network switch troubleshooting (key lessons learned):**
   - Initially connected the VM to **Default Switch (NAT)** — this gave the VM an isolated `172.27.x.x` address that other PCs on the real network could never reach. This was the root cause of early "cannot be contacted" domain-join errors.
   - Attempted an **External switch bound to Wi-Fi**, which caused the host's own Wi-Fi to disconnect. Fixed by ensuring **"Allow management operating system to share this network adapter"** was checked in Virtual Switch Manager.
   - Host had a **Network Bridge** adapter combining Wi-Fi + Ethernet; this wasn't selectable in the Hyper-V External switch dropdown, so the bridge was bypassed — the External switch was bound directly to the physical Wi-Fi adapter (`Realtek RTL8852BE WIFI`) instead.
   - **Lesson:** for a print/AD server that must be reachable by other PCs, an External switch bound to a real physical adapter (Wi-Fi or Ethernet) — not Default Switch — is required. Ethernet is more stable long-term if available.
4. Once the VM was on the real `192.168.100.x` network, a static IP was configured inside the VM (see Section 3).

---

## 3. Static IP Configuration (Server)

Set via Control Panel → Network and Sharing Center → Change adapter settings → Ethernet → Properties → IPv4:

- IP address: `192.168.100.230`
- Subnet mask: `255.255.255.0` (note: Windows auto-fills subnet masks based on IP class, e.g. `255.255.0.0` for the earlier `172.27.x.x` range — this must be manually corrected to match the real network's mask, don't accept the auto-fill blindly)
- Default gateway: `192.168.100.1`
- Preferred DNS: `127.0.0.1`

**Issue encountered — IP address conflict:** the server's static IP briefly duplicated the address already assigned (via DHCP) to the host's own shared network adapter on the External switch. This caused the VM to fall back to a self-assigned APIPA address (`169.254.x.x`), breaking gateway connectivity, DNS resolution, and internet access simultaneously. Confirmed via `ipconfig /all` showing `(Duplicate)` next to the IP. **Resolved by reassigning the server to a different, verified-unused static address** (`192.168.100.230`) outside the router's DHCP pool.

---

## 4. Active Directory Domain Services (AD DS)

1. Installed the **AD DS** role via Server Manager → Add Roles and Features.
2. Promoted the server to a Domain Controller: **Add a new forest**, root domain `officecorp.local`.
3. Set a DSRM password during promotion.
4. Created Organizational Units under the domain:
   - `Office Computers`
   - `Office Users`
5. **Issue encountered — missing DNS SRV records:** after promotion (which happened while network settings were still being corrected), the `_msdcs.officecorp.local` forest DNS zone and its SRV records (`_ldap._tcp.dc._msdcs.officecorp.local`) were missing or incomplete, causing domain-join to fail with:
   > *"An AD DC for the domain 'officecorp.local' could not be contacted... error 0x000005B4 ERROR_TIMEOUT"*
   - Verified using: `nslookup -type=SRV _ldap._tcp.dc._msdcs.officecorp.local <server IP>`
   - Fixed by restarting the **Netlogon** service (forces DNS record re-registration) and confirming the zone structure in DNS Manager (`_msdcs.officecorp.local` should exist as its own top-level forward lookup zone, containing `dc → _tcp → _ldap` SRV record).

---

## 5. Joining Office PCs to the Domain

1. On each PC: Settings → System → About → Domain or workgroup → Change → Domain → `officecorp.local`.
2. **Issue encountered — DNS typo:** the office laptop's Wi-Fi adapter had its Preferred DNS server manually entered as `192.167.100.230` (typo: `.167` instead of `.168`), a nonexistent network. This caused the PC's *default* DNS resolver to fail even though manual `nslookup` queries against the correct IP succeeded. Fixed by correcting the DNS server address in the adapter's IPv4 settings.
3. Once DNS was correct, domain join succeeded and prompted for domain admin credentials (`officecorp\Administrator`).
4. After joining, computer objects landed in the **default "Computers" container** rather than the intended `Office Computers` OU — this had to be manually corrected (see Section 6), since GPOs linked only to `Office Computers` do not apply to objects sitting in the default container.

---

## 6. Group Policy — USB Storage Blocking

1. Group Policy Management → `Domains → officecorp.local → Office Computers` (note: OUs are nested one level deeper than the Forest/Domains root view).
2. Right-click `Office Computers` → **Create a GPO in this domain, and Link it here** → named `Block USB Storage`.
3. Edited the GPO: `Computer Configuration → Policies → Administrative Templates → System → Removable Storage Access → "All Removable Storage classes: Deny all access"` → **Enabled**.
4. **Issue encountered — policy not applying:**
   - Root cause #1: the test laptop's computer object was in the default `Computers` container, not `Office Computers` — moved via Active Directory Users and Computers → right-click computer → **Move…** → `Office Computers`.
   - Root cause #2: Removable Storage Access policies require a **full restart**, not just `gpupdate /force`, to take effect.
5. **Verification method:** `gpresult /r /scope:computer` on the client (the `/scope:computer` flag avoids a benign "user does not have RSoP data" error that appears when running an elevated session). Confirmed `Block USB Storage` appears under *Applied Group Policy Objects*.
6. **Result confirmed working** — USB drives are denied access on domain-joined, correctly-placed machines after a restart.

---

## 7. Time-Based Website Restriction (8:30 AM – 5:30 PM)

Native Group Policy has no built-in time-based website blocking. Implemented using **DNS blackhole zones** (via the DC's built-in DNS role) combined with **Scheduled Tasks** that toggle the zones on/off.

### 7.1 Blackhole DNS zones created
In DNS Manager → Forward Lookup Zones → New Zone → Primary Zone (left empty, no records):
- `facebook.com`, `fb.com`
- `instagram.com`
- `tiktok.com`, `tiktokcdn.com`
- `youtube.com`, `youtu.be`, `m.youtube.com`
- `netflix.com`

### 7.2 Scheduled Tasks
- **"Block Sites - Start"** — Daily trigger at **8:30 AM** — runs PowerShell:
  ```powershell
  Enable-DnsServerZone -Name 'facebook.com'; Enable-DnsServerZone -Name 'fb.com'; Enable-DnsServerZone -Name 'instagram.com'; Enable-DnsServerZone -Name 'tiktok.com'; Enable-DnsServerZone -Name 'tiktokcdn.com'; Enable-DnsServerZone -Name 'youtube.com'; Enable-DnsServerZone -Name 'youtu.be'; Enable-DnsServerZone -Name 'm.youtube.com'; Enable-DnsServerZone -Name 'netflix.com'
  ```
- **"Block Sites - End"** — Daily trigger at **5:30 PM** — same command list, using `Disable-DnsServerZone` instead.

### 7.3 Category-based restriction (illegal/adult/piracy content)
Not handled via manual domain lists (impractical — these sites rotate domains constantly). Recommended approach instead: point the server's DNS **Forwarders** (DNS Manager → server Properties → Forwarders tab) at a category-based filtering service such as **OpenDNS / Cisco Umbrella (free tier)**, which blocks broad categories (piracy, adult content, gambling, etc.) automatically. *(Not yet configured as of this document — optional next step.)*

### 7.4 Limitations noted
- Blocking is by **domain name only** — doesn't stop access via raw IP address (edge case, low practical risk for typical office use).
- Additional CDN/video-delivery domains for some platforms may not be fully covered by the base domain list above.

---

## 8. Printer Setup & Print Activity Tracking

1. Installed the **Print and Document Services** role via Add Roles and Features.
2. Print Management → Printers → Add Printer, using the printer's IP address.
   - **Issue encountered — network mismatch:** the printer was initially on a different Wi-Fi access point than the server, making it unreachable. Resolved by moving the printer onto the same `192.168.100.x` network as the server.
3. Shared the printer (Printer Properties → Sharing → **Share this printer**).
4. Deployed via Group Policy: Print Management → right-click printer → **Deploy with Group Policy** → created/selected GPO `Deploy Office Printer`, applied to **Office Computers** (per-machine).
   - **Issue encountered — GPO not applying:** same root cause as Section 6 — needed to confirm the GPO was actually linked to `Office Computers`, and the laptop's computer object was correctly placed in that OU.
5. Enabled logging: Event Viewer → `Applications and Services Logs → Microsoft → Windows → PrintService → Operational` → **Enable Log**. This records **Event ID 307** per completed print job (username, document name, printer, page count).
6. **Verified working:** test prints from the domain-joined laptop appear automatically in Settings → Printers & scanners (after `gpupdate /force` + restart), print successfully, and generate matching Event ID 307 entries in the server's Operational log.

### 8.1 Known limitation
Native Windows print logging captures **metadata only** (who printed what file, when, how many pages) — it does **not** retain or allow viewing of the actual printed document content. For content-level auditing, a third-party tool (e.g. PaperCut, Print Audit) would be required; not implemented here.

---

## 9. Employee (Domain User) Accounts

1. Created via Active Directory Users and Computers → right-click **directly on the `Office Users` OU** → New → User (creating directly inside the OU avoids a move step).
2. Set a temporary password with **"User must change password at next logon"** checked.
3. **Issue encountered:** an account created outside the OU (at the same level as `Office Users`) could not be moved or deleted — blocked by the **"Protect object from accidental deletion"** flag, checked by default on new AD objects. Resolved by enabling **View → Advanced Features** in ADUC, opening the object's **Object** tab, and unchecking that flag before deleting/moving.
4. Standard employee accounts are **not** added to Domain Admins or the local Administrators group, so they behave as standard (non-admin) users on domain-joined PCs by default — no extra configuration required for this baseline.
5. **Not yet configured:** a Restricted Groups GPO to explicitly enforce/lock down local Administrators group membership office-wide (recommended next step, not yet implemented).

---

## 10. Outstanding / Recommended Next Steps

- [ ] Set up **Restricted Groups** GPO to formally enforce standard-user-only access on office PCs.
- [ ] Configure **OpenDNS/Cisco Umbrella** forwarding for category-based filtering (illegal/adult/gambling content).
- [ ] Create additional employee accounts inside `Office Users` as needed, following the process in Section 9.
- [ ] Move each additional office PC's computer object into `Office Computers` upon joining (do not assume this happens automatically).
- [ ] Build a PowerShell reporting script to aggregate Event ID 307 entries into a per-user/per-period printing report (raw log entries only exist currently — no summarized report yet).
- [ ] Plan for conversion from the 180-day Windows Server evaluation license to a licensed copy before expiry, if this remains the permanent production server.
- [ ] Consider moving off Wi-Fi to a wired Ethernet connection for the External switch for long-term stability, if not already done.

---

## 11. Key Troubleshooting Commands Reference

| Purpose | Command |
|---|---|
| Full network config | `ipconfig /all` |
| Flush/re-register DNS | `ipconfig /flushdns` / `ipconfig /registerdns` |
| Force Group Policy update | `gpupdate /force` |
| Check applied GPOs (computer only, avoids RSoP user error) | `gpresult /r /scope:computer` |
| Visual applied-policy viewer | `rsop.msc` |
| Test SRV record for domain controller discovery | `nslookup -type=SRV _ldap._tcp.dc._msdcs.officecorp.local <server IP>` |
| Restart AD record registration | Restart the **Netlogon** service via `services.msc` |
| View ARP table (MAC-to-IP mapping, useful for IP conflict diagnosis) | `arp -a` |
