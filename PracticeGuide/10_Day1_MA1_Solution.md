# 10 — Day 1 Morning (MA1) — Walkthrough Solution

**Target time:** 90 min work + 90 min documentation. (Test gives you 3 hours total.)
**Login:** `Competitor0 / CharterDressing` on the workstation. ESXi `amuser / Mt Blanc` (per MA1 line 60).
**Deliverable:** ONE document on the desktop, named with your country code, containing:
1. Executive summary of the network's security state (≤ 500 words).
2. Table 1 with the **two most critical** Apache/website vulnerabilities (risk, severity, identification, remediation).

> Marks at stake (Crit A1):
>
> | Aspect | Type | Max K | What earns it |
> |---|---|---|---|
> | A1 D31 | Meas | 2.0 | identify ≥2 valid Apache vulns |
> | A1 D32 | Judg | 2.0 | executive summary quality |
> | A1 D37 | Judg | 1.0 | 1st vuln write-up quality |
> | A1 D42 | Judg | 1.0 | 2nd vuln write-up quality |
> | **Total K** | | **6.0** | |

---

## Phase 1 — Reconnaissance (20 min)

### Step 1.1 — Map the network from AMClient1
**Why:** confirm which VMs exist and what they expose.
**Tools:** AMClient1 → cmd.
```cmd
ping 172.16.100.10
ping 172.16.100.13
nslookup www.grimshay.ca 172.16.100.10
```
**Expected:** DC and webserver respond. DNS resolves www.grimshay.ca → 172.16.100.13.

### Step 1.2 — Browse the website
**Tools:** Chrome on AMClient1.
- Visit `http://www.grimshay.ca` → notice it redirects to https.
- Visit `https://www.grimshay.ca` → certificate warning (self-signed).
- Click **Advanced → Continue** → Basic-Auth prompt.
- Try `alice / P@ssw0rd` → loads the members page.
**Note for the report:** self-signed cert is expected for an internal site, **but** TLS protocol pin is missing. Save a screenshot of the cert details.

### Step 1.3 — SSH to the webserver as competitor
**Tools:** PuTTY on AMClient1.
```
Host: 172.16.100.13   User: competitor   Password: P@ssw0rd
```

---

## Phase 2 — Vulnerability hunting (30 min)

Work through this checklist in order. Each one maps to a known item in the marking-scheme judge notes for A1 row 32.

