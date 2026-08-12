# 04 — Block USB Ports via Group Policy

> Prerequisite: server must be a Domain Controller with an OU structure and joined PCs. See `03-Active-Directory-Setup.md`.

This is a fully native Group Policy feature — no third-party software needed.

## Steps

1. Server Manager → **Tools** → **Group Policy Management**.
2. Right-click your **Office Computers** OU → **Create a GPO in this domain, and Link it here...**
3. Name it clearly, e.g. `Block USB Storage`.
4. Right-click the new GPO → **Edit** to open the Group Policy Management Editor.
5. Navigate to:
   ```
   Computer Configuration > Policies > Administrative Templates > System > Removable Storage Access
   ```
6. Double-click **"All Removable Storage classes: Deny all access"** → set to **Enabled** → Apply/OK.
   - This blocks read/write/execute on USB drives, external hard drives, and similar removable media.
   - For more granular control, use the individual policies instead:
     - **Removable Disks: Deny write access**
     - **Removable Disks: Deny read access**

## Testing

1. On a domain-joined test PC, open Command Prompt as Administrator and run:
   ```
   gpupdate /force
   ```
   (Group Policy normally refreshes automatically every 90–120 minutes; forcing it applies changes immediately.)
2. Plug in a USB drive — it should fail to be recognized or show an access denied error.

## Notes
- This policy applies at the **Computer Configuration** level, so it affects the PC regardless of which user logs in.
- To exempt specific PCs (e.g., an IT admin machine), move that computer object to a different OU that this GPO isn't linked to, or use **Security Filtering** on the GPO to exclude specific computer groups.
