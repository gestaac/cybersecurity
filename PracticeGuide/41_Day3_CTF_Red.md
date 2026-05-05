# 41 — Day 3 CTF: OWASP Juice Shop — Medium (★3–★4)

**Time:** 6 hours.
**Pre-req:** finish all ★1 + ★2 challenges from `40_…`.

> Medium challenges require chaining 2–3 simple flaws or carefully crafted payloads. They're worth more raw points but cost more time. Aim for ~70% of these in 5 hours.

---

## Tools you'll lean on heavily

| Tool | Used for |
|---|---|
| **Burp Repeater** | Tweaking captured requests (90% of medium challenges) |
| **Burp Intruder** (Community has speed throttle but works) | Brute-forcing tokens, IDOR scans |
| **jwt_tool** | Forging tokens |
| **CyberChef** | Encoding/decoding/HMAC/JWT inspection |
| **DevTools → Network** | Watching websocket + REST traffic |
| **sqlmap** (rare for Juice Shop, but works on /rest/products/search) | Automated SQLi |

---

## Category 1 — Broken Authentication

### Reset Jim's Password (★4)
**Description:** Reset Jim's password using a security question.

**Steps:**
1. Login page → *Forgot Password* → email `jim@juice-sh.op` → security question reveals: *"Your eldest siblings middle name?"*
2. Jim is a Star Trek reference (James Kirk). His brother is George Samuel Kirk → middle name **`Samuel`**.
3. New password: anything that meets policy.
4. Submit → challenge solved.

> Lesson: security questions whose answers are public knowledge are the same as no security at all.

