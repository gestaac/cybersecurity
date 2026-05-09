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

### ✅ Reality confirmed by chief Marlon: 3-day competition

| Day in spreadsheet | Confirmed reality at competition |
|---|---|
| Day 1 = Crit A (MA1+MA2) | **Day 1 = MA1 only (CMS pentest)** — `testpacakge_pdf/WSA2025_TP54_MA1_*.pdf` |
| Day 2 = Crit B "Cyber Security Incident Response, Digital Forensics, Application Security" | **Day 2 morning = MA2 hardening (Crit A2–A8)** + **Day 2 afternoon = Crit B IR/Forensics/AppSec** using Security Onion. Marlon confirmed Security Onion is a Day-2 setup that needs internet alongside OpenVPN. |
| Day 3 = Crit C "Red CTF" | **Day 3 morning = Red CTF** — VulnHub boot-to-root + Juice Shop exploitation |
| Day 4 = Crit D "Blue CTF" | ❌ **No Day 4. Crit D shifts into Day 3 afternoon as Blue CTF** (chief explicitly confirmed: "no task from Day 4, it will be Day 3 already") |

**Spreadsheet row 19 explicitly names Crit B as "Cyber Security Incident Response, Digital Forensics, Application Security" — NOT MA2 hardening.** MA2 belongs to Crit A2–A8. See B section below for the per-flag table.

### Confirmed mark distribution per day (3-day model)

| Day | Source criteria | Marks |
|---|---|---|
| Day 1 (MA1 CMS pentest) | Crit A1 | ~6 |
| Day 2 morning (MA2 hardening) | Crit A2–A8 | ~19 |
| Day 2 afternoon (IR + Forensics + AppSec, Security Onion) | Crit B | 25 |
| Day 3 morning (Red CTF) | Crit C | 25 |
| Day 3 afternoon (Blue CTF) | Crit D | 25 |
| **Total** | | **100** |

K-values total 100 overall. The chief may revise row names before competition; the day-of-marking column (1/2/3/4) and K-totals stay valid.

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
| 1 | MA1 CMS pentest walkthrough | `10_…` |
| 2 AM | MA2 hardening — pfSense + LinSRV1 + AD/GPOs + PKI + verification | `20_…` through `30_…` |
| 2 PM | IR + Forensics + AppSec — Security Onion install + Day-2 Crit B playbooks | `07_…`, `40_…`, `41_…`, `42_…` |
| 3 AM | Red CTF — VulnHub boot-to-root + Juice Shop ★1–★6 | `50_…`, `51_…`, `52_…` |
| 3 PM | Blue CTF — PCAP / memory / disk / log forensics | `53_…` |

When the chief releases the official ASEAN flag→target mapping for Days 2–3, we replace the placeholder per-flag rows in this file with real per-flag K-values.

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

## Criterion B — Day 2 (Max 25) — Cyber Security Incident Response, Digital Forensics, Application Security

> 🚨 **CORRECTION (2026-05-09):** The spreadsheet row 19 explicitly names Crit B as **"Cyber Security Incident Response, Digital Forensics, Application Security"**. This is **NOT** MA2 hardening — MA2 is part of Criterion A (A2-A8). Crit B is a separate Day-2 IR/Forensics/AppSec module covered by `40_…`, `41_…`, `42_…` and uses **Security Onion** (built per `07_…`).
>
> The Lyon-style row names (HO/CM Flags) ARE the actual flag-style challenge format — each flag has H1/H2 hint bonuses. The marking is per-flag (Got it Y/N + bonus K-marks for not using hints).

### Day 2 (chief's restructure) actually contains BOTH

| Component | Maps to | Marks |
|---|---|---|
| MA2 hardening (firewall, AD, PKI, LinSRV1, clients) | Crit A2–A8 (rows 47–126) | ~19.4 K |
| IR + Forensics + AppSec (Security Onion + flag challenges) | **Crit B** (rows 130–167) | 25 K |
| **Day 2 total** | | **~44 K** |

### B1 — "HO Flags" — Forensics-style hide-and-seek (rows 131–148)

Per row 19 spreadsheet title: these are Digital Forensics flags. Build skills via `40_Day2_IR_Forensics.md`.

| Row | Flag ID | Base K | +No H1 | +No H2 | Max | Solved in |
|---|---|---|---|---|---|---|
| D131 | HO-01 | 0.84 | 0.21 | 1.04 | 2.09 | `40_…` Walkthrough 1–3 |
| D134 | HO-02 | 0.28 | 0.07 | 0.34 | 0.69 | `40_…` |
| D137 | HO-03 | 0.42 | 0.10 | 0.52 | 1.04 | `40_…` |
| D140 | HO-04 | 0.97 | 0.24 | 1.22 | 2.43 | `40_…` |
| D143/D144 | HO-05a/b | 1.63 + 1.50 | — | — | 3.13 | `40_…` |
| D145/D146 | HO-06a/b | 1.78 + 1.00 | — | — | 2.78 | `40_…` (advanced) |
| D147/D148 | HO-07a/b | 1.78 + 1.00 | — | — | 2.78 | `40_…` (advanced) |
| **B1 subtotal (max with no hints)** | | | | | **~13 K** | |

