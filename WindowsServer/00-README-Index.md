# SMB Office Server Setup — Guide Index

A complete build-out of a Windows Server 2022 VM (via Hyper-V) providing Active Directory, Group Policy management, USB restriction, time-based website blocking, and printer activity tracking for a small-to-medium office.

## Suggested order

1. **01-HyperV-Setup.md** — Enable Hyper-V on the host PC, create the VM, and configure networking (the trickiest part — especially Wi-Fi vs. Ethernet).
2. **02-Windows-Server-Install.md** — Download and install Windows Server 2022 inside the VM.
3. **03-Active-Directory-Setup.md** — Promote the server to a Domain Controller and join office PCs (required before any Group Policy features will work).
4. **04-Block-USB-GPO.md** — Block USB storage devices office-wide.
5. **05-Restrict-Website-Access.md** — Restrict specific websites during working hours.
6. **06-Print-Tracking.md** — Deploy the shared printer and track per-user print activity.

## Key things to keep in mind throughout
- The VM needs a **stable, network-reachable connection** (External virtual switch on **Ethernet**, not Wi-Fi or NAT) before Active Directory, Group Policy, or printer sharing will work for other PCs.
- Everything here uses **native Windows Server features** — no paid third-party software required, though free/open-source alternatives are noted where useful (DNS filtering services, PyKota, PaperCut free tier).
- This is built on the **evaluation edition** of Windows Server 2022. It's fully functional for 180 days; for permanent production use, plan to convert to a licensed copy.
