# 53 — Day 3 (Stage B) — Blue CTF Playbook

**What this file covers:** the **defensive / forensics** side of the Day 3 CTF — what the marking scheme calls **"Blueday Flags"** (Criterion D, 25 marks).

**Marks at stake:** 25 K — the entire Crit D pool. Per chief's confirmation, Day 4 collapses into Day 3, so these marks are still earnable on Day 3 alongside Red CTF (Crit C).

**Skill level assumed:** none. We assume zero prior CTF experience but you've already worked through `40_…`, `41_…`, `42_…`.

**Time:** 90 min per practice run. Repeat 4+ times.

---

## What is a "Blue CTF"? (read once)

In CTF tradition:
- **Red team** = attackers. They exploit, pwn, get root, exfiltrate flags.
- **Blue team** = defenders. They analyze, hunt, attribute, and reconstruct what attackers did.

**Blue CTF challenges** give you evidence (PCAPs, memory dumps, disk images, logs, malware samples) and ask you to **answer questions** about an attack that already happened.

Typical Blue CTF question styles:
- *"What time did the attacker first connect?"*
- *"What's the C2 domain?"*
- *"What command did the attacker run after escalating to root?"*
- *"What malware family is this dropper?"*
- *"What was the password for user XYZ?"*
- *"What PID was the malicious process?"*

Each correct answer = a flag (often submitted in `flag{...}` format with the answer baked in, like `flag{2026-05-09T14:32:11}` or `flag{evil-c2.com}`).

---

## Blue CTF vs Red CTF — when to choose which

You can't do both at the same time on Day 3. **Pick by category strength.**

| Sign you should do Blue first | Sign you should do Red first |
|---|---|
| You're stronger at log/file analysis than exploitation | You're confident in nmap/Burp/Metasploit |
| The Red CTF target seems hard / over-fortified | You see an obvious exploitation angle |
| There are more Blue flags available with H1/H2 hints not consumed | Red flags are simpler / shorter |

**Team strategy** (per `91_Team_Strategy.md`):
- Member A (Pentest Lead) = Red CTF
- Member B (Hardening Lead, Day 2 IR experience) = Blue CTF
- Sync every 90 min on flag count

---

## The 6-step Blue CTF methodology

This builds on the IR loop in `40_…` but is more structured for time-boxed CTF flags.

| Step | Question | Time-box |
|---|---|---|
| **1. Categorize** | "Is this PCAP / memory / disk / log / malware? Pick the right toolchain." | 30 sec |
| **2. Big-picture** | "What's the time span? Hosts involved? Volume?" | 3 min |
| **3. Find the smoking gun** | "Where's the attacker's first move?" | 10 min |
| **4. Walk the timeline** | "What did they do next? After that?" | 10 min |
| **5. Extract the answer** | "What specific value does the question ask for?" | 5 min |
| **6. Submit flag** | "Format correctly, double-check, submit." | 1 min |

Total: ~30 min per flag. With 15 flags in Crit D, you have ~24 min/flag. **Speed matters.**

---

## Tools for Blue CTF (have these ready on Kali + a Windows VM)

| Tool | What it does | OS |
|---|---|---|
| **Wireshark** | PCAP visual analysis | Both |
| **tshark** | PCAP CLI scripting | Linux |
| **Network Miner** | PCAP file extraction GUI | Windows |
| **Volatility 3** | Memory forensics | Linux (`vol`) |
| **Autopsy** | Disk image GUI | Both |
| **The Sleuth Kit** (`fls`, `icat`, etc.) | Disk forensics CLI | Linux |
| **strings + grep** | Find anything in any binary | Linux |
| **xxd / hexdump** | Bytes view | Linux |
| **CyberChef** | Decode/decrypt strings | Browser (offline OK) |
| **HxD** | Hex editor (Windows) | Windows |
| **PEStudio** | PE static analysis | Windows |
| **olevba / oledump** | Office macro analysis | Linux |
| **peepdf** | PDF analysis | Linux |
| **john / hashcat** | Crack hashes | Linux |
| **chainsaw** | Windows event log triage | Linux |
| **EVTXEcmd / Hayabusa** | Sigma-style EVTX rules | Linux |
| **plaso / log2timeline** | Disk-image timeline build | Linux |

> 💡 **Pre-install ALL of these BEFORE competition** while you have internet. Many are not in default Kali.

---

## Walkthrough 1 — PCAP-only Blue challenge (30 min target)

