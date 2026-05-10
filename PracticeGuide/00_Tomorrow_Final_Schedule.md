# 00 — Tomorrow's Final Practice Schedule (D-1, last day before competition)

> **Print this file.** Tick boxes as you go. Run it with your teammate side-by-side.

**Confirmed competition layout (chief Marlon, 2026-05-10):**
- Day 1 — VAPT (MA1 CMS pentest) → 2 flags
- Day 2 — Security Hardening / Infrastructure setup (MA2)
- Day 3 morning — Red Teaming (OWASP Juice Shop)
- Day 3 afternoon — Blue Teaming "Analysis and Exploitation" (still OWASP Juice Shop, confirmed)

**45 flags total on Day 3 = randomly drawn from the 111 OWASP Juice Shop challenges.**

**Realistic K target with 1 day prep: 51 – 65 K out of 100** → solid top-3, fighting range for 2nd.

---

## 🧠 What to memorize with your teammate (drill these together)

Before you start practising, the team should agree these are MUST-MEMORIZE items. Quiz each other tonight, then quiz again tomorrow morning:

| Memorize | Why it matters | Look here |
|---|---|---|
| **CMS-identification command** `whatweb http://192.168.2.1` | Day 1 target is random VulnHub VM — must identify CMS first before picking exploit | Block 1 / `MA1_CMS_Cheatsheet.md` |
| **MA1 privesc checklist** `sudo -l` → SUID `find` → cron → kernel | Day 1 root access — universal across any Linux box | Block 1 / `MA1_CMS_Cheatsheet.md` Step 9 |
| **Hashcat modes** 7900 (Drupal) / 400 (WP, Joomla 3+) / 1800 (Linux shadow) | Wrong mode = no crack, no flag — must match the hash you find | Block 1 / `MA1_CMS_Cheatsheet.md` |
| **CMS scanner per CMS** `droopescan` (Drupal) / `wpscan` (WordPress) / `joomscan` (Joomla) | If you run the wrong scanner you waste 20 min | `MA1_CMS_Cheatsheet.md` Step 3+4 |
| **SQLi pattern** `' OR 1=1--` (with trailing space) | Solves 5+ Juice Shop challenges (Login Admin, Bender, Jim, etc.) | Block 4 Wave 2 |
| **JWT forging steps** (the 8-step recipe) | High-value challenge, unlocks several others | Block 6 Wave 4 |
| **MA2 banner text** `WorldSkills ASEAN Manila` / `Authorized access only` | Exact wording — typo = lost mark | `MA2_Cheatsheet.md` |
| **Snort FIN rule** `alert tcp any any <> $HOME_NET any (flags: F; msg: "Possible FIN scan"; sid: 100001;)` | Type verbatim or it doesn't trigger | `MA2_Cheatsheet.md` |
| **Share permissions** `Marketing=R, Executive=FC` reading `park.jpg` | Marking scheme says wrong values — trust the PDF | `MA2_Cheatsheet.md` |
| **All 4 logins** (MA1 / MA2 / ESXi / Kali) | Wasting 5 min on a forgotten password = wasted points | `MA2_Cheatsheet.md` |
| **Save target + file naming** Desktop → `PHL_Team1_<Module>_<Type>.pdf` | Wrong filename or location = unread by judges | `MA2_Cheatsheet.md` |

> **Drill method:** Member A asks, Member B recites from memory. Switch. 3 wrong answers in a row on any item → re-write that line by hand 3 times.

---

## 📖 Plain-English glossary (read this once before starting)

These terms appear all over the Juice Shop blocks. Skim once, refer back if confused.

