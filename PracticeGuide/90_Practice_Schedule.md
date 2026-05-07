# 90 — 2-Week Practice Schedule

This calendar assumes you and your teammate can practice **3 hours per evening** weekdays + **6 hours per day** on weekends. Adjust as needed.

> Total commitment: ~50 hours over 2 weeks.

---

## Week 1 — Setup + Day 1 mastery

### Day 1 (Mon) — Tooling install (3 h)
- [ ] Both teammates: install VMware Workstation Pro 17.
- [ ] Download all 4 OS ISOs.
- [ ] Install workstation tools (Chrome, PuTTY, WinSCP, Wireshark, Nmap, OpenVPN Connect).
- [ ] **Reference:** `01_Setup_Tools.md`

### Day 2 (Tue) — ESXi + topology (3 h)
- [ ] Install ESXi 8 on team server (3-PC mode) OR set up VMnets (single-PC mode).
- [ ] Create 5 vSwitches + port groups (PG-Internet, PG-LAN, PG-DMZ, PG-Servers, PG-MA1-CMS).
- [ ] Confirm ESXi reachable from PC1 and PC2.
- [ ] **Reference:** `02_Setup_Topology.md` (3-PC) or `02b_…` (single-PC)

### Day 3 (Wed) — Build MA1 environment (CMS pentest target + Kali, 3 h)
- [ ] CMS target VM — Drupal 7 LAMP install on CentOS or Ubuntu (60 min)
- [ ] Bake in the deliberate flaws (weak `john` user + sudo NOPASSWD vim privesc) (15 min)
- [ ] Import Kali Linux VM, set static IP `192.168.2.2` (30 min)
- [ ] Verify `nmap 192.168.2.1` from Kali shows ports 22 + 80
- [ ] Snapshots taken
- [ ] **Reference:** `03_Setup_VMs_MA1.md`

### Day 4 (Thu) — Build MA2 environment, part 1 (3 h)
- [ ] ISP (40 min)
- [ ] pfSense base install (40 min)
- [ ] WINSRV1 promote to manila.com (60 min)
- [ ] WINSRV3 partial CA install (40 min)
- [ ] **Reference:** `04_Setup_VMs_MA2.md` Parts 1–4

### Day 5 (Fri) — Build MA2 environment, part 2 (3 h)
- [ ] WINSRV4 root CA + sign WINSRV3 CSR (45 min)
- [ ] LinSRV1 base install (45 min)
- [ ] Client1, Client2, Client3 (90 min)
- [ ] All snapshots taken
- [ ] **Reference:** `04_Setup_VMs_MA2.md` Parts 5–7

### Day 6 (Sat) — Day 1 dry-run #1 (6 h)
- [ ] **Morning:** MA1 walkthrough — 3 hours, no time pressure first run.
  - Practice the recon, vuln finding, executive summary writing.
- [ ] **Lunch break.**
- [ ] **Afternoon:** MA2 walkthrough — 3 hours. Both teammates work in parallel:
  - Person A: Firewall + PKI + Verification.
  - Person B: LinSRV1 + WinSRV1 AD.
  - Sync every 30 min.
- [ ] After: tally `99_Marking_Map.md`. Note what you missed.
- [ ] **Reference:** `10_…` through `30_…`

### Day 7 (Sun) — Day 1 dry-run #2 (6 h)
- [ ] **Restore all snapshots first.** Time it like the competition (3h + 3h).
- [ ] Aim for ≥ 80% of marks (target 20/25 of Criterion A).
- [ ] Identify your weak spots — write them on a sticky note.

---

## Week 2 — Repeat dry-runs + CTF skill drills

### Day 8 (Mon) — MA1 (Day 1 pentest) dry-run #3, against the clock (3 h)
- [ ] Restore snapshots, no peeking at the guide.
- [ ] Run the 4-task MA1 pentest in ≤ 2.5 hours.
- [ ] Goal: full Q&A answers + 150-word summary + top-3 risks completed.
- [ ] **Reference:** `10_Day1_MA1_Solution.md`

### Day 9 (Tue) — MA2 (Day 2 hardening) dry-run #3, against the clock (3 h)
- [ ] Restore snapshots.
- [ ] Run firewall + LinSRV1 + WinSRV1 + PKI + verification in ≤ 5 hours.
- [ ] Goal: ≥ 22/25 marks for Criterion A2–A8.
- [ ] **Reference:** `20_…` through `30_…`

