# 50 — Day 3 (CTF morning) — OWASP Juice Shop: Methodology + Easy (★1–★2)

> Day plan reminder: Day 1 = MA1+MA2; **Day 3 morning = Security Hardening (`24_…`)**; **Day 3 (this file) + Day 4 = CTF.**

**Time budget:** ~3 hours warm-up on Day 3 morning, then VulnHub afternoon. Adjust if the actual schedule varies.
**Target:** `http://localhost:3000` (your local instance from `05_Setup_JuiceShop.md`).

> 🚫 No internet, no AI tools. The Pwning OWASP Juice Shop ebook is the only outside reference allowed (because it ships with the project itself, not because rules permit external write-ups — verify with the chief).

> 🎯 Easy challenges (★1–★2) typically score the bulk of "free" marks. Aim to solve **all of them** in the first 90 minutes.

---

## Part A — Universal methodology for every Juice Shop challenge

```
1. Read the challenge description on the score-board (#/score-board).
2. Note the category (Injection, XSS, Broken Auth, etc.).
3. Open Burp Suite → ensure Firefox is proxying through 127.0.0.1:8080.
4. Reproduce the relevant action in the UI; watch the request in Burp.
5. Modify the request to exploit the vuln; observe the response.
6. The flag is automatically credited when Juice Shop detects the exploit.
   Either:
     - a notification appears in the UI ("Challenge solved!"), or
     - the score-board ticks the row green.
7. Submit the flag in CTFd (when scored separately at competition).
```

### Burp Suite quick-start
- *Proxy → Intercept off* (we just want HTTP history).
- *HTTP history* shows every request — right-click any → **Send to Repeater**.
- Repeater: edit headers/body → Send → see response in right pane.
- *Decoder* tab — base64/URL/JWT decode in 1 click.

### Browser DevTools quick-start
- F12 → **Network** → see all requests + responses.
- **Application → Local Storage / Session Storage / Cookies** → JWT lives in `token`.
- **Console** → run JS directly: `localStorage.token`.

---

## Part B — Score-board first (the meta-challenge)

### Challenge: "Score Board" (★1)
**Description:** Find the score-board.

**Solution:**
- Visit `http://localhost:3000/#/score-board` directly.
- The Angular routing exposes it but no link points to it.

> Why it works: Juice Shop's SPA has unlinked routes. View HTML source → find `main.js` → search for "score-board" string → confirms the route.

---

## Part C — Easy challenges (★1) walkthroughs

### 1. Privacy Policy (★1, Miscellaneous)
**Step:** Click menu (top-left) → *About Us* → at bottom click *Customer Privacy* link → *or* directly visit `/#/privacy-security/privacy-policy`.

### 2. DOM XSS (★1, XSS)
**Step:** In the search bar, type:
```html
<iframe src="javascript:alert(`xss`)">
```
Submit → an alert pops.

> Real-world: Angular's `bypassSecurityTrustHtml()` was misused on the search field.

### 3. Bonus Payload (★1, XSS)
**Step:** Search for:
```html
<iframe width="100%" height="166" scrolling="no" frameborder="no" allow="autoplay" src="https://w.soundcloud.com/player/?url=https%3A//api.soundcloud.com/tracks/771984076&color=%23ff5500&auto_play=true&hide_related=false&show_comments=true&show_user=true&show_reposts=false&show_teaser=true"></iframe>
```
Same DOM-XSS sink, different payload — the special "bonus" string Juice Shop watches for.

### 4. Confidential Document (★1, Sensitive Data Exposure)
**Step:** Visit `http://localhost:3000/ftp/acquisitions.md`.

> Real-world: a static-file route (`/ftp/`) exposes internal docs. The file extension is whitelisted (`.md`).

### 5. Login Admin (★2, Injection)
**Step:** On login page, email = `' OR 1=1 --` (with trailing space), password = anything → submit. You're now `admin@juice-sh.op`.

> Real-world: classic SQL injection in `SELECT * FROM Users WHERE email = '...' AND password = '...'`. The `--` comments out the password check.

### 6. Login MC SafeSearch (★2, Injection)
**Step:** Email = `mc.safesearch@juice-sh.op` then exploit known leaked password from a YouTube video — well-known answer is `Mr. N00dles`.
*Alt:* SQLi on email field with `mc.safesearch@juice-sh.op'--`.

### 7. Repetitive Registration (★2, Improper Input Validation)
**Step:** Register a new user → set "security question / answer" twice with different values → server only checks the latest, accepts duplicates.
**How:** intercept the POST `/api/Users` in Burp → modify `repeatPassword` to differ from `password` → still 201 Created.

### 8. Five-Star Feedback (★1, Broken Access Control)
**Step:**
1. Log in as admin (challenge 5 above).
2. Visit `/#/administration` → see all feedback list.
3. Click trash icon to delete the *one-star* feedback that's tagged "feedback".
4. Score-board credits the challenge.

### 9. Error Handling (★1, Security Misconfiguration)
**Step:** Force any error — easiest is to send a malformed JSON body to `/api/Feedbacks`:
```bash
curl -X POST http://localhost:3000/api/Feedbacks -H "Content-Type: application/json" -d "{bad}"
```
Server returns a stack trace including file paths.

