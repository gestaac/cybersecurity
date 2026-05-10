# MA2 Cheat-Sheet (1 page, handwrite this for the venue)

> 🧠 **MEMORIZE WITH TEAMMATE — THE ENTIRE FILE.** Paper allowed at competition. Kinetic memory beats re-reading.

## How to drill this with your teammate (60 min total)

1. **Round 1 — Read together** (15 min): both read this file aloud once, ask "what's confusing?" after every section.
2. **Round 2 — Handwrite** (20 min each, in parallel): both copy the file by hand on A4 paper. No copying from each other; copy from the screen. **Kinetic memory locks it in.**
3. **Round 3 — Quiz each other** (20 min): close the file. Take turns asking:
   - "What's the exact banner title text?"
   - "What's the Snort FIN scan rule, word for word?"
   - "What share permissions for Marketing? For Executive?"
   - "Which user reads `park.jpg`?"
   - "What port is sshd on LinSRV1?"
   - "What's the Executive PSO password length?"
   - "Name all 7 GPOs."
4. **Round 4 — Re-write the parts you missed** (5 min): each wrong answer = re-write that line.

**Goal:** by bedtime, both of you can recite every section below without looking. At the venue, your handwritten copy is your safety net — but it should be *backup*, not primary.

---

## A2 — pfSense Firewall (target 4.95 K)

**Login:** browse from Client1 to `https://172.16.100.254` → `admin / pfsense` (default — change to `P@ssw0rd` first).

**DHCP on LAN:** range `172.16.100.50–150`, DNS `192.168.2.10` (WINSRV1), GW `172.16.100.254`, domain `manila.com`.

**Aliases (Networks):** `LAN_NET=172.16.100.0/24`, `DMZ_NET=192.168.1.0/24`, `SERVERS_NET=192.168.2.0/24`. **Hosts:** `LINSRV1=192.168.1.10`, `WINSRV1=192.168.2.10`. **Ports:** `AD_PORTS=53 88 135 137 138 389 443 445 464 636 3268 3269`, `WEB_PORTS=80 443`, `PKI_PORT=9389`.

**LAN rules:**
1. Pass `LAN_NET → LINSRV1` ports `22,53,80,443`
2. Pass `LAN_NET → SERVERS_NET` `AD_PORTS`
3. Pass `LAN_NET → SERVERS_NET` `9389` (PKI)
4. Pass `LAN_NET → WAN net` any
5. Block any (default deny)

**DMZ rules:** Pass `DMZ_NET → SERVERS_NET` `AD_PORTS` (LinSRV1 joins AD); block else.

**WAN/NAT:** port-forward TCP 80, TCP 443, UDP 53, TCP 53 → `192.168.1.10`.

**OpenVPN:** UDP 1194, server cert from **WINSRV3 CA (NOT self-signed)**, group `VPNGroup`, user `VPNUser / P@ssw0rd`, tunnel `10.8.0.0/24`, push `172.16.100.0/24, 192.168.2.0/24`.

**Snort FIN scan rule (custom.rules):**
```
alert tcp any any <> $HOME_NET any (flags: F; msg: "Possible FIN scan"; sid: 100001;)
```
Action `alert`, flags `F` only. **NOT XMAS.**

> 🧠 **MEMORIZE EXACTLY** — this rule must be typed verbatim. Common mistakes: writing `flags: X` (XMAS leftover from Lyon scheme), wrong sid number, forgetting `<>` (bidirectional). Quiz your teammate: "Recite the FIN scan rule from memory." Both should pass before sleep.

**Block site:** DNS Resolver Host Override `www.starcity.com.ph → 127.0.0.1`.

---

## A3 — LinSRV1 (CentOS, target 3.15 K)

- Domain-join `manila.com`: `realm join --user=Administrator manila.com`
- `sshd_config`: `Port 2022`, `PermitRootLogin no`, `AllowUsers C1 C2`. Restart sshd.
- `firewalld`: zone `public` → services `ssh http https`. `firewall-cmd --reload`.
- Password complexity: edit `/etc/security/pwquality.conf` → `minlen=12 dcredit=-1 ucredit=-1 ocredit=-1 lcredit=-1`. Ageing in `/etc/login.defs` → `PASS_MAX_DAYS 90 PASS_MIN_DAYS 1`.
- SELinux: `setenforce 1` + edit `/etc/selinux/config` → `SELINUX=enforcing`. Confirm `httpd_t` context on `/var/www/html`.
- Apache HTTPS: cert+key signed by WINSRV3 CA in `/etc/pki/tls/`.

---

## A4 — WINSRV1 AD (target 2.30 K + downstream verification)

**Login:** `MANILA\Administrator / P@ssw0rd`.

**7 GPOs** (link to `manila.com` root unless stated):

1. **Default Domain Policy** — pwd length **8**, history **30**.
2. **Executive-PSO** (fine-grained) — pwd length **16**, applies to group **Executive**, precedence 10. Test user M004 password = `P@ssw0rdP@ssw0rd`.
3. **LoginBanner**:
   - Title: **`WorldSkills ASEAN Manila`**
   - Text: **`Authorized access only`**

   > 🧠 **MEMORIZE EXACT WORDING.** Lyon-leftover marking row D119 says "WorldSkills Lyon" — that's WRONG. Type "WorldSkills ASEAN Manila" exactly (with capital A in ASEAN, capital M in Manila). One typo = lost mark.
