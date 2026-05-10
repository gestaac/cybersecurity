# 30 — Day 2 (MA2) — Functional Verification from Clients

**Target time:** 45 min.
**Run together** at the end of MA2. Both teammates execute the checks from the appropriate client. **If any step fails, the marking row fails — fix and re-test.**

> Marks at stake: **A6 (Client1) + A7 (Client2) + A8 (Client3) ≈ 8.45 K** = the largest single-file scoring opportunity in MA2.

---

## 📖 How to read this file (beginner orientation)

Every test below has the same structure:
- **Where to run it:** Client1, Client2, or Client3 (different VMs)
- **Login:** which AD user to log in as
- **Tools:** what app to open (Chrome, cmd, PuTTY, etc.)
- **Step-by-step clicks/commands**
- **What "success" looks like** (so you know when to take the screenshot)
- **Marks earned** (the row in the marking scheme)
- **If it fails:** one specific fix

📸 **Screenshot every successful test.** Judges may ask for proof.

### How to switch between Client VMs
1. Open VMware Workstation on PC1 (or PC2)
2. Top tab bar → click the tab labeled `Client1`, `Client2`, or `Client3`
3. If VM is powered off → click green **Power On**
4. Wait for Windows login screen
5. Login per the section below

### Common Windows actions you'll do repeatedly
- **Open cmd:** Press `Win+R` → type `cmd` → Enter
- **Open PowerShell:** Press `Win+R` → type `powershell` → Enter
- **Open Chrome:** Click Chrome icon on taskbar OR `Win+R` → type `chrome` → Enter
- **Open Event Viewer:** Press `Win+R` → type `eventvwr.msc` → Enter
- **Sign out:** Win+X → "Shut down or sign out" → Sign out
- **Switch user:** Win+L → click "Other user" → enter different credentials
- **Take screenshot:** Press `Windows+Shift+S` → drag a region → screenshot copies to clipboard, then Ctrl+V into a Word doc

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

> 🧠 **Switch to Client1 VM first.** Open VMware Workstation tab labeled `Client1`. Power on if needed. Wait for the login screen.

Log on as `MANILA\M001` / `P@ssw0rd` (Marketing user from MA2 PDF Table 3).
- At Windows login screen → click **Other user** (bottom-left) if M001 isn't shown
- User: `M001` (or `MANILA\M001` if domain prefix needed)
- Password: `P@ssw0rd`
- Click → wait for desktop to load

### A6.1 Reach allowed Internet site

**Why it earns marks:** firewall must allow LAN → Internet for the allowed site.

**Step-by-step:**
1. Open **Chrome** (taskbar icon, or Win+R → `chrome`)
2. In the URL bar, type: `http://www.nationalmuseum.gov.ph`
3. Press Enter

**✅ Success looks like:** the National Museum website loads with text + images.
**❌ If it fails:** "site can't be reached" → check pfSense LAN→WAN rule (Step 4.1 of `20_…`). Run `nslookup www.nationalmuseum.gov.ph` in cmd — should resolve to `10.0.0.10` (ISP).

📸 Screenshot the loaded page.

**Marks:** [Crit A6 D97 K=0.3]

### A6.2 Restricted Internet site blocked

**Why it earns marks:** firewall must block `www.starcity.com.ph` per project requirements.

**Step-by-step:**
1. Same Chrome → URL bar
2. Type: `http://www.starcity.com.ph` → Enter

