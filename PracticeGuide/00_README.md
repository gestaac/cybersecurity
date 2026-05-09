# Team 1 — WSA2025 Cyber Security Practice Guide

Step-by-step practice playbook for the WorldSkills ASEAN Manila 2025 Cyber Security skill (skill 54). Takes **Team 1 — you, your teammate, and the ESXi server (your 3rd team unit)** — from **zero installed software** to a **full dry-run** of every gradeable activity across the **3-day competition**.

> 👥 **This guide is for both team members.** It's the single source of truth you both work from — same files, same role split, same marking-map. See **`91_Team_Strategy.md`** for the per-day "who does what" plan.

Every step in this guide tells you:
- **Why** we do it (one sentence, plain English).
- **Where** to do it (which VM, which window).
- **Tools** (the specific app or command).
- **Commands / Clicks** (exact text — copy/paste).
- **Expected output** (what success looks like).
- **Marks earned** — exact row in the marking scheme, e.g. `[Crit A2 D49 K=0.2]`.
- **If it fails** — one specific recovery action.

---

## Day plan at a glance (CONFIRMED 3-day structure)

| Day | What competitors do | Source | Marks |
|---|---|---|---|
| **Day 1 (6 h)** | **MA1 — CMS pentest.** Information gathering → CMS vuln assessment → user/root privilege escalation → 150-word executive summary + top-3 risks. | `testpacakge_pdf/WSA2025_TP54_MA1_*.pdf` | Crit A1 = ~6 K |
| **Day 2 (6 h)** | **Morning: MA2 Security Hardening** (pfSense + OpenVPN + Snort + 7 GPOs + share + audit + LinSRV1 + PKI). **Afternoon: IR + Forensics + AppSec** using Security Onion (PCAP analysis, malware IR, web app testing). | `testpacakge_pdf/WSA2025_TP54_MA2_*.pdf` + Crit B flag scenarios | Crit A2–A8 (~19 K) + **Crit B (25 K)** = ~44 K |
| **Day 3 (6 h)** | **CTF (Red + Blue combined).** Member A on Red side (VulnHub + Juice Shop exploitation). Member B on Blue side (PCAP / memory / disk forensics). **Day 4's marks merged into Day 3** per chief's confirmation. | No PDF — random pick at venue | Crit C (25 K) + Crit D (25 K) = 50 K |

> ⚠️ The marking scheme spreadsheet has a 4-day skeleton with Lyon-leftover row names (HO/CM/ODD/CC/CS Flags). The **actual ASEAN competition is 3 days** — chief confirmed. **Day 4's marks (Crit D, "Blueday Flags," 25 marks) collapse into Day 3 CTF** per chief's explicit confirmation. Score totals still sum to 100 across the 3 days.

> 📊 Detailed Day-of-Marking analysis in **`99_Marking_Map.md`** → top section.

---

## The two-stage practice model (read this first)

**At the actual competition, organisers pre-install all VMs on your ESXi server before you arrive.** Win Server 2022 is already installed on WINSRV1/3/4 with AD/CA roles configured. Kali is already at 192.168.2.2. The CMS target is already running. pfSense base is done. LinSRV1 base is done. **You don't install anything during the competition** — you just configure rules, harden services, and pentest the CMS target. That's what earns marks.

**During practice, you have to do BOTH:**

### Stage A — Setup (the work organisers normally do for you) — ONE TIME, ~15 hours
You build all the VMs from scratch so your ESXi has the same starting state competitors would receive.

