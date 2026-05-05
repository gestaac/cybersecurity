# 22 — Day 1 PM (MA2) — WINSRV1 (AD GPOs, Share, Audit)

**Target time:** 60 min.
**Owner:** Person B (after `21_…`).
**Login:** WINSRV1 console as `MANILA\Administrator / P@ssw0rd`. Tools used: *Group Policy Management*, *Active Directory Users & Computers*, *Server Manager → File Services*, PowerShell.

> Marks at stake (Crit A4): rows 80–89 → **K total ≈ 2.3 directly**, plus enables A6/A7 Client-side checks worth ~3 more.
>
> ⚠️ **Plus** four MA2 deliverables that have NO marking row (domain pwd policy, fine-grained pwd policy, control GPO, registry GPO). Do them anyway — small time, big risk if marking scheme gets patched.

---

## Step 1 — Domain Password Policy (8-char, monthly)

> **Note:** MA2 line 198 requires this; marking scheme has no aspect. Do it anyway.

**Where:** *Group Policy Management → Forest → manila.com → Default Domain Policy → Edit*
**Path:** *Computer Configuration → Policies → Windows Settings → Security Settings → Account Policies → Password Policy*

| Setting | Value |
|---|---|
| Enforce password history | 24 |
| Maximum password age | 30 days |
| Minimum password age | 1 day |
| Minimum password length | 8 |
| Password must meet complexity requirements | Enabled |
| Store passwords using reversible encryption | Disabled |

Apply, then `gpupdate /force`.
**Marks:** *(no aspect — but required.)*

---

## Step 2 — Fine-Grained Password Policy (10-char for `executive`)

> Required by MA2 line 199; no aspect.

**Tools:** *Active Directory Administrative Center (DSAC)* → *manila* → *System → Password Settings Container*.

- *New → Password Settings*
- Name: `executive-PSO`
- Precedence: 10
- Min length: 10
- Apply To: group `executive`
- Save.

Verify:
```powershell
Get-ADFineGrainedPasswordPolicy executive-PSO
```

---

## Step 3 — Login banner GPO (title + text)

**Where:** *GPM → manila.com → New GPO → "LoginBanner"* → Edit.
**Path:** *Computer Configuration → Policies → Windows Settings → Security Settings → Local Policies → Security Options*

| Setting | Value |
|---|---|
| Interactive logon: Message title for users attempting to log on | `WorldSkills ASEAN Manila` |
| Interactive logon: Message text for users attempting to log on | `authorized access only` |

> ⚠️ Marking scheme G119 has the wrong title ("WorldSkills Lyon"). The MA2 project is the authoritative source — type **WorldSkills ASEAN Manila**.

Link the GPO to the domain root.
**Marks:** [Crit A6 D119 K=0.3] verified at Client login.

---

## Step 4 — `control` GPO (block Control Panel for accounting)

> Required (MA2 line 202); no marking aspect.

- New GPO `control` → link to OU containing accounting users (or filter by group).
- *User Configuration → Policies → Administrative Templates → Control Panel → Prohibit access to Control Panel and PC settings* → Enabled.
- Security filtering: remove *Authenticated Users*, add *accounting*.

---

## Step 5 — `registry` GPO (block reg tools for Manila)

> Required (MA2 line 203); no marking aspect.

- New GPO `registry`.
- *User Configuration → Policies → Administrative Templates → System → Prevent access to registry editing tools* → Enabled.
- Security filtering: remove *Authenticated Users*, add *Manila*.

---

## Step 6 — `google` GPO (Chrome enterprise homepage)

### 6.1 Extract & install ADMX/ADML
1. Copy `googleChromeEnterpriseBundle64.zip` from `C:\Users\Administrator\Documents\` to `C:\Temp\`.
2. Right-click → Extract All.
3. Open `Configuration\admx\`. Copy `chrome.admx` and `google.admx` to `C:\Windows\PolicyDefinitions\`. Copy `en-US\*.adml` files to `C:\Windows\PolicyDefinitions\en-US\`.

### 6.2 Create the GPO
- New GPO `google` → link to manila.com root.
- *Computer Configuration → Policies → Administrative Templates → Google → Google Chrome → Startup, Home page and New tab page*
  - "Configure the home page URL" → Enabled → URL: `http://w3.manila.com`
  - "Show Home button on toolbar" → Enabled
  - "Action to take on startup" → Enabled → "Open a list of URLs"
  - "URLs to open on startup" → `http://w3.manila.com`

**Marks:** [Crit A4 D81 K=0.3] google policy exists at domain level; [Crit A6 D117/D118 K=0.4] verified at Client.

---

## Step 7 — `certenroll` GPO (autoenroll certificates)

- New GPO `certenroll` → link to manila.com root.
- *Computer Configuration → Policies → Windows Settings → Security Settings → Public Key Policies*
  - *Certificate Services Client – Auto-Enrollment* → Enabled, both checkboxes ticked (Renew expired… + Update certificates that use templates).
- *User Configuration → Policies → Windows Settings → Security Settings → Public Key Policies* → same.

> Cert template "Workstation Authentication" or "Computer" must be set to **Auto-enroll** in WINSRV3 CA console: *Certificate Templates → Manage → Workstation Authentication → Properties → Security → Domain Computers Read+Enroll+Autoenroll*.

**Marks:** [Crit A4 D80 K=0.4] certenroll exists/autoenroll; [Crit A7 D116 K=0.3] verified.

---

## Step 8 — `pictures` share (AGLP best practice)

