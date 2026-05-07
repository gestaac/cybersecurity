# 22 — Day 2 (MA2) — WINSRV1 (AD GPOs, Share, Audit)

**Target time:** 75 min.
**Owner:** Person B (after `21_…`).
**Login:** WINSRV1 console as `MANILA\Administrator / P@ssw0rd`. Tools used: *Group Policy Management*, *Active Directory Users & Computers*, *Server Manager → File Services*.

> Source of truth for this file: **`testpacakge_pdf/WSA2025_TP54_MA2_actual_en_final (1).pdf` page 11–12 (WINSRV1 section)**. The original docx version had different GPO names; the PDF is now the canonical task list.

> All deliverables in this file are explicitly listed in the MA2 PDF. Each step earns marks (the marking-scheme row names are Lyon-leftovers and don't match — but the chief evaluates against the PDF).

---

## Step 1 — Domain Password Policy (8-char + history of 30)

**Per MA2 PDF page 11:** *"Create a password policy that requires all domain users' passwords to be 8 characters in length, and keep a history of 30 past passwords."*

**Where:** *Group Policy Management → Forest → manila.com → Default Domain Policy → Edit*
**Path:** *Computer Configuration → Policies → Windows Settings → Security Settings → Account Policies → Password Policy*

| Setting | Value |
|---|---|
| **Enforce password history** | **30 passwords remembered** |
| Maximum password age | 30 days |
| Minimum password age | 1 day |
| **Minimum password length** | **8** |
| Password must meet complexity requirements | Enabled |
| Store passwords using reversible encryption | Disabled |

Apply, then in PowerShell: `gpupdate /force`.

**Verify:** `net accounts` on any domain client → "Length of password history maintained: 30".

---

## Step 2 — Fine-Grained Password Policy — 16-char for Executive

**Per MA2 PDF page 11:** *"Create a fine-grained password policy that requires members of the executive group to have a 16-character long password. Change password to `P@ssw0rdP@ssw0rd` during testing for one member of executive group."*

**Tools:** *Active Directory Administrative Center (DSAC)* → *manila (local)* → *System → Password Settings Container*.

- Right-click → *New → Password Settings*.
- *Name:* `Executive-PSO`
- *Precedence:* `10`
- ☑ *Enforce minimum password length:* **16**
- ☑ *Enforce password history:* 30
- ☑ *Password must meet complexity requirements*
- *Directly Applies To:* click **Add** → search → select group **Executive** → OK.
- OK to save.

**Test (per PDF):** pick one Executive member (e.g., M004).
- Right-click M004 → Reset Password → set to **`P@ssw0rdP@ssw0rd`** (exactly 16 chars).
- UNTICK *"User must change password at next logon"*.
- OK.

**Verify:**
```powershell
Get-ADFineGrainedPasswordPolicy Executive-PSO
Get-ADUserResultantPasswordPolicy M004
# Should return Executive-PSO, MinPasswordLength=16
```

---

## Step 3 — Login banner GPO (title + text)

**Per MA2 PDF page 11:**
- *"Create a login banner/title that says 'WorldSkills ASEAN Manila'"*
- *"Create a login banner/text that says 'Authorized access only'"*

**Where:** *GPM → manila.com → New GPO → name `LoginBanner` → Edit*.
**Path:** *Computer Configuration → Policies → Windows Settings → Security Settings → Local Policies → Security Options*

| Setting | Value |
|---|---|
| Interactive logon: Message title for users attempting to log on | `WorldSkills ASEAN Manila` |
| Interactive logon: Message text for users attempting to log on | `Authorized access only` |

Link the GPO to the **manila.com** domain root.

**Verify at Client1:** sign out → see banner → click OK → sign-in proceeds.

---

## Step 4 — `lockout` GPO (3 attempts / 60-second lockout)

**Per MA2 PDF page 11:** *"Create a GPO called 'lockout' that will lock accounts after 3 failed logon attempts for all domain users. Account duration lockout is 60 seconds."*

**Where:** *GPM → manila.com → New GPO → name `lockout` → Edit*.
**Path:** *Computer Configuration → Policies → Windows Settings → Security Settings → Account Policies → Account Lockout Policy*

| Setting | Value |
|---|---|
| Account lockout threshold | **3 invalid logon attempts** |
| Account lockout duration | **1 minute** (= 60 seconds) |
| Reset account lockout counter after | 1 minute |

> ⚠️ **Important:** Account Lockout Policy must be set in the **Default Domain Policy** OR a GPO linked to the domain root for it to apply to **all domain users**. Linking it to an OU only applies to computer accounts in that OU, NOT user accounts. The PDF says "all domain users" — link to the domain root.

Link `lockout` GPO to **manila.com** root.

**Verify:** from Client1, attempt to log in as M001 with wrong password 3 times → account locked. Wait 60 sec → can log in again.

---

## Step 5 — `restrict control panel` GPO (everyone except Executive)

**Per MA2 PDF page 11:** *"Create a GPO called 'restrict control panel' that will restrict access to the control panel — which is only applicable to all users, except for the executive group."*

**Where:** *GPM → manila.com → New GPO → name `restrict control panel` → Edit*.
**Path:** *User Configuration → Policies → Administrative Templates → Control Panel*

| Setting | Value |
|---|---|
| **Prohibit access to Control Panel and PC settings** | **Enabled** |

Link the GPO to **manila.com** root (so it applies domain-wide).

**Security filtering — exclude Executive:**
1. Click the GPO → *Delegation* tab → **Advanced**.
2. Add the **Executive** group → set to *Read* + **Apply group policy = Deny** (deny takes precedence).
3. Keep *Authenticated Users* with *Apply* (so it applies to everyone else).

**Verify:**
- Login as M001 (Marketing) → *Settings* / Control Panel → blocked.
- Login as M004 (Executive) → Control Panel → accessible.

---

## Step 6 — `disabled add and remove program panel` GPO (Executive only)

**Per MA2 PDF page 11:** *"Create a GPO called 'disabled add and remove program panel' that does not allow executive group to use control panel to add new program, uninstall or change a program."*

**Where:** *GPM → manila.com → New GPO → name `disabled add and remove program panel` → Edit*.
**Path:** *User Configuration → Policies → Administrative Templates → Control Panel → Add or Remove Programs*

| Setting | Value |
|---|---|
| **Remove Add or Remove Programs** | **Enabled** |
| Hide Add New Programs page | Enabled |
| Hide Change or Remove Programs page | Enabled |
| Hide Add/Remove Windows Components page | Enabled |

> Modern Windows uses *Settings → Apps* instead of legacy "Add/Remove Programs". Also enable:
> *User Configuration → Admin Templates → Windows Components → Settings page* → **Hide app pages** (lists `installed-apps`, `installed-apps-features`, etc.).

Link the GPO to **manila.com** root.

**Security filtering — apply ONLY to Executive:**
1. *Scope* tab → remove *Authenticated Users*.
2. **Add → Executive** group.

**Verify:**
- Login as M004 (Executive) → try to open *Settings → Apps* → blocked / page hidden.
- Login as M001 (Marketing) → *Settings → Apps* → opens normally.

---

## Step 7 — `autolock` GPO (Executive only, 10 sec inactivity)

**Per MA2 PDF page 11:** *"Create a GPO called 'autolock' that will auto lock screen after 10 seconds of inactivity — only applicable to the executive group."*

**Where:** *GPM → manila.com → New GPO → name `autolock` → Edit*.
**Path:** *Computer Configuration → Policies → Windows Settings → Security Settings → Local Policies → Security Options*

| Setting | Value |
|---|---|
| Interactive logon: Machine inactivity limit | **10** seconds |

> The Machine inactivity setting is computer-side, but the PDF says "only applicable to the executive group". The cleanest GUI implementation:
> - Set the GPO at *Computer Configuration* level above.
> - Apply security filtering by **Executive** group → in WMI filter or via "Loopback Processing" / item-level targeting.
>
> **Simpler practical approach (recommended for 10-second deadline):**
> - Use *User Configuration → Policies → Administrative Templates → Control Panel → Personalization → "Screen saver timeout"* → **10 seconds**, plus
> - *User Configuration → Admin Templates → Control Panel → Personalization → "Password protect the screen saver"* → **Enabled**, plus
> - *User Configuration → Admin Templates → Control Panel → Personalization → "Force specific screen saver"* → **Enabled**, value `scrnsave.scr`.

Link to manila.com root.

**Security filtering — apply ONLY to Executive:**
- *Scope* → remove *Authenticated Users*, add **Executive**.

**Verify:** login as M004 → idle for 10 seconds → screen locks. Login as M001 → idle 10 sec → screen does NOT lock.

---

## Step 8 — `certenroll` GPO (autoenroll certs)

**Per MA2 PDF page 11:** *"Create a new Group Policy Object called 'certenroll' so that computers on the domain will automatically receive a certificate from the issuing CA through application of the GPO. These GPO's should be set to autoenroll."*

- *GPM → New GPO → `certenroll` → Edit*.
- *Computer Configuration → Policies → Windows Settings → Security Settings → Public Key Policies → Certificate Services Client – Auto-Enrollment*:
  - Configuration Model: **Enabled**
  - ☑ Renew expired certificates, update pending certificates, and remove revoked certificates
  - ☑ Update certificates that use certificate templates
- Same under *User Configuration → Policies → Windows Settings → Security Settings → Public Key Policies*.

> On WINSRV3 CA console: *Certificate Templates → Manage* → duplicate **Workstation Authentication** → name it `Workstation-AutoEnroll` → Security tab → add **Domain Computers** with Read+Enroll+Autoenroll. Then *New → Certificate Template to Issue* → choose `Workstation-AutoEnroll`.

Link `certenroll` GPO to manila.com root.

---

## Step 9 — `pictures` share (Marketing=R, Executive=FC)

**Per MA2 PDF page 12:** *"Create a share on WinSRV1 following best practices at the local path C:\\shares\\pictures shared as 'pictures' that will allow the Read access (R) to the Marketing group, Full Control (FC) for Executive group, no access for anyone else."*

> ⚠️ **This is different from the older docx**: groups are now **Marketing** + **Executive** (was customer service / Graphics / IT). Use the new ones from the PDF.

### 9.1 — Create the folder + share
On WINSRV1:
```powershell
mkdir C:\shares\pictures -Force
New-SmbShare -Name "pictures" -Path "C:\shares\pictures" `
   -FullAccess "MANILA\Executive" `
   -ReadAccess "MANILA\Marketing"
```

### 9.2 — NTFS permissions (best-practice)
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

Add-Ace "MANILA\Executive"     "FullControl"
Add-Ace "MANILA\Marketing"     "ReadAndExecute"
Add-Ace "SYSTEM"               "FullControl"
Add-Ace "MANILA\Domain Admins" "FullControl"

Set-Acl C:\shares\pictures $acl
```

### 9.3 — Place `park.jpg` in the share
**Per MA2 PDF page 12:** *"In the C:\\shares\\pictures folder, you will find a picture called 'park.jpg'..."*

```powershell
# Place a real picture if you have one; otherwise generate:
[byte[]](0..255) | Set-Content C:\shares\pictures\park.jpg -Encoding Byte
```

> ⚠️ Filename is **`park.jpg`** in the PDF (was `manila.jpg` / `france.jpg` in older versions). Use **park.jpg**.

---

## Step 10 — Audit `park.jpg` access

**Per MA2 PDF page 12:** *"set auditing on this file so it is logged when read by a member of any group."*

### 10.1 — Enable Object Access auditing in GPO
- *GPM → manila.com → New GPO → `audit-share` → Edit*.
- *Computer Configuration → Policies → Windows Settings → Security Settings → Advanced Audit Policy Configuration → Object Access* → "Audit File System" → ☑ Success ☑ Failure.
- Link to manila.com root.

### 10.2 — SACL on the file
```powershell
$audit = New-Object System.Security.AccessControl.FileSystemAuditRule(
  "Everyone", "ReadData", "None", "None", "Success,Failure")
$acl = Get-Acl C:\shares\pictures\park.jpg
$acl.AddAuditRule($audit)
Set-Acl C:\shares\pictures\park.jpg $acl
```

**Verify from Client1:** login as M001 (Marketing) → `\\winsrv1\pictures\park.jpg` → file opens. On WINSRV1 *Event Viewer → Security* → Event ID 4663 should appear with Subject = M001, Object = park.jpg, Access = ReadData.

---

## Step 11 — Table 2 (GPO Recommendations — top 3)

**Per MA2 PDF page 12:** *"The client would like you to recommend three other GPOs which should be created to help better secure the domain."*

The PDF table has four columns: *Name of Policy / Path to Setting / Effects of Applying / Why over Other Choices*. Three solid recommendations (defensible, low-risk):

| Name | Path | Effects | Why this over alternatives |
|---|---|---|---|
| **Disable LLMNR/NBT-NS** | Computer Config → Policies → Admin Templates → Network → DNS Client → "Turn off multicast name resolution" Enabled | Blocks the most common AD credential-theft attack (Responder/Inveigh poisoning) by removing fallback name resolution | Targets a known, high-impact attack vector with virtually no business cost. DNS still works for legitimate lookups. |
| **AppLocker default rules in Audit mode** | Computer Config → Policies → Windows Settings → Security Settings → Application Control Policies → AppLocker | Inventories every executable that runs across the domain; can be promoted to Enforce later | Preferred over Software Restriction Policies (SRP) because AppLocker supports per-user rules and modern publishers. Audit-first lowers business risk during rollout. |
| **SMB Signing required for client + server** | Computer Config → Policies → Windows Settings → Security Settings → Local Policies → Security Options → "Microsoft network client/server: Digitally sign communications (always)" → Enabled | Prevents SMB relay attacks (NTLM relay to AD/file shares — common privilege escalation path) | Mandatory signing closes the SMB relay window; alternative is requiring Kerberos-only which is more disruptive. SMB signing has minor (1–5%) perf cost — acceptable in a domain. |

Save Table 2 + the executive summary into a single PDF on the desktop with country code.

---

## Final verification

```powershell
gpresult /h C:\Temp\gp.html
# Open in browser → verify all GPOs present:
#   - Default Domain Policy (password 8-char + history 30)
#   - LoginBanner
#   - lockout
#   - restrict control panel
#   - disabled add and remove program panel
#   - autolock
#   - certenroll
#   - audit-share
#   - Executive-PSO (fine-grained, applies to M004 + S001)
```

**Snapshot:** `WINSRV1-policies-applied`.

---

## Mark map for this file

| Deliverable | Step | K (best-guess vs Lyon-leftover scheme) |
|---|---|---|
| Domain pwd policy 8-char + history 30 | 1 | enables A4/A7 |
| FGPP 16-char Executive | 2 | enables A4/A7 |
| LoginBanner | 3 | [Crit A6/A7 D119 K=0.3] |
| lockout GPO | 4 | enables A4 |
| restrict control panel GPO | 5 | enables A4 |
| disabled add/remove programs GPO | 6 | enables A4 |
| autolock GPO | 7 | enables A4 |
| certenroll GPO | 8 | [Crit A4 D80 K=0.4], enables A6 D104 |
| pictures share Marketing=R/Executive=FC | 9 | [Crit A4 D82/D83 K=0.4] |
| Audit park.jpg | 10 | [Crit A7 D121 K=0.5] |
| Table 2 GPO recommendations | 11 | [Crit A4 D84 K=0.7 Judg max 3] |
| **Direct A4 K total** | | **~2.3+** |

Next file: **`23_Day1_MA2_PKI.md`**.

---

