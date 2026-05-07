# 23 — Day 2 (MA2) — PKI (WINSRV3 Issuing CA)

**Target time:** 20 min.
**Owner:** Person A (after Firewall) or Person B (after AD).
**Login:** WINSRV3 console as `MANILA\Administrator / P@ssw0rd`.

> Marks at stake (Crit A5): row 95 → **K = 0.2** directly, but PKI underpins LinSRV1 cert (A3 D78), AD certenroll (A4 D80), Client checks (A6 D103/D104, A7 D114), VPN cert (A2 D57). Treat as enabling work.

---

## Step 1 — Confirm CertSvc is running

The practice setup left CertSvc **stopped** (CSR pending). It should now be started because WINSRV4 already signed the CSR during initial build (`04_Setup_VMs_MA2.md` Step 5.3).

```powershell
Get-Service CertSvc
Start-Service CertSvc
```
**Expected:** `Status: Running`.

If the issuing CA cert was never installed (i.e., you skipped Step 5.3 in setup):
```powershell
certutil -installCert C:\winsrv3.cer
Start-Service CertSvc
```

---

## Step 2 — Publish + duplicate the templates we need

**Tools:** *Server Manager → Tools → Certification Authority* (`certsrv.msc`).

### 2.1 Publish "Workstation Authentication" for autoenrollment
- Right-click *Certificate Templates → Manage* → opens template console.
- Right-click **Workstation Authentication → Duplicate Template**.
- General tab: name `Workstation-AutoEnroll`, Validity 1 year.
- Security tab: add **Domain Computers** → tick **Read + Enroll + Autoenroll**.
- OK.
- Back in CA console → right-click *Certificate Templates → New → Certificate Template to Issue* → choose `Workstation-AutoEnroll`.

### 2.2 Publish "Web Server" template (for IIS + LinSRV1)
- Right-click **Web Server → Duplicate Template** → name `Web-Server-Manila`.
- Security tab: add **Domain Computers** → Read + Enroll (no autoenroll for web servers).
- Issue it via *New → Certificate Template to Issue → Web-Server-Manila*.

**Marks:** [Crit A5 D95 K=0.2] CA has issued certs.

---

## Step 3 — Issue a Web Server cert for the IIS site on WINSRV3

Per **MA2 PDF page 11**: *"Through auto-enrollment, WinSRV3 should receive a certificate for the Web server in the IIS installation. (Website: https://webtest.manila.com)"*. Strictly speaking IIS Web Server isn't an autoenroll template, so we request it manually.

```powershell
# On WINSRV3
$tpl = "Web-Server-Manila"
certreq -enroll -machine -q $tpl
# OR via IIS Manager → Server Certificates → Create Domain Certificate
```

Then in **IIS Manager** → *Default Web Site → Bindings → Add → https → choose the new cert → port 443*.

Verify from any LAN client: `https://webtest.manila.com` → green padlock, chain `Manila-Root-CA → WINSRV3 → webtest.manila.com`.

**Marks:** [Crit A7 D114 K=0.3] verified at Client2.

---

## Step 4 — Issue a cert to LinSRV1 web server

This is the cert used in `21_Day1_MA2_LinSRV1.md` Step 7.

If you already issued it during Step 7 there, skip. Otherwise:
1. Take the CSR you generated on LinSRV1 (`/tmp/manila.csr`) — copy via WinSCP to `C:\manila.csr` on WINSRV3.
2. ```cmd
   certreq -submit -attrib "CertificateTemplate:Web-Server-Manila" C:\manila.csr C:\manila.cer
   ```
3. Copy `C:\manila.cer` back to LinSRV1, install per Step 7.3.

**Marks:** [Crit A3 D78 K=0.3] LinSRV1 https with PKI cert; [Crit A6 D103 K=0.4] verified at Client1.

---

## Step 5 — Issue OpenVPN server cert (if not done in `20_…` Step 5.1)

```powershell
# On WINSRV3, in PowerShell as admin:
$tpl = "Web-Server-Manila"   # or duplicate one with EKU "Server Authentication" only
certreq -enroll -q $tpl
# Export the new cert (with private key) as PFX, copy to pfSense for OpenVPN config
```

**Marks:** [Crit A2 D57 K=0.25] OpenVPN cert from CA, not self-signed.

---

## Step 6 — Verify autoenrollment from a domain client

After Client1 is domain-joined (functional test in `30_…`):
```cmd
gpupdate /force
certutil -pulse
certutil -store -user My
```
**Expected:** at least one cert with subject containing the client name, issuer `WINSRV3-CA`.

**Marks:** [Crit A6 D104 K=0.5] cert auto-enrolled to client.

---

## Snapshot

ESXi → WINSRV3 → snapshot `WINSRV3-CA-issuing-ready`.

---

## Mark map for this file

| Aspect | K | Step |
|---|---|---|
| A5 D95 CA check | 0.2 | 1–2 |
| A2 D57 OpenVPN cert (enables) | 0.25 | 5 |
| A3 D78 LinSRV1 https (enables) | 0.3 | 4 |
| A4 D80 certenroll (enables) | 0.4 | 2.1 |
| A6 D103 cert chain at Client1 (enables) | 0.4 | 4 |
| A6 D104 GPO cert at Client (enables) | 0.5 | 6 |
| A7 D114 cert at Client2 (enables) | 0.3 | 3 |
| **Direct + enabling total** | **~2.35** | |

Next file: **`30_Day1_MA2_Verification.md`**.
