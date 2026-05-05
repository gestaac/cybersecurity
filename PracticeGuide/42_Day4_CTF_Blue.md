# 42 — Day 4 CTF: OWASP Juice Shop — Hard (★5–★6)

**Time:** 6 hours.
**Pre-req:** ★1–★4 challenges done (`40_…`, `41_…`).

> The hardest Juice Shop challenges chain 3+ vulnerabilities: SSRF + RCE + crypto + JWT, etc. Don't expect to solve all of them — pick the 5 highest-payoff ones for your team.

---

## Tools you must have ready

| Tool | Use |
|---|---|
| **jwt_tool** | Forge JWTs (every hard challenge needs JWT manipulation) |
| **Burp Repeater + Intruder** | Send-and-tweak |
| **CyberChef** | XOR, base64, ROT, AES, HMAC |
| **Python 3 + requests** | Custom exploits |
| **Hashcat** | Crack leaked password hashes |
| **xxd / hexdump** | Inspect binary uploads |
| **Ghidra** | If a binary artefact is in `/ftp` |

---

## Category — JWT abuse (★5)

### Forged Signed JWT (★6)
**Description:** Forge an essentially unsigned JWT to take over admin.

**Steps:**
1. Login → DevTools → `localStorage.token` → copy.
2. Decode with `jwt_tool`:
   ```
   jwt_tool eyJhbGciOi...
   ```
3. Header: `{"alg":"RS256","typ":"JWT"}`. Payload contains your user data + `email`.
4. Set `alg` to `none` (Juice Shop accepts):
   ```
   jwt_tool <token> -X a -I -hc kid -hv ../../../../../../../../etc/passwd
   ```
5. Modify payload `email` to `jwtn3d@juice-sh.op` (the staged victim) → ★6 solves.

### Forged Signed JWT — RSA / Public Key Reuse (★6)
1. RS256 → use Juice Shop's own public key as HMAC secret with `alg:HS256`.
2. Public key sits at `/encryptionkeys/jwt.pub`.
3. `jwt_tool <token> -X k -pk jwt.pub` → forge new JWT signed with the public key as HMAC secret.
4. Submit → admin login.

### JWT Issues (the meta) — multiple ★5 challenges chained.

---

## Category — SSRF + Cloud-meta-style (★6)

### SSRF (★6)
**Description:** Use SSRF to make Juice Shop request a URL on its own loopback.

**Steps:**
1. Profile photo upload → URL field accepts external image.
2. Submit `http://127.0.0.1:3000/api/Users/1` as the photo URL.
3. Server fetches that URL with admin context → response logged in upload metadata.
4. ★6 fires.

### Login as Tom (★4) — uses leaked SSRF response.

---

## Category — XXE (★5–★6)

### XXE Tier 1 (★4)
**Description:** Submit XML feedback that includes external entity.

**Steps:**
1. *Complaint* form (Premium membership only) → upload an XML file:
   ```xml
   <?xml version="1.0"?>
   <!DOCTYPE foo [<!ENTITY xxe SYSTEM "file:///etc/passwd">]>
   <root>&xxe;</root>
   ```
2. Server parses without disabling DTDs → response includes `/etc/passwd`.
3. ★4 fires.

### XXE Tier 2 (★5)
Same form → cause **server denial-of-service** with billion laughs (`<!ENTITY a "lol"><!ENTITY b "&a;&a;&a;..."> ...`).

### Deluxe Membership (★5) — required to access XXE form. Use forged JWT or:
- Buy via SQLi-injected coupon (★4 Forged Coupon trick from `41_…`).
- Or NoSQL update on user role.

---

## Category — Server-Side RCE / Code Injection (★6)

### Imaginary Challenge (★6)
**Description:** Solve a challenge that doesn't exist on the score-board.

Hidden challenge — only fires if you trigger a specific debug parameter:
1. Inspect `main.js` → search "challenge.imaginary".
2. Visit a URL that POSTs to `/rest/admin/save-test-result?challengeId=99` with a forged HMAC.
3. ★6 — and bragging rights.

