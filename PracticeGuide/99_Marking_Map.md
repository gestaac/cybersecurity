# 99 — Master Marking Map

Cross-reference of every aspect in `WSA2025_54_Cyber_Security_marking_scheme_RevisedScore.xlsx` and the file in this guide that earns it.

> Use as a checklist after every dry-run: tick the **Earned?** column to know your raw score.

---

## Day of Marking — what the spreadsheet says vs reality

Verified directly from column **C** (Day of Marking) of the spreadsheet:

| Sub-criterion ID | Title in spreadsheet | Day column |
|---|---|---|
| A1 | Morning of Day 1 — security assessments | **1** |
| A2 | Firewall | **1** |
| A3 | LINSRV1 | **1** |
| A4 | WinSRV1 (AD) | **1** |
| A5 | WINSRV3 | **1** |
| A6 | Client1 | **1** |
| A7 | Client2 (Internal) | **1** |
| A8 | Client3 (External) | **1** |
| B1 | HO Flags | **2** |
| B2 | Casual Malware Flags | **2** |
| C1 | ODD Flags | **3** |
| C2 | From Cache Cache portion of CTF | **3** |
| D1 | Blueday Flags | **4** |

### What this means literally

The **spreadsheet** has a 4-day skeleton:
- Day 1: Criterion A (MA1 + MA2 work)
- Day 2: Criterion B — "HO Flags" + "Casual Malware Flags" (CTF-style names)
- Day 3: Criterion C — "ODD Flags" + "Cache-Cache Flags" (CTF-style names)
- Day 4: Criterion D — "Blueday Flags" (CTF-style names)

### ⚠️ The spreadsheet's 4-day skeleton doesn't match reality

**Confirmed competition structure (per chief Marlon):** 3 days, not 4.

| Day in spreadsheet | Reality at competition |
|---|---|
| Day 1 (MA1+MA2 = Crit A) | **Day 1 = MA1 (CMS pentest only)** — `testpacakge_pdf/WSA2025_TP54_MA1_*.pdf` |
| Day 2 (Lyon CTF flag names) | **Day 2 = MA2 (Security Hardening)** — `testpacakge_pdf/WSA2025_TP54_MA2_*.pdf`. The MA2 PDF IS the Day 2 module Marlon mentioned; the Lyon flag-name rows are placeholders. |
| Day 3 + Day 4 (Lyon CTF flag names) | **Day 3 = CTF** — random-pick from VulnHub + Juice Shop. Just 1 day, not 2. |

The marking-scheme **K-values still total 25 per criterion = 100 overall**. The chief will likely rebalance row names before competition; until then the K totals are still meaningful.

### Why the discrepancy exists

The spreadsheet was **copy-pasted from the Lyon WSC 2024 Cyber Security marking scheme** with only partial localisation:
- Criterion **titles** were updated for ASEAN.
- Criterion **B/C/D row names** (HO/CM/ODD/CC/CS prefixes) were NOT updated — they're Lyon challenge names.
- The **Day-of-Marking column** stayed valid (1/2/3/4).
- The **K-values stayed valid** (sum to 25 per criterion).

So the **DAY numbers and TITLES are reliable**; the **row names are not**.

### How our guide aligns

| Day | What the guide covers | File(s) |
|---|---|---|
| 1 | MA1 + MA2 walkthroughs (matches spreadsheet exactly) | `10_…`, `20_…` to `30_…` |
| 2 | MA2 (full Security Hardening per PDF) | `20_…` through `30_…` |
| 3 | CTF: Juice Shop ★1–★4 + first VulnHub VM (boot-to-root) | `50_…`, `51_…` |
| 4 | CTF: harder VulnHub + Juice Shop ★5–★6 | `52_…` |

When the chief releases the official **Day 2 deliverables doc** and the **ASEAN flag → target mapping** for Days 3–4, we replace the placeholder rows in this file with real aspect-by-aspect K-values.

---

## Criterion A — Day 1 (Max 25)

### A1 — Morning of Day 1, CMS Pentest (per MA1 PDF — 4 tasks)
| Row | Aspect | K | Solved in | Earned? |
|---|---|---|---|---|
| D31 | Information Gathering (services + secret) — Task 1 | 2.0 | `10_…` Task 1 | ☐ |
| D32 | Executive summary (≤150 words, top 3 risks) | 2.0 | `10_…` Task 4 | ☐ |
| D37 | CMS Vulnerability Assessment + sensitive info (Task 2) | 1.0 | `10_…` Task 2 | ☐ |
| D42 | System Security Weaknesses + privesc to root (Task 3) | 1.0 | `10_…` Task 3 | ☐ |
| **A1 subtotal** | | **6.0** | | |

