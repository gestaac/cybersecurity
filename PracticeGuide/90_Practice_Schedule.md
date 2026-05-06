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
- [ ] Install ESXi 8 on team server.
- [ ] Create 5 vSwitches + port groups (PG-Internet, PG-LAN, PG-DMZ, PG-Servers, PG-MA1-LAN).
- [ ] Confirm ESXi reachable from both laptops.
- [ ] **Reference:** `02_Setup_Topology.md`

### Day 3 (Wed) — Build MA1 environment (3 h)
- [ ] DC.grimshay.local (90 min)
- [ ] www.grimshay.ca with deliberate flaws (60 min)
- [ ] AMClient1 + AMClient2 (30 min)
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

## Week 2 — Repeat Day 1 + CTF skill drills

### Day 8 (Mon) — Day 1 dry-run #3, against the clock (6 h)
- [ ] Restore snapshots, no peeking at the guide.
- [ ] Goal: ≥ 22/25.

### Day 9 (Tue) — Day 2 prep: Security Onion + OpenVPN (3 h)
- [ ] Download Security Onion 2.4 ISO (~9 GB).
- [ ] Build SO Eval VM (16 GB RAM, 300 GB disk, 2 NICs).
- [ ] Run setup wizard, verify SOC web UI loads.
- [ ] Install Wazuh agent on a Linux + Windows test client.
- [ ] Trigger `curl http://testmyids.com/uid/index.html` — confirm alert in SOC UI.
- [ ] Snapshot SO clean.
- [ ] **Reference:** `07_Setup_SecurityOnion.md`, `24_Day2_SecurityHardening.md` Phase 1

### Day 10 (Wed) — Day 2 dry-run + CTF tooling (3 h)
- [ ] Day 2 dry-run: deploy SO + OpenVPN service (1.5 h).
- [ ] CTF tooling: Node.js 20 LTS + Juice Shop ZIP from `https://github.com/juice-shop/juice-shop/releases/latest` running on `http://localhost:3000`.
- [ ] Import Kali, attach to VMnet1 Host-Only with DHCP.
- [ ] Install Burp, jwt_tool, sqlmap, ffuf, GTFOBins offline.
- [ ] Build CyberChef offline.
- [ ] Download Pwning OWASP Juice Shop PDF + EPUB.
- [ ] In parallel: download all 8 VulnHub VMs (`06_…` Part F) → `D:\VulnHub\`.
- [ ] **Reference:** `24_…`, `05_…`, `06_…`

### Day 11 (Thu) — Juice Shop ★1–★3 (3 h)
- [ ] Find the score-board (★1).
- [ ] Solve every ★1 + ★2 challenge.
- [ ] Start ★3 — at least 5 of them.
- [ ] **Reference:** `50_Day3_CTF_Playbook.md`

### Day 12 (Fri) — VulnHub Boot-to-Root drill #1 (3 h)
- [ ] Spin up **Basic Pentesting: 1** → root.
- [ ] Spin up **DC-1** → 5 flags + root via SUID find.
- [ ] Internalise: Phase 1–5 of the boot-to-root playbook.
- [ ] **Reference:** `51_Day3_VulnHub_BootToRoot.md`, `52_Day4_CTF_Hard.md` Walkthroughs 1+3

### Day 13 (Sat) — Mock full 4-day run (6 h)
- [ ] Hour 0–1: MA1 mock (compressed, 1 h).
- [ ] Hour 1–2.5: MA2 mock (compressed, 1.5 h).
- [ ] Hour 2.5–4: Day 2 mock — re-deploy SO + Wazuh agent + investigate one alert (1.5 h).
- [ ] Hour 4–4.5: Juice Shop sprint — 6 challenges in 30 min, no ebook.
- [ ] Hour 4.5–6: Pick a VulnHub VM you HAVEN'T done (e.g. Mr. Robot, DC-2, Kioptrix Level 1). Try root in 1.5 hours.
- [ ] Score yourself.

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
  - **Security Onion 2.4 ISO (~9 GB).**
  - Kali Linux VM image.
  - All 8 recommended VulnHub VMs (`06_…` Part F) — about 50 GB.
  - Juice Shop offline ZIP + Pwning Juice Shop PDF/EPUB.
  - HackTricks PDF, GTFOBins offline mirror, LinPEAS/WinPEAS, LinEnum.sh.
  - **Wazuh agent installers** (Linux RPM + Windows MSI).
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
- [ ] You can deploy a Security Onion Eval install from snapshot in ≤ 30 min and onboard a Wazuh agent.
- [ ] You can stand up a Linux OpenVPN server (with cert chain) in ≤ 45 min.
- [ ] Your teammate can execute any MA2 step you can (no single point of failure).

Last file: **`99_Marking_Map.md`**.
