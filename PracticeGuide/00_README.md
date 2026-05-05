# Team 1 — WSA2025 Cyber Security Practice Guide

This is a step-by-step practice playbook for the WorldSkills ASEAN Manila 2025 Cyber Security skill (skill 54).
You and your teammate will go from **zero installed software** to a **full dry-run** of every gradeable activity in MA1, MA2 and the CTF days.

Every step in this guide tells you:

- **Why** we do it (one sentence, plain English).
- **Where** to do it (which VM, which window).
- **Tools** (the specific app or command you'll use).
- **Commands / Clicks** (exact text — copy/paste).
- **Expected output** (what success looks like — so you know to move on).
- **Marks earned** — the exact row in the marking scheme that this step satisfies, e.g. `[Crit A2 D49 K=0.2]`.
- **If it fails** — one specific recovery action.

---

## How to use this guide

1. Read `01_Setup_Tools.md` first — it lists every download and how to install it.
2. Then `02_Setup_Topology.md` — physical wiring + virtual networking.
3. Then build the practice VMs from `03_Setup_VMs_MA1.md` and `04_Setup_VMs_MA2.md`. (Do these once. Snapshot every VM after install.)
4. Practice the actual deliverables from the `10_…` through `30_…` files.
5. CTF practice from `40_…` through `42_…`.
6. Use `90_Practice_Schedule.md` as a 2-week calendar.
7. After every dry-run, open `99_Marking_Map.md` and tick what you completed — it shows you what % of marks you'd have earned.

---

## Day plan at a glance

| Day | What competitors do | Marking Criterion | Max marks |
|---|---|---|---|
| **Day 1 — morning (3 h)** | MA1: Assess Apache/website security on grimshay.local. Write 2 vulnerabilities + executive summary. | A1 (4 aspects) | ~part of 25 |
| **Day 1 — afternoon (3 h)** | MA2: Build pfSense, harden LinSRV1, configure WinSRV1 GPOs/share/audit, finish PKI on WinSRV3, verify from clients. | A2–A8 | rest of 25 |
| **Day 2 (6 h)** | CTF on **OWASP Juice Shop** — easy challenges (★1–★2). | Criterion B | 25 |
| **Day 3 (6 h)** | CTF on **OWASP Juice Shop** — medium (★3–★4). | Criterion C | 25 |
| **Day 4 (6 h)** | CTF on **OWASP Juice Shop** — hard (★5–★6). | Criterion D | 25 |
| **Total** | | | **100** |

> ⚠️ The official Test Project Development plan only lists 3 competition days. The marking scheme has 4 marking days. Treat this as 4 days for safety; if the schedule is later confirmed as 3, Days 3 and 4 will be compressed into Days 2 and 3.

---

## Time budget per Day 1 file (rehearse against the clock)

| File | Target time |
|---|---|
| `10_Day1_MA1_Solution.md` | 90 min (then 90 min buffer for documentation/proof-reading) |
| `20_Day1_MA2_Firewall.md` | 75 min |
| `21_Day1_MA2_LinSRV1.md` | 60 min |
| `22_Day1_MA2_WinSRV1_AD.md` | 60 min |
| `23_Day1_MA2_PKI.md` | 20 min |
| `30_Day1_MA2_Verification.md` | 45 min |
| **MA2 afternoon total** | ~4.3 h (you have 3h, so split between two of you) |

**Team split recommendation** during MA2:
- **Person A** does `20_Firewall` → `23_PKI` → `30_Verification`.
- **Person B** does `21_LinSRV1` → `22_WinSRV1_AD` → joins `30_Verification`.
- Sync every 30 min on a shared notepad.

---

## Reading the marking-scheme tags

Every step ends with one or more tags like:

```
**Marks earned:** [Crit A2 D49 K=0.2] DHCP handled by firewall
```

- `Crit A2` → Sub-criterion **A2 = Firewall** (rows in the spreadsheet starting at A47)
- `D49` → the specific aspect row in the spreadsheet (Aspect Type column D, row 49)
- `K=0.2` → max marks for that aspect

Total all the K-values across all the steps you successfully complete and you have your raw score for that criterion (max 25).

---

## Five rules for competition day (memorise)

1. **Read the deliverable line literally.** If it says "WorldSkills ASEAN Manila", type that exactly. The marking scheme has a known typo ("WorldSkills Lyon"), but experts mark from the test project, not the spreadsheet — write what the project asks.
2. **Save with your country code in the filename.** Example: `PHL_Team1_Vulnerabilities.pdf`. Save to the **Desktop** of the competitor workstation.
3. **Functional test from a client, every time.** Configuration on the server is worthless if a client can't actually use it. Always verify from Client1, Client2 or Client3.
4. **Snapshot before risky changes.** pfSense and WINSRV3 in particular — snapshot via ESXi before any irreversible step.
5. **No internet, no AI tools, no external write-ups during the CTF.** This is an automatic disqualification per `Capture-The-Flag-Regional-Skills-Olympics-Cybersecurity.docx`.

---

## File index

| File | Topic |
|---|---|
| `00_README.md` | This file |
| `01_Setup_Tools.md` | Tool downloads + installs |
| `02_Setup_Topology.md` | Physical + virtual network setup |
| `03_Setup_VMs_MA1.md` | Build the MA1 (grimshay.local) practice environment |
| `04_Setup_VMs_MA2.md` | Build the MA2 (manila.com) practice environment |
| `05_Setup_JuiceShop.md` | Install + configure OWASP Juice Shop (the actual CTF target) |
| `10_Day1_MA1_Solution.md` | Day 1 morning — solution walkthrough |
| `20_Day1_MA2_Firewall.md` | Day 1 PM — pfSense / OpenVPN / Snort |
| `21_Day1_MA2_LinSRV1.md` | Day 1 PM — CentOS hardening |
| `22_Day1_MA2_WinSRV1_AD.md` | Day 1 PM — AD GPOs + share + audit |
| `23_Day1_MA2_PKI.md` | Day 1 PM — Issuing CA + cert distribution |
| `30_Day1_MA2_Verification.md` | Day 1 PM — Functional tests from clients |
| `40_Day2_CTF_Playbook.md` | Day 2 — Juice Shop methodology + ★1–★2 walkthroughs |
| `41_Day3_CTF_Red.md` | Day 3 — Juice Shop ★3–★4 walkthroughs |
| `42_Day4_CTF_Blue.md` | Day 4 — Juice Shop ★5–★6 walkthroughs |
| `90_Practice_Schedule.md` | 2-week practice calendar |
| `99_Marking_Map.md` | Master mark map — every aspect → which step solves it |