> ⚠️ Lyon row names (D31/D32/D37/D42) don't match the actual MA1 PDF tasks 1:1. The K-values still total 6.0; the **mapping is best-guess** until the chief publishes an updated marking scheme aligned to the 4-task PDF structure.

### A2 — Firewall (pfSense)
| Row | Aspect | K | Solved in | Earned? |
|---|---|---|---|---|
| D48 | admin password set (Meas) | 0.2 | `20_…` Step 1 | ☐ |
| D49 | DHCP from FW (Meas) | 0.2 | `20_…` Step 2 | ☐ |
| D51 | LAN interface rules base (Meas) | 0.6 | `20_…` Step 4.1 | ☐ |
| D52 | LAN→DMZ ssh (Meas) | 0.3 | `20_…` Step 4.1 | ☐ |
| D53 | Servers iface rules (Meas) | 0.1 | `20_…` Step 4.3 | ☐ |
| D54 | DMZ iface rules (Meas) | 0.4 | `20_…` Step 4.2 | ☐ |
| D55 | WAN/NAT (Meas) | 0.4 | `20_…` Step 4.4 | ☐ |
| D56 | OpenVPN installed (Meas) | 0.7 | `20_…` Step 5 | ☐ |
| D57 | OpenVPN cert from CA, not self-signed (Meas) | 0.25 | `20_…` Step 5.1 + `23_…` Step 5 | ☐ |
| D58 | Snort installed (Meas) | 0.5 | `20_…` Step 6.1–6.3 | ☐ |
| D59 | Snort tracking events (Meas) | 0.3 | `20_…` Step 6.4 | ☐ |
| D60 | FW best practice (Judg, max 3) | 1.0 | `20_…` Step 7 | ☐ |
| **A2 subtotal** | | **4.95** | | |

### A3 — LinSRV1 (CentOS)
| Row | Aspect | K | Solved in | Earned? |
|---|---|---|---|---|
| D66 | Computer account in AD/DNS (Meas) | 0.2 | `21_…` Step 1 | ☐ |
| D67 | LinSRV1 joined manila.com (Meas) | 0.2 | `21_…` Step 1 | ☐ |
| D68 | sshd_config: 2022/no root/AllowUsers (Meas) | 0.2 | `21_…` Step 3 | ☐ |
| D70 | firewalld active (Meas) | 0.2 | `21_…` Step 4 | ☐ |
| D71 | firewalld services correct (Meas) | 0.3 | `21_…` Step 4 | ☐ |
| D72 | Linux password complexity + ageing (Meas) | 0.55 | `21_…` Step 5 | ☐ |
| D73 | sudo works for C1/C2 (Meas) | 0.2 | `21_…` Step 2 | ☐ |
| D74 | firewalld active (post-reboot) (Meas) | 0.2 | `21_…` Step 4 | ☐ |
| D75 | firewalld services correct (post-reboot) | 0.3 | `21_…` Step 4 | ☐ |
| D76 | sestatus enforcing (Meas) | 0.2 | `21_…` Step 6 | ☐ |
| D77 | httpd context (selinux) (Meas) | 0.3 | `21_…` Step 6 | ☐ |
| D78 | website using PKI cert (Meas) | 0.3 | `21_…` Step 7 + `23_…` Step 4 | ☐ |
| **A3 subtotal** | | **3.15** | | |

### A4 — WinSRV1 (AD)
| Row | Aspect | K | Solved in | Earned? |
|---|---|---|---|---|
| D80 | certenroll GPO (autoenroll) (Meas) | 0.4 | `22_…` Step 8 | ☐ |
| D81 | (Lyon-leftover, was google GPO — replaced by lockout/restrict-CP/disabled-add-remove/autolock per MA2 PDF) | 0.3 | `22_…` Steps 4–7 | ☐ |
| D82 | pictures share exists (Meas) | 0.2 | `22_…` Step 9 | ☐ |
| D83 | share permissions Marketing=R/Executive=FC (Meas) | 0.2 | `22_…` Step 9 | ☐ |
| D84 | Table 2 GPO recommendations (Judg, max 3) | 0.7 | `22_…` Step 11 | ☐ |
| D89 | Best-practice perm setup (Judg, max 3) | 0.5 | `22_…` Step 9 | ☐ |
| **A4 subtotal** | | **2.3** | | |