| What you install/build | Time | File |
|---|---|---|
| PC1 + PC2 host tooling (VMware Workstation, browsers, PuTTY, Burp, Wireshark, Nmap, OpenVPN Connect, Node.js + Juice Shop, etc.) | 3 h | `01_Setup_Tools.md`, `08_Setup_Workstations_HostOS.md` |
| Windows host prep on PC1 + PC2 (folder layout, AV exclusions, OVF Tool) | 1 h | `08_Setup_Workstations_HostOS.md` |
| Network rig (TP-Link router + switch + 3 boxes + ESXi port groups) | 2 h | `02_Setup_Topology.md` + `02c_Setup_ESXi_Server.md` |
| **CMS pentest target VM** (CentOS/Ubuntu + LAMP + Drupal 7 + weak `john` user + sudo NOPASSWD vim privesc) | 1 h | `03_Setup_VMs_MA1.md` Part 2 |
| **Kali Linux VM** (import VMware image, set static IP `192.168.2.2`) | 30 min | `03_Setup_VMs_MA1.md` Part 3 |
| **Security Onion VM** (Day 2 IR/Forensics platform — install + so-setup + connect to mirror port) | 3 h (mostly unattended) | `07_Setup_SecurityOnion.md` |
| **MalwareLab VM** (Win10 sandbox for malware analysis — disconnected NIC + analysis tools) | 30 min | `41_Day2_MalwareIR.md` |
| **ISP VM** (CentOS + dnsmasq + Apache + 2 self-signed test sites) | 40 min | `04_Setup_VMs_MA2.md` Part 1 |
| **pfSense** (4 NICs, base install only — leave unconfigured) | 40 min | `04_Setup_VMs_MA2.md` Part 2 |
| **WINSRV1** (Win Server 2022 install + promote to DC manila.com + AD users from Table 3) | 90 min | `04_Setup_VMs_MA2.md` Part 3 |
| **WINSRV3** (Win Server 2022 + AD CS Subordinate CA half-built — leave CSR pending) | 60 min | `04_Setup_VMs_MA2.md` Part 4 |
| **WINSRV4** (Win Server 2022 + Standalone Root CA + sign WINSRV3 CSR once) | 45 min | `04_Setup_VMs_MA2.md` Part 5 |
| **LINSRV1** (CentOS + httpd unhardened — that's the starting state) | 45 min | `04_Setup_VMs_MA2.md` Part 6 |
| **Client1, Client2, Client3** (Win 10 + Chrome + PuTTY + Wireshark; Client3 also Nmap + OpenVPN) | 90 min | `04_Setup_VMs_MA2.md` Part 7 |
| **Juice Shop install** (Node.js + ZIP, runs on host or Kali) | 30 min | `05_Setup_JuiceShop.md` |
| **VulnHub VMs** (download + import the recommended 8) | 90 min | `06_Setup_VulnHub.md` |

**End of Stage A: snapshot every VM.** From this point on, you can restore to "competition starting state" in 30 seconds.

### Stage B — Practice the deliverables (the work that earns marks) — REPEAT 3+ TIMES
With snapshots in place, every dry-run starts from the same clean state. You practise the actual graded work:

| Day | What you do (this earns marks) | File(s) |
|---|---|---|
| Day 1 | MA1 — pentest the CMS target from Kali (Information Gathering → CMS exploit → user pwd crack → root privesc → 150-word report + top-3 risks) | `10_Day1_MA1_Solution.md` |
| Day 2 | **Morning** (Crit A2–A8): MA2 hardening — pfSense rules + OpenVPN + Snort, LinSRV1 hardening, GPOs + share + audit on WINSRV1, finish CA on WINSRV3, verify from Client1/2/3 | `20_…` through `30_…` |
| Day 2 | **Afternoon** (Crit B = 25 K): IR + Forensics + AppSec via Security Onion — alert triage, PCAP analysis, malware analysis, web app testing | `40_…`, `41_…`, `42_…` |
| Day 3 | **Member A — Red CTF** (Crit C = 25 K): VulnHub boot-to-root + Juice Shop exploitation | `50_…`, `51_…`, `52_…` |
| Day 3 | **Member B — Blue CTF** (Crit D = 25 K): PCAP / memory / disk forensics + log triage | `53_Day3_BlueCTF.md` |

**Why two stages:**
- Stage A is a **one-time prep cost** (~15 hours total). Without it, your ESXi is empty and you have nothing to practise on.
- Stage B is the **actual training** (each full Day 1+2+3 dry-run = ~12 hours). Run it 3+ times to build muscle memory.
- After Stage A, every dry-run costs 30 seconds of snapshot restore + 12 hours of practice. Without snapshots, every mistake = re-installing VMs = hours wasted.

> 🔑 **The whole point** of Stage A is to build what the organisers would hand you on competition day. Stage B is what actually trains you to win.

---

## How to use this guide (file order)

### Phase 1 — Stage A setup (one-time, ~15 hours over the first week)
1. **`01_Setup_Tools.md`** — every download + install instruction.
2. **`02_Setup_Topology.md`** — physical wiring (TP-Link + switch + PC1 + PC2 + ESXi server).
3. **`02c_Setup_ESXi_Server.md`** — install ESXi on the 3rd PC. If "No Network Adapters" → see `02d_Setup_ESXi_Custom_ISO.md`.
4. **`08_Setup_Workstations_HostOS.md`** — Windows host prep on PC1 + PC2 (folder layout, OVF Tool, browser bookmarks, SSH aliases).
5. **`03_Setup_VMs_MA1.md`** — CMS pentest target + Kali (2 VMs on 192.168.2.0/24).
6. **`04_Setup_VMs_MA2.md`** — manila.com environment (10 VMs across 4 VLANs, including Security Onion for Day 2 PM).
7. **`05_Setup_JuiceShop.md`** — Juice Shop on host or Kali.
8. **`06_Setup_VulnHub.md`** — download + import VulnHub VMs.

### Phase 2 — Stage B practice (repeat 3+ times in week 2)
9. **`10_Day1_MA1_Solution.md`** — Day 1 MA1 CMS pentest walkthrough (4 tasks).
10. **`20_Day1_MA2_Firewall.md`** through **`30_Day1_MA2_Verification.md`** — Day 2 morning MA2 hardening (5 files).
11. **`40_Day2_IR_Forensics.md`** + **`41_Day2_MalwareIR.md`** + **`42_Day2_AppSec.md`** — Day 2 afternoon IR/Forensics/AppSec (Crit B = 25 K).
12. **`50_Day3_CTF_Playbook.md`** + **`51_Day3_VulnHub_BootToRoot.md`** + **`52_Day3_CTF_Advanced.md`** — Day 3 Red CTF (Crit C = 25 K).
13. **`53_Day3_BlueCTF.md`** — Day 3 Blue CTF (Crit D = 25 K).

### Phase 3 — Track and improve
12. **`90_Practice_Schedule.md`** — 2-week calendar with day-by-day tasks.
13. **`91_Team_Strategy.md`** — role split + parallelization + comms rules for both team members.
14. **`99_Marking_Map.md`** — tick what you completed after every dry-run; running raw score per criterion.

---

## How the team uses this guide together

You're 2 humans + 1 ESXi server. The day-by-day playbooks (`10_…`, `20_…`, `50_…`) tell you *what to do*. **`91_Team_Strategy.md`** tells you *who does it, in what order, and who verifies* — see it for:

- Default role split (Member A = Pentest Lead, Member B = Hardening Lead — swap in Week 2)
- ESXi server discipline (snapshots, exclusive consoles, "VM Wrangler" of the day)
- Per-day handoff points (e.g. WINSRV3 cert → pfSense OpenVPN at hour 2:30 of Day 2)
- Comms cadence, decision rules, tiebreakers
- What to do if your teammate is late / sick / disconnected

Read `91_Team_Strategy.md` together **before** starting Stage A practice. Override the default role split based on each other's strengths.

---

## Five rules for competition day (memorise)

1. **Read the deliverable line literally.** Type wording exactly as the PDF says.
2. **Save with your country code in the filename.** Example: `PHL_Team1_MA1_Report.pdf`. Save to the **Desktop**.
   - **MA1 workstation login:** `competitor1a / Boracay@14!`
   - **MA2 workstation login:** `competitor1b / Tagaytay_62&L`
   - **MA2 ESXi login:** `wsauser / Andres@9V4` at `192.168.1.1`
3. **Functional test from a client, every time.** Server config is worthless if Client1/2/3 can't actually use it.
4. **Snapshot before risky changes.** Especially pfSense and WINSRV3.
5. **No internet, no AI tools, no external write-ups during the CTF.** Automatic DQ per CTF rules.

---

## Reading the marking-scheme tags

Every step ends with a tag like:
```
**Marks earned:** [Crit A2 D49 K=0.2] DHCP handled by firewall
```

- `Crit A2` → Sub-criterion (A2 = Firewall, A3 = LINSRV1, etc.).
- `D49` → exact row in the spreadsheet (column D, row 49).
- `K=0.2` → max marks for that aspect.

Sum K-values across completed steps = your raw score per criterion (max 25 each, 100 total).

---

## File index

### Setup files (read once during practice prep)
| File | Topic |
|---|---|
| `00_README.md` | This file |
| `01_Setup_Tools.md` | Tool downloads + installs (host hardware budget, OS ISOs, Burp, jwt_tool, Kali, etc.) |
| `02_Setup_Topology.md` | 3-PC ESXi physical setup (TP-Link router + switch + 2 PCs + ESXi server) |
| `02c_Setup_ESXi_Server.md` | ESXi 8 install on the 3rd PC |
| `02d_Setup_ESXi_Custom_ISO.md` | **Troubleshooting: build custom ESXi ISO with Realtek driver** (only if stock ISO fails with "No Network Adapters") |
| `03_Setup_VMs_MA1.md` | Build the MA1 CMS pentest target + Kali |
| `04_Setup_VMs_MA2.md` | Build the manila.com environment (pfSense, AD, PKI, LinSRV1, clients) |
| `05_Setup_JuiceShop.md` | OWASP Juice Shop install + Burp config |
| `06_Setup_VulnHub.md` | Download + import recommended VulnHub VMs |
| `07_Setup_SecurityOnion.md` | **Security Onion 2.4 install** (Day 2 IR/Forensics platform) |
| `08_Setup_Workstations_HostOS.md` | Windows host prep + browser + SSH + snapshots + backups |

### Day-by-day deliverables (repeat as practice)
| File | Day | Topic |
|---|---|---|
| `10_Day1_MA1_Solution.md` | **Day 1** | CMS pentest walkthrough (4 tasks) |
| `20_Day1_MA2_Firewall.md` | **Day 2 AM** | pfSense + OpenVPN + Snort |
| `21_Day1_MA2_LinSRV1.md` | **Day 2 AM** | CentOS hardening |
| `22_Day1_MA2_WinSRV1_AD.md` | **Day 2 AM** | 7 GPOs + share + audit |
| `23_Day1_MA2_PKI.md` | **Day 2 AM** | Issuing CA + cert distribution |
| `30_Day1_MA2_Verification.md` | **Day 2 AM** | Functional tests from Client1/2/3 |
| `40_Day2_IR_Forensics.md` | **Day 2 PM** | Security Onion + 5-step IR loop + PCAP analysis (B1) |
| `41_Day2_MalwareIR.md` | **Day 2 PM** | Static + dynamic malware analysis (B2) |
| `42_Day2_AppSec.md` | **Day 2 PM** | OWASP Top 10 + Burp/ZAP/sqlmap workflow |
| `50_Day3_CTF_Playbook.md` | **Day 3 Red** | Juice Shop ★1–★2 + general CTF methodology |
| `51_Day3_VulnHub_BootToRoot.md` | **Day 3 Red** | Boot-to-root playbook + medium VMs |
| `52_Day3_CTF_Advanced.md` | **Day 3 Red** | Hard VulnHub VMs + Juice Shop ★5–★6 |
| `53_Day3_BlueCTF.md` | **Day 3 Blue** | PCAP / memory / disk / log forensics (Crit D) |

### Reference (use after every dry-run)
| File | Topic |
|---|---|
| `90_Practice_Schedule.md` | 2-week practice calendar |
| `91_Team_Strategy.md` | Role split + per-day "who does what" + comms rules |
| `99_Marking_Map.md` | Master mark map — every aspect → which step solves it |

> ⚠️ File names with `Day1_MA2_` prefix are historical — the content covers Day 2 (the MA2 hardening day). The chief originally split it as morning/afternoon of Day 1; the actual competition gives MA2 a full Day 2.

---

## What to do tonight

Quickest win to build momentum:

1. **30 min:** Read `01_Setup_Tools.md` Section 0 (host budget) + Sections 1–2 (hypervisor + ISOs).
2. **30 min:** Wire the rig per `02_Setup_Topology.md` (PC1 + PC2 + ESXi via switch).
3. **30 min:** Verify ESXi UI reachable from PC1 → `https://192.168.1.1` per `02c_…`.
4. **45 min:** Build the Drupal 7 CMS target VM on ESXi per `03_…` Part 2.
5. **5 min:** Boot Kali, set static IP, run `nmap 192.168.2.1` to confirm wiring.

That's a productive 2-hour first session. By tomorrow you can run your first MA1 pentest dry-run.