### 10. Outdated Allowlist (★1, Vulnerable Components)
**Step:** Visit `/redirect?to=https://blockchain.info/address/1AbKfgvw9psQ41NbLi8kufDQTezwG8DRZm` — Juice Shop blocks unknown URLs but allows whitelisted ones, including a deprecated bitcoin domain. The ★1 is just navigating to that whitelisted URL.

### 11. Missing Encoding (★1, XSS)
**Step:** Click any product → click any image → image filename is rendered without HTML-encoding in some places. The challenge fires automatically when you upload a profile photo with name `<svg/onload=alert(1)>.jpg`.

### 12. Repetitive Registration (★2)
Already covered in #7.

### 13. Zero Stars (★2, Improper Input Validation)
**Step:** Submit a feedback with the rating slider set to **0** (Angular UI prevents this; use Burp to intercept POST `/api/Feedbacks` and set `rating: 0`).

### 14. Email Leak (★2, Sensitive Data Exposure)
**Step:** Visit `/rest/user/whoami` while not logged in — endpoint returns user list including admin email when called specific way. Or check `/api/Users` — but that's blocked unless admin. Easier path: register, then visit `/api/Feedbacks` — the API leaks user references.
*Best:* admin login → `/#/administration` shows all emails.

### 15. Login Bender (★3, Broken Authentication)
**Step:** Email = `bender@juice-sh.op'--`, password = anything → in. (Same SQLi as Admin login but for a different user.)

### 16. Visual Geo Stalking (★2, Sensitive Data Exposure)
**Step:** During login, click *"Forgot Password"* → enter `bjoern.kimminich@gmail.com` → security question is "your eldest siblings middle name?" — the answer can be derived from a photo posted on his social media (in the prebuilt CTF sandbox the answer is **`Samuel`** for Jim, but this question is for Bjoern → answer **`Mortimer`** based on the staged metadata).
*Spoiler:* look at the user's profile photo for EXIF location → it points to a real address → google-maps-equivalent local browsing... in CTF use the cheat from the ebook.

> If a challenge asks you to OSINT something — Juice Shop ships the artefacts inside the app. Look at uploaded profile photos with `exiftool`.

### 17. Reset Jim's Password (★4, Broken Authentication) — covered in `41_…`
Listed here for awareness.

### 18. Login Jim (★3, Broken Authentication)
**Step:** Same SQLi `jim@juice-sh.op'--`.

### 19. Admin Section (★2, Broken Access Control)
**Step:** Visit `/#/administration` directly while not admin — page renders even though API endpoints reject. The challenge fires on URL access.

### 20. View Basket (★2, Broken Access Control)
**Step:** Login as any user → DevTools → Application → Local Storage → see `bid` (basket id). Change to another user's id (e.g. `1`) → reload basket page → see another user's basket.
Or in Burp: GET `/rest/basket/2` while authed as user 1 → leaks data.

### 21. Forgotten Sales Channel (★2, Sensitive Data Exposure)
**Step:** Visit `/#/track-result?id=5267-0a87...` (the order-tracking URL is leaked in the legacy interface). The trick: visit `/#/track-result/new` or look at the `Last-Login-IP` page. Quick win: append `?orderID=` to old paths.

### 22. View Cart Item (★2)
Similar to #20.

### 23. Login Jim (★3) — already noted.

### 24. CAPTCHA Bypass (★3, Broken Anti Automation) — covered in `41_…`

---

## Part D — When the score-board says "solved" but CTFd doesn't

If the practice CTFd shows nothing scored:
1. Make sure CTFd was imported with the **same CTF key** you used for `juice-shop-ctf` (Step 4.2 in `05_…`).
2. The flag format CTFd expects is the SHA-1 of `(challenge name + key)`. Juice Shop calls back to its built-in trigger, then a webhook can score CTFd. In practice you'll just submit the flag manually — find it via the `juice-shop-ctf` zip → `OWASP_Juice_Shop.<ts>.flagsformatted.txt`.

At the actual competition the organizers handle the integration; you only need to **trigger** the challenge in Juice Shop.

---

## Part E — Time pacing for Day 3 morning

| Time | Action |
|---|---|
| 0:00–0:15 | Both teammates open Juice Shop side-by-side. Find score-board. |
| 0:15–1:30 | Solve every ★1 (≈ 8 challenges). |
| 1:30–3:00 | Solve every ★2 (≈ 12 challenges). |
| 3:00–4:30 | Start ★3 — see `41_Day3_CTF_Red.md`. |
| 4:30–5:30 | Continue, lean on ebook hints. |
| 5:30–6:00 | Documentation + report PDF. |

---

## Mark map placeholder

The marking-scheme rows for Day 3 morning (HO-/CM- prefixes) are Lyon leftovers. When the chief releases the ASEAN Juice Shop challenge mapping, fill in:

| ASEAN flag id | Juice Shop challenge | ★ | Solved | Hint used |
|---|---|---|---|---|

Track this row by row during the actual CTF.

Next file: **`51_Day3_VulnHub_BootToRoot.md`** — boot-to-root playbook + medium walkthroughs.
