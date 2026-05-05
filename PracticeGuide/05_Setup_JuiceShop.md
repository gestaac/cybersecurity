# 05 — Setup OWASP Juice Shop (the actual CTF target)

The chief (Marlon) confirmed: **the CTF will use OWASP Juice Shop**. Juice Shop is a deliberately-vulnerable Node.js + Angular web shop with **100+ challenges** across the OWASP Top 10. It exports natively to **CTFd** (the official scoring platform listed in `Capture-The-Flag-Regional-Skills-Olympics-Cybersecurity.docx`).

You **must** install Juice Shop locally on your Kali VM (or any Linux/Windows box) so you can practice the exact same target before competition day.

> Time: 30 min first install, 5 min on subsequent runs.

---

## What is OWASP Juice Shop?

- A modern web app (Angular SPA + Node.js/Express REST API + SQLite by default).
- Every challenge corresponds to a real OWASP-Top-10 class flaw.
- Built-in **score-board** at `/#/score-board` that ranks 100+ challenges by difficulty (1 ★ easy → 6 ★ insane).
- **Free + open-source**: `https://owasp.org/www-project-juice-shop/`
- Source: `https://github.com/juice-shop/juice-shop`
- The companion ebook **"Pwning OWASP Juice Shop"** (Björn Kimminich) walks every challenge — also free: `https://pwning.owasp-juice.shop/`

---

## Step 1 — Install (pick ONE of three methods)

### Method A — Docker (recommended, 2 min)
**Pre-req:** Docker Desktop on Windows or Docker on Linux.

Install Docker Desktop on Windows (free for personal use): `https://www.docker.com/products/docker-desktop/`

Then:
```bash
docker pull bkimminich/juice-shop
docker run --rm -d -p 3000:3000 --name juiceshop bkimminich/juice-shop
```

**Verify:** open `http://localhost:3000` in a browser — you should see the OWASP Juice Shop store.

To stop:
```bash
docker stop juiceshop
```

To restart later:
```bash
docker run --rm -d -p 3000:3000 --name juiceshop bkimminich/juice-shop
```

> **Why Docker:** clean wipe between practice runs (`--rm` deletes everything when stopped, so the next run starts at challenge zero).

### Method B — Pre-built ZIP (no Docker, 5 min)
1. Go to GitHub releases: `https://github.com/juice-shop/juice-shop/releases/latest`
2. Download the platform-appropriate archive:
   - Windows: `juice-shop-<version>_node20_windows_x64.zip`
   - Linux: `juice-shop-<version>_node20_linux_x64.tgz`
3. Extract → open the folder in a terminal:
   ```
   cd juice-shop_<version>
   npm start
   ```
4. Browse to `http://localhost:3000`.

### Method C — From source (15 min, only if A and B fail)
Pre-req: Node.js 20.x LTS (`https://nodejs.org/`) + git.
```bash
git clone https://github.com/juice-shop/juice-shop.git
cd juice-shop
npm install            # ~3 min
npm start              # ~30 sec to boot
```
Open `http://localhost:3000`.

---

## Step 2 — Verify the install

In your browser:
- `http://localhost:3000` → loads the store front.
- Open DevTools (F12) → **Console** tab → you should see a few green-text "Welcome" messages from Juice Shop.

Try one quick challenge to confirm everything works:
- Click the menu icon (top-left) → **About Us** → scroll to bottom → click the link with text "check out our terms of use" → that triggers the *"Privacy Policy"* challenge ★.
- Then visit `http://localhost:3000/#/score-board` — the score-board challenge should now show as solved.

---

## Step 3 — Stage the companion ebook for offline use

You **cannot** access the internet during the CTF. Download the ebook now and keep it on your USB.

### Option 1 — PDF/EPUB from Leanpub (free)
1. Go to `https://leanpub.com/juice-shop`
2. Set price to **$0** (free) → Get the book → download both PDF and EPUB.
3. Save to your USB as `Pwning_Juice_Shop.pdf`.

### Option 2 — GitBook web export
1. Visit `https://pwning.owasp-juice.shop/`
2. Use browser → *Print → Save as PDF* for the chapters you need most:
   - **Part I — Hacking preparations** (essentials)
   - **Part II — Challenge hunting** (every challenge solution!)
   - **Appendix A — Challenge solutions** (one-line spoilers)

### Option 3 — Clone the repo
```bash
git clone https://github.com/juice-shop/pwning-juice-shop.git
```
The whole book is in `*.md` files — readable offline in any editor.

> 🚨 **Critical:** the appendix has **every challenge solution**. Read Part II to learn the *technique*, then check Appendix A only if stuck. During real competition you won't have it open while solving — practice without it on your last 2 dry-runs.

---

## Step 4 — CTFd export (so you can score yourself)

