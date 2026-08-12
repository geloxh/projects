# 02 — Windows Server Installation (Inside the Hyper-V VM)

## Download

**Official Microsoft Evaluation Center — Windows Server 2022:**
https://www.microsoft.com/en-us/evalcenter/download-windows-server-2022

- Register with an email to unlock the download.
- Download format: **ISO** (not VHD). You install it yourself onto the virtual hard disk you created — a VHD would be a pre-built, already-installed disk, which isn't what the Evaluation Center provides for this product.
- The evaluation edition expires in **180 days**, and must activate online within the first **10 days** to avoid automatic shutdown. It can later be converted to a licensed retail version without reinstalling.
- Windows Server 2025 is also available and fully supported if you'd rather start on the newer release — same download process, same feature set for everything in this guide. 2022 has simply been out longer, so slightly more third-party software/driver compatibility has been confirmed against it.

## Choosing the installation type

During setup, you'll be asked to choose between:

- **Windows Server 2022 (Desktop Experience)** ✅ — **choose this one.** Full graphical interface: Server Manager, Group Policy Management Console, Print Management, Active Directory Users and Computers, etc. — everything in these guides assumes this option.
- **Windows Server 2022 (Server Core)** — command-line/PowerShell only, no desktop. More secure, smaller footprint, but a much steeper learning curve. Not used in this guide.

## Installing inside the VM

1. In Hyper-V Manager, make sure the VM's DVD drive is pointed at the downloaded ISO (this was set during VM creation under *Installation Options*, or can be set later in VM **Settings > IDE Controller > DVD Drive**).
2. Start the VM and connect to its console (double-click the VM in Hyper-V Manager).
3. Boot from the ISO, select language/region, click **Install now**.
4. Choose **Windows Server 2022 Standard (Desktop Experience)**.
   - Standard edition is sufficient — Datacenter is built for large-scale virtualization hosting and isn't needed for an SMB print/AD/GPO server.
5. Accept the license terms, choose **Custom: Install Windows Server only**, select the virtual hard disk you created, and proceed.
6. The VM will install and reboot a few times automatically.
7. Set the local Administrator password when prompted.
8. Log in — you'll land on the Windows Server desktop with **Server Manager** opening automatically.

## Next steps
Once installed and able to log in, continue to:
- `03-Active-Directory-Setup.md` — turn this server into a Domain Controller
- `06-Print-Tracking.md` — if you only need print tracking without AD/GPO restrictions