**Scenario:** organisers give you `breach.pcap` (~50 MB). 5 questions:
1. What's the attacker's IP?
2. What service did they exploit?
3. What credentials did they brute-force?
4. What file did they exfiltrate?
5. What's the contents of the exfiltrated file? (= flag)

### Step 1 — Categorize
File ends in `.pcap` → packet capture → start with Wireshark + tshark.

### Step 2 — Big-picture (3 min)
```bash
capinfos breach.pcap
# Time span: 2026-05-09 14:00 → 14:45
# Packets: 124,500
# Avg size: 850 bytes

tshark -r breach.pcap -q -z conv,ip | head -10
# Top talker pair: 192.168.7.50 ↔ 192.168.1.100
# 70 MB transferred in 30 min — anomaly!
```

The 192.168.7.50 host stands out. Likely the attacker.

### Step 3 — Find the smoking gun (10 min)

```bash
# What protocols did 192.168.7.50 use?
tshark -r breach.pcap -Y "ip.addr==192.168.7.50" -q -z io,phs | head -30
# eth -> ip -> tcp -> ssh    (high counts)
# eth -> ip -> tcp -> http   (low counts)
```

So attacker used SSH and HTTP. SSH brute-force? Open in Wireshark:
```bash
wireshark breach.pcap &
```
Filter: `ip.addr==192.168.7.50 and tcp.port==22` → see hundreds of failed handshakes followed by one success.

**Q1 answer:** attacker IP = `192.168.7.50`
**Q2 answer:** service = SSH (port 22)

### Step 4 — Walk the timeline (10 min)

Brute-force shows many failures + one success. Pivot to "what did they do after success":
```bash
tshark -r breach.pcap -Y "ip.src==192.168.7.50 and tcp.port==22" -T fields -e frame.time -e tcp.flags
```

Mark the successful auth time. After that, look for HTTP / SCP / FTP traffic from that host:
```bash
tshark -r breach.pcap -Y "ip.src==192.168.7.50 and http" -T fields -e frame.time -e http.host -e http.request.uri
```

You see HTTP request to `/upload?file=secret.zip`.

**Q3 answer:** look in the SSH packet at the moment of success. SSH is encrypted, so you can't read creds directly. **BUT** — if the brute-force was done with a tool like `hydra`, the attacker likely used a wordlist. The marking scheme typically gives you the cracked credential as a known wordlist entry. So check:
```bash
# Did they use FTP first? (common pivot)
tshark -r breach.pcap -Y "ftp" -T fields -e ftp.request.command -e ftp.request.arg
```
You see `USER admin / PASS Summer2024!`.

**Q3 answer:** `admin / Summer2024!`

### Step 5 — Extract the file

```bash
mkdir extracted && cd extracted
tshark -r ../breach.pcap --export-objects http,. -Y "http"
ls -la
file *
# secret.zip — Zip archive, password protected
```

**Q4 answer:** `secret.zip`

```bash
# Crack the zip password
fcrackzip -u -D -p /usr/share/wordlists/rockyou.txt secret.zip
# password: hello123

unzip -P hello123 secret.zip
cat secret.txt
# flag{exf1ltr4t3d_pcap_blue}
```

**Q5 answer:** `flag{exf1ltr4t3d_pcap_blue}` ← **the actual flag for marking**

### Total time
Step 1: 30s. Step 2: 3 min. Step 3: 10 min. Step 4: 10 min. Step 5: 5 min. = **28 min** for 5 sub-flags.

---

## Walkthrough 2 — Memory dump challenge (45 min target)

**Scenario:** `infected.raw` (8 GB memory dump). Question: what process did the malware inject into?

### Step 1 — Identify the OS
```bash
vol -f infected.raw windows.info
# Windows 10 x64 build 19041
```

### Step 2 — Process list + tree
```bash
vol -f infected.raw windows.pslist > pslist.txt
vol -f infected.raw windows.pstree > pstree.txt
```

Open `pstree.txt`. **Look for parent-child anomalies**:
- `winword.exe` → `cmd.exe` → `powershell.exe` → `notepad.exe` ← suspicious chain
- `services.exe` → `svchost.exe` → `unknown.exe` ← weird child of svchost

### Step 3 — Detect injections
```bash
vol -f infected.raw windows.malfind > malfind.txt
```

Output lists processes with injected code (`MZ` headers in heap, `RWX` permissions, etc.).

Example:
```
PID 4892   notepad.exe   0x7ff0... PAGE_EXECUTE_READWRITE   MZ header found
```

**Answer:** malware injected into `notepad.exe` PID 4892.