| Term | What it means in 1 sentence | Why it matters |
|---|---|---|
| **OWASP Juice Shop** | A deliberately-broken online shop website used as a hacking practice target | This is your Day 3 target — 111 challenges, 45 randomly selected become flags |
| **Score Board** | The built-in challenge tracker inside Juice Shop at `/#/score-board` | Every challenge you "solve" turns green here automatically |
| **Burp Suite** | A tool that sits between your browser and the website so you can see + modify every request | Without it you can only do what the UI lets you — Burp lets you change *anything* |
| **Proxy** | Burp's traffic interceptor running on `127.0.0.1:8080` | Firefox sends traffic here so Burp sees it |
| **SQLi (SQL Injection)** | Typing database commands into a normal text field to fool the server | Example: putting `' OR 1=1--` in a login form → server logs you in as admin |
| **XSS (Cross-Site Scripting)** | Putting JavaScript into a text field so it runs in someone's browser | Example: typing `<script>alert(1)</script>` in a search box pops an alert |
| **DOM XSS** | XSS where the browser itself runs the bad code (no server involved) | Same idea, different mechanism — Juice Shop's search bar is vulnerable |
| **JWT (JSON Web Token)** | A signed string the server gives you after login that proves who you are | If you can forge it, you can become anyone — including admin |
| **JWT forging** | Tricking the server into accepting a fake token by changing the signing method to "none" | Lets you log in as any user without their password |
| **Path traversal** | Using `../` or `%00` tricks in URLs to reach files you shouldn't see | Example: `/ftp/file.md%2500.bak` lets you grab `.bak` files |
| **UNION SQLi** | A SQL trick that pastes a second query's results into the first query's output | Lets you read tables you shouldn't see (like the database schema) |
| **Regex bomb (ReDoS)** | A search pattern carefully crafted to make the server take forever to process | Submitting it crashes the server — Juice Shop has a challenge for this |
| **API endpoint** | A URL the website talks to in the background (like `/api/Users`) | You can call these directly with Burp instead of using the UI |
| **Local Storage** | A small storage box in your browser where the site keeps your login token | DevTools (F12) → Application tab → see + edit it |
| **K (marks)** | Marking-scheme score units (max 100 across the competition) | Each block in this schedule shows the K it targets |
| **Crit A1, A2, etc.** | Sections of the marking scheme (Criterion A1 = MA1, A2-A8 = MA2, B/C/D = Juice Shop) | Tells you which row in the spreadsheet a step earns marks for |

> **First time using Burp?** Read `00_Beginner_Burp_Quickstart.md` first — 10-min walkthrough with troubleshooting. Short version: start Burp → "Temporary project" → "Use Burp defaults" → "Start Burp". Then in Firefox: Settings → search "proxy" → Manual proxy → `127.0.0.1` port `8080`. To see traffic: Burp → Proxy → HTTP history.

> **Shaky on Linux commands?** Read `00_Beginner_Linux_Commands.md` first — every command you'll use tomorrow with examples.

> **First time using Juice Shop?** Open `http://127.0.0.1:3000` in Firefox. The store front loads. Add `/#/score-board` to the URL → that's your challenge tracker. As you complete each challenge, its row turns green automatically. **You don't submit flags — solving the challenge IS the flag.**

---

## 0. Pre-flight setup (07:30 – 08:00, 30 min)

> 🚨 **At competition (setup day BEFORE Day 1):** before any practice, run the Network Discovery sequence in `Network_Discovery_Cheatsheet.md` to verify both NICs got IPs and you can reach ESXi (`192.168.10.10`) + CTFD (`192.168.10.100`).

Member A:
- [ ] Confirm MA1 Drupal target VM boots from snapshot
- [ ] Confirm Kali boots, can `nmap 192.168.2.1`
- [ ] At venue: also run `ip a` + `sudo arp-scan -l` to verify network state

Member B:
- [ ] Install Juice Shop on Kali via Docker:
  ```bash
  sudo apt update && sudo apt install -y docker.io
  sudo systemctl enable --now docker
  sudo docker pull bkimminich/juice-shop
  sudo docker run -d -p 3000:3000 --restart unless-stopped --name juice bkimminich/juice-shop
  curl http://127.0.0.1:3000      # expect 200 OK
  ```
- [ ] Open Firefox → `http://127.0.0.1:3000/#/score-board` → bookmark
- [ ] Configure Burp proxy 127.0.0.1:8080 + import CA cert into Firefox

Both:
- [ ] Both PCs charged + ESXi up + network rig connected
- [ ] Kettle/coffee on. Snacks staged.

---

## 1. MA1 Lock-in (08:00 – 10:00, 2 h) → target 6 / 6 K (Crit A1)

Both members run **independent timed dry-runs** on separate Kali instances. Target: ≤ 90 min each.