> ⚠️ **MA2 PDF (page 11) lists 7 GPOs as required — many are uncredited in the Lyon-leftover marking-scheme rows.** All are documented in `22_…`:
> - Domain pwd policy 8-char + history 30 → Step 1
> - Fine-grained 16-char for Executive → Step 2
> - LoginBanner ("WorldSkills ASEAN Manila" / "Authorized access only") → Step 3
> - `lockout` GPO (3 attempts / 60 sec) → Step 4
> - `restrict control panel` (everyone except Executive) → Step 5
> - `disabled add and remove program panel` (Executive only) → Step 6
> - `autolock` (Executive only, 10 sec inactivity) → Step 7
> - `certenroll` (autoenroll certs) → Step 9
> - `pictures` share (Marketing=R, Executive=FC) + `park.jpg` audit → Steps 10–11
>
> Do all of them — these are explicit MA2 PDF requirements, regardless of whether the marking scheme has aligned rows.

### A5 — WINSRV3 (CA)
| Row | Aspect | K | Solved in | Earned? |
|---|---|---|---|---|
| D95 | CA shows certs issued (Meas) | 0.2 | `23_…` Step 1–2 | ☐ |
| **A5 subtotal** | | **0.2** | | |

### A6 — Client1 functional tests
| Row | Aspect | K | Solved in | Earned? |
|---|---|---|---|---|
| D97 | Reach Internet website allowed (Meas) | 0.3 | `30_…` A6.1 | ☐ |
| D98 | Restricted website blocked (Meas) | 0.3 | `30_…` A6.2 | ☐ |
| D99 | DHCP from PFSense functional (Meas) | 0.2 | `30_…` A6.3 | ☐ |
| D100 | ssh to LinSRV1 port 2022 as C1 (Meas) | 0.4 | `30_…` A6.4 | ☐ |
| D101 | sudo as C1 works (Meas) | 0.3 | `30_…` A6.5 | ☐ |
| D102 | https traffic over FW (Meas) | 0.3 | `30_…` A6.6 | ☐ |
| D103 | PKI chain check (Meas) | 0.4 | `30_…` A6.7 | ☐ |
| D104 | GPO certenroll worked (Meas) | 0.5 | `30_…` A6.8 | ☐ |
| D105 | SNORT logged web (Meas) | 0.5 | `30_…` A6.9 | ☐ |
| D106 | SNORT FIN-scan rule present (Meas — Lyon row says XMAS, MA2 PDF page 10 says FIN) | 0.3 | `30_…` A6.10 | ☐ |
| D107 | GPO recommendations applied (Judg, max 3) | 0.6 | `22_…` Step 10 + `30_…` | ☐ |
| **A6 subtotal** | | **4.1** | | |

### A7 — Client2 functional tests
| Row | Aspect | K | Solved in | Earned? |
|---|---|---|---|---|
| D113 | Linux ssh as domain user (Meas) | 0.3 | `30_…` A7.1 | ☐ |
| D114 | Cert chain at https://webtest (Meas) | 0.3 | `30_…` A7.2 | ☐ |
| D115 | DNS resolves www.manila.com (Meas) | 0.2 | `30_…` A7.3 | ☐ |
| D116 | certenroll GPO scope (Meas) | 0.3 | `30_…` A7.4 | ☐ |
| D117 | (Lyon-leftover, was Chrome homepage — replaced by lockout/autolock verification per MA2 PDF) | 0.2 | `30_…` A7.5 | ☐ |
| D118 | (Lyon-leftover) | 0.2 | replaced by lockout/restrict-CP verification at client | ☐ |
| D119 | Login banner (Meas) | 0.3 | `30_…` A7.6 | ☐ |
| D120 | Share readable as Marketing (M001) + writable as Executive (M004) per MA2 PDF page 12 | 0.3 | `30_…` A7.7 | ☐ |
| D121 | Auditing logged (Meas) | 0.5 | `30_…` A7.8 | ☐ |
| **A7 subtotal** | | **2.6** | | |

### A8 — Client3 (External)
| Row | Aspect | K | Solved in | Earned? |
|---|---|---|---|---|
| D123 | OpenVPN dial-in (Meas) | 0.5 | `30_…` A8.1 | ☐ |
| D124 | DNS over VPN (Meas) | 0.3 | `30_…` A8.2 | ☐ |
| D125 | Reach DMZ website from outside (Meas) | 0.5 | `30_…` A8.3 | ☐ |
| D126 | FIN scan triggers Snort (Meas — Lyon row says XMAS, MA2 PDF page 10 says FIN) | 0.4 | `30_…` A8.4 | ☐ |
| **A8 subtotal** | | **1.7** | | |

### Criterion A grand total
| | K |
|---|---|
| A1 | 6.0 |
| A2 | 4.95 |
| A3 | 3.15 |
| A4 | 2.3 |
| A5 | 0.2 |
| A6 | 4.1 |
| A7 | 2.6 |
| A8 | 1.7 |
| **Total Criterion A K-marks** | **25.0** |