### Step 4 — Dump and analyze
```bash
vol -f infected.raw windows.dumpfiles --pid 4892
strings file.4892.dmp | grep -iE "flag\{|http|c2"
```

Often the C2 / flag is in the dumped memory.

---

## Walkthrough 3 — Disk image challenge (60 min target)

**Scenario:** `victim.E01` (40 GB disk image). Question: what file did the user delete that contains the flag?

### Step 1 — Open in Autopsy
```bash
sudo apt install -y autopsy
autopsy &
# Browser opens to Autopsy UI
```

1. Create new case → add data source → `victim.E01`.
2. Run all default modules (~20 min for 40 GB).

### Step 2 — Check Recent Files / Deleted Files
- **Tree view** → expand "Deleted Files" → look at filenames + timestamps.
- **Keyword search:** `flag{` → instant hit if any deleted file contains it.

### Step 3 — Recovered file content
Right-click any deleted file → Extract → save to disk. Open with appropriate viewer.

### Step 4 — Look at browser history
- Tree → "Web History" → see URLs visited.
- Often the flag is in a Pastebin URL or note-taking app's saved page.

### Step 5 — Email content
- Tree → "Email Messages" → search for keywords.

### Step 6 — Bash / PowerShell history
For Linux disks: `/home/<user>/.bash_history`.
For Windows: `C:\Users\<user>\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadline\ConsoleHost_history.txt`.

---

## Walkthrough 4 — Windows event log analysis (30 min target)

**Scenario:** `Security.evtx` (Windows event log). Question: what time did the attacker successfully log in?

### Step 1 — Convert to readable format
```bash
sudo apt install -y python3-evtx
evtx_dump.py Security.evtx > security.xml
```

### Step 2 — Use Hayabusa for Sigma-style scanning
```bash
# Pre-clone Hayabusa repo before competition
~/tools/hayabusa/hayabusa csv-timeline -d ./logs -o output.csv
```

Output is a CSV with severity-ranked events. Look for:
- Failed logons (event 4625)
- Successful logon after failures (event 4624)
- Account lockout (event 4740)
- Privilege use (event 4673, 4674)

### Step 3 — Filter for the smoking gun
```bash
grep "4624" security.xml | grep -B2 -A30 "<TimeCreated"
```

Find the successful logon with `LogonType=10` (RDP) or `LogonType=3` (network) following many `4625` failures.

### Step 4 — Note timestamp + user
**Flag format:** typically `flag{2026-05-09T14:32:11}` or `flag{admin}`.

---

## Walkthrough 5 — Encoded / encrypted strings (20 min target)

CTF organisers love hiding flags inside encoded strings.

### Common encodings (try in this order)
| Looks like | Try |
|---|---|
| `ZmxhZ3thYmNkfQ==` (ends with `=`) | base64: `echo "..." \| base64 -d` |
| `666c61677b6162636 4 7d` (only hex chars) | hex: `echo "..." \| xxd -r -p` |
| `synt{noqkre}` | ROT13: `echo "..." \| tr 'A-Za-z' 'N-ZA-Mn-za-m'` |
| `RyDeyrwMfNqBqL` (mixed alphanumeric) | base32 or base58 |
| Random-looking with `==` at end | base64 (try first always) |
| Long hex starting with key | XOR (try common keys) |

### CyberChef workflow
1. Open `gchq.github.io/CyberChef` (or in SO web UI).
2. Paste the encoded string in **Input**.
3. Drag operations to **Recipe**: e.g., `From Base64`, `From Hex`, `XOR`.
4. Output appears live.

CyberChef has **"Magic"** operation that auto-detects encoding. Use it as a first guess.

---

## Walkthrough 6 — Hash cracking (15 min target)