**✅ Success looks like:** "This site can't be reached" / "ERR_CONNECTION_TIMED_OUT" / "Server IP address could not be found."
**❌ If it fails (page LOADS — that's bad):** the block isn't working. Re-check pfSense DNS Resolver host override (`20_…` Step 4.5) — should sinkhole `www.starcity.com.ph` to `127.0.0.1`.

📸 Screenshot the "site can't be reached" error.

**Marks:** [Crit A6 D98 K=0.3]

### A6.3 DHCP from pfSense

**Why it earns marks:** pfSense (not WINSRV1) must hand out LAN IPs.

**Step-by-step:**
1. Open cmd: Win+R → `cmd` → Enter
2. Type these commands one at a time:
   ```cmd
   ipconfig /release
   ipconfig /renew
   ipconfig /all
   ```

**✅ Success looks like (in `ipconfig /all` output):**
```
IPv4 Address. . . . . . . . . . . : 172.16.100.X     (X between 50 and 150)
Subnet Mask . . . . . . . . . . . : 255.255.255.0
Default Gateway . . . . . . . . . : 172.16.100.254   (the pfSense LAN IP)
DHCP Server . . . . . . . . . . . : 172.16.100.254   (pfSense, NOT WINSRV1)
DNS Servers . . . . . . . . . . . : 192.168.2.10     (WINSRV1)
Connection-specific DNS Suffix    : manila.com
```

**❌ If it fails:**
- DHCP from wrong server → check pfSense DHCP service is enabled on LAN (`20_…` Step 2)
- Times out → pfSense DHCP not running

📸 Screenshot the `ipconfig /all` output.

**Marks:** [Crit A6 D99 K=0.2]

### A6.4 SSH to LinSRV1 as C1, port 2022

**Why it earns marks:** sshd config must listen on 2022 with C1 allowed (`21_…` Step 3).

**Step-by-step:**
1. Open **PuTTY** (Start menu → PuTTY, OR Win+R → `putty`)
2. In the PuTTY config window:
   - **Host Name:** `192.168.1.10`
   - **Port:** `2022` (NOT 22!)
   - **Connection type:** SSH
3. Click **Open**
4. If "PuTTY Security Alert" pops up about a host key → click **Accept**
5. At the `login as:` prompt, type: `C1@manila.com` (or just `C1` if using local fallback user)
6. At the `password:` prompt, type: `P@ssw0rd` (won't show as you type — normal)
7. Press Enter

**✅ Success looks like:** prompt becomes `[C1@LinSRV1 ~]$` — you're logged in.

**❌ If it fails:**
- "Network error: Connection refused" → sshd not on 2022. SSH from console as root and check `cat /etc/ssh/sshd_config | grep Port` — must be `Port 2022`.
- "Permission denied (publickey,password)" → AllowUsers blocking. Check `cat /etc/ssh/sshd_config | grep AllowUsers` — must include `C1` and `C2`.
- Connection times out → firewalld blocking 2022. On LinSRV1 console: `sudo firewall-cmd --permanent --add-port=2022/tcp && sudo firewall-cmd --reload`

📸 Screenshot the PuTTY window with the `[C1@LinSRV1]$` prompt visible.

**Marks:** [Crit A6 D100 K=0.4]

### A6.5 sudo as C1

**Why it earns marks:** C1 must have sudo (per `21_…` Step 2).

**Step-by-step (continuing in the same PuTTY session from A6.4):**
1. At the `[C1@LinSRV1 ~]$` prompt, type:
   ```bash
   sudo useradd fred
   ```
2. If prompted `[sudo] password for C1:` → type `P@ssw0rd` (silent typing)
3. Press Enter

**✅ Success looks like:** no error message — back to `[C1@LinSRV1 ~]$` prompt. The user `fred` was added.

Verify:
```bash
id fred
# Output: uid=1002(fred) gid=1002(fred) groups=1002(fred)
```

**❌ If it fails:**
- "C1 is not in the sudoers file" → `21_…` Step 2 wasn't completed. SSH as root, run:
  ```bash
  echo "C1 ALL=(ALL) ALL" | sudo tee /etc/sudoers.d/c1c2
  echo "C2 ALL=(ALL) ALL" | sudo tee -a /etc/sudoers.d/c1c2
  sudo chmod 440 /etc/sudoers.d/c1c2
  ```

📸 Screenshot the sudo command + `id fred` output.

**Marks:** [Crit A6 D101 K=0.3]

### A6.6 HTTPS to LinSRV1 over firewall

**Why it earns marks:** firewall LAN→DMZ HTTPS must be open + LinSRV1 must serve HTTPS.

**Step-by-step:**
1. Open Chrome (or click the existing Chrome tab from A6.1)
2. URL bar → type: `https://www.manila.com` → Enter
3. If prompted to log in (HTTP basic auth from `21_…` Step 7.4):
   - Username: `M001` (or any domain user)
   - Password: `P@ssw0rd`

**✅ Success looks like:** the LinSRV1 website loads. Padlock icon in URL bar (no red warning).

**❌ If it fails:**
- "ERR_CONNECTION_REFUSED" → httpd not running on LinSRV1: `sudo systemctl start httpd`
- "ERR_CERT_AUTHORITY_INVALID" / red lock icon → root CA not trusted. Test A6.7 next.
- Auth keeps prompting → LDAP backend wrong. Check `21_…` Step 7.4 vhost config.

📸 Screenshot the loaded site WITH the padlock visible.

**Marks:** [Crit A6 D102 K=0.3]

### A6.7 PKI chain check (no certificate warning)

**Why it earns marks:** root CA from WINSRV3 must be trusted via GPO autoenroll → no cert warning at clients.

**Step-by-step:**
1. On the loaded `https://www.manila.com` page in A6.6
2. Click the **🔒 padlock** in the URL bar
3. Click **"Connection is secure"**
4. Click **"Certificate is valid"**
5. A "Certificate Viewer" window opens. Click the **Details** tab.
6. In the "Certificate Hierarchy" tree, verify the chain:
   ```
   Manila-Root-CA
     └─ WINSRV3-CA
        └─ www.manila.com
   ```

**✅ Success looks like:** all 3 certs in the chain shown, no red X marks anywhere.

**❌ If it fails:**
- Chain breaks at "Manila-Root-CA" with red X → root CA not trusted by Client1. Run `gpupdate /force` in cmd, then refresh page.
- Only `www.manila.com` shows (no chain) → LinSRV1 didn't include intermediate CA cert. Re-run `21_…` Step 7.3.

📸 Screenshot the Certificate Viewer's chain diagram.

**Marks:** [Crit A6 D103 K=0.4]

### A6.8 GPO certenroll worked

**Why it earns marks:** the `certenroll` GPO must auto-issue a cert to this client.

**Step-by-step:**
1. Open cmd
2. Type:
   ```cmd
   gpupdate /force
   ```
   Wait for "Computer Policy update has completed successfully."
3. Force cert enrollment:
   ```cmd
   certutil -pulse
   ```
4. Verify cert is in the personal store:
   ```cmd
   certutil -store -user My
   ```

**✅ Success looks like:** in the output of `certutil -store -user My`, at least one cert with:
- `Issuer: CN=WINSRV3-CA, ...`
- Subject contains your computer name OR M001

Also verify GPO is applied:
```cmd
gpresult /v | findstr /i certen
```
Output should show `certenroll` listed under "Applied Group Policy Objects".

**❌ If it fails:**
- No certs in user store → certenroll GPO not linked. On WINSRV1, GPM → manila.com → confirm certenroll GPO is linked to domain root.
- Cert template not available → on WINSRV3 CA, ensure `Workstation-AutoEnroll` template is published (`23_…` Step 2.1)

📸 Screenshot certutil output + gpresult output.

**Marks:** [Crit A6 D104 K=0.5]

### A6.9 SNORT logged web traffic to Internet

**Why it earns marks:** Snort must log all traffic going out to Internet (per `20_…` Step 6).

**Step-by-step:**
1. From Client1, open Chrome
2. URL bar → `https://172.16.100.254` (pfSense WebUI)
3. Login: `admin / P@ssw0rd` (or `pfsense` if you didn't change it — but you should have, per `20_…` Step 1)
4. Top menu → **Services** → **Snort**
5. Top tab inside Snort → **Alerts**
6. Look at recent entries

**✅ Success looks like:** entries showing source IP from `172.16.100.x` (your Client1 IP) going to `10.0.0.10` or `10.0.0.20` (the ISP-hosted websites). Timestamps from your recent A6.1 visit.

**❌ If it fails:**
- Empty alerts page → Snort not running on WAN. Snort → Snort Interfaces → WAN → click ▶ (Start)
- Alerts present but no recent entries → Snort logging level wrong. Snort → Global Settings → enable "Send alerts to system log"

📸 Screenshot the Alerts page with recent entries highlighted.

**Marks:** [Crit A6 D105 K=0.5]

### A6.10 SNORT FIN scan rule present

**Why it earns marks:** custom FIN scan rule (sid 100001) must be configured per MA2 PDF page 10.

**Step-by-step (still in pfSense WebUI from A6.9):**
1. Snort → **Snort Interfaces** tab → click **WAN** row's pencil edit icon
2. Top sub-tabs → **Rules**
3. Category dropdown → **custom.rules**
4. Look for the rule line:
   ```
   alert tcp any any <> $HOME_NET any (flags: F; msg: "Possible FIN scan"; sid: 100001;)
   ```
5. Confirm checkbox next to it is ☑ (enabled)

**✅ Success looks like:** the FIN scan rule is visible, checked, sid is `100001`.

**❌ If it fails (rule missing):**
- Snort → WAN → Rules → custom.rules → click Edit (top-right)
- Paste the rule above
- Save → restart Snort interface

> ⚠️ **Lyon-leftover trap:** marking row D106 says "XMAS scan." MA2 PDF page 10 says **FIN scan**. Use FIN. If chief Marlon directed you to follow the marking scheme literally, ALSO add a second rule for XMAS:
> ```
> alert tcp any any <> $HOME_NET any (flags: SRPFAU; msg: "Possible XMAS scan"; sid: 100002;)
> ```

📸 Screenshot the custom.rules page with the FIN rule visible.

**Marks:** [Crit A6 D106 K=0.3]

### A6.11 Banner before logon

**Why it earns marks:** the LoginBanner GPO (`22_…` Step 3) must show banner BEFORE login.

**Step-by-step:**
1. From Client1 desktop → Win+L (lock screen)
2. Click anywhere to dismiss the lock → you'll see a **banner pop-up**:
   - **Title bar:** `WorldSkills ASEAN Manila`
   - **Body text:** `Authorized access only`
   - **OK button** to dismiss
3. Click OK → THEN you see the login screen

**✅ Success looks like:** banner appears with EXACT text shown above. You must click OK to proceed.

**❌ If it fails:**
- No banner → LoginBanner GPO not linked. On WINSRV1 → GPM → confirm LoginBanner GPO is linked to manila.com root.
- Wrong text → re-edit GPO per `22_…` Step 3 — text is **case-sensitive**.
- Banner shows AFTER login → wrong path used. Must be at `Computer Configuration → Security Settings → Local Policies → Security Options`.

📸 Screenshot the banner with title + body visible.

**Marks:** [Crit A6/A7 D119 K=0.3] (this is an A7 row in the scheme but tested at any client)

---

## A7 — Tests from Client2 (LAN)

> 🧠 **Switch to Client2 VM.** VMware Workstation tab → Client2 → Power On → wait for login screen.

Log on as `MANILA\C2` / `P@ssw0rd` (IT user, Singapore — has SSH access on LinSRV1).
- Click **Other user**
- User: `C2` (or `MANILA\C2`)
- Password: `P@ssw0rd`

### A7.1 SSH as a domain user

**Why it earns marks:** Linux must accept domain users via SSH (per `21_…` Step 2).

**Step-by-step:**
1. Open PuTTY
2. **Host Name:** `192.168.1.10`, **Port:** `2022`, **Connection:** SSH
3. Click **Open**
4. `login as:` → `C2@manila.com`
5. `password:` → `P@ssw0rd`

**✅ Success looks like:** prompt becomes `[C2@manila.com@LinSRV1 ~]$` — domain credential accepted.

**❌ If it fails:**
- "Permission denied (publickey,password)" → domain join didn't work properly. On LinSRV1 console: `realm list` should show manila.com joined.
- "Login timed out" → DMZ→Servers AD ports rule missing. Re-check `20_…` Step 4.2.

📸 Screenshot the PuTTY prompt showing the C2@manila.com login.

**Marks:** [Crit A7 D113 K=0.3]

### A7.2 https://webtest.manila.com — IIS cert chain check

**Why it earns marks:** WINSRV3 IIS must serve a CA-signed cert (no warning at clients).

**Step-by-step:**
1. Open Chrome
2. URL bar → `https://webtest.manila.com` → Enter
3. Verify NO red warning, padlock icon visible
4. Click padlock → Connection is secure → Certificate is valid → check chain

**✅ Success looks like:**
- Page loads without "Your connection is not private" warning
- Padlock is closed/green
- Cert chain: `Manila-Root-CA → WINSRV3-CA → webtest.manila.com`

**❌ If it fails:**
- Cert warning → root CA not pushed via GPO. Run `gpupdate /force` and retry.
- "DNS_PROBE_FINISHED_NXDOMAIN" → DNS A record missing for webtest. On WINSRV1 → DNS Manager → manila.com zone → add A record `webtest → 192.168.2.30`.

📸 Screenshot the page WITH padlock visible + cert chain dialog.

**Marks:** [Crit A7 D114 K=0.3]

### A7.3 DNS check from CLI

**Why it earns marks:** WINSRV1 DNS must resolve internal hostnames.

**Step-by-step:**
1. Open cmd
2. Run:
   ```cmd
   ping www.manila.com
   nslookup www.manila.com
   ```

**✅ Success looks like:**
```
Pinging www.manila.com [192.168.1.10] with 32 bytes of data:
Reply from 192.168.1.10: bytes=32 time=1ms TTL=64
...

> nslookup www.manila.com
Server:  WINSRV1.manila.com
Address: 192.168.2.10
Name:    www.manila.com
Address: 192.168.1.10
```

**❌ If it fails:**
- nslookup fails → DNS A record missing on WINSRV1. Add it: WINSRV1 → DNS Manager → manila.com → New A record → `www → 192.168.1.10`.
- Ping replies but from wrong IP → conflicting DNS entries. Check DNS for duplicate `www` records.

📸 Screenshot both commands' output.

**Marks:** [Crit A7 D115 K=0.2]

### A7.4 GPO scope of certenroll at this client

**Step-by-step:**
1. cmd
2. Run:
   ```cmd
   gpupdate /force
   gpresult /v > %temp%\gpresult.txt
   notepad %temp%\gpresult.txt
   ```
3. In Notepad → Ctrl+F → search `certenroll` → confirm it's listed under "Applied Group Policy Objects"

**✅ Success looks like:** Notepad shows `certenroll` in the applied GPOs section.

**❌ If it fails (certenroll missing):**
- GPO not linked to manila.com → fix on WINSRV1 GPM
- Security filtering excludes Client2 → check GPO's Scope tab

📸 Screenshot the gpresult output showing certenroll.

**Marks:** [Crit A7 D116 K=0.3]

### A7.5 Lockout policy works (verifies `22_…` Step 4)

**Step-by-step:**
1. Win+L to lock the screen
2. Click "Other user"
3. User: `M001`, Password: `wrongpass1` → Enter (login fails)
4. Try again: `M001` / `wrongpass2` → fails
5. Try third time: `M001` / `wrongpass3` → fails
6. Now try CORRECT password: `M001` / `P@ssw0rd` → should be **rejected** (account locked)

**✅ Success looks like:** message: *"The referenced account is currently locked out and may not be logged on to."*

7. Wait 60 seconds (per the lockout duration)
8. Try `M001` / `P@ssw0rd` again → should now succeed

**❌ If it fails (account never locks):**
- lockout GPO not linked → Account Lockout Policy must be at the **Default Domain Policy** (or any GPO linked to the domain ROOT, not an OU)
- Re-check `22_…` Step 4

📸 Screenshot the lockout message.

**Marks:** verifies `lockout` GPO. Also relates to D119 if banner appears at the same screen.

### A7.6 Banner before logon (Client2 verification)

Same as A6.11 but verified from Client2. Win+L → confirm banner shows with exact title + text.

**Marks:** [Crit A7 D119 K=0.3]

### A7.7 Share access — Marketing (R), Executive (FC), others (deny)

**Why it earns marks:** the `pictures` share permissions must enforce per-group access (`22_…` Step 9).

**Per MA2 PDF page 12** — Marketing = Read, Executive = Full Control. Other groups = no access.

#### As M001 (Marketing user — Read access expected)

1. Sign out from C2 (Win+X → Sign out)
2. Sign in as `M001` / `P@ssw0rd`
3. Win+R → type: `\\winsrv1\pictures` → Enter
4. File Explorer opens showing `park.jpg`
5. **Read test:** double-click `park.jpg` → image opens in Photos viewer ✅
6. **Write test (should FAIL):** right-click in the share → New → Folder → try to create folder
   - Expected: "You'll need to provide administrator permission" or "Access denied"

#### As M004 (Executive user — Full Control)

1. Sign out
2. Sign in as `M004` / `P@ssw0rdP@ssw0rd` (16-char password from FGPP)
3. Win+R → `\\winsrv1\pictures` → Enter
4. **Read test:** double-click `park.jpg` → opens ✅
5. **Write test (should SUCCEED):** right-click → New → Folder → create "TestFolder"
   - Should create successfully ✅
6. **Delete test:** right-click "TestFolder" → Delete → succeeds

#### As M002 (Customer Service — should be DENIED)

1. Sign out
2. Sign in as `M002` / `P@ssw0rd`
3. Win+R → `\\winsrv1\pictures` → Enter
4. Expected: "You don't have permission to access \\winsrv1\pictures..." OR prompts for credentials and rejects M002.

**❌ If wrong perms work:**
- M001 can write → NTFS perms wrong. Re-run `22_…` Step 9.2 PowerShell ACL block.
- M002 can access → security filter wrong on share permissions.

📸 Screenshots: M001 reading park.jpg, M004 creating folder, M002 access denied.

**Marks:** [Crit A7 D120 K=0.3]

### A7.8 Audit log on WINSRV1 (verifies `22_…` Step 10)

**Why it earns marks:** reading `park.jpg` must be logged via SACL + Object Access auditing.

**Step-by-step:**
1. From Client2 (still as M004 after A7.7 write test) → also as M001 → access `\\winsrv1\pictures\park.jpg` (this generates the audit event)
2. Switch to **WINSRV1** VM (VMware Workstation tab)
3. Login as `MANILA\Administrator` / `P@ssw0rd`
4. Open Event Viewer: Win+R → `eventvwr.msc` → Enter
5. Left tree → **Windows Logs → Security**
6. Right pane → **Filter Current Log...**
7. **Event IDs:** type `4663` → OK
8. Look at filtered events → find recent ones with `park.jpg`

**✅ Success looks like:** Event 4663 entries with:
- **Subject Account Name:** M001 (or whichever user accessed the file)
- **Object Name:** `C:\shares\pictures\park.jpg`
- **Access Mask:** `0x1` (Read) or similar

**❌ If no events:**
- SACL not set on park.jpg → re-run `22_…` Step 10.2
- Object Access auditing not enabled → re-run `22_…` Step 10.1

📸 Screenshot Event Viewer with Event 4663 details visible.

**Marks:** [Crit A7 D121 K=0.5]

---

## A8 — Tests from Client3 (External, via OpenVPN)

> 🧠 **Switch to Client3 VM.** This client is on the "Internet" side (gets DHCP from ISP, NOT pfSense). It is NOT domain-joined.

Login as the **local administrator** on Client3:
- User: `Administrator` (no domain prefix)
- Password: `P@ssw0rd`

### A8.1 OpenVPN dial-in as VPNUser

**Why it earns marks:** OpenVPN server on pfSense must accept domain user authentication (`20_…` Step 5).

**Pre-requisites:**
- The `.ovpn` config file (exported from pfSense `20_…` Step 5.5) must be on Client3's Desktop or USB
- OpenVPN Connect client must be installed on Client3

**Step-by-step:**
1. Open **OpenVPN Connect** (taskbar or Start menu)
2. Click **+ icon** → **File** tab → **BROWSE** → select the `.ovpn` file from Desktop
3. Click **CONNECT**
4. Username/password prompt:
   - Username: `VPNUser`
   - Password: `P@ssw0rd`
5. Click **OK**

**✅ Success looks like:**
- Status changes to **CONNECTED** (green icon)
- Tunnel IP shown — should be in `10.8.0.0/24` range (e.g. `10.8.0.6`)
- Bytes in/out increasing

**❌ If it fails:**
- "AUTH_FAILED" → wrong password OR LDAP backend not configured. Check pfSense → System → User Manager → Authentication Servers → LDAP test.
- "TLS handshake failed" → cert mismatch. OpenVPN server cert must be from WINSRV3 CA, not self-signed.
- "Connection timed out" → WAN rule for UDP 1194 missing. Check `20_…` Step 5.4.

📸 Screenshot OpenVPN Connect showing CONNECTED state + tunnel IP.

**Marks:** [Crit A8 D123 K=0.5]

### A8.2 DNS over VPN

**Why it earns marks:** with VPN connected, DNS queries should be tunneled.

**Step-by-step (with VPN still connected):**
1. Open cmd
2. Run:
   ```cmd
   nslookup www.manila.com
   ```

**✅ Success looks like:**
```
Server:  WINSRV1.manila.com
Address: 192.168.2.10

Name:    www.manila.com
Address: 192.168.1.10
```

The query went through the VPN to WINSRV1 (the DNS server on the SERVERS net).

> ⚠️ **Lyon-leftover trap:** marking row D124 says *"should resolve to external interface IP of firewall"* — that's Lyon's interpretation. ASEAN PDF doesn't specify; the safest interpretation is the internal IP (192.168.1.10) because that's where the DMZ website actually lives.

**❌ If it fails:**
- Server shows ISP DNS (10.0.0.x) → VPN didn't push DNS. Check OpenVPN config: must include `push "dhcp-option DNS 192.168.2.10"`.

📸 Screenshot the nslookup output.

**Marks:** [Crit A8 D124 K=0.3]

### A8.3 Reach the DMZ website

**Why it earns marks:** with VPN, external client should reach internal DMZ website.

**Step-by-step:**
1. Open Chrome
2. URL → `https://www.manila.com` → Enter
3. Log in with M001/P@ssw0rd if HTTP basic auth prompt appears

**✅ Success looks like:** site loads, padlock visible, no cert warning.

**❌ If it fails:**
- "ERR_CONNECTION_TIMED_OUT" → VPN routing not pushing the right networks. Check OpenVPN server config: tunnel network `10.8.0.0/24`, push `192.168.1.0/24` and `192.168.2.0/24`.

📸 Screenshot the loaded site over VPN.

**Marks:** [Crit A8 D125 K=0.5]

### A8.4 FIN scan triggers Snort

**Why it earns marks:** Snort's custom FIN-scan rule (sid 100001) must trigger when an attacker scans the WAN.

**Step-by-step:**
1. **Disconnect VPN first** (this scan must come from the "Internet" side, not from inside the VPN)
   - OpenVPN Connect → click the connected profile → DISCONNECT
2. Open cmd OR Zenmap (Nmap GUI)
3. Find pfSense's WAN IP — it has DHCP from ISP, so it should be `10.0.0.X`. Confirm by browsing `https://10.0.0.<the-IP>` to see pfSense WebUI from outside.
   - Or guess: usually `10.0.0.1` or check ISP's DHCP lease
4. Run a FIN scan:
   ```cmd
   nmap -sF -p 1-1024 10.0.0.<pfsense-wan-ip>
   ```
   `-sF` = FIN scan (sets only the FIN flag in TCP packets)

**✅ Success looks like:**
- Switch to a Client1 PuTTY session OR open pfSense WebUI from Client1 (still on LAN) → Services → Snort → Alerts
- Recent alert entry: `Possible FIN scan` with sid `100001`
- Source IP = Client3's IP, dest IP = pfSense WAN

**❌ If it fails:**
- No alert → custom rule not in custom.rules. Re-add per `20_…` Step 6.4.
- Alert shows different sid → wrong sid. Must be `100001`.

📸 Screenshot the alert page on pfSense showing the FIN scan entry.

**Marks:** [Crit A8 D126 K=0.4]

> ⚠️ **Lyon-leftover trap:** marking row D126 says "Possible XMAS Scan" sid 100001. PDF says FIN. If chief Marlon insists on Lyon-style scoring, also add a second rule:
> ```
> alert tcp any any <> $HOME_NET any (flags: SRPFAU; msg: "Possible XMAS scan"; sid: 100002;)
> ```
> Then run: `nmap -sX -p 1-1024 <pfsense-wan-ip>` (XMAS scan) — generates alert for sid 100002.

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

Day 2 (MA2) is now complete. CTF practice from **`50_Day3_CTF_Playbook.md`** onward.
