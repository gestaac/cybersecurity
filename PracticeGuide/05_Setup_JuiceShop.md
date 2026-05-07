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

## Step 1 — Install (pick ONE of two methods)

> Both methods need **Node.js 20.x LTS** installed first: download from `https://nodejs.org/` → install with default options → confirm with `node --version` (should print `v20.x.x`).

### Method A — Pre-built ZIP (recommended, 5 min)
1. Go to GitHub releases: `https://github.com/juice-shop/juice-shop/releases/latest`.
2. Download the platform-appropriate archive:
   - Windows: `juice-shop-<version>_node20_windows_x64.zip`
   - Linux: `juice-shop-<version>_node20_linux_x64.tgz`
3. Extract to a folder (e.g. `D:\juice-shop\`).
4. Open a terminal in that folder:
   ```
   cd D:\juice-shop\juice-shop_<version>
   npm start
   ```
5. Wait ~30 sec → browse to `http://localhost:3000`.

To stop: press **Ctrl+C** in the terminal.
To restart: re-run `npm start`.

### Method B — From source (15 min, only if Method A fails)
Pre-req: Node.js 20.x LTS + git (`https://git-scm.com/downloads`).
```bash
git clone https://github.com/juice-shop/juice-shop.git
cd juice-shop
npm install            # ~3 min — pulls all dependencies
npm start              # ~30 sec to boot
```
Open `http://localhost:3000`.

> **Why these methods:** simpler than Docker for Windows users new to security tooling. No daemon to manage, no virtualisation conflicts with VMware. The trade-off is reset takes a couple more steps (Step 7 below) but it's not significant.

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

## Step 4 — Score yourself (built-in score-board, no CTFd needed)

Juice Shop has its own built-in score-board at `http://localhost:3000/#/score-board`. Each challenge auto-ticks green the moment you trigger it. **For practice, this is enough** — you don't need a separate CTFd to track progress.

### 4.1 Use the native score-board
1. Bookmark `http://localhost:3000/#/score-board`.
2. As you solve, the row turns green and shows the points you'd score in CTFd terms.
3. Filter by category, ★ difficulty, or status (solved/unsolved) using the toolbar at the top.

### 4.2 (Optional) Generate a flag-list file for offline reference
If you want a text file with every challenge's expected flag (handy if a flag fails to register and you need to debug):

```bash
npm install -g juice-shop-ctf-cli
juice-shop-ctf
```
Wizard prompts:
- CTF framework: pick anything (e.g. CTFd)
- Juice Shop URL: `http://localhost:3000`
- CTF key (any string): `team1-practice`
- Insert hints: **paid** (so you practice both with and without)
- Country mapping: skip
Output: `OWASP_Juice_Shop.<timestamp>.zip` containing the flag list.

> If you really want a separate CTFd UI later (when Marlon confirms the venue platform), the install isn't covered here — too many moving parts without Docker. The native score-board is functionally identical for practice.

---

## Step 5 — Configure Burp Suite + Firefox (one-time, ~10 min)

The single most important setup outside Juice Shop itself. Without this, you can't see/modify any HTTP traffic during the CTF.

### 5.1 Set Firefox to use Burp as proxy
**Why:** so all browser traffic flows through Burp where you can intercept/replay it.

1. Open **Burp Suite Community** → *Temporary project → Use Burp defaults → Start Burp*.
2. Burp UI → *Proxy → Proxy settings* → confirm a listener is on `127.0.0.1:8080` (default).
3. In **Firefox**: *Settings → search "proxy" → Network Settings → Settings*.
4. Choose **Manual proxy configuration**:
   - HTTP Proxy: `127.0.0.1`  Port: `8080`
   - ✅ Tick *"Also use this proxy for HTTPS"* (or *"Use this proxy server for all protocols"* depending on Firefox version).
   - No Proxy For: leave empty (or `localhost` if you want to bypass for some local pages — but for Juice Shop you DO want it through Burp, so leave empty).
5. OK to save.

> 💡 **Better workflow:** install the **FoxyProxy** Firefox add-on (`https://addons.mozilla.org/firefox/addon/foxyproxy-standard/`) so you can toggle the proxy on/off with one click instead of editing settings each time.

### 5.2 Install Burp's CA certificate in Firefox

Without this, every HTTPS site shows a certificate error because Burp re-signs traffic with its own CA.

1. With proxy active, in Firefox visit: `http://burp` (or `http://burpsuite`).
2. Top-right corner of the page → click **CA Certificate** → downloads `cacert.der`.
3. In Firefox: *Settings → Privacy & Security → scroll to Certificates → View Certificates → Authorities tab → Import*.
4. Select the downloaded `cacert.der`.
5. Tick ✅ **"Trust this CA to identify websites"** → OK.
6. Restart Firefox.

**Verify:** browse to `https://www.google.com` → should load with **no** certificate warning. (Burp's HTTP history will also show the request.)

### 5.3 Initial walkthrough (do this once)

Familiarise both teammates with Juice Shop's UI:

1. **Browse the store** — note prices, products, search bar (top), basket, login.
2. **Find the score-board** (★1 challenge): `http://localhost:3000/#/score-board`. Bookmark it.
3. **DevTools → Network tab** — refresh the page, observe `/api/Products`, `/api/Quantitys`, `/rest/user/whoami`, `/api/Users` etc.
4. **Burp Suite → Proxy → HTTP history** — confirm Juice Shop requests now appear (proxy + cert install both worked).
5. **Login as admin** with the easiest SQLi:
   - Email: `' OR 1=1 --`
   - Password: anything
   - → logged in as `admin@juice-sh.op`. The score-board ticks off the *"Login Admin"* challenge.

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

### From ZIP install
```
# Stop with Ctrl+C in the terminal running 'npm start'
# Then delete the local DB:
del data\juiceshop.sqlite     # Windows
# or:  rm data/juiceshop.sqlite   on Linux
# Then start again:
npm start
```

### From source install
Same — `npm start` writes its DB into `data/juiceshop.sqlite`. Delete that file and re-run.

### Tip — keep a clean copy
After a fresh `npm install`, copy the whole folder to `juice-shop-CLEAN/`. To reset, delete the working folder, copy the clean one back, run `npm start`. Avoids re-running `npm install` every time.

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
- [ ] Juice Shop pre-built ZIP (Windows + Linux variants)
- [ ] Node.js 20 LTS installer (Windows + Linux)
- [ ] Pwning OWASP Juice Shop PDF + EPUB
- [ ] Burp Suite Community installer
- [ ] Firefox installer + Burp's CA cert (`cacert.der`)
- [ ] FoxyProxy add-on `.xpi` file (for offline install)
- [ ] CyberChef offline build (`dist/` folder)
- [ ] jwt_tool, sqlmap, ffuf

---

Next: jump to **`50_Day3_CTF_Playbook.md`** — methodology + walkthrough of every ★1 and ★2 challenge.
