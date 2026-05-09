# Team 1 — Strategy & Role Split

How **you, your teammate, and the ESXi server** operate as a 3-unit team across all 3 competition days.

This file sits ON TOP of the day-by-day playbooks (`10_…`, `20_…`, `30_…`, `50_…`). The playbooks tell you **what to do**. This file tells you **who does it, in what order, and who verifies**.

---

## 1. The team is 3 units, not 2

| Unit | Role | What it owns |
|---|---|---|
| **Member A — "Pentest Lead"** | Drives Day 1 (MA1 attack) and Day 3 Red side. Strong on Kali, nmap, Burp, sqlmap, exploitation. | PC1 → `competitor1a / Boracay@14!` |
| **Member B — "Hardening Lead"** | Drives Day 2 (MA2 hardening) and Day 3 Blue side. Strong on Windows AD, GPOs, pfSense, PKI, Snort, log triage. | PC2 → `competitor1b / Tagaytay_62&L` |
| **ESXi server** | The shared lab. Hosts every VM. Snapshot vault. Treat it like a teammate — both of you can console into it but only one at a time touches a VM's settings. | `wsauser / Andres@9V4` at `192.168.1.1` |

> 🔄 **Swap roles in Week 2 of practice.** Try one full dry-run with A=Hardening / B=Pentest. Whichever role split scores higher → that's your final competition assignment.

---

## 2. ESXi server discipline (the 3rd team member)

The ESXi box is shared infra. Two competitors fighting over the same VM = lost time.

### Rules

| Rule | Why |
|---|---|
| **Both members log in to ESXi UI from their own PC's browser** (`https://192.168.1.1`) | Each of you sees VM state in real time — no "wait, is it on?" |
| **Snapshot every VM before Day 1 starts** | One revert = back to clean state in 30 seconds |
| **Only one member edits a given VM's settings at once** (CPU, RAM, NIC) | ESXi will silently overwrite the other's change |
| **Console sessions are exclusive** — if Member A is in WINSRV1 console, Member B uses RDP from PC2 | Two console clients = mouse fights |
| **Designate a "VM Wrangler" each day** | One person owns "is the lab healthy?" — opens consoles, watches CPU/RAM, restores snapshots if anything breaks |
| **Never delete a snapshot during the competition** | Even if it looks orphaned. Wait until after Day 3 |

### Pre-day checklist (5 min before each day starts)

- [ ] Both PCs can reach ESXi UI
- [ ] All VMs needed today are powered ON and reachable (ping each)
- [ ] Snapshot named `pre-day-N` exists on every VM
- [ ] Shared notepad open on PC1 desktop (`Team1_Worklog.txt`)
- [ ] Both members logged into their own competitor account, **not** sharing one

---

## 3. Day 1 split — MA1 CMS Pentest (6 h)

**Morning kickoff (10 min):** read PDF together, agree on scope of the 4 deliverables.