### 8.1 Create the folder + share
On WINSRV1:
```powershell
mkdir C:\shares\pictures -Force
New-SmbShare -Name "pictures" -Path "C:\shares\pictures" `
   -FullAccess "MANILA\IT" `
   -ChangeAccess "MANILA\Graphics" `
   -ReadAccess "MANILA\customer service"
```

### 8.2 NTFS permissions (the part judges check most)
```powershell
$acl = Get-Acl C:\shares\pictures
# Disable inheritance + remove existing
$acl.SetAccessRuleProtection($true,$false)
foreach ($r in $acl.Access) { $acl.RemoveAccessRule($r) | Out-Null }

function Add-Ace($identity, $rights) {
  $rule = New-Object System.Security.AccessControl.FileSystemAccessRule(
    $identity, $rights, "ContainerInherit,ObjectInherit", "None", "Allow")
  $acl.AddAccessRule($rule)
}

Add-Ace "MANILA\IT"               "FullControl"
Add-Ace "MANILA\Graphics"         "Modify"
Add-Ace "MANILA\customer service" "ReadAndExecute"
Add-Ace "SYSTEM"                  "FullControl"
Add-Ace "MANILA\Domain Admins"    "FullControl"

Set-Acl C:\shares\pictures $acl
```

> **Best-practice tips that earn the Judg points** (D89):
> - Inheritance disabled.
> - No "Authenticated Users" or "Everyone".
> - No explicit Deny.
> - Permissions on **NTFS**, not on the share.
> - Follows AGLP (Account → Global → Local → Permission) — we put permissions on the AD groups directly.

**Marks:** [Crit A4 D82 K=0.2] share exists; [Crit A4 D83 K=0.2] perms work; [Crit A4 D89 K=0.5–0.7 Judg].

### 8.3 Place `manila.jpg`
```powershell
Copy-Item .\manila.jpg C:\shares\pictures\manila.jpg -Force
# (or generate one)
[byte[]](0..255) | Set-Content C:\shares\pictures\manila.jpg -Encoding Byte
```

> ⚠️ Marking scheme H120 says "france.jpg" — ignore. The file must be `manila.jpg` per MA2 line 207.

---

## Step 9 — Audit `manila.jpg` access

### 9.1 Enable Object Access auditing in GPO
- New GPO `audit-share` → link to root.
- *Computer Configuration → Policies → Windows Settings → Security Settings → Advanced Audit Policy Configuration → Object Access* → "Audit File System" → Success + Failure.

### 9.2 SACL on the file
```powershell
$audit = New-Object System.Security.AccessControl.FileSystemAuditRule(
  "Everyone", "ReadData", "None", "None", "Success,Failure")
$acl = Get-Acl C:\shares\pictures\manila.jpg
$acl.AddAuditRule($audit)
Set-Acl C:\shares\pictures\manila.jpg $acl
```
**Verify:** from Client1 as `gfxguy / P@ssw0rd` → `\\winsrv1\pictures\manila.jpg` → open → on WINSRV1 *Event Viewer → Security* → Event ID 4663 should appear.

**Marks:** [Crit A7 D121 K=0.5] auditing logs read; [Crit A7 D120 K=0.3] share accessible.

---

## Step 10 — Table 2 (GPO Recommendations)

> Marking scheme A4 D84 (K=0.7 Judg, max=3) wants **3 recommended GPOs** beyond what was assigned, with rationale.

Add to MA2 Appendix Table 2 the following three (defensible, well-known, low-risk):

| Policy | Path | Effect | Why this over alternatives |
|---|---|---|---|
| **Account Lockout** (5 attempts / 15 min lockout / 15 min reset) | Computer Config → Policies → Windows Settings → Security Settings → Account Policies → Account Lockout Policy | Slows password-guessing attacks against domain accounts. | Cheap to deploy, no user-experience cost beyond locked-out users; complements password complexity. |
| **Disable LLMNR/NBT-NS** | Computer Config → Policies → Admin Templates → Network → DNS Client → "Turn off multicast name resolution" Enabled | Stops the most common AD credential-stealing technique (Responder/Inveigh poisoning). | Targets a known attack vector with virtually no business cost (DNS still works). |
| **AppLocker default rules in Audit then Enforce** | Computer Config → Policies → Windows Settings → Security Settings → Application Control Policies → AppLocker | Blocks unsigned/unknown executables — strong defense vs malware droppers. | Preferred over SRP because per-user rules; "audit first" lowers business risk during rollout. |

> Save Table 2 in the appendix doc on the desktop along with MA1's vulnerabilities. Final filename: `PHL_Team1_Day1_Appendix.pdf`.

**Marks:** [Crit A4 D84 K=0.7 Judg max 3] + [Crit A6 D107 K=0.6 Judg max 3] (when verified at clients).

---

## Snapshot & sanity

```powershell
gpresult /h C:\Temp\gp.html
# open in browser → verify all GPOs present and in scope
```

Take a snapshot `WINSRV1-policies-applied`.

---

## Mark map for this file

| Aspect | K | Step |
|---|---|---|
| A4 D80 certenroll GPO | 0.4 | 7 |
| A4 D81 google GPO | 0.3 | 6 |
| A4 D82 share exists | 0.2 | 8.1 |
| A4 D83 share perms | 0.2 | 8.2 |
| A4 D84 Table 2 GPO recs (Judg) | 0.7 | 10 |
| A4 D89 share best-practice (Judg) | 0.5 | 8.2 |
| **Direct total** | **2.3** | |
| (also enables A6/A7 D107/D116/D117/D118/D119/D120/D121 ≈ 2.6) | | |

Next file: **`23_Day1_MA2_PKI.md`**.