### Check A — Apache TLS configuration
```bash
sudo grep -RiE "SSLProtocol|SSLCipherSuite" /etc/httpd
```
**Expected:** *no SSLProtocol line found in /etc/httpd/conf.d/*.
**Finding:** `SSLProtocol` not pinned → TLS 1.0/1.1 still allowed.
**Marks:** counts toward A1 D31 + a strong candidate for vuln write-up.

Verify exposure with openssl:
```bash
openssl s_client -connect www.grimshay.ca:443 -tls1_1
```
If handshake succeeds → TLS 1.1 is enabled — **vulnerability confirmed**.

### Check B — LDAP authentication mode
```bash
sudo grep -i "AuthLDAPURL" /etc/httpd/conf.d/*.conf
```
**Expected:** `AuthLDAPURL "ldap://172.16.100.10/..."` — **starts with `ldap://` not `ldaps://`**.
**Finding:** site sends usernames/passwords to AD in **cleartext**. Easy to confirm with Wireshark.

Wireshark proof:
1. Start capture on AMClient1's NIC.
2. Open private-window Chrome → log in to https://www.grimshay.ca with a fake user `evil / qwerty`.
3. Filter `ldap` → look at the `bindRequest` packet → fields `name=Anorbert@grimshay.local` (the bind DN) and `simple` password are visible.
4. Save the capture as `ldap_clear.pcapng` for the report.

### Check C — Private key permissions
```bash
ls -l /etc/pki/tls/private/grimshay.key
```
**Expected:** `-rw-r--r--` (mode 644). World-readable private key — anyone with shell can copy it.

### Check D — SELinux state
```bash
sudo sestatus
```
**Expected:** `SELinux status: disabled`.
**Finding:** SELinux not enforcing → no MAC layer if Apache is exploited.

### Check E — File / folder permissions on web root
```bash
ls -ld /var/www/grimshay
```
**Expected:** `drwxrwxrwx`. Anyone (even nobody) can write to web root → trivial defacement.

### Check F — Service account
```bash
grep -E "^User|^Group" /etc/httpd/conf/httpd.conf
```
**Expected:** `User apache / Group apache` (default).
**Finding:** uses generic apache user with default privileges; a service-specific account with `nologin` shell is recommended.

---

## Phase 3 — Pick the **two most critical** (5 min)

Of the 5+ vulnerabilities you will find, the **two with the worst combination of likelihood × impact** are:

1. **LDAP bind in cleartext (Check B)** — credentials of every login go on the wire to the DC. Anyone with switch-port access can capture domain creds.
2. **Missing SSLProtocol pin / weak TLS (Check A)** — the site can be downgraded to TLS 1.0/1.1 by an attacker; well-known POODLE/BEAST class.

(Choose 2 from your real assessment if it differs — but in the practice environment the answer is these two. Both score full Judg=3 if explained well.)

---

## Phase 4 — Write the deliverable (90 min)

### Step 4.1 — Open the appendix
On AMClient1 desktop: open `WSA2025_TP54_MA1_actual_en.docx` in Word → go to *Appendix 1*.

### Step 4.2 — Executive summary (≤ 500 words)
**Marks:** [Crit A1 D32 K=2.0, max Judg=3]

Write in plain non-technical language. Use this skeleton:

```
Executive Summary

We assessed the security configuration of the www.grimshay.ca members-area
website hosted on a Linux server with Apache HTTPD. The site is intended to
be accessible only to staff in the AD group "webusers". Overall the site
WORKS as intended — authorised users can log in and view the page. However,
several configuration choices put member credentials and the underlying
server at material risk and should be remediated before this site is exposed
to a wider audience.

Highest-impact findings (full detail in Table 1):

1. The website authenticates users against Active Directory using the
   unencrypted LDAP protocol (TCP/389) instead of LDAPS (TCP/636). This means
   every login submitted through the website travels across the network in
   plain text — including the username and password. Any party with access to
   the local switch could capture credentials by passive eavesdropping. Risk:
   HIGH. Likelihood: HIGH (no special tools needed).

2. The TLS configuration of the site does not pin a minimum protocol version.
   A capable attacker on the same network can force the browser to negotiate
   downward to legacy TLS 1.0 / 1.1, which contain known cryptographic
   weaknesses (POODLE, BEAST). Risk: MEDIUM. Likelihood: MEDIUM.

Other findings observed but not selected for detailed write-up include
SELinux being disabled, world-readable web root, and the use of the default
apache service account. Each compounds the impact of any web-app compromise.

Recommendations are documented in Table 1. Remediation is straightforward
and can be implemented in under one working day with no service downtime.

Word count: ~270 / 500 max.
```

### Step 4.3 — Table 1, vulnerability #1
**Marks:** [Crit A1 D31 K=2.0, D37 K=1.0, max Judg=3 each]

| Field | Value |
|---|---|
| **Vulnerability** | LDAP bind sent in cleartext (port 389, not 636) |
| **Where found** | `/etc/httpd/conf.d/grimshay.conf` — `AuthLDAPURL "ldap://172.16.100.10/..."` |
| **Risk** | HIGH — leaks AD bind credentials and member sign-in credentials on every authenticated request |
| **Severity (CVSS-like)** | 8.6 (Network / Low complexity / No auth needed / High Confidentiality) |
| **Identification (proof)** | Wireshark capture (file `ldap_clear.pcapng`) on AMClient1 shows `bindRequest` packets with the bind DN `CN=Administrator,CN=Users,DC=grimshay,DC=local` and `simple` password field in clear text. |
| **Remediation** | Change the URL to `ldaps://172.16.100.10:636/...`. Ensure the AD root CA is trusted by the Linux box (`update-ca-trust`). Disable port 389 on the firewall between webserver and DC. |

### Step 4.4 — Table 1, vulnerability #2
**Marks:** [Crit A1 D42 K=1.0, max Judg=3]

| Field | Value |
|---|---|
| **Vulnerability** | TLS protocol/cipher policy not pinned |
| **Where found** | `/etc/httpd/conf.d/grimshay.conf` — no `SSLProtocol` or `SSLCipherSuite` directive |
| **Risk** | MEDIUM — site can be negotiated down to TLS 1.0/1.1 with weak ciphers |
| **Severity** | 6.5 (downgrade-only attack, requires on-path attacker) |
| **Identification (proof)** | `openssl s_client -connect www.grimshay.ca:443 -tls1_1` succeeds (handshake completes). |
| **Remediation** | Add to vhost: `SSLProtocol TLSv1.2 TLSv1.3` and `SSLCipherSuite HIGH:!aNULL:!MD5:!RC4`. Restart `httpd`. Re-test with `nmap --script ssl-enum-ciphers -p 443 www.grimshay.ca`. |

### Step 4.5 — Save & combine
Per MA1 line 55: **combine** the answer documents into a single file on the desktop, labelled with your country code.

```
File name suggestion:  PHL_Team1_MA1_Appendix.pdf  (or .docx)
Save to: C:\Users\Competitor0\Desktop\
```

> **Tip:** keep both the Word file *and* the PDF. Sometimes experts ask for the editable one.

---

## Time check

| Phase | Target | Real |
|---|---|---|
| Recon | 0:20 | __ |
| Vuln hunt | 0:30 | __ |
| Pick top 2 | 0:05 | __ |
| Write & polish | 1:30 | __ |
| Buffer | 0:35 | __ |

---

## Marks summary for this file

| Aspect | K-value | Earned by step |
|---|---|---|
| A1 D31 (≥2 vulns identified, must be Apache-related) | 2.0 | Phase 2 + Table 1 entries |
| A1 D32 (executive summary quality) | 2.0 | Step 4.2 |
| A1 D37 (vuln #1 detail) | 1.0 | Step 4.3 |
| A1 D42 (vuln #2 detail) | 1.0 | Step 4.4 |
| **Total** | **6.0** | |

Next file: **`20_Day1_MA2_Firewall.md`**.