References:
- **Primary walkthrough: `PracticeGuide/10_Day1_MA1_Solution_v2_DC1.md`** ⭐ (DC-1 VulnHub VM — closest to what chief will hand you tomorrow)
- DC-1 import + IP setup: `PracticeGuide/DC1_Import_Setup.md`
- Original Drupal walkthrough (older example): `PracticeGuide/10_Day1_MA1_Solution.md`
- **CMS adaptation cheat-sheet: `PracticeGuide/MA1_CMS_Cheatsheet.md`** ⭐

> 🚨 **CRITICAL — Day 1 target = randomly-picked VulnHub VM.** Chief Marlon confirmed they pick the CMS from `vulnhub.com`. The CMS could be Drupal (your practice), WordPress, Joomla, MediaWiki, or other. Step 1 (recon) and Steps 5-11 (user crack → privesc → report) are identical for every CMS. Only Steps 2-4 (identify CMS, find exploit, exploit) change. Read `MA1_CMS_Cheatsheet.md` to know which tool maps to which CMS.

Phases:
- [ ] Phase 1 — `nmap -sC -sV -T4 192.168.2.1` + find secret message (HTML comment / robots / CHANGELOG) → **Crit A1 D31 = 2.0 K**
- [ ] Phase 2a — Identify CMS: `whatweb http://192.168.2.1` → note CMS name + version
- [ ] Phase 2b — Run CMS-specific scanner: `droopescan` (Drupal) / `wpscan` (WordPress) / `joomscan` (Joomla) → see `MA1_CMS_Cheatsheet.md` per-CMS section → **Crit A1 D37 = 1.0 K**
- [ ] Phase 2c — Exploit via msfconsole or manual exploit. If Drupal 7 → Drupalgeddon2 (your practice). If WordPress → vulnerable plugin / weak admin pwd. If Joomla → SQLi or known CVE. Dump users from CMS database.
- [ ] Phase 3 — hashcat with the **correct mode** for the CMS hash format → SSH in → `cat secret.txt` → privesc to root via `sudo -l` / SUID → `cat /root/proof.txt` → **Crit A1 D42 = 1.0 K**

  > 🧠 **MEMORIZE the 4 hashcat modes** (because you don't know which CMS yet):
  > - **`-m 7900`** = Drupal 7 (`$S$`)
  > - **`-m 400`** = WordPress / Joomla 3+ (`$P$` or `$H$`)
  > - **`-m 1800`** = Linux `/etc/shadow` (`$6$`)
  > - **`-m 11`** or **`-m 400`** = Joomla (older / newer)
  >
  > Drill: "Hashcat mode for $S$?" → "7900". "Hashcat mode for $P$?" → "400". Both teammates must answer instantly.

  > 🧠 **MEMORIZE the universal privesc checklist** (one of these usually works):
  > 1. `sudo -l` → if `NOPASSWD` on any binary → check GTFOBins offline mirror
  > 2. `find / -perm -4000 -type f 2>/dev/null` → SUID binaries → check GTFOBins
  > 3. `cat /etc/crontab` + `ls -la /etc/cron.*` → writable cron job
  > 4. `uname -a` → kernel exploit
  >
  > Most common one-liner from sudo path: `sudo /usr/bin/vim -c ':!/bin/bash'`. Both must type from memory in under 10 seconds.
- [ ] Phase 4 — write 150-word executive summary + top-3 risk table → save as `PHL_Team1_MA1_Report.pdf` on Desktop → **Crit A1 D32 = 2.0 K**

At 09:30: swap reports, peer-review for 30 min. Catch each other's gaps.

**Exit gate:** both produce a clean `PHL_Team1_MA1_Report.pdf` in ≤ 90 min.

---

## 2. MA2 Paper-walk (10:00 – 12:00, 2 h) → target 15 – 19 K (Crit A2-A8)

You can't build the 10 MA2 VMs in time. Organisers pre-build them at the venue. Your job today: **memorise the config** so you can fly through tomorrow.

Reference files (read in your assigned track):

**Member A — Firewall + PKI track**
- [ ] `PracticeGuide/20_Day1_MA2_Firewall.md` (pfSense + OpenVPN + Snort)
- [ ] `PracticeGuide/23_Day1_MA2_PKI.md` (CA cert chain)
- [ ] MA2 PDF pages on FW + PKI

**Member B — Linux + AD + Verification track**
- [ ] `PracticeGuide/21_Day1_MA2_LinSRV1.md` (sshd 2022, firewalld, SELinux)
- [ ] `PracticeGuide/22_Day1_MA2_WinSRV1_AD.md` (7 GPOs + share)
- [ ] `PracticeGuide/30_Day1_MA2_Verification.md` (Client1/2/3 tests)

**Each writes ONE A4-page handwritten cheat-sheet** by hand. Use `PracticeGuide/MA2_Cheatsheet.md` as the template. Paper is allowed at the venue.

**At 11:30 — quiz each other for 30 min.** Each wrong answer = re-write that line.

K targets within Crit A2-A8:
- A2 Firewall = 4.95 K
- A3 LinSRV1 = 3.15 K
- A4 WinSRV1 AD/GPOs/share = 2.30 K
- A5 WINSRV3 CA = 0.20 K
- A6 Client1 verification = 4.10 K
- A7 Client2 verification = 2.60 K
- A8 Client3 (External/VPN) = 1.70 K
- **Total possible: 19 K**

---

## 3. Lunch + reset (12:00 – 13:00, 1 h)

Real food. Step away from screens for 30 min. Don't skip — afternoon is the longest block.

---

## 4. Juice Shop Marathon Round 1 — ★1 + ★2 (13:00 – 17:00, 4 h) → target 18 – 22 K

This is **the highest-leverage 4 hours of the entire prep day.** Each Juice Shop challenge solved = 1 of the 45 random flags = direct K.

Reference: `PracticeGuide/05_Setup_JuiceShop.md` + `PracticeGuide/50_Day3_CTF_Playbook.md`

Setup (5 min):
- [ ] Both members on separate Kali boxes, separate Juice Shop containers
- [ ] Score-board open: `http://127.0.0.1:3000/#/score-board`
- [ ] Solution guide open in another tab: `https://pwning.owasp-juice.shop`
- [ ] Burp running on each (proxy 8080, CA cert imported)

### Wave 1 (13:00 – 14:00, 1 h) — every ★1 (8 challenges)

> **What ★1 means:** the easiest tier. These are warm-up challenges — most are "navigate to a hidden URL" or "paste a payload." No complex tools needed beyond Burp + Firefox.

- [ ] **Score Board** — type `http://127.0.0.1:3000/#/score-board` in Firefox URL bar. *Why it works:* Juice Shop hides this page from the menu but the URL still works. Solved instantly on visit.
- [ ] **DOM XSS** — in the top-bar search box, paste `<iframe src="javascript:alert('xss')">` and press Enter. An alert pops. *Why it works:* the search field renders HTML without sanitising it.
- [ ] **Bonus Payload** — same search box, paste this exact long string:
  ```html
  <iframe width="100%" height="166" scrolling="no" frameborder="no" allow="autoplay" src="https://w.soundcloud.com/player/?url=https%3A//api.soundcloud.com/tracks/771984076&color=%23ff5500&auto_play=true&hide_related=false&show_comments=true&show_user=true&show_reposts=false&show_teaser=true"></iframe>
  ```
  *Why it works:* Juice Shop watches for this specific "bonus" string and credits it.
- [ ] **Confidential Document** — visit `http://127.0.0.1:3000/ftp/acquisitions.md`. *Why it works:* a directory of internal files is exposed at `/ftp/`.
- [ ] **Error Handling** — open Burp → Repeater → send `POST /api/Feedbacks` with body `{bad}` (broken JSON). Server returns a stack trace including file paths. *Why it works:* the server doesn't catch malformed JSON cleanly.
- [ ] **Missing Encoding** — visit `/assets/public/images/uploads/😼-#zatschi-#whoneedsfourlegs-1572600969477.jpg` (literally that emoji + name). *Why it works:* the upload filename wasn't URL-encoded, leaking the file.
- [ ] **Privacy Policy** — visit `http://127.0.0.1:3000/#/privacy-security/privacy-policy`. *Why it works:* Juice Shop counts visiting this page as the challenge.
- [ ] **Repetitive Registration** — go to register page, fill in everything, submit, then in Burp's HTTP history right-click the POST `/api/Users` → Send to Repeater → change the `repeatPassword` field to differ from `password` → Send. The server still creates the account. *Why it works:* Juice Shop only checks one of the two password fields.

> **Stuck on any challenge?** Open `https://pwning.owasp-juice.shop/` in another tab. Scroll to the challenge name. Read the official solution. Move on. **No shame in checking — speed matters tomorrow.**

### Wave 2 (14:00 – 15:30, 1.5 h) — split ★2 between members

> **What ★2 means:** still easy but you start using Burp to modify requests. Most are SQL injection (typing DB commands in login fields) or visiting hidden admin pages.

**Member A:**
- [ ] **Login Admin** — go to login page. Email field: `' OR 1=1--` (note the space after `--`). Password: anything. Submit → you're logged in as `admin@juice-sh.op`. *Why it works:* the server's SQL query becomes `WHERE email = '' OR 1=1-- AND password = '...'`. The `--` comments out the password check. **This is classic SQL injection — your most-used trick.**

  > 🧠 **MEMORIZE this exact pattern**: `' OR 1=1--` (apostrophe, space, OR, space, 1, equals, 1, two dashes, space). You'll reuse it on Login Bender, Login Jim, and many others. Both teammates should be able to type this from memory in under 5 seconds.
- [ ] **Admin Section** — once logged in as admin, visit `http://127.0.0.1:3000/#/administration`. *Why it works:* the admin page exists; this challenge fires on URL visit.
- [ ] **View Basket** — log in as any user → press F12 → Application tab → Local Storage → find `bid` (basket id). Change it to `1`. Reload basket page. You see another user's basket. *Why it works:* the server trusts the basket ID from the browser without checking ownership.
- [ ] **Five-Star Feedback** — admin login → visit `/#/administration` → see feedback list → click trash icon next to the 1-star feedback. *Why it works:* admins can delete; the challenge is to find and delete the negative review.
- [ ] **Password Strength** — fires automatically when you log in as admin (you already did this for Login Admin). *Why it works:* admin's password (`admin123`) is in common wordlists.

**Member B:**
- [ ] **Reset Jim's Password** — click login → "Forgot Password" → enter `jim@juice-sh.op` → security question is "Your eldest siblings middle name?" → answer is **`Samuel`** (it's leaked in a product review for the "Apple Juice (1000ml)" product — read the review). Set new password. *Why it works:* the security answer is stored in plaintext in product reviews.
- [ ] **Login Bender** — login page. Email: `bender@juice-sh.op'--`. Password: anything. *Why it works:* same SQLi pattern as admin login but targeting a different user.
- [ ] **Login Jim** — same pattern: email `jim@juice-sh.op'--`, any password.
- [ ] **Zero Stars** — submit any feedback in the UI (rating must be ≥ 1 in UI). In Burp HTTP history, right-click POST `/api/Feedbacks` → Send to Repeater → change `"rating":1` to `"rating":0` → Send. *Why it works:* the UI prevents 0-star but the server doesn't.
- [ ] **Email Leak** — admin login → `/#/administration` → the user list shows all emails. *Why it works:* the admin page leaks user emails that should be private.

