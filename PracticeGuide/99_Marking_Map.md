# 99 — Master Marking Map

Cross-reference of every aspect in `WSA2025_54_Cyber_Security_marking_scheme_RevisedScore.xlsx` and the file in this guide that earns it.

> Use as a checklist after every dry-run: tick the **Earned?** column to know your raw score.

---

## Criterion A — Day 1 (Max 25)

### A1 — Morning of Day 1, Security Assessments
| Row | Aspect | K | Solved in | Earned? |
|---|---|---|---|---|
| D31 | ≥ 2 Apache vulnerabilities identified (Meas) | 2.0 | `10_…` Phase 2 | ☐ |
| D32 | Executive summary (Judg, max 3) | 2.0 | `10_…` Step 4.2 | ☐ |
| D37 | 1st vulnerability write-up (Judg, max 3) | 1.0 | `10_…` Step 4.3 | ☐ |
| D42 | 2nd vulnerability write-up (Judg, max 3) | 1.0 | `10_…` Step 4.4 | ☐ |
| **A1 subtotal** | | **6.0** | | |

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
| D80 | certenroll GPO (autoenroll) (Meas) | 0.4 | `22_…` Step 7 | ☐ |
| D81 | google GPO (Meas) | 0.3 | `22_…` Step 6 | ☐ |
| D82 | pictures share exists (Meas) | 0.2 | `22_…` Step 8.1 | ☐ |
| D83 | share permissions (Meas) | 0.2 | `22_…` Step 8.2 | ☐ |
| D84 | Table 2 GPO recommendations (Judg, max 3) | 0.7 | `22_…` Step 10 | ☐ |
| D89 | Best-practice perm setup (Judg, max 3) | 0.5 | `22_…` Step 8.2 | ☐ |
| **A4 subtotal** | | **2.3** | | |

> ⚠️ **Uncredited but required in MA2:**
> - Domain password policy 8-char/monthly → `22_…` Step 1
> - Fine-grained 10-char password policy for `executive` → Step 2
> - `control` GPO restricting Control Panel for accounting → Step 4
> - `registry` GPO blocking reg tools for Manila users → Step 5
>
> Do them anyway — small time, big risk if marking scheme gets patched.

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
| D106 | SNORT XMAS rule present (Meas) | 0.3 | `30_…` A6.10 | ☐ |
| D107 | GPO recommendations applied (Judg, max 3) | 0.6 | `22_…` Step 10 + `30_…` | ☐ |
| **A6 subtotal** | | **4.1** | | |

### A7 — Client2 functional tests
| Row | Aspect | K | Solved in | Earned? |
|---|---|---|---|---|
| D113 | Linux ssh as domain user (Meas) | 0.3 | `30_…` A7.1 | ☐ |
| D114 | Cert chain at https://webtest (Meas) | 0.3 | `30_…` A7.2 | ☐ |
| D115 | DNS resolves www.manila.com (Meas) | 0.2 | `30_…` A7.3 | ☐ |
| D116 | certenroll GPO scope (Meas) | 0.3 | `30_…` A7.4 | ☐ |
| D117 | Chrome homepage forced (Meas) | 0.2 | `30_…` A7.5 | ☐ |
| D118 | google GPO applied (Meas) | 0.2 | `30_…` A7.5 | ☐ |
| D119 | Login banner (Meas) | 0.3 | `30_…` A7.6 | ☐ |
| D120 | Share readable as graphics (Meas) | 0.3 | `30_…` A7.7 | ☐ |
| D121 | Auditing logged (Meas) | 0.5 | `30_…` A7.8 | ☐ |
| **A7 subtotal** | | **2.6** | | |

### A8 — Client3 (External)
| Row | Aspect | K | Solved in | Earned? |
|---|---|---|---|---|
| D123 | OpenVPN dial-in (Meas) | 0.5 | `30_…` A8.1 | ☐ |
| D124 | DNS over VPN (Meas) | 0.3 | `30_…` A8.2 | ☐ |
| D125 | Reach DMZ website from outside (Meas) | 0.5 | `30_…` A8.3 | ☐ |
| D126 | XMAS scan triggers Snort (Meas) | 0.4 | `30_…` A8.4 | ☐ |
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

## Criterion B — Day 2 (Max 25) — Security Hardening: SOC + IR + Forensics + AppSec

