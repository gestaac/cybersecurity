# 30 — Day 1 PM (MA2) — Functional Verification from Clients

**Target time:** 45 min.
**Run together** at the end of MA2. Both teammates execute the checks from the appropriate client. **If any step fails, the marking row fails — fix and re-test.**

> Marks at stake: **A6 (Client1) + A7 (Client2) + A8 (Client3) ≈ 8.45 K** = the largest single-file scoring opportunity in MA2.

---

## Pre-flight (before starting verification)

- [ ] pfSense rules + DHCP saved
- [ ] LinSRV1 sshd on 2022, https serving manila.crt
- [ ] WINSRV1 GPOs linked, share created
- [ ] WINSRV3 CA running, templates issued
- [ ] Client1, Client2 domain-joined to manila.com
  ```powershell
  Add-Computer -DomainName manila.com -Credential (Get-Credential) -Restart
  ```
- [ ] Client3 has DHCP from ISP (10.0.0.x)
- [ ] All clients run `gpupdate /force` once before verification

---

## A6 — Tests from Client1 (LAN)

Log on as `MANILA\Anorbert` / `P@ssw0rd` unless told otherwise.

### A6.1 Reach allowed Internet site
**Tools:** Chrome.
- Visit `http://www.nationalmuseum.gov.ph` → page loads.
**Marks:** [Crit A6 D97 K=0.3]

### A6.2 Restricted Internet site blocked
- Visit `http://www.starcity.com.ph` → connection fails / page does not load.
- (If using DNS sinkhole, browser shows "site can't be reached".)
**Marks:** [Crit A6 D98 K=0.3]

### A6.3 DHCP from pfSense
**Tools:** cmd.
```cmd
ipconfig /release
ipconfig /renew
ipconfig /all
```
**Expected:** IP in 172.16.100.50–150, DNS 192.168.2.10, Gateway 172.16.100.254, Domain `manila.com`.
**Marks:** [Crit A6 D99 K=0.2]

### A6.4 SSH to LinSRV1 as C1, port 2022
**Tools:** PuTTY → host `192.168.1.10` port `2022` user `C1@manila.com` password `P@ssw0rd` (or local C1).
**Expected:** prompt `[C1@LinSRV1 ~]$`.
**Marks:** [Crit A6 D100 K=0.4]

### A6.5 sudo as C1
```bash
sudo useradd fred
```
**Expected:** prompt for C1 password, then user added (no errors).
**Marks:** [Crit A6 D101 K=0.3]

### A6.6 HTTPS to LinSRV1 over firewall
- Chrome → `https://www.manila.com` → page loads.
**Marks:** [Crit A6 D102 K=0.3]

### A6.7 PKI chain check
- Click padlock → *Connection is secure → Certificate is valid* → chain shows `Manila-Root-CA → WINSRV3 → www.manila.com`.
**Marks:** [Crit A6 D103 K=0.4]

### A6.8 GPO certenroll worked
```cmd
gpresult /v | findstr /i certen
certutil -store -user My
```
**Expected:** `certenroll` listed; user store has at least one cert from WINSRV3-CA.
**Marks:** [Crit A6 D104 K=0.5]

### A6.9 SNORT logged web traffic to Internet
- On pfSense WebUI as admin/`P@ssw0rd`: *Services → Snort → Alerts*.
- Recent entries should show traffic from 172.16.100.x to 10.0.0.10/.20.
**Marks:** [Crit A6 D105 K=0.5]

### A6.10 SNORT XMAS rule present
- *Snort → WAN tab → Rules → custom.rules* → SID 100001 enabled.
**Marks:** [Crit A6 D106 K=0.3]

### A6.11 Banner before logon
- Log out → log back in → banner title `WorldSkills ASEAN Manila`, body `authorized access only` appears before credential entry.
**Marks:** [Crit A6 D119 K=0.3] (this is technically an A7 row in the scheme but tested at any client)

---

## A7 — Tests from Client2 (LAN)

Log on as `MANILA\mratt` / `P@ssw0rd`.

### A7.1 SSH from putty as a domain user
- PuTTY → `mratt@manila.com@192.168.1.10:2022`.
**Expected:** login succeeds.
**Marks:** [Crit A7 D113 K=0.3]

### A7.2 https://webtest.manila.com (WINSRV3 IIS) chain check
- Chrome → `https://webtest.manila.com` → no warning, chain Manila-Root-CA → WINSRV3 → webtest.
**Marks:** [Crit A7 D114 K=0.3]