### Wave 3 (15:30 – 17:00, 1.5 h) — finish ★2 + start ★3

> **What ★3 means:** medium difficulty. You'll need Burp to modify JSON bodies, and SQLi gets more advanced (UNION queries to read other tables).

- [ ] Whoever finishes Wave 2 first **helps the other catch up** — pair-debug, don't push ahead.
- [ ] **Forged Coupon** — log in → add items to basket → checkout → in Burp catch POST `/rest/basket/.../checkout` → modify `couponData` to a future date. *Why it works:* coupon validation is client-side only.
- [ ] **Forged Feedback** — log in as user1 → submit feedback → in Burp catch POST `/api/Feedbacks` → add `"UserId": 2` to the JSON body → Send. *Why it works:* the API trusts the UserId field instead of the logged-in session.
- [ ] **Database Schema** — go to product search. In the search box paste:
  ```
  qwert')) UNION SELECT sql,2,3,4,5,6,7,8,9 FROM sqlite_schema--
  ```
  Press Enter. The product results now contain the database schema (table definitions). *Why it works:* the search uses raw SQL with an unfiltered query — UNION pastes the sqlite schema into the result.
- [ ] **GDPR Data Erasure** — admin login → `/#/privacy-security/data-export` → request data → in Burp modify the export user — too complex for now, skip if stuck after 10 min.
- [ ] **Christmas Special** — search products for `Christmas`. The result includes a hidden Christmas product (id 10) marked deleted. Add it to basket via Burp: POST `/api/BasketItems` body `{"BasketId":<your_bid>,"ProductId":10,"quantity":1}`. *Why it works:* deleted products aren't really deleted, just hidden from the UI.