Juice Shop ships a CTFd-compatible export. To set up a local CTFd + Juice Shop to mirror the competition platform:

### 4.1 Install CTFd locally (Docker)
```bash
git clone https://github.com/CTFd/CTFd.git
cd CTFd
docker-compose up -d
```
Open `http://localhost:8000` → first-run wizard, create admin account.

### 4.2 Generate Juice Shop's CTFd challenges file
```bash
npm install -g juice-shop-ctf-cli
juice-shop-ctf
```
Wizard prompts:
- CTF framework: **CTFd**
- Juice Shop URL: `http://localhost:3000`
- CTF key (any string): `team1-practice`
- Insert hints: **paid** (so you practice both with and without hints)
- Country mapping: skip
Output: `OWASP_Juice_Shop.<timestamp>.zip`

### 4.3 Import into CTFd
- CTFd admin → **Config → Backup → Import** → upload the zip → import challenges + flags.
- Now your local CTFd has all Juice Shop challenges with their flags. You can practice against the same scoring system you'll see at the venue.

---

## Step 5 — Initial walkthrough (10 min, do this once)

Familiarise both teammates with the UI:

1. **Browse the store** — note prices, products, search bar (top), basket, login.
2. **Find the score-board** (★ challenge): `http://localhost:3000/#/score-board`. Bookmark it.
3. **Open DevTools → Network tab** — refresh the page, observe `/api/Products`, `/api/Quantitys`, `/rest/user/whoami`, `/api/Users` etc.
4. **Open Burp Suite** → set Firefox proxy to 127.0.0.1:8080 → Burp installs CA cert → re-load Juice Shop → confirm requests show in Burp's HTTP history.
5. **Login as admin** with the easiest SQLi (you'll learn it formally in `40_…`):
   - Email: `' OR 1=1 --`
   - Password: anything
   - → logged in as `admin@juice-sh.op`.

---

## Step 6 — Tools that pair with Juice Shop (install if not yet)

| Tool | Purpose | Install |
|---|---|---|
| **Burp Suite Community** | THE tool for every challenge | https://portswigger.net/burp/communitydownload |
| **Browser DevTools** | Built into Firefox/Chrome (F12) | — |
| **CyberChef offline** | Encode/decode/encrypt | clone `https://github.com/gchq/CyberChef` → `npm run build` → `dist/` is offline-usable |
| **jwt_tool** | JWT manipulation | `pip install jwt-tool` or `git clone https://github.com/ticarpi/jwt_tool` |
| **sqlmap** | Auto-SQLi (some Juice Shop endpoints are blind) | `apt install sqlmap` |
| **ffuf / gobuster** | Endpoint enum | `apt install ffuf gobuster` |
| **Postman / Hoppscotch** | Replaying API calls | `https://hoppscotch.io/download` (offline-installable) |
| **OWASP ZAP** | Burp alternative | `apt install zaproxy` |

> Burp + Firefox + DevTools + the ebook is the minimum viable kit.

---

## Step 7 — Reset between runs

Juice Shop tracks your progress in a local SQLite file. To restart from scratch:

### Docker
```bash
docker stop juiceshop          # the --rm flag wipes data
docker run --rm -d -p 3000:3000 --name juiceshop bkimminich/juice-shop
```

### Source / ZIP
```bash
# Stop with Ctrl+C, then:
rm data/juiceshop.sqlite       # if it exists
npm start
```

### Restore from a snapshot
Take a Kali VM snapshot **after** install but **before** practice, so you can go back to "fresh install + tools loaded" in one click.

---

## Step 8 — Practice routine (use with `90_Practice_Schedule.md`)

| Session | Target |
|---|---|
| First time | Just play. Find the score-board. Solve any 5 ★1 challenges. |
| Day 10 | Solve **all ★1 + ★2** without using the ebook. |
| Day 11 | Solve **all ★3** with ebook for hints only. |
| Day 12 | Attempt **★4 and ★5**. |
| Day 13 | Mock 3-hour timed sprint. |
| Day 14 | Reset, mock 6-hour run; only check ebook *after* trying for 30 min. |

> Track every solved challenge in a spreadsheet: name, category, ★, time taken, hints used. This becomes your CTF report.

---

## What to bring to competition

USB contents:
- [ ] Juice Shop offline ZIP (so you have it even if container registry is unreachable)
- [ ] Pwning OWASP Juice Shop PDF + EPUB
- [ ] CTFd local copy (in case practice between rounds)
- [ ] Burp Suite Community installer
- [ ] Firefox + Burp's CA cert
- [ ] CyberChef offline build
- [ ] jwt_tool, sqlmap, ffuf

---

Next: jump to **`40_Day2_CTF_Playbook.md`** — methodology + walkthrough of every ★1 and ★2 challenge.