> ✅ K-totals match the spreadsheet's Criterion A max of 25.

---

## Criterion B — Day 2 (Max 25) — MA2 Security Hardening (per MA2 PDF)

> ✅ **MA2 PDF IS the Day 2 module** (chief confirmed). The Lyon-leftover row names (HO Flags, Casual Malware Flags) in the spreadsheet don't match the actual MA2 deliverables, but K-totals still sum to 25. The chief will likely rebalance row names before competition; meanwhile the MA2 PDF is the source of truth for what to do.

| Lyon-leftover row | K placeholder | Actual MA2 deliverable | Solved by following |
|---|---|---|---|
| HO-01 to HO-07 (with H1/H2 bonuses) | ~13.0 | pfSense rules + OpenVPN + Snort + AD GPOs (lockout, certenroll, autolock, restrict CP, disable add/remove) | `20_…`, `22_…` |
| CM-01 to CM-08 (with H1/H2 bonuses) | ~12.0 | LinSRV1 hardening + PKI completion + share/audit + functional verification from clients | `21_…`, `23_…`, `30_…` |
| **Criterion B total (paper)** | **25.0** | | |

Track each MA2 deliverable as: ☐ done / ☐ verified-from-client / time / notes.

---

## Criterion C+D — Day 3 (Max 50) — CTF: VulnHub random pick + Juice Shop

> Per chief: **Day 3 is CTF only**. Random-pick from VulnHub, plus Juice Shop. The marking scheme has 50 marks split across Crit C (25) + Crit D (25), but functionally it's all one CTF day.

| Lyon-leftover row | K placeholder | Actual likely target | Solved by following |
|---|---|---|---|
| ODD Flag 01–12 (Crit C, ~17.5 K) | 17.5 | Mid VulnHub VM (DC-1, DC-2, Mr. Robot, Basic Pentesting 1) — user shell + root + per-stage flags | `51_…` boot-to-root, `52_…` Walkthroughs 2–4 |
| Cache-Cache Flag 01–03 (Crit C, ~7.5 K) | 7.5 | Juice Shop ★1–★4 (login admin, reset Jim, UNION SQLi, JWT, vulnerable lib) | `50_…`, `51_…` Section X |
| CS-01 to CS-15 (Crit D, 25 K) | 25.0 | Harder VulnHub (DC-3+, Kioptrix, Sunset) OR Juice Shop ★5–★6 (Forged JWT, SSRF, XXE, Premium Paywall) | `52_…` advanced walkthroughs |
| **Crit C+D total** | **50.0** | | |

> When chief releases the actual ASEAN flag → target mapping (closer to competition day), replace the placeholder rows above with the real flag IDs.

---

## Grand total

| Criterion | Day | Max | Your score |
|---|---|---|---|
| A — MA1 (CMS pentest) + MA2 (Hardening) | 1 + 2 | 25 | __ |
| B — MA2 deliverables (overlapping w/ A2–A8) | 2 | 25 | __ |
| C — Day 3 CTF | 3 | 25 | __ |
| D — Day 3 CTF advanced | 3 | 25 | __ |
| **Total** | | **100** | __ |

> ⚠️ The marking scheme overlaps Crit A and Crit B in covering MA2 work (since both reference MA2 deliverables). The chief will likely rebalance these. Aim to complete every MA2 deliverable cleanly — it scores under whichever criterion the rebalanced scheme uses.

---

## Risk register (known marking-scheme defects to be aware of)

| Defect in marking scheme | Mitigation |
|---|---|
| Day-of-Marking column has 4 days; actual competition is 3 days | Practice covers all 3 days; the Day 4 row is Lyon-leftover |
| All B/C/D flag names (HO/CM/ODD/CC/CS) are Lyon-leftover | Per MA1+MA2 PDFs and chief confirmation: Day 1 = MA1 pentest, Day 2 = MA2 hardening, Day 3 = CTF |
| Marking scheme rows haven't been re-aligned to MA1/MA2 PDFs yet | The chief will likely publish a revised marking scheme before competition. K-totals (25 per criterion, 100 total) still hold. |
| Older docx version of marking scheme had banner = "WorldSkills Lyon" | The MA2 PDF says **"WorldSkills ASEAN Manila"** — type this exactly per `22_…` Step 3. |
| MA1 PDF tasks (Information Gathering / CMS Vuln / System Weaknesses / Report) don't map 1:1 to the 4 Lyon Crit-A1 row names | Aim for completing all 4 PDF tasks correctly + a clean 150-word report; total K stays 6.0 |

---

End of guide.