If a question gives you a hash (MD5, SHA1, NTLM, etc.):
```bash
# Identify hash type
hashid '<hash>'

# Crack with hashcat (faster, GPU)
hashcat -m 0 hash.txt /usr/share/wordlists/rockyou.txt    # MD5
hashcat -m 100 hash.txt rockyou.txt                        # SHA1
hashcat -m 1000 hash.txt rockyou.txt                       # NTLM
hashcat -m 1800 hash.txt rockyou.txt                       # SHA512crypt (Linux shadow)

# Or with john
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

The cracked password = often the flag.

---

## Sample challenges to practice ON

| Source | Difficulty | What it teaches |
|---|---|---|
| **CyberDefenders.org** | Beginner → Pro | PCAP + memory + disk forensics. Best free practice for Blue CTF |
| **DFIR.training challenges** | Intermediate | Full incident scenarios |
| **MalwareTrafficAnalysis.net** | Intermediate | PCAP-only puzzles |
| **HackTheBox Sherlocks** | Various | Modern Blue scenarios (need internet) |
| **TryHackMe Blue rooms** | Beginner | Guided walkthroughs |
| **Volatility past CTFs** | Advanced | Memory forensics deep dives |
| **FlareOn (past years)** | Advanced | Reverse engineering — mostly malware |

**Pre-download** PCAPs and memory dumps to your USB before competition.

---

## Practice routine — Blue CTF (90 min per session)

| Time | Activity |
|---|---|
| 0:00–0:05 | Pick 1 challenge from CyberDefenders or saved practice |
| 0:05–0:35 | Solve flag #1 (target: 30 min) |
| 0:35–1:05 | Solve flag #2 (target: 30 min) |
| 1:05–1:25 | Solve bonus flag #3 (if time) |
| 1:25–1:30 | Write down where you got stuck — review tomorrow |

Goal: solve any 3-flag challenge in 90 min by Run 4.

---

## Hint-bonus reminder

Crit D flags follow the same H1/H2 bonus structure as Crit B:

| Flag | Base K | +No H1 | +No H2 | Max |
|---|---|---|---|---|
| CS-01 | 0.54 | 0.13 | 0.66 | 1.33 |
| CS-06 | 1.97 | 0.22 | — | 2.19 |
| CS-10 | 1.75 | — | — | 1.75 |

**Total Crit D possible if no hints used:** ~25 K. With hints: ~16-18 K.

**The H2 bonus on most flags is BIGGER than the base.** Solving without H2 alone can double your score on a flag. **Avoid hints aggressively.**

---

## Common Blue CTF pitfalls

| Pitfall | Fix |
|---|---|
| Open Wireshark on a 50 MB PCAP and freeze for 10 min | Always start with `capinfos` and `tshark -q -z` for top-level summary first |
| Spend an hour in memory analysis when answer was in the EVTX | Match toolchain to question style |
| Decode every base64 you see, even useless ones | Only decode strings near suspicious context |
| Submit flag with extra whitespace | Always strip trailing `\n`, paste-then-trim |
| Take H1 hint at minute 5 | Try without hints for at least 15 min — the bonus is huge |
| Don't check what file type the evidence is | `file <evidence>` is always your first command |

---

## Cheat sheet — copy to your team notes

```
BLUE CTF 6-STEP LOOP
─────────────────────
1. file <evidence>      ← pcap? memory? disk? exe?
2. capinfos / vol info / autopsy preview
3. find smoking gun (top talkers, malfind, recent files)
4. timeline (sort by @timestamp / pstree / web history)
5. extract answer (specific to question)
6. format flag, submit

TOOL PER EVIDENCE TYPE
───────────────────────
.pcap   → tshark, Wireshark, NetworkMiner
.raw    → vol, strings
.E01    → Autopsy, sleuthkit
.evtx   → evtx_dump, Hayabusa, Chainsaw
.exe    → strings, PEStudio, IDA Free
.docx   → olevba, oledump
.pdf    → peepdf

DECODE QUICK WINS
──────────────────
echo "..." | base64 -d        # base64
echo "..." | xxd -r -p        # hex
echo "..." | tr 'A-Za-z' 'N-ZA-Mn-za-m'   # ROT13
CyberChef "Magic" operation   # auto-detect

HASHCAT MODES (most common)
────────────────────────────
0    MD5
100  SHA1
1000 NTLM
1800 SHA512crypt (Linux shadow)
2500 WPA-PSK
13100 Kerberos AS-REP
```

---

## What you've learned by end of this file

✅ Difference between Red and Blue CTF mindsets
✅ The 6-step Blue methodology
✅ PCAP analysis at speed (capinfos → tshark → Wireshark)
✅ Memory forensics with Volatility 3
✅ Disk forensics with Autopsy + Sleuth Kit
✅ Windows event log analysis with Hayabusa
✅ Decoding common encodings via CyberChef
✅ Hash cracking with hashcat / john
✅ Where to practice (CyberDefenders, MTA, etc.)
✅ Hint-bonus strategy

---

End of file. With `40_…`, `41_…`, `42_…`, and this file (`53_…`), you have full coverage of Crit B (Day 2) + Crit D (Day 3 Blue side). Combined with the existing `50_/51_/52_` Red CTF guides, all 100 marks have practice material.
