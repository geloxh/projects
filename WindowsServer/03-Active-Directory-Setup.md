# 03 — Active Directory & Domain Setup

This is the prerequisite for Group Policy features (USB blocking, website restriction, printer deployment). Group Policy only pushes to computers joined to a domain.

> ⚠️ Before starting: make sure the VM has stable network connectivity reachable by other PCs on your office network (an **External virtual switch bound to Ethernet** — see `01-HyperV-Setup.md`). A NAT/Default Switch VM cannot be reached by other computers and domain join will fail.

## Step 1 — Set a static IP on the server
1. Control Panel → Network and Sharing Center → Change adapter settings.
2. Right-click your Ethernet adapter → Properties → IPv4 → set a **static IP**, subnet mask, and gateway matching your network.
3. Set **Preferred DNS server** to `127.0.0.1` (the server will become its own DNS server once promoted).

## Step 2 — Install Active Directory Domain Services (AD DS)
1. Server Manager → **Manage** → **Add Roles and Features**.
2. Proceed through the wizard, check **Active Directory Domain Services**.
3. Accept additional required features when prompted → Install.

## Step 3 — Promote the server to a Domain Controller
1. In Server Manager, click the yellow warning flag → **Promote this server to a domain controller**.
2. Choose **Add a new forest**.
3. Enter a root domain name, e.g. `officecorp.local` (using `.local` avoids conflicts with real public domains).
4. Set a Directory Services Restore Mode (DSRM) password.
5. Continue through the remaining default options → the server runs prerequisite checks, then reboots automatically to complete promotion.
6. After reboot, log in with the **domain** administrator account (e.g. `OFFICECORP\Administrator`) instead of the local one.

## Step 4 — Create Organizational Units (OUs)
1. Server Manager → **Tools** → **Active Directory Users and Computers**.
2. Right-click your domain name → **New > Organizational Unit**.
3. Create OUs to organize objects, e.g.:
   - `Office Computers`
   - `Office Users`
4. You'll target Group Policies at these OUs in later steps.

## Step 5 — Join office PCs to the domain
On each office PC:
1. Settings → System → About → **Domain or workgroup** (or Control Panel → System) → **Change settings** → **Change...**
2. Select **Domain**, enter your domain name (e.g. `officecorp.local`).
3. Enter domain admin credentials when prompted.
4. Reboot the PC.
5. Back on the server, in Active Directory Users and Computers, find the PC under **Computers** and move it into your **Office Computers** OU.

## Next steps
- `04-Block-USB-GPO.md`
- `05-Restrict-Website-Access.md`
- `06-Print-Tracking.md`