### Successful RCE DoS (★6)
**Description:** Crash the Juice Shop process via a sleep loop in B2B XML order.

**Steps:**
1. B2B order endpoint accepts XML.
2. Use a payload with billion-laughs or an `<xsl:script>` that calls `system("sleep 100000")`.
3. Server exhausts loop → process times out / restarts.
4. ★6.

### B2B Order Injection (★5)
Same XML endpoint, simpler payload — XPath injection retrieves prices outside scope.

---

## Category — Crypto (★5–★6)

### Premium Paywall (★5)
**Description:** Read paywalled content without paying.

**Steps:**
1. The page checks a base64 token in URL `/this/page/is/hidden/behind/an/incredibly/high/paywall/<token>`.
2. Reverse the token's encoding (XOR with constant).
3. Use the leaked logic (in `main.js` you can read the algorithm) to compute the bypass token.
4. ★5.

### Forged Coupon (★4) — covered in `41_…`.

### Cryptanalysis: Weird Crypto (★1) — basic.

---

## Category — Vulnerable Components (★5)

### Known Vulnerable Library — Critical (★5)
1. Find a library in `package.json` with a known **CVE-with-exploit** (not just CVE).
2. Submit the form: `<library-name> <version> (CVE-XXXX-YYYY)` exactly.
3. Examples often used: `marsdb 0.6.11`, `sanitize-html < 1.4.3`.
4. ★5 fires when the right combo is named.

### NoSQL Manipulation (★5) — chain with vulnerable mongo-style packages.

---

## Category — Insecure Deserialization (★6)

### Type Juggling (★5)
**Description:** Bypass auth using JS `==` quirks.

POST `/rest/user/login` with body:
```json
{ "email": "admin@juice-sh.op", "password": [false] }
```
Server compares `password == hash(false)` → may return 200 in vulnerable middleware versions. ★5.

---

## Category — Multi-step admin compromise (★6)

### Login Admin via Vulnerable Library (★6 chain)
Steps:
1. Find vuln in `marsdb` allowing query injection.
2. Use it on the `/rest/products/search?q=` endpoint to dump admin credentials.
3. Crack the bcrypt hash with hashcat (`-m 3200`) using rockyou.
4. Login → ★6.

---

## Practice triage for hard challenges

If you can solve **5 of these ★5–★6 challenges in 6 hours**, you're top-tier. Realistic strategy:

| Priority | Challenge | Why |
|---|---|---|
| 1 | Forged Signed JWT (★6) | One technique, big payoff |
| 2 | SSRF (★6) | Easy if you've done one before |
| 3 | XXE Tier 1 (★4) — already covered, extend to ★5 DoS | Same code path |
| 4 | Premium Paywall (★5) | Read main.js, deterministic |
| 5 | User Credentials via UNION SQLi (★4 from yesterday — chain to ★6 admin compromise) | Logical chain |

Skip:
- ❌ Imaginary Challenge (mostly novelty)
- ❌ Successful RCE DoS (risky on shared infra; might disqualify)
- ❌ B2B Order Injection unless you've practised XPath

---

## Practice plan for these (do during Week 2 setup)

| Day | Goal |
|---|---|
| Day 11 | Forge JWT 5 times until it's muscle memory |
| Day 12 | Read `main.js` of your local Juice Shop end-to-end (it's only ~5 MB) — find every `challenge.solve` call, understand triggers |
| Day 13 | Mock 6-h with hard challenges only — see how many you can crack |

---

## Documentation reminder

Every solved challenge:
1. Note the technique (1–2 sentences).
2. Screenshot the score-board confirmation.
3. Save the request/response from Burp (Save Item → .req file).
4. Add to `team1_CTF_Report.pdf` as a section.

---

## Mark map (placeholder)

| ASEAN flag id | Juice Shop challenge | ★ | Solved | Hint? |
|---|---|---|---|---|

End of Day 4 playbook. Final file: **`90_Practice_Schedule.md`** (already updated to reference Juice Shop) and **`99_Marking_Map.md`**.
