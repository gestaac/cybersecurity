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

### Day 9 (Tue) — Juice Shop install + tooling (3 h)
- [ ] Install Docker Desktop or Node 20.
- [ ] Run Juice Shop locally — verify `http://localhost:3000`.
- [ ] Install Burp Suite, import CA cert into Firefox.
- [ ] Install jwt_tool, sqlmap, ffuf, gobuster.
- [ ] Build CyberChef offline (`npm run build`).
- [ ] Download Pwning OWASP Juice Shop PDF + EPUB.
- [ ] Optional: stand up local CTFd, import Juice Shop challenge zip.
- [ ] **Reference:** `05_Setup_JuiceShop.md`, `01_Setup_Tools.md` section 5

### Day 10 (Wed) — Juice Shop ★1–★2 drill (3 h)
- [ ] Find the score-board (★1).
- [ ] Solve every ★1 challenge.
- [ ] Solve every ★2 challenge.
- [ ] Track in spreadsheet: name / ★ / time / hints used.
- [ ] **Reference:** `40_Day2_CTF_Playbook.md`

### Day 11 (Thu) — Juice Shop ★3–★4 drill (3 h)
- [ ] SQL injection deep-dive (Login Admin/Bender/Jim, UNION attacks).
- [ ] JWT manipulation (alg=none, key confusion).
- [ ] CAPTCHA bypass, Forged Coupon.
- [ ] Reset Jim's Password and similar security-question challenges.
- [ ] **Reference:** `41_Day3_CTF_Red.md`

### Day 12 (Fri) — Juice Shop ★5–★6 drill (3 h)
- [ ] Forge admin JWT.
- [ ] SSRF via profile-photo URL.
- [ ] XXE in B2B/Complaint XML upload.
- [ ] Premium Paywall token reverse.
- [ ] Read `main.js` end-to-end at least once.
- [ ] **Reference:** `42_Day4_CTF_Blue.md`

### Day 13 (Sat) — Mock 4-day full run (6 h)
- [ ] Hour 0–1: MA1 mock (compressed, 1 h).
- [ ] Hour 1–3: MA2 mock (compressed, 2 h).
- [ ] Hour 3–6: Juice Shop sprint — fresh container, no ebook, see how many challenges you solve.
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
- Backup USB with all ISOs + tools.
- Offline references: HackTricks PDF, GTFOBins HTML, this guide.
- Notepad (paper) for sketching network diagrams during MA2.

---

## Confidence thresholds (you are ready when…)

- [ ] You can build the entire MA2 environment from snapshots in ≤ 45 min.
- [ ] You can complete MA2 deliverables in ≤ 2.5 hours (leaving 30-min buffer).
- [ ] You score ≥ 22/25 on Criterion A in 2 consecutive dry-runs.
- [ ] You can solve **every Juice Shop ★1 + ★2** in under 90 minutes from a fresh container.
- [ ] You can solve **at least 8 of the ★3–★4** in 3 hours.
- [ ] You can forge a Juice Shop admin JWT from memory (no ebook).
- [ ] Your teammate can execute any MA2 step you can (no single point of failure).

Last file: **`99_Marking_Map.md`**.
