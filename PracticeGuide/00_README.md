# Team 1 — WSA2025 Cyber Security Practice Guide

Step-by-step practice playbook for the WorldSkills ASEAN Manila 2025 Cyber Security skill (skill 54). Takes you and your teammate from **zero installed software** to a **full dry-run** of every gradeable activity across the **3-day competition**.

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
| **Day 1 (6 h)** | **MA1 — CMS pentest.** Information gathering → CMS vuln assessment → user/root privilege escalation → 150-word executive summary + top-3 risks. | `testpacakge_pdf/WSA2025_TP54_MA1_*.pdf` | Crit A1 = ~6 K (more once chief rebalances) |
| **Day 2 (6 h)** | **MA2 — Security Hardening.** pfSense rules + OpenVPN + Snort + 7 GPOs + AD password policies + share + audit + LinSRV1 hardening + PKI completion + client verification. | `testpacakge_pdf/WSA2025_TP54_MA2_*.pdf` | Crit A2–A8 = ~19 K + uncredited GPOs |
| **Day 3 (6 h)** | **CTF.** Random-pick from VulnHub-style boot-to-root targets + OWASP Juice Shop (per chief Marlon's confirmation). | No PDF — random pick at venue | Crit B/C/D = remaining 75 |

> ⚠️ The marking scheme spreadsheet has a 4-day skeleton with Lyon-leftover row names (HO/CM/ODD/CC/CS Flags). The **actual ASEAN competition is 3 days** — chief confirmed. Marking-scheme rows for Days 2/3/4 will likely be rebalanced before competition. Score totals still sum to 100.

> 📊 Detailed Day-of-Marking analysis in **`99_Marking_Map.md`** → top section.

---

## How to use this guide (in order)

### Phase 1 — Setup (one-time, ~6 hours over 2 days)
1. **`01_Setup_Tools.md`** — every download + install instruction.
2. **`02_Setup_Topology.md`** (or **`02b_Setup_SinglePC_Practice.md`** if you only have one PC) — wiring + virtual networks.
3. **`02c_Setup_ESXi_Server.md`** — only if using the 3-PC ESXi rig.
4. **`08_Setup_Workstations_HostOS.md`** — Windows host prep + best practices.

### Phase 2 — Build the practice VMs (one-time, ~4 hours)
5. **`03_Setup_VMs_MA1.md`** — CMS pentest target + Kali (2 VMs on 192.168.2.0/24).
6. **`04_Setup_VMs_MA2.md`** — manila.com environment (9 VMs across 4 VLANs).
7. **`05_Setup_JuiceShop.md`** — Juice Shop on host or Kali.
8. **`06_Setup_VulnHub.md`** — download + import VulnHub VMs.

### Phase 3 — Practice the deliverables (repeat 3+ times)
9. **`10_Day1_MA1_Solution.md`** — Day 1 (full day) MA1 CMS pentest walkthrough (4 tasks).
10. **`20_Day1_MA2_Firewall.md`** through **`30_Day1_MA2_Verification.md`** — Day 2 hardening (5 files).
11. **`50_Day3_CTF_Playbook.md`** + **`51_Day3_VulnHub_BootToRoot.md`** + **`52_Day3_CTF_Advanced.md`** — Day 3 CTF playbooks.

### Phase 4 — Track and improve
12. **`90_Practice_Schedule.md`** — 2-week calendar.
13. **`99_Marking_Map.md`** — tick what you completed after every dry-run.

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
| `02b_Setup_SinglePC_Practice.md` | Single-PC alternative (VMware Workstation only, no ESXi) |
| `02c_Setup_ESXi_Server.md` | ESXi 8 install on the 3rd PC |
| `03_Setup_VMs_MA1.md` | Build the MA1 CMS pentest target + Kali |
| `04_Setup_VMs_MA2.md` | Build the manila.com environment (pfSense, AD, PKI, LinSRV1, clients) |
| `05_Setup_JuiceShop.md` | OWASP Juice Shop install + Burp config |
| `06_Setup_VulnHub.md` | Download + import recommended VulnHub VMs |
| `08_Setup_Workstations_HostOS.md` | Windows host prep + browser + SSH + snapshots + backups |

### Day-by-day deliverables (repeat as practice)
| File | Day | Topic |
|---|---|---|
| `10_Day1_MA1_Solution.md` | **Day 1** | CMS pentest walkthrough (4 tasks) |
| `20_Day1_MA2_Firewall.md` | **Day 2** | pfSense + OpenVPN + Snort |
| `21_Day1_MA2_LinSRV1.md` | **Day 2** | CentOS hardening |
| `22_Day1_MA2_WinSRV1_AD.md` | **Day 2** | 7 GPOs + share + audit |
| `23_Day1_MA2_PKI.md` | **Day 2** | Issuing CA + cert distribution |
| `30_Day1_MA2_Verification.md` | **Day 2** | Functional tests from Client1/2/3 |
| `50_Day3_CTF_Playbook.md` | **Day 3** | Juice Shop ★1–★2 + general CTF methodology |
| `51_Day3_VulnHub_BootToRoot.md` | **Day 3** | Boot-to-root playbook + medium VMs |
| `52_Day3_CTF_Advanced.md` | **Day 3** | Hard VulnHub VMs + Juice Shop ★5–★6 |

### Reference (use after every dry-run)
| File | Topic |
|---|---|
| `90_Practice_Schedule.md` | 2-week practice calendar |
| `99_Marking_Map.md` | Master mark map — every aspect → which step solves it |

> ⚠️ File names with `Day1_MA2_` prefix are historical — the content covers Day 2 (the MA2 hardening day). The chief originally split it as morning/afternoon of Day 1; the actual competition gives MA2 a full Day 2.

---

## What to do tonight

Quickest win to build momentum:

1. **30 min:** Read `01_Setup_Tools.md` Section 0 (host budget) + Sections 1–2 (hypervisor + ISOs).
2. **20 min:** Disable Hyper-V/WSL2/Memory Integrity (`02b_…` Section C.2).
3. **30 min:** Configure VMnets per `02b_…` Section D, OR install ESXi per `02c_…`.
4. **45 min:** Build the Drupal 7 CMS target VM per `03_…` Step 2.
5. **5 min:** Boot Kali, set static IP, run `nmap 192.168.2.1` to confirm wiring.

That's a productive 2-hour first session. By tomorrow you can run your first MA1 pentest dry-run.