### B2 — "Casual Malware Flags" — Malware analysis + IR (rows 150–167)

Build skills via `41_Day2_MalwareIR.md`.

| Row | Flag ID | Base K | +No H1 | +No H2 | Max | Solved in |
|---|---|---|---|---|---|---|
| D150 | CM-01 | 0.28 | 0.07 | 0.34 | 0.69 | `41_…` Walkthrough 1 |
| D153 | CM-02 | 0.83 | 0.21 | 1.04 | 2.08 | `41_…` |
| D156 | CM-03 | 0.42 | 0.10 | 0.52 | 1.04 | `41_…` |
| D159 | CM-04 | 0.28 | 0.07 | 0.34 | 0.69 | `41_…` |
| D162 | CM-05 | 1.25 | 0.14 | — | 1.39 | `41_…` |
| D164 | CM-06 | 1.57 | 0.17 | — | 1.74 | `41_…` (medium) |
| D166 | CM-07 | 1.04 | — | — | 1.04 | `41_…` |
| D167 | CM-08 | 1.39 | — | — | 1.39 | `41_…` (advanced) |
| **B2 subtotal (max with no hints)** | | | | | **~12 K** | |

### Where AppSec fits

The **"Application Security"** part of Crit B's title doesn't have its own dedicated rows in the spreadsheet — it's likely embedded in some HO/CM challenges that test web-app exploitation against a vulnerable target. Coverage in `42_Day2_AppSec.md`.

### B grand total
| | K |
|---|---|
| B1 (HO Flags) | ~13.0 |
| B2 (CM Flags) | ~12.0 |
| **Crit B total** | **25.0** |

> ⚠️ **Strategy:** every flag has up to 2× its base value as hint-avoidance bonuses. Practice WITHOUT hints. Burning hint H2 typically loses more than the base value of finding the flag.

---

## Criterion C — Day 3 Red CTF (Max 25) — Offensive: ODD + Cache-Cache

> Per chief: **Day 3 is CTF combined**. Crit C = Red side (offensive — exploitation, web, crypto, pwn). Crit D = Blue side (defensive — covered separately below).

### C1 — "ODD Flags" — Offensive challenges (rows 172–200)

Coverage in `50_Day3_CTF_Playbook.md`, `51_Day3_VulnHub_BootToRoot.md`, `52_Day3_CTF_Advanced.md`.

| Row | Flag ID | Base K | +No H1 | +No H2 | Max | Solved in |
|---|---|---|---|---|---|---|
| D172 | ODD-01 | 0.28 | 0.07 | 0.34 | 0.69 | `50_…` |
| D175 | ODD-02 | 0.56 | 0.14 | 0.69 | 1.39 | `51_…` |
| D178 | ODD-03 | 0.56 | 0.14 | 0.69 | 1.39 | `51_…` |
| D181 | ODD-04 | 0.97 | 0.24 | 1.22 | 2.43 | `51_…` |
| D184 | ODD-05 | 0.42 | 0.10 | 0.52 | 1.04 | `51_…` |
| D187 | ODD-06 | 0.42 | 0.10 | 0.52 | 1.04 | `52_…` |
| D190 | ODD-07 | 0.62 | 0.07 | — | 0.69 | `52_…` |
| D192 | ODD-08 | 1.25 | 0.14 | — | 1.39 | `52_…` |
| D194/D195/D196 | ODD-09a/b/H | 1.50 + 1.00 + 0.28 | — | — | 2.78 | `52_…` |
| D197 | ODD-10 | 1.74 | — | — | 1.74 | `52_…` |
| D198 | ODD-11 | 1.04 | — | — | 1.04 | `52_…` |
| D199/D200 | ODD-12a/b | 1.43 + 1.00 | — | — | 2.43 | `52_…` |
| **C1 subtotal (max with no hints)** | | | | | **~17.5 K** | |

### C2 — "Cache-Cache" Flags (rows 202–206)

| Row | Flag ID | Base K | Solved in |
|---|---|---|---|
| D202 | CC-01 | 1.74 | `50_…` Juice Shop |
| D203/D204 | CC-02a/b | 1.43 + 1.00 | `51_…` |
| D205/D206 | CC-03a/b | 1.78 + 1.00 | `52_…` |
| **C2 subtotal** | | **~7.5 K** | |

### Crit C grand total
| | K |
|---|---|
| C1 (ODD Flags) | 17.5 |
| C2 (CC Flags) | 7.5 |
| **Crit C total** | **25.0** |

---

## Criterion D — Day 4 (collapsed into Day 3) — Blue CTF (Max 25)

> 🚨 **Spreadsheet says Day 4** but chief explicitly confirmed **Day 4 collapses into Day 3**. The 25 K of Crit D shifts into Day 3 alongside Crit C.

> Crit D = "Blueday Flags" / CS-prefix = **Blue-side defensive CTF** (PCAP analysis, memory forensics, log triage, malware classification). Coverage: `53_Day3_BlueCTF.md`.