### Reset Bender's Password (★4)
**Q:** *"Company you first worked for as adult?"*
**A:** **`Stok'd & Loaded`** (or per ebook current answer; he's a robot, "stok'd" was a bartender job — answer derived from Futurama trivia).

### Reset Bjoern's Password (Owasp Juice Shop creator) (★5)
**Q:** *"Name of your favorite pet?"* → answer pulled from his real social media → answer **`Zaya`**. (Verify via ebook.)

### Reset Morty's Password (★5)
**Q:** *"Name of your favorite pet?"* → answer **`Snuffles`** but Juice Shop's anti-brute-force throttles you. Use Burp **Repeater** to send manually after waiting 1 second between requests; the brute-force lockout has a window.

### CAPTCHA Bypass (★3)
**Description:** Submit 10 feedbacks within 10 sec.

**Steps:**
1. Submit a normal feedback → intercept POST `/api/Feedbacks` in Burp.
2. Right-click → **Send to Repeater**.
3. The body has `captcha`, `captchaId` — observe Juice Shop's check is server-side AND the captcha doesn't refresh between submissions if you don't reload.
4. In Repeater, **send 10 times rapidly** (Ctrl+R, Ctrl+G is "send group" in Pro; Community: just spam Send button).
5. Each succeeds because captcha + captchaId reused.
6. After 10 within 10 seconds → ★3 solved.

### Login Amy (★3)
**Description:** Log in as Amy with her actual password.

Her hint says password is "Bjoern's wife's name + symbols + number" → ebook-confirmed: `K1f.....................` (a long string with bjoern's wife's name in it). Try `Kif.....................` patterns; or use hashcat against the leaked DB.

### Login MC SafeSearch (★2 — already in 40)
Reminder.

### GDPR Data Theft (★3)
**Description:** Steal someone's personal data without using injection.

**Steps:**
1. Login as admin (★2 SQLi).
2. Visit `/api/Users` → returns full user list, including hashed passwords + emails.
3. Pick any victim (e.g. Bender) → visit `/rest/user/data-export` while logged in as that victim — but you're admin. Switch user via JWT hack:
4. Take admin's JWT from `localStorage.token` → decode at jwt.io / CyberChef → swap `email` claim to `bender@juice-sh.op` → re-sign with the leaked HMAC secret (challenge below) → use the new JWT.

---

## Category 2 — Injection (deeper)

### Christmas Special (★4)
**Description:** Order a product no longer available.

**Steps:**
1. Visit `/rest/products/search?q=` → returns all 24 products including "Christmas Super-Surprise-Box (2014 Edition)".
2. Try to add to basket → "product unavailable".
3. Inspect basket POST request `/api/BasketItems` body:
   ```
   { "BasketId": 1, "ProductId": 10, "quantity": 1 }
   ```
4. The unavailable product has ID 10 (or similar). Send the POST manually in Burp Repeater with `ProductId: 10` → returns 200 OK; product added.
5. Checkout → ★4 solved.

> Real-world: client-side filtering only; server doesn't validate `ProductId`.

### User Credentials (★4) — SQLi UNION
**Steps:**
1. `/rest/products/search?q=` is the SQLi sink.
2. Test union: `qwert')) UNION SELECT '1','2','3','4','5','6','7','8','9'--`
3. The endpoint returns 9 columns — confirm:
   `qwert')) UNION SELECT id,email,password,'4','5','6','7','8','9' FROM Users--`
4. Now product list includes user credentials in fields. ★4 solved when you see the leaked passwords in the response.

### Database Schema (★3)
**Steps:**
1. Same SQLi point: `qwert')) UNION SELECT sql,'1','2','3','4','5','6','7','8' FROM sqlite_master--`
2. Returns CREATE TABLE statements for every table (SQLite metadata).
3. ★3 fires when the response contains schema info.

### Ephemeral Accountant (★3)
**Steps:**
1. Same UNION, now insert a fake user:
   ```
   ')) UNION SELECT '15','acc0untant@juice-sh.op','12345','admin','accounting','deletedAt','9','10','11' FROM Users WHERE id=1--
   ```
2. Then login as `acc0untant@juice-sh.op` / `12345`.
3. Or trigger via the dedicated SQL: `UNION SELECT '99','acc0untant@juice-sh.op','admin','99','99','9','99','9','99' FROM Users WHERE deletedAt IS NOT NULL--`
4. ★3 solved.

### NoSQL Injection — Multiple Likes (★3)
**Description:** Make a product appear in your basket multiple times.

**Steps:**
1. POST `/api/BasketItems` with `quantity: -1` works for some items → balance fix.
2. Better: **NoSQL** version — the *reviews* mongo endpoint accepts:
   ```
   { "id": { "$ne": null }, "message": "spam" }
   ```
   to update many docs at once.
3. The exact challenge fires when you up-vote the same review 5+ times via the like endpoint with NoSQL trick.

### Login Christopher (★3) — covered, same SQLi pattern.

---

## Category 3 — Broken Access Control

### View Another User's Cart (★3)
**Steps:**
1. Login as user A.
2. DevTools → `localStorage.token` → JWT.
3. Burp Repeater: GET `/rest/basket/2` (where 2 is another user's basket id) → leaks contents.
4. ★3 fires.

### Forged Review (★3)
**Steps:**
1. Login as user.
2. Submit a review on any product. Intercept POST `/rest/products/reviews`:
   ```json
   { "id": "<your-id>", "message": "test", "author": "you@x.com" }
   ```
3. Replay with `"author": "admin@juice-sh.op"` → review appears under admin's name.
4. ★3 solved.

### Manipulate Basket (★3)
**Steps:**
1. POST `/api/BasketItems` directly with **negative quantity**:
   `{ "BasketId": 1, "ProductId": 1, "quantity": -100 }`
2. Server stores -100; basket total goes negative.
3. ★3 solved.

### Update Multiple Product Reviews (★4) — NoSQL multi-update
Same as NoSQL Multiple Likes, but the trick is editing reviews you don't own:
```json
{ "id": { "$ne": -1 }, "message": "owned by attacker" }
```
PATCH `/rest/products/reviews` with above → updates EVERY review.

### Access Log (★4)
Visit `/support/logs` → directory listing → click any `.log` file → leaks server logs.
*Or:* `/ftp` directory listing exposed via SSRF / poodle on `quarantine`.

### Admin Section already covered (★2).

### Easter Egg (★4)
**Steps:**
1. Visit `/ftp` directory listing.
2. Click `eastere.gg` — denied (file extension blocked, only `.md` and `.pdf` allowed).
3. Bypass with poison-NULL-byte / double-extension: `/ftp/eastere.gg%2500.md` → file content reveals base64 string.
4. Decode base64 → ROT13 → reveal coordinates / phrase → submit.

### Premium Paywall (★5) — covered in `42_…`.

---

## Category 4 — Broken Cryptography

### Forged Coupon (★4)
**Steps:**
1. Find any coupon code (e.g. footer of holiday season page or in Confidential Document leak).
2. Coupon format: base64-encoded `name-mmddyy` Z85-encoded then XORed with date.
3. Use the ebook's `juice-shop-coupon-cracker` algorithm offline: generate coupons with 99% discount.
4. Apply at checkout → ★4 solved.

### Weird Crypto (★1) — already in ★1 set; uses Md5 / btoa-style obfuscation.

### Imaginary Challenge (★6, Crypto) — covered later.

---

## Category 5 — Vulnerable Components

### Vulnerable Library (★3)
**Description:** Inform shop about a vulnerable library used.

**Steps:**
1. Visit *Contact Us* form.
2. Type message exactly:
   `bassmaster 1.5.1 (CVE-2014-7205)` (or whatever vulnerable lib version Juice Shop bundles in current release — check `package.json` in your local install).
3. Submit → ★3.

> Real-world: forces you to read `package.json` and find a known-vulnerable dep. Reading the lock-file gives you both name and version.

### Known Vulnerable Library (★4) — same form, different lib (e.g. `sanitize-html 1.4.2`).

---

## Category 6 — Sensitive Data Exposure

### Access Log (★4) — covered in BAC.

### Backup Leakage (★3)
Visit `/ftp/package.json.bak` → returns 200 (one of the few extensions whitelisted) → contains old dev dependencies revealing more vulnerable libs.

### Forgotten Developer Backup (★4)
`/ftp/coupons_2013.md.bak` — find coupon plus archive metadata.

---

## Category 7 — XSS

### Reflected XSS (★3)
**Steps:** Visit `/#/track-result?id=<script>alert(1)</script>` → tracking page reflects ID without sanitization.

### Persistent XSS (★4)
**Steps:** Submit a feedback with `<script>alert('xss')</script>` as the comment. Admin page loads it. ★4 fires when admin views feedback.

### Server-Side XSS (★5)
**Steps:** Admin section → user list → set your username via Burp PATCH /api/Users/<id> with `<iframe src="javascript:alert('xss')">` → renders server-side in admin emails. (★5).

---

## Category 8 — Broken Anti-Automation

### Login Twin (★3)
Open *two browsers* (or normal + private) → log in as same user concurrently. Juice Shop's session model permits, ★3 fires.

### Multiple Likes (★3)
Same review like endpoint hammered with Burp Repeater 5x in 1 second.

---

## Category 9 — Improper Input Validation

### Forged Feedback (★3)
**Steps:** Submit feedback while logged out. POST body has `UserId: null`. Change to `UserId: 1` (admin) → admin posts feedback he never wrote.

### Upload Size (★3)
**Steps:** Profile photo upload limited to 100 KB. Bypass:
1. Intercept POST `/file-upload`.
2. Modify `Content-Length` header to lie about size.
3. Or upload a 50 KB file and append junk after upload via Burp.

### Upload Type (★3)
**Steps:** Restrict to `.jpg/.png` client-side. Burp: change `Content-Type: image/png` but file is `.exe`. Server accepts. ★3.

### Repetitive Registration (★2) — already in 40.

### Persisted XSS via REST (★4) — covered in XSS.

---

## Category 10 — Misc

### Deprecated Interface (★3)
The legacy interface allows uploading XML files. Submit any XML → response 200. ★3 fires.

### Forgotten Sales Channel (★2) — covered.

### Christmas Special (★4) — covered.

### CSRF (★3, Broken Authentication)
Find an external page that POSTs to Juice Shop. Built-in: visit `/promotion` while logged in → triggers internal CSRF demo → ★3 fires.

---

## Part Z — Triage tips

If a challenge is taking >30 minutes:
1. Re-read the description for keywords (★ category, hint about technique).
2. Check the ebook **Part II** for the technique (not the answer).
3. Take H1 hint inside Juice Shop (settings cog → hints) — costs no marks (Juice Shop hints are inline, the WSC hint penalty applies only to the ASEAN scoring system).
4. Move on, return at end.

---

## Mark map (placeholder — fill once ASEAN flag mapping released)

| ASEAN flag id | Juice Shop challenge | ★ | Solved | Hint? |
|---|---|---|---|---|

Next file: **`42_Day4_CTF_Blue.md`** — hard challenges (★5–★6).