> 🎯 **Marking hits:** Crit B (HO + CM) = 25 K + part of Crit C (ODD) = 5 K → block target ≈ 18 – 22 K

**Hard rule:** 10-minute timeout per challenge. Stuck → check `pwning.owasp-juice.shop` → learn → move on.

---

## 5. Snack break + score check (17:00 – 17:30, 30 min)

Pull up the scoreboard. Count solved.

| Solved by 17:00 | Trajectory |
|---|---|
| 30+ | On track for 2nd place |
| 20 – 30 | On track for 3rd |
| < 20 | Push harder in evening — call out which categories you avoided |

---

## 6. Juice Shop Round 2 — ★3 + ★4 + selected ★5 (17:30 – 21:00, 3.5 h) → target 12 – 18 K

### Wave 4 (17:30 – 19:00, 1.5 h) — finish ★3 in parallel

- [ ] **Forged Review** — log in → click any product → write a review → in Burp catch PUT `/rest/products/<id>/reviews` → change `author` to another user's email. *Why it works:* the review API trusts the author field from the request body.
- [ ] **Manipulate Basket** — log in as user1 → add an item → in Burp catch POST `/api/BasketItems` → change `BasketId` to `2` (another user's basket). *Why it works:* same broken access control pattern as View Basket.
- [ ] **Reset Bender's Password** — Forgot Password → `bender@juice-sh.op` → security question "Name of your favorite pet?" → answer **`Stop the Calculator`** (clue: his catchphrase from Futurama).
- [ ] **JWT Forging** ⭐ (do TOGETHER — high-value, unlocks several other challenges):

  > 🧠 **MEMORIZE these 8 steps as a team drill** — one of you reads, the other does, then swap. By bedtime both should be able to perform JWT forging without looking.

  **Step-by-step (this is the hardest challenge of the day, but you can do it):**
  1. Log in as admin (Login Admin SQLi from earlier).
  2. Press F12 → Application tab → Local Storage → copy the `token` value (a long string with two dots).
  3. Open `https://jwt.io` (allowed offline if cached, otherwise use the `jwt_tool` CLI). Paste the token.
  4. The token has 3 parts: header, payload, signature. The header says `"alg": "HS256"`.
  5. Change the header to `{"typ":"JWT","alg":"none"}`. Change the payload's `email` to `jwtn3d@juice-sh.op`.
  6. The new token is just `header.payload.` (with no signature — alg is "none").
  7. In Burp, replay any authenticated request (e.g. GET `/rest/user/whoami`) but replace the `Authorization: Bearer ...` header with your forged token.
  8. Server accepts it → challenge solved.

  *Why it works:* the JWT library accepts `alg: none`, meaning "no signature required." This is a real-world vulnerability that hit several apps in 2015-2018.

### Wave 5 (19:00 – 20:30, 1.5 h) — top ★4 + ★5 picks

- [ ] **Easter Egg** — visit `http://127.0.0.1:3000/ftp/eastere.gg`. Server says blocked. Try `http://127.0.0.1:3000/ftp/eastere.gg%2500.md`. *Why it works:* the `%2500` is a URL-encoded null byte that bypasses the file-extension whitelist (server reads it as `.md` ✓ but the OS opens `.egg`).
- [ ] **Forgotten Sales Backup** — same trick: `http://127.0.0.1:3000/ftp/coupons_2013.md.bak%2500.md`. *Why it works:* same null-byte path traversal.
- [ ] **Misplaced Signature File** — `http://127.0.0.1:3000/ftp/incident-support.kdbx`. *Why it works:* `.kdbx` is whitelisted accidentally; the file is sensitive (KeePass DB).
- [ ] **Reset Bjoern's Password** — Forgot Password → `bjoern@owasp.org` → security question is "company you first worked for" → answer **`SC$#%TF-Cool!`** (it's a hint in his profile; alternatively read the ebook for the exact answer).
- [ ] **Successful RCE DoS** — submit feedback (logged-in user). In the comment field, paste a "regex bomb": `/((a+)+)+$/`. Wait. Server hangs. *Why it works:* the regex tries to match in exponential time and pegs the CPU — Denial of Service via regex (called "ReDoS").

