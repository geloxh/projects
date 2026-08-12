# 01 — Hyper-V Setup (Windows 11 Pro)

## Requirements
- Windows 11 **Pro, Enterprise, or Education** (Home does not support Hyper-V)
- CPU virtualization support (Intel VT-x / AMD-V) enabled in BIOS/UEFI

### Confirm requirements
1. Press `Win + R`, type `winver`, Enter — confirm edition is **Pro/Enterprise/Education**.
2. Open Task Manager (`Ctrl+Shift+Esc`) → **Performance** tab → **CPU** → check the **Virtualization** field shows **Enabled**.
3. Optional double-check: open Command Prompt and run:
   ```
   systeminfo
   ```
   Look for the **Hyper-V Requirements** section. If it says *"A hypervisor has been detected"*, a hypervisor is already active — this is normal and means virtualization is working.
4. If virtualization is **Disabled**, restart into BIOS/UEFI (F2/F10/Del/Esc depending on manufacturer) and enable **Intel VT-x**, **Intel Virtualization Technology**, **AMD-V**, or **SVM Mode**.

## Enable Hyper-V
1. Start menu → search **"Turn Windows features on or off"** → open it.
2. Check **Hyper-V** (this auto-selects Hyper-V Management Tools and Hyper-V Platform) → OK.
3. Restart when prompted.
4. After reboot, search Start menu for **Hyper-V Manager** and open it.

To check if Hyper-V is already enabled without opening the GUI, run in PowerShell (Admin):
```powershell
Get-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V-All
```
`State : Enabled` means it's already on.

---

## Creating a Virtual Machine

1. In Hyper-V Manager → Actions pane → **New > Virtual Machine**.
2. Name the VM (e.g., `PrintServer`).
3. Allocate memory — 4 GB is fine for a small print/AD server.
4. **Configure Networking** — choose a virtual switch (see below).
5. **Create a virtual hard disk**:
   - Default location: `C:\Users\Public\Documents\Hyper-V\Virtual Hard Disks\`
   - Size: **60–80 GB** is comfortable for Windows Server + print logs (dynamically expanding — doesn't consume full size immediately, just needs the headroom on your host drive).
6. **Installation Options** — choose *"Install an operating system from a bootable image file"* and browse to your Windows Server ISO.

---

## Virtual Switch Options — Which to Choose

| Option | What it does | Use for a print/AD server? |
|---|---|---|
| **Not Connected** | No network access at all | ❌ No |
| **Default Switch** | Built-in NAT switch; VM gets internet through the host, but other PCs on the network **cannot reach it** | ⚠️ Fine for testing only |
| **External (Ethernet)** | Bridges VM directly onto your physical network — VM gets its own real IP, reachable by all office PCs | ✅ **Recommended — this is what you want for production** |
| **External (Wi-Fi)** | Same idea, but Wi-Fi adapter bridging is unreliable and can knock the *host* offline | ⚠️ Avoid if possible |

### Creating an External Switch
1. Hyper-V Manager → Actions pane → **Virtual Switch Manager**.
2. Select **External** → **Create Virtual Switch**.
3. Choose your **Ethernet adapter** (preferred) from the dropdown.
4. Check **"Allow management operating system to share this network adapter"** — this lets your host PC keep its own network access while the VM uses the same physical adapter.
5. Apply → OK.

### If your host loses Wi-Fi after creating an External switch
This is a known Hyper-V/Wi-Fi limitation, not a bug — Wi-Fi drivers often don't support the bridging Hyper-V needs.

**Best fix — use Ethernet instead:**
1. Connect an Ethernet cable from the host to your router/switch.
2. Confirm Windows shows "Ethernet" connected (system tray).
3. Hyper-V Manager → Virtual Switch Manager → remove the broken Wi-Fi-based External switch (or create a new one) bound to the **Ethernet** adapter instead.
4. Check "Allow management operating system to share this network adapter" → Apply → OK.
5. In the VM's **Settings > Network Adapter**, select the new Ethernet-based switch → Apply → OK.
6. Start the VM, run `ipconfig` inside it to confirm it received a valid IP (not `169.254.x.x`).

**If Ethernet isn't available yet — fall back to Default Switch (NAT):**
1. Shut down the VM.
2. Remove the broken Wi-Fi External switch (Virtual Switch Manager → select it → Remove). Host Wi-Fi should reconnect automatically (toggle Wi-Fi off/on if not).
3. VM **Settings > Network Adapter** → select **Default Switch** → Apply → OK.
4. Start VM, run `ipconfig` — expect a `172.x.x.x` NAT address.

> ⚠️ **Important:** On Default Switch (NAT), the VM has internet access but is **invisible to other PCs on your network**. This is fine for installing Windows Server and testing roles, but domain join, Group Policy, and shared printing from other office PCs will **not work** until you get a working External switch (Ethernet is the reliable path).

### Fixing a VM that was created with the wrong network adapter
1. Shut down the VM (Hyper-V Manager → right-click → Shut Down / Turn Off).
2. Confirm/create the correct External switch (Virtual Switch Manager).
3. Right-click VM → **Settings** → **Network Adapter** → set the **Virtual switch** dropdown to the correct switch → Apply → OK.
4. Start the VM → run `ipconfig` to verify.