### A7.3 DNS check from CLI
```cmd
ping www.manila.com
nslookup www.manila.com
```
**Expected:** resolves to `192.168.1.10`.
**Marks:** [Crit A7 D115 K=0.2]

### A7.4 GPO scope of certenroll
```cmd
gpresult /v | findstr "certen"
```
**Expected:** `certenroll` GPO applied.
**Marks:** [Crit A7 D116 K=0.3]

### A7.5 Chrome homepage forced to w3.manila.com
- Open Chrome → cold-start opens `http://w3.manila.com` (or attempts to). Try changing homepage in settings → relaunch → setting reverts.
**Marks:** [Crit A7 D117 K=0.2] + [D118 K=0.2]

### A7.6 Banner before logon
- Same banner test as A6.11 if not already covered.
**Marks:** [Crit A7 D119 K=0.3]

### A7.7 Share access as `gfxguy` (Modify)
- Sign out → Sign in as `MANILA\gfxguy` / `P@ssw0rd`.
- Win+R → `\\winsrv1\pictures\manila.jpg` → file opens.
- Right-click file → Save As → save back to share → succeeds (Modify rights).
**Marks:** [Crit A7 D120 K=0.3]

### A7.8 Audit log on WINSRV1
- On WINSRV1 → *Event Viewer → Security* → filter Event ID 4663 → see `gfxguy` reading `manila.jpg`.
**Marks:** [Crit A7 D121 K=0.5]

---

## A8 — Tests from Client3 (Internet, external client)

Client3 has only DHCP from ISP and **no domain membership**. Use local Administrator account.

### A8.1 OpenVPN dial-in as `vpnuser`
- Open OpenVPN Connect → import the `.ovpn` file from `20_…` Step 5.5.
- Connect → enter `vpnuser` / `P@ssw0rd` (LDAP backend on pfSense queries WINSRV1 for VPN Users group).
**Expected:** tunnel green; tunnel IP from 10.8.0.0/24.
**Marks:** [Crit A8 D123 K=0.5]

### A8.2 DNS over VPN
```cmd
nslookup www.manila.com
```
**Expected:** resolves to internal LinSRV1 (192.168.1.10) once VPN routes the query.
*Or* per the marking text: "should resolve to external interface IP of firewall" if you opted for split DNS that returns the public IP.
**Marks:** [Crit A8 D124 K=0.3]

### A8.3 Reach the DMZ website
- Chrome → `https://www.manila.com` → loads with valid cert.
**Marks:** [Crit A8 D125 K=0.5]

### A8.4 XMAS scan triggers Snort
**Tools:** Zenmap (Nmap GUI) on Client3.
- Disconnect VPN first (we want this to come from "Internet").
- Run: `nmap -sX -p 1-1024 <pfSense WAN IP>`
**Expected on pfSense:** *Services → Snort → Alerts* shows `Possible XMAS scan` SID 100001.
**Marks:** [Crit A8 D126 K=0.4]

---

## Quick fail-fix table

| Symptom | Likely cause | Fix |
|---|---|---|
| Client1 can't get DHCP | pfSense DHCP not enabled on LAN | `20_…` Step 2 |
| LinSRV1 SSH refused | sshd port 2022 not in firewalld or selinux | `21_…` Step 3 + 4 |
| `https://www.manila.com` cert warning | Client trust root missing | Either GPO root push (autoenroll) or `Computer Cert Store → Trusted Root → import` Manila-Root-CA |
| Share path 0x80004005 | NTFS perms blocking, or SMB signing mismatch | Re-check `Get-Acl C:\shares\pictures` |
| Snort alerts page empty | Snort interface not started (▶ button) | `20_…` Step 6.5 |
| OpenVPN auth fail | LDAP backend filter wrong | re-check User Manager → LDAP test |

---

## Mark map for this file

| Aspect | K | |
|---|---|---|
| A6 D97–D107 | 4.0 (incl. judg D107=0.6) | LAN test set |
| A7 D113–D121 | 2.45 | LAN test #2 |
| A8 D123–D126 | 1.7 | Internet/VPN |
| **Total** | **~8.15 + 0.3 banner** | |

If Day 1 totals (A1+A2+A3+A4+A5+A6+A7+A8) are tracked correctly you should be on pace for **23–25 / 25 of Criterion A**.

Next file: **`40_Day2_CTF_Playbook.md`**.