### Day 10 (Wed) — CTF tooling install + Juice Shop ★1–★2 (3 h)
- [ ] Node.js 20 LTS + Juice Shop ZIP running on `http://localhost:3000`.
- [ ] Burp + Firefox + CA cert imported (`05_…` Step 5).
- [ ] Solve every Juice Shop ★1 + ★2 challenge.
- [ ] In parallel: download all 8 VulnHub VMs (`06_…` Part F) → `D:\VulnHub\`.
- [ ] **Reference:** `05_…`, `06_…`, `50_…`

### Day 11 (Thu) — VulnHub Boot-to-Root drill #1 (3 h)
- [ ] Spin up **Basic Pentesting: 1** → root.
- [ ] Spin up **DC-1** → 5 flags + root via SUID find.
- [ ] Internalise: Phase 1–5 of the boot-to-root playbook.
- [ ] **Reference:** `51_Day3_VulnHub_BootToRoot.md`, `52_Day3_CTF_Advanced.md` Walkthroughs 1+3

### Day 12 (Fri) — VulnHub Boot-to-Root drill #2 (3 h)
- [ ] Spin up **Mr. Robot: 1** → 3 keys.
- [ ] Spin up **DC-2** → wpscan + rbash escape + sudo git privesc.
- [ ] **Reference:** `52_Day3_CTF_Advanced.md` Walkthroughs 2+4

### Day 13 (Sat) — Mock full 3-day run (6 h)
- [ ] Hour 0–2: MA1 mock (full pentest + report).
- [ ] Hour 2–5: MA2 mock (full hardening — pfSense + GPOs + LinSRV1 + PKI + verify).
- [ ] Hour 5–6: Pick a VulnHub VM you HAVEN'T done (e.g. Kioptrix Level 1, Basic Pentesting 2). Try root.
- [ ] Score yourself against `99_Marking_Map.md`.

### Day 14 (Sun) — Reset + final polish (3 h)
- [ ] Re-snapshot every clean VM.
- [ ] Review your weakest area for 90 min.
- [ ] Pack the USB stick: ISOs, tool installers, offline references, your runbook.
- [ ] Sleep early before competition day.

---

## Daily routine

Each session:
1. **5 min stand-up** — what we'll cover today.
2. **15 min review** — yesterday's notes, weak spots.
3. **2 h focused execution** — follow the guide.
4. **20 min retro** — what worked, what didn't, what to fix tomorrow.

---

## What to bring to the competition

- USB stick (per Infrastructure-List): 4 patch cords, your laptop with VMware Workstation, your ESXi server, unmanaged switch, power extension.
- Backup USB(s) — minimum **128 GB total** (or use a **1 TB external SSD** — recommended given that PCs only have 500 GB internal each) — with:
  - All Day-1 OS ISOs (pfSense, CentOS, Win Server 2022, Win 10).
  - Kali Linux VM image.
  - All 8 recommended VulnHub VMs (`06_…` Part F) — about 50 GB.
  - Juice Shop offline ZIP + Pwning Juice Shop PDF/EPUB.
  - HackTricks PDF, GTFOBins offline mirror, LinPEAS/WinPEAS, LinEnum.sh.
  - SecLists, rockyou.txt extracted.
  - This entire `PracticeGuide/` folder.
- Notepad (paper) for sketching network diagrams during MA2.

---

## Confidence thresholds (you are ready when…)

- [ ] You can build the entire MA2 environment from snapshots in ≤ 45 min.
- [ ] You can complete MA2 deliverables in ≤ 2.5 hours (leaving 30-min buffer).
- [ ] You score ≥ 22/25 on Criterion A in 2 consecutive dry-runs.
- [ ] You can solve **every Juice Shop ★1 + ★2** in under 90 minutes from a fresh container.
- [ ] You can solve **at least 8 of the ★3–★4** in 3 hours.
- [ ] You can forge a Juice Shop admin JWT from memory (no ebook).
- [ ] You can root **at least 4 VulnHub VMs** (e.g. Basic Pentesting 1, DC-1, DC-2, Mr. Robot) without peeking at write-ups.
- [ ] You can recite the Phase 1–5 boot-to-root playbook (Discover → Enumerate → Foothold → Privesc → Loot) and the top 5 Linux privesc checks.
- [ ] You can build a CMS pentest target VM (CMS + weak user + privesc) from snapshot in ≤ 15 min.
- [ ] You can stand up a Linux OpenVPN server (with cert chain) in ≤ 45 min.
- [ ] Your teammate can execute any MA2 step you can (no single point of failure).

Last file: **`99_Marking_Map.md`**.