### Wave 6 (20:30 – 21:00, 30 min) — Blue-side category sweep

> **What this means:** chief confirmed Blue Teaming on Day 3 PM is "Analysis and Exploitation" of Juice Shop — likely focused on the categories below. Open the score-board, filter by category, and at least *attempt* every challenge in each. Even partial progress sometimes scores.

How to filter on the score-board: open `/#/score-board` → click the **"Category"** dropdown at the top → pick one.

- [ ] **Sensitive Data Exposure** — challenges where the server leaks data it shouldn't (emails, files, customer info). Try every one.
- [ ] **Cryptographic Issues** — weak passwords, broken crypto, exposed keys. Most are "find the key in the code or files."
- [ ] **Security Misconfiguration** — server settings that leak info (error pages, default creds, debug endpoints). Mostly URL-visit challenges.
- [ ] **Vulnerable Components** — known-vulnerable libraries used by Juice Shop. Often "find the version" type challenges.
- [ ] **Improper Input Validation** — like Repetitive Registration. Common pattern: client-side validation only; bypass via Burp.

> 🎯 **Marking hits:** Crit C (ODD + CC) = 25 K + Crit D (CS / Blueday) = 25 K → block target ≈ 12 – 18 K

**Random-selection insight:** since the 45 flags are randomly drawn from 111, **breadth beats depth**. Touch as many categories as possible — don't camp on one ★6 for 2 hours. **Skip > camp.**