4. **lockout** — threshold **3**, duration **1 minute**, reset **1 minute**.
5. **restrict control panel** — User Cfg → Admin Templates → Control Panel → "Prohibit access to Control Panel and PC settings" = Enabled. Security filter: deny **Executive**.
6. **disabled add and remove program panel** — User Cfg → Admin Templates → Control Panel → Add or Remove Programs → "Remove Add or Remove Programs" = Enabled. Security filter: scope = **Executive only**.
7. **autolock** — Computer Cfg → Security Options → "Interactive logon: Machine inactivity limit" = **10 seconds**. Filter: **Executive only**. (Also set screen-saver timeout 10 sec / password protect / force `scrnsave.scr` under User Cfg.)
8. **certenroll** — Public Key Policies → Certificate Services Auto-Enrollment **Enabled**, both Computer and User Cfg. Issuing template: duplicate Workstation Authentication, give Domain Computers Read+Enroll+Autoenroll.

**Pictures share** at `C:\shares\pictures` shared as `pictures`:
- **Marketing = Read**, **Executive = Full Control**, no one else.
- File **`park.jpg`** in folder. Audit ReadData on the file (Object Access auditing GPO + SACL → Everyone, Success+Failure).

> 🧠 **MEMORIZE — Lyon-row TRAPS.** Marking-scheme row D83 says "CS=R, Graphics=Mod, IT=FC" — WRONG. Row D120 says "graphics user" + "france.jpg" — WRONG. Use **Marketing=R, Executive=FC, file `park.jpg`**. The MA2 PDF page 12 wins, always.

**Table 2 GPO recommendations** (write top 3):
1. Disable LLMNR/NBT-NS (DNS Client → "Turn off multicast name resolution" Enabled)
2. AppLocker default rules in Audit mode
3. SMB Signing required (client + server "Digitally sign communications (always)" Enabled)

---

## A5 — WINSRV3 (Subordinate CA, target 0.2 K)

- Issued certs visible in CA console (`certsrv.msc`).
- CSR signed by WINSRV4 standalone root CA.

---

## A6/A7/A8 — Client verification (target 8.4 K combined)

**Client1 (LAN domain user, e.g. C1):**
- Reach allowed external website → OK
- Reach `www.starcity.com.ph` → BLOCKED (sinkholed)
- DHCP IP from 172.16.100.50–150
- `ssh -p 2022 C1@linsrv1` → in
- `sudo -l` → works
- `https://www.manila.com` → cert valid (chain to WINSRV4 root)
- certenroll auto-issued user/computer cert (`certmgr.msc`)
- Snort logs web traffic; FIN-scan rule visible in WAN tab

**Client2 (Internal, domain user e.g. C2):**
- SSH to LinSRV1 as domain user
- HTTPS chain valid at `https://webtest.manila.com`
- `nslookup www.manila.com` resolves
- certenroll GPO scope shows on this client
- Login banner shown at logon
- Marketing user (M001) reads `\\winsrv1\pictures\park.jpg`; Executive (M004) writes to share. Event Viewer Security 4663 logged.

**Client3 (External via OpenVPN):**
- OpenVPN dial-in succeeds (cert from WINSRV3 CA)
- DNS over VPN resolves `www.manila.com`
- Reach DMZ web from outside
- FIN scan from C3 triggers Snort alert (use `nmap -sF`)

---

## Logins to memorise

> 🧠 **MEMORIZE — both teammates must recite from memory.** Quiz: "What's the MA2 workstation password?" "What's the ESXi user?" "Where's ESXi reachable?"

| System | User | Password |
|---|---|---|
| MA1 workstation (Day 1) | `competitor1a` | `Boracay@14!` |
| MA2 workstation (Day 2) | `competitor1b` | `Tagaytay_62&L` |
| ESXi (MA2) at 192.168.1.1 | `wsauser` | `Andres@9V4` |
| Kali (MA1) | `kali` | `kali` |
| WINSRV1 / Admin | `MANILA\Administrator` | `P@ssw0rd` |
| pfSense (after change) | `admin` | `P@ssw0rd` |
| OpenVPN test user | `VPNUser` | `P@ssw0rd` |

---

## Save targets

> 🧠 **MEMORIZE — every saved file follows this pattern.** Wrong location or wrong name = lost mark even if content is perfect.

- All deliverables → **Desktop** of the competitor workstation.
- File-name pattern → **`PHL_Team1_<Module>_<Type>.pdf`** (e.g. `PHL_Team1_MA1_Report.pdf`).

---

## High-risk traps (memorise — Lyon-leftover marking-scheme rows are WRONG)

| Marking row says | What you actually do |
|---|---|
| Banner "WorldSkills Lyon" (D119) | "WorldSkills ASEAN Manila" / "Authorized access only" |
| XMAS scan (D106 / D126) | **FIN scan** with `flags: F` |
| Share `CS=R, Graphics=Mod, IT=FC` (D83) | **Marketing=R, Executive=FC** |
| Audit `france.jpg` (D120) | **`park.jpg`** by Marketing user |
| Login as "Anorbert" (D97) | Use Table 3 users (M001-M004 / S001 / C1 / C2) |
| Login `mratt@manila.com` (D113) | Use C2 (IT user from Table 3) |
| "google GPO" (D81 Chrome homepage) | Not in MA2 PDF — skip |

When in doubt, **MA2 PDF wording > marking-scheme row text**.
