# 06 — Track Printing Activity

> Prerequisite: for GPO-based printer deployment, the server should be a Domain Controller with joined PCs. See `03-Active-Directory-Setup.md`. The Print and Document Services role itself does not strictly require AD, but deployment via Group Policy does.

## Step 1 — Install the Print and Document Services role
1. Server Manager → **Manage** → **Add Roles and Features**.
2. Check **Print and Document Services**, accept additional required features → Install.

## Step 2 — Add and share the printer
1. Server Manager → **Tools** → **Print Management**.
2. Right-click **Printers** under your server → **Add Printer**.
3. Add the printer using its **IP address** (TCP/IP port) rather than relying on auto-discovery, so you know exactly which port it's using. Install the correct manufacturer driver.
4. Right-click the printer → **Printer Properties** → **Sharing** tab → check **Share this printer**, give it a clear share name (e.g. `HP-Office-2F`).

## Step 3 — Deploy the printer automatically via Group Policy
1. In Print Management, right-click the printer → **Deploy with Group Policy**.
2. Choose or create a GPO (e.g. `Deploy Office Printer`).
3. Check **"The computers that this GPO applies to (per machine)"** → Add → OK.
4. This automatically installs the printer connection on every domain-joined PC in scope — no manual per-PC setup needed.

## Step 4 — Enable print activity logging
1. Open **Event Viewer**.
2. Navigate to:
   ```
   Applications and Services Logs > Microsoft > Windows > PrintService
   ```
3. Right-click **Operational** → **Enable Log**.
   - This captures **Event ID 307** for every completed print job: username, document name, printer, and **page count**.

## Step 5 — Verify
1. On the server, run:
   ```
   gpupdate /force
   ```
2. On a domain-joined test PC, confirm the printer appears automatically under **Settings > Printers & scanners** without manual setup.
3. Print a test page from the test PC.
4. Back on the server, check **PrintService > Operational** in Event Viewer for a new **Event ID 307** entry with the correct user, document, and page count.

## Getting usable reports (not just raw log entries)
Event ID 307 entries sit in the Event Viewer log as individual events — not a report. To get per-user/per-period totals, pull and aggregate them with PowerShell, e.g.:

```powershell
Get-WinEvent -LogName "Microsoft-Windows-PrintService/Operational" |
  Where-Object { $_.Id -eq 307 } |
  Select-Object TimeCreated, @{n='User';e={$_.Properties[2].Value}}, @{n='Pages';e={$_.Properties[3].Value}}, @{n='Printer';e={$_.Properties[4].Value}} |
  Export-Csv -Path "C:\PrintReports\print_log.csv" -NoTypeInformation
```
> Field indices under `Properties[]` can vary slightly by Windows Server build — verify against a sample event in Event Viewer (Details tab, Friendly View) before relying on this in production, and adjust indices if needed.

For a fuller solution with dashboards, quotas, and cost tracking out of the box, consider open-source **PyKota** or the free tier of **PaperCut NG** rather than building all reporting by hand.

## Free / open-source alternatives worth knowing about
- **PyKota** — open-source print quota/accounting (Python-based), works with Linux/CUPS print servers; Windows clients print through a Samba/CUPS server that handles accounting.
- **PaperCut NG (free tier)** — not open-source, but free for small user/printer counts; industry-standard reporting and quotas.