---

## 7. Pack + final checks (21:00 – 21:30, 30 min)

Backpack:
- [ ] PC1, PC2, ESXi server (3 boxes)
- [ ] Patch cords ×4+
- [ ] Unmanaged switch
- [ ] Power strip / extension

USB / 1 TB SSD:
- [ ] All OS ISOs (pfSense, CentOS, Win Server 2022, Win 10, Kali)
- [ ] **Juice Shop offline tarball** (CRITICAL)
- [ ] Pwning OWASP Juice Shop PDF
- [ ] rockyou.txt
- [ ] GTFOBins offline mirror, LinPEAS
- [ ] HackTricks PDF
- [ ] The entire `PracticeGuide/` folder

Memorised + on cheat-sheet:
- [ ] Both handwritten MA2 cheat-sheets (Member A's + B's)
- [ ] Country prefix: **`PHL_Team1_…`** for every saved file
- [ ] Save target: **Desktop** of competitor workstation
- [ ] MA1 login: `competitor1a / Boracay@14!`
- [ ] MA2 login: `competitor1b / Tagaytay_62&L`
- [ ] ESXi: `wsauser / Andres@9V4` at `192.168.1.1`

Notepad + 2 pens.

---

## 8. Sleep (21:30 – 06:00 next day)

**Hard rule: lights out by 22:00. Sleep ≥ 7 hours.**

Tired teams lose more marks to silly mistakes than they gain from one more cram hour. Trust the prep.

---

## Score map summary (realistic)

| Block | K targeted | Cumulative |
|---|---|---|
| 1. MA1 lock-in | 6 | 6 |
| 2. MA2 paper-walk (banked for venue) | 15 – 19 | 21 – 25 |
| 4. Juice Shop ★1 + ★2 + early ★3 | 18 – 22 | 39 – 47 |
| 6. Juice Shop ★3 + ★4 + ★5 + Blue sweep | 12 – 18 | **51 – 65** |

**Realistic landing zone: 51 – 65 K** → solid top-3, fighting range for 2nd.

---

## Two non-negotiables

1. **Both teammates must complete each block.** No "I'll catch up later." Solo gaps = team gaps.
2. **At every block transition, do a 5-min sync** — what worked, what didn't, who's strong on what. Tomorrow's sync rhythm at home becomes your sync rhythm at the venue.

---

## Day-of-competition reminders (read morning-of, not now)

1. **Read deliverable lines literally.** Type wording exactly as the PDF says. ("WorldSkills ASEAN Manila", not "Lyon".)
2. **Save with country code** — `PHL_Team1_<module>_…` on Desktop.
3. **Functional test from a client, every time.** Server config is worthless if Client1/2/3 can't actually use it.
4. **Snapshot before risky changes** (pfSense and WINSRV3 especially).
5. **No internet, no AI tools, no external write-ups during the CTF.** Automatic DQ.

Good luck. Execute clean. The plan is enough.