> 🎯 Per chief Marlon's confirmation: Day 2 = **Security Hardening from scratch**, named tools = **Security Onion** + **OpenVPN**.
> The marking-scheme title for this criterion (*"Cyber Security Incident Response, Digital Forensics, Application Security"*) matches a SOC-build module, not a CTF.
> The HO-/CM- flag rows are **Lyon-2024 leftovers** — they will likely be replaced by Day 2 build/IR/forensics aspects when the official Day 2 doc lands.

| Lyon-leftover row | K placeholder | Likely Day 2 deliverable | Solved by following |
|---|---|---|---|
| HO-01 to HO-07 (with H1/H2 bonuses) | ~13.0 | Security Onion deployment + Wazuh onboarding (Linux + Windows) + log-source forwarding | `24_…` Phase 1; `07_Setup_SecurityOnion.md` |
| CM-01 to CM-08 (with H1/H2 bonuses) | ~12.0 | OpenVPN server install + cert chain + IR investigation + forensic findings + AppSec hardening | `24_…` Phases 2–5 |
| **Criterion B total (paper)** | **25.0** | | |

Track each step solved as: deliverable / outcome / time / observations.

---

## Criterion C — Day 3 (Max 25) — CTF: Juice Shop ★1–★4 and/or 1st VulnHub VM

| Lyon-leftover row | K placeholder | Likely target | Solved by following |
|---|---|---|---|
| ODD Flag 01–12 (+ H1/H2) | ~17.5 | Mid VulnHub VM (e.g. DC-1, DC-2, Mr. Robot) — flags map to user shell + root + per-stage flags | `51_…` boot-to-root, `52_…` Walkthroughs 2,3,4 |
| Cache-Cache Flag 01–03 | ~7.5 | Juice Shop ★1–★4 (login admin, reset Jim, UNION SQLi, JWT, vulnerable lib) | `50_…`, `51_…` Section X |
| **Criterion C total** | **25.0** | | |

---

## Criterion D — Day 4 (Max 25) — Harder VulnHub VM and/or Juice Shop ★5–★6

| Lyon-leftover row | K placeholder | Likely target | Solved by following |
|---|---|---|---|
| CS-01 to CS-15 + hint bonuses | 25.0 | Harder VulnHub (DC-3+, Kioptrix Level 1, Sunset series) + Juice Shop ★5–★6 (Forged JWT, SSRF, XXE, Premium Paywall) | `52_…` Walkthroughs 5,6 + Juice Shop hard section |
| **Criterion D total** | **25.0** | | |

> When chief releases the official Day 2 deliverables doc + flag→target mapping, replace the placeholder columns above with the actual rows.

---

## Grand total

| Criterion | Max | Your score |
|---|---|---|
| A — Day 1 | 25 | __ |
| B — Day 2 | 25 | __ |
| C — Day 3 | 25 | __ |
| D — Day 4 | 25 | __ |
| **Total** | **100** | __ |

---

## Risk register (known marking-scheme defects to be aware of)

| Defect in marking scheme | Mitigation |
|---|---|
| G119 says banner = "WorldSkills Lyon" — **wrong** | Type "WorldSkills ASEAN Manila" (per project) |
| H120 says "france.jpg" — **wrong** | Use "manila.jpg" (per project) |
| 4 missing aspects (domain pwd 8-char, FGPP 10-char, control GPO, registry GPO) | Implement them anyway — covered in `22_…` |
| Day 4 exists in marking scheme but not in test plan | Practice for 4 days; if 3, you'll be over-prepared |
| Day 2 = Security Hardening (per chief), not CTF | Build SOC + OpenVPN practice rig — see `24_…` and `07_…` |
| All B/C/D flag names are Lyon-leftover | Confirmed by chief: Day 2 = SOC build, Days 3–4 CTF = **Juice Shop + VulnHub random pick** |
| Chief has not yet released the Day 2 deliverables doc (no MA3-equivalent) | Build everything in `24_…` based on Marlon's named tools; verify when doc lands |
| Chief has not yet released ASEAN flag → challenge mapping | Track every challenge you solve — when mapping drops, fill in placeholders above |
| Day 2 may need internet during the build (Marlon mentioned it) | Confirmed reading: tools are **pre-staged on competition VMs**; internet only needed during organisers' prep AND during your at-home practice. No internet on competition day. |

---

End of guide.
