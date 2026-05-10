# `_legacy/` — Obsolete files (do NOT use for tomorrow's competition)

These 8 guide files were moved here on **2026-05-10** after chief Marlon confirmed the final 3-day competition structure.

## Why these files are obsolete

The old guide assumed:
- Day 2 PM = Crit B "IR + Forensics + AppSec" using Security Onion
- Day 3 = Red CTF (VulnHub boot-to-root + Juice Shop) + Blue CTF (PCAP/memory/disk forensics)

The **actual** competition (chief's confirmed plan):
- Day 2 = Security Hardening (full day, MA2 only)
- Day 3 morning = Red Teaming on **OWASP Juice Shop**
- Day 3 afternoon = Blue Teaming on **OWASP Juice Shop** (Analysis and Exploitation)

So none of the IR / forensics / VulnHub content matches the real competition.

## What's in here

| File | Was for | Why obsolete |
|---|---|---|
| `06_Setup_VulnHub.md` | Day 3 Red CTF — install 8 VulnHub VMs | Day 3 is Juice Shop only |
| `07_Setup_SecurityOnion.md` | Day 2 PM IR platform | Day 2 PM is still hardening |
| `40_Day2_IR_Forensics.md` | Crit B "HO Flags" forensics walkthroughs | HO Flags map to Juice Shop challenges |
| `41_Day2_MalwareIR.md` | Crit B "CM Flags" malware analysis | CM Flags map to Juice Shop challenges |
| `42_Day2_AppSec.md` | Crit B AppSec sub-block | AppSec folded into Day 3 Juice Shop |
| `51_Day3_VulnHub_BootToRoot.md` | Day 3 Red CTF VulnHub | Not in competition |
| `52_Day3_CTF_Advanced.md` | Day 3 Red CTF advanced VulnHub | Juice Shop ★5-★6 covered in `00_Tomorrow_Final_Schedule.md` Wave 5 |
| `53_Day3_BlueCTF.md` | Day 3 Blue CTF (PCAP/memory/disk) | Day 3 Blue is Juice Shop |

## Why kept (not deleted)

1. **Recovery** — if chief revises again, content is one move-back away
2. **Reference** — some skills (PCAP analysis, memory forensics) are useful for future competitions
3. **Git history** — easier to see what changed without combing through deleted files

## Use the active guide instead

Active practice files (in the parent `PracticeGuide/` folder):

| For | File |
|---|---|
| **TOMORROW'S SCHEDULE** | `00_Tomorrow_Final_Schedule.md` |
| MA2 memorization | `MA2_Cheatsheet.md` |
| First-time Burp users | `00_Beginner_Burp_Quickstart.md` |
| First-time Linux users | `00_Beginner_Linux_Commands.md` |
| Day 1 MA1 walkthrough | `10_Day1_MA1_Solution.md` |
| Day 2 MA2 walkthroughs | `20_…` through `30_…` |
| Day 3 Juice Shop methodology | `50_Day3_CTF_Playbook.md` |
| Juice Shop install | `05_Setup_JuiceShop.md` |