| Time | Member A (Pentest Lead) | Member B (Support / Writer) |
|---|---|---|
| 0:00–0:30 | Information gathering — `nmap -A`, banner grab, `dirb`/`gobuster`, identify CMS version | Watch over A's shoulder, take notes for the report |
| 0:30–1:30 | Run vulnerability scan tools (`nikto`, `whatweb`, `wpscan`/Drupal-equivalent), document findings | Start drafting report shell — header, section titles, target IP, scope statement |
| 1:30–2:30 | Exploit the user-level vuln (e.g. Drupalgeddon → web shell or weak SSH password) | Continue drafting Section 1 (Methodology) and Section 2 (Findings — fill from A's screen) |
| 2:30–3:30 | Privilege escalate to root (sudo NOPASSWD vim, kernel exploit, etc.) | Take screenshots from A's screen for evidence appendix |
| 3:30–4:30 | Confirm root access, dump `/etc/shadow`, prove impact | Write the **150-word executive summary** + Top-3 risks table |
| 4:30–5:30 | Help B with technical details for the writeup | Polish report — formatting, screenshots, recommendations |
| 5:30–6:00 | Both: review, save as `PHL_Team1_MA1_Report.pdf` to **Desktop**, double-check filename | Both: sanity check — file opens, content is readable |

### Critical handoff in Day 1

When Member A gets a working exploit, **immediately tell B which CVE / technique to write up** — don't wait until end of day. The report is graded on clarity, not just findings.

### Decision rule
If A and B disagree on which 2 vulnerabilities to feature in the executive summary: **A picks**, because A has the deepest technical context. B can flag concerns once but must defer.

---

## 4. Day 2 split — MA2 Hardening (morning) + IR/Forensics/AppSec (afternoon)

**Day 2 has 10 VMs (Security Onion = 10th) and ~44 marks** — split as **MA2 hardening (~19 K) + Crit B IR/Forensics/AppSec (25 K)**. Parallelize aggressively across both halves of the day.

### Day 2 Morning split — MA2 Hardening (~19 K)

| VM / Task | Owner | Why |
|---|---|---|
| **pfSense** (rules, OpenVPN, Snort) | **Member A** | Network-heavy — fits A's networking strength |
| **LINSRV1** (CentOS hardening) | **Member A** | Linux CLI → Member A's lane |
| **WINSRV1** (DC + 7 GPOs + share + audit) | **Member B** | Heaviest Windows/AD task |
| **WINSRV3** (Issuing CA + cert distribution) | **Member B** | PKI flows naturally after WINSRV1 |
| **Client1, Client2** functional verification | **Member B** | Member B already has WINSRV1 context |
| **Client3** (external — VPN dial-in, Nmap FIN scan) | **Member A** | Member A built the firewall, knows what to test |

### Day 2 Afternoon split — Crit B IR / Forensics / AppSec (25 K)

| Task | Owner | Why |
|---|---|---|
| **Security Onion** alert triage + Hunt queries (B1 HO Flags) | **Member B** | Member B has Day 2 morning monitoring context |
| **PCAP analysis** with Wireshark + tshark (B1) | Both alternating | Joint reading speeds it up |
| **Malware analysis** static + dynamic (B2 CM Flags) | **Member A** | A has Kali/CLI strengths from Day 1 pentest |
| **AppSec** scanning with Burp + sqlmap + ZAP | **Member A** | A's exploitation muscle memory transfers directly |
| **Investigation report writing** | **Member B** | Same role as Day 1 MA1 report writer |

> 💡 **Member A spends Day 2 afternoon on offensive-style tasks** (malware analysis = "what does this exploit do" = pentest mindset). **Member B handles defensive-style tasks** (alert triage, hunt, report). Same brain types, different artifacts.

### Day 2 Timeline (6 h total — morning hardening + afternoon Crit B)

| Time | Member A (PC1) | Member B (PC2) |
|---|---|---|
| **MORNING — MA2 Hardening (~19 K)** | | |
| 0:00–0:15 | Read MA2 PDF Section A2 + A3 | Read MA2 PDF Section A4 + A5 |
| 0:15–1:00 | pfSense — admin pwd, LAN DHCP, 5 rule sets | WINSRV1 — domain pwd policy + FGPP Executive |
| 1:00–1:30 | pfSense — OpenVPN with cert from WINSRV3 (waits briefly) | WINSRV1 — 7 GPOs (lockout, restrict CP, autolock, banner) |
| 1:30–2:00 | pfSense — Snort install + FIN scan rule | WINSRV1 — pictures share + park.jpg audit |
| 2:00–2:30 | LinSRV1 — sshd 2022, AllowUsers, pwd aging, sudoers, SELinux | WINSRV3 — finish issuing CA + Web Server cert |
| 2:30–3:00 | Client3 — VPN dial + FIN scan trigger | Client1/2 — share access + cert chain check |
| **AFTERNOON — Crit B IR/Forensics/AppSec (25 K)** | | |
| 3:00–3:30 | Pick first AppSec target (Juice Shop or staged web app) — recon + nuclei + ZAP scan | Open Security Onion → triage all alerts since Day 2 start |
| 3:30–4:30 | Manual probe OWASP Top 10 → exploit + extract flag | Pivot from suspicious alerts → Hunt queries → extract HO flags |
| 4:30–5:30 | Malware sample static analysis (`strings`, PEStudio) → CM flags | PCAP analysis (`tshark`, Wireshark "Follow Stream") → HO flags |
| 5:30–5:45 | Both — write up findings (AppSec finding + malware report) | Both — write up findings (Investigation case notes) |
| 5:45–6:00 | Both: cross-check `99_Marking_Map.md` — every aspect ticked? | Both: snapshot final state, prepare Q&A notes for experts |

### Handoff points (the crucial 3)

1. **WINSRV3 cert → pfSense OpenVPN** — Member B must finish issuing CA and export the OpenVPN server cert *by ~hour 2:30*, otherwise A is blocked.
2. **WINSRV3 cert → LINSRV1 HTTPS** — Member B issues the web server cert, hands the `.cer` + private key to A by ~hour 4:30.
3. **WINSRV1 GPOs applied → Client1/2 verification** — Member B must `gpupdate /force` on both clients before functional tests can run (~hour 5:00).

### Decision rule for Day 2
If a hardening step (e.g. SELinux enforcing) breaks a service the marking depends on (e.g. httpd): **the owning member decides whether to roll back or troubleshoot**. The other member must NOT touch that VM — wait, support, suggest. One pilot per cockpit.

---

## 5. Day 3 split — CTF (6 h)

CTF is **per-category strength**. Don't just split alphabetically — play to who's stronger.

### Suggested split

| Category | Owner | Reason |
|---|---|---|
| **Web exploitation** (SQLi, XSS, SSRF, JWT, IDOR) | **Member A** | Day 1 pentest experience transfers directly |
| **Boot-to-root VulnHub** | **Both alternating** | One drives keyboard, other reads writeup notes / suggests next angle |
| **Crypto** (RSA, hashes, Vigenere) | **Member B** | Patient analytical work |
| **Forensics** (PCAP, memory, disk) | **Member B** | Wireshark + log triage = Day 2 muscle memory |
| **Reverse engineering / pwn** | Whoever has more comfort with Ghidra | Likely A — but pre-test in practice |

### Daily flow

1. **First 30 min — both read challenge list together.** Pick low-hanging fruit (Juice Shop ★1–2) for fast points.
2. **Solo work for 90 min** — each on their assigned categories.
3. **30-min sync** — share found flags, share blockers, ask "do you see something I missed?"
4. **Solo work for 90 min** — second pass on whatever remains.
5. **Final 60 min** — pair-program on the hardest remaining challenge.
6. **Last 30 min — submit all flags, double-check format `flag{...}`, no typos.**

### Shared flag log

Open `Team1_Flags.txt` on a shared OneNote / shared text file:
```
[12:34] [Web] Juice Shop SQLi → flag{sql_inj_admin_login}
[12:51] [Crypto] Vigenere XOR → flag{caesar_was_here}
[13:10] [Forensics] PCAP HTTP basic auth → flag{leaked_password_in_pcap}
```

Every flag goes here the moment you find it. Single source of truth, no double-entry, no "did you submit that one?"

---

## 6. Comms + decision rules (apply all 3 days)

### Comms cadence

| When | What |
|---|---|
| **Hour mark** (every 60 min) | 30-second status check — "I'm at step X, blocker Y, ETA Z" |
| **Whenever stuck >15 min** | Tell teammate. Either pair on it or skip and come back |
| **Before any irreversible action** | Snapshot the VM. Tell teammate. Take a breath |
| **End of each phase** | Tick the marking-map row. Both members confirm |

### Decision rules

1. **Owner decides on owned VM.** Member A doesn't second-guess WINSRV1 mid-config. Member B doesn't second-guess pfSense mid-config.
2. **Time pressure trumps perfection.** If something is at minute 240/360 and not working, it's worth abandoning to grab the next deliverable. Move on, come back if time.
3. **The marking scheme is the only judge.** Don't argue what's "correct" or "best practice" if it earns no marks. Earn marks first; argue elegance never.
4. **Rest > heroics.** Lunch break. Bathroom break. Stretch. A 10-minute break in hour 4 is worth more than 10 grumpy minutes of mistakes.

### Tiebreaker
If you and your teammate disagree on a strategic call (e.g. "should we attempt the hard CTF?"), **time-box it 60 seconds** then **defer to the day's owning member** (Member A on Day 1, Member B on Day 2, alternate per challenge on Day 3).

---

## 7. Pre-competition rehearsal plan

### Week 1 — Stage A setup (~15 hours, both members involved)
- Both members install host tooling on their own PC
- One member (whichever has stronger network skills) builds the ESXi rig
- Split VM builds: A builds Linux side (CMS, Kali, ISP, LinSRV1), B builds Windows side (WINSRV1/3/4, Client1/2/3), pfSense built together (4-NIC config benefits from 4 eyes)
- Both review every VM together at end of Stage A — "snapshot ceremony"

### Week 2 — Stage B dry-runs (3+ runs)
| Run | Day | Notes |
|---|---|---|
| Run 1 | Day 1 + Day 2 + Day 3 | Default role split (A=Pentest, B=Hardening). Track time per phase |
| Run 2 | Day 1 + Day 2 + Day 3 | **Swap roles** — A=Hardening, B=Pentest. Find each member's weak categories |
| Run 3 | Day 1 + Day 2 + Day 3 | Lock in best role split. Polish weak areas |
| Run 4 (optional) | Day 2 + Day 3 only | Day 2 has the most marks — extra rep here |

After each run, both members fill in `99_Marking_Map.md` together. The diff between Run 1 score and Run 3 score = your improvement curve.

---

## 8. What to do if your teammate is late / sick / disconnected

- **Member alone for >30 min:** focus on owned VMs only. Don't touch teammate's VMs.
- **Teammate disconnected from ESXi:** check console state, snapshot before touching anything they were configuring.
- **Solo for entire day:** abandon parallelization, work serially, accept lower score, prioritize highest-K-mark items first per `99_Marking_Map.md`.

---

## 9. Customizing this strategy

The role split above is a **default**. Override it by editing this file once you and your teammate have done your first dry-run together. Specifically tune:

- [ ] Who's stronger at Linux CLI?
- [ ] Who's stronger at Windows AD/GPOs?
- [ ] Who's stronger at writing (the MA1 report)?
- [ ] Who's stronger at exploitation vs hardening?
- [ ] Who's faster at typing / clicking through GUIs?
- [ ] Who's calmer under time pressure (= Day 3 lead)?

After Run 1 of practice, fill the table at the top with your actual names and assigned roles. From then on, this file is your competition contract.

---

## 10. Quick reference — who logs in where on Day 1

| Resource | Member A | Member B |
|---|---|---|
| PC1 (workstation) | `competitor1a / Boracay@14!` | — |
| PC2 (workstation) | — | `competitor1b / Tagaytay_62&L` |
| ESXi UI | `wsauser / Andres@9V4` | `wsauser / Andres@9V4` |
| Kali (192.168.2.2) | root / **whatever you set** | view-only via SSH from PC1 |
| CMS Target (192.168.2.1) | attack from Kali | view-only via SSH from PC1 |
| WINSRV1 | view-only (RDP) | full admin |
| WINSRV3 | view-only (RDP) | full admin |
| pfSense web GUI | full admin | view-only |
| LinSRV1 | full admin (SSH 2022) | view-only |
| Client1, Client2 | view-only | full local admin |
| Client3 | full local admin | view-only |

**View-only** here just means "don't change settings unless asked" — both members can ping/test/observe from any machine, that's expected.