| Row | Flag ID | Base K | +No H1 | +No H2 | Max | Solved in |
|---|---|---|---|---|---|---|
| D211 | CS-01 | 0.54 | 0.13 | 0.66 | 1.33 | `53_…` Walkthrough 1 |
| D214 | CS-02 | 0.87 | 0.22 | 1.10 | 2.19 | `53_…` Walkthrough 1 |
| D217 | CS-03 | 0.69 | 0.18 | 0.88 | 1.75 | `53_…` Walkthrough 1 |
| D220 | CS-04 | 0.35 | 0.09 | 0.44 | 0.88 | `53_…` Walkthrough 4 |
| D223 | CS-05 | 0.35 | 0.09 | 0.44 | 0.88 | `53_…` Walkthrough 4 |
| D226 | CS-06 | 1.97 | 0.22 | — | 2.19 | `53_…` Walkthrough 2 |
| D228/D229/D230 | CS-07a/b/H | 1.18 + 1.18 + 0.26 | — | — | 2.62 | `53_…` Walkthrough 2 |
| D231 | CS-08 | 1.97 | 0.22 | — | 2.19 | `53_…` Walkthrough 3 |
| D233 | CS-09 | 1.19 | 0.13 | — | 1.32 | `53_…` Walkthrough 3 |
| D235 | CS-10 | 1.75 | — | — | 1.75 | `53_…` Walkthrough 5 |
| D236 | CS-11 | 0.88 | — | — | 0.88 | `53_…` Walkthrough 5 |
| D237 | CS-12 | 1.32 | — | — | 1.32 | `53_…` Walkthrough 6 |
| D238/D239 | CS-13a/b | 1.19 + 1.00 | — | — | 2.19 | `53_…` advanced |
| D240/D241 | CS-14a/b | 1.19 + 1.00 | — | — | 2.19 | `53_…` advanced |
| D242 | CS-15 | 1.32 | — | — | 1.32 | `53_…` advanced |
| **Crit D total (max)** | | | | | **~25 K** | |

> ⚠️ **The H2 bonuses on most CS flags are LARGER than the base.** Practice without hints. Skip a flag rather than burn an H2 on a flag whose base is < 0.5.

---

## Combined Day 3 (Crit C + Crit D) — practice strategy

| Aspect | Crit C (Red) | Crit D (Blue) |
|---|---|---|
| Marks | 25 | 25 |
| Member | A (Pentest Lead) | B (Hardening Lead) |
| Tools | Burp, sqlmap, msf, nmap | Wireshark, vol, Autopsy, Hayabusa |
| Practice file | `50_…`, `51_…`, `52_…` | `53_…` |
| Time per flag | ~20 min | ~25 min |
| Hint strategy | Aggressive solving, never use H1/H2 | Same |

---

## Grand total

| Criterion | Day in chief's restructure | Max | Your score |
|---|---|---|---|
| A — Enterprise Infrastructure Security (MA1 + MA2) | Day 1 (MA1) + Day 2 morning (MA2) | 25 | __ |
| B — IR + Forensics + AppSec (Security Onion) | Day 2 afternoon | 25 | __ |
| C — Red CTF (offensive) | Day 3 morning | 25 | __ |
| D — Blue CTF (defensive) | Day 3 afternoon | 25 | __ |
| **Total** | | **100** | __ |

---

## Risk register (known marking-scheme defects to be aware of)

| Defect in marking scheme | Mitigation |
|---|---|
| Day-of-Marking column has 4 days; chief confirmed 3-day comp | Day 4 (Crit D) collapses into Day 3 |
| Row D81 mentions "google GPO" (Chrome homepage) | NOT in MA2 PDF — Lyon-leftover. Skip. |
| Row D83 says share permissions: `CS=R, Graphics=Mod, IT=FC` | MA2 PDF page 12 actual: `Marketing=R, Executive=FC`. Follow PDF. |
| Row D97 says login as "Anorbert" | MA2 PDF Table 3 has M001/M002/M003/M004/S001/C1/C2. Use those. |
| Row D113 says login as "mratt@manila.com" | Use C2 (IT user from Table 3). |
| Row D119 says banner "WorldSkills Lyon" | MA2 PDF says **"WorldSkills ASEAN Manila"** — type that exactly. |
| Row D120 says "graphics user" + "france.jpg" / "manila.jpg" | MA2 PDF says Marketing user reads `park.jpg`. |
| Row D106, D126 say "Christmas / XMAS scan" | MA2 PDF page 10 says **FIN scan** (`flags: F`). Implement FIN. |
| MA1 marking has only 4 judgment rows for the 4 PDF tasks | The 9 question-answers in MA1 PDF are graded as overall pentest quality, not per-question. Top-2 vulnerabilities + executive summary = full marks. |
| All B/C/D flag names are Lyon-leftover (HO/CM/ODD/CC/CS prefixes) | The K-values + hint structure are valid. The actual flags will be ASEAN-specific but the per-flag scoring formula stays the same. |
| Crit B was previously misinterpreted as MA2 hardening | **Corrected 2026-05-09:** Crit B = IR/Forensics/AppSec (per spreadsheet row 19). MA2 = part of Crit A. |

---

End of guide.
