# 21 — Day 2 (MA2) — LINSRV1 Hardening (CentOS Stream 9)

**Target time:** 60 min.
**Owner:** Person B.
**Login:** From Client1 → PuTTY → `192.168.1.10` (port 22 first, port 2022 after Step 3) as `root / P@ssw0rd`.

> Marks at stake (Crit A3): rows 66–78 → **K total ≈ 3.15**.

---

## Step 1 — Domain join `manila.com`
**Why:** lets domain users (C1, C2) ssh in with AD credentials.
**Pre-req:** WINSRV1 reachable via 192.168.2.10 from LinSRV1 (firewall rule DMZ→Servers AD ports must already be in place per `20_…`).

```bash
sudo timedatectl set-timezone Asia/Manila
sudo systemctl enable --now chronyd
sudo realm discover manila.com
sudo realm join -U Administrator manila.com
# enter P@ssw0rd

# verify
realm list
id Administrator@manila.com
```
**Expected:** `realm list` shows `manila.com` joined; `id` shows the user.
**Marks:** [Crit A3 D66 K=0.2] computer account in AD/DNS; [Crit A3 D67 K=0.2] joined.
**If it fails:** check `sudo cat /etc/resolv.conf` shows `192.168.2.10`; check `realm discover manila.com` resolves.

---

## Step 2 — Allow C1 + C2 domain users (and create local fallbacks)
**Why:** marking scheme accepts either domain or local accounts.

Create matching local accounts as a safety net (per MA2 PDF page 12: *"If domain integration fails, create local users C1 and C2 matching the expected credentials"*):
```bash
sudo useradd -m C1
sudo useradd -m C2
sudo passwd C1   # P@ssw0rd12 (will pass complexity once we set it)
sudo passwd C2   # P@ssw0rd12
```

Add C1 + C2 to sudoers:
```bash
sudo bash -c 'echo "C1 ALL=(ALL) ALL" > /etc/sudoers.d/c1c2'
sudo bash -c 'echo "C2 ALL=(ALL) ALL" >> /etc/sudoers.d/c1c2'
sudo chmod 440 /etc/sudoers.d/c1c2
```
For domain users C1@manila.com and C2@manila.com:
```bash
sudo bash -c 'echo "%domain\\ admins ALL=(ALL) ALL" > /etc/sudoers.d/domain'
# alternative: explicit domain user
sudo bash -c 'echo "C1@manila.com ALL=(ALL) ALL" >> /etc/sudoers.d/domain'
sudo bash -c 'echo "C2@manila.com ALL=(ALL) ALL" >> /etc/sudoers.d/domain'
sudo chmod 440 /etc/sudoers.d/domain
```
**Marks:** [Crit A3 D73 K=0.2] sudo works for C1/C2; [Crit A6 D101 K=0.3] confirmed via Client1 later.

---

## Step 3 — SSH hardening (port 2022, no root, only C1/C2)
**Why:** project requirement.

Edit `/etc/ssh/sshd_config`:
```bash
sudo sed -i 's/^#Port 22/Port 2022/' /etc/ssh/sshd_config
sudo sed -i 's/^#PermitRootLogin .*/PermitRootLogin no/' /etc/ssh/sshd_config
sudo sed -i 's/^#MaxAuthTries.*/MaxAuthTries 3/' /etc/ssh/sshd_config
sudo bash -c 'echo "AllowUsers C1 C2 C1@manila.com C2@manila.com" >> /etc/ssh/sshd_config'
```

> **Per MA2 PDF page 13:** *"Limit maximum authentication attempts to 3."* — that's the `MaxAuthTries 3` line above.

SELinux still allows port 22 only by default — add 2022:
```bash
sudo dnf install -y policycoreutils-python-utils
sudo semanage port -a -t ssh_port_t -p tcp 2022
```

Open the firewall service for the new port:
```bash
sudo firewall-cmd --permanent --remove-service=ssh
sudo firewall-cmd --permanent --add-port=2022/tcp
```
Restart sshd:
```bash
sudo systemctl restart sshd
```
**Verify** from a *new* PuTTY session before closing the current one:
- new connection → `192.168.1.10` port `2022` user `C1` → success.
- `ssh root@192.168.1.10 -p 2022` → **denied**.

**Marks:** [Crit A3 D68 K=0.2] sshd_config; [Crit A6 D100 K=0.4] verified from Client1.

---

## Step 4 — Local firewall (firewalld) — services + reboot-safe
```bash
sudo systemctl enable --now firewalld
sudo firewall-cmd --permanent --add-port=2022/tcp
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --permanent --add-service=kerberos
# Remove anything not needed:
sudo firewall-cmd --permanent --remove-service=cockpit 2>/dev/null
sudo firewall-cmd --reload
sudo firewall-cmd --list-all --permanent
```
**Expected list:** ports `2022/tcp`, services `http https kerberos`. No `ssh`, no `cockpit`, no `dhcpv6-client` if not needed.
**Marks:** [Crit A3 D70 K=0.2] firewalld active; [Crit A3 D71 K=0.3] correct services.

---

## Step 5 — Password complexity & ageing
**Why:** project requires:
- 10 chars min length
- changed every 30 days
- min change interval 25 days
- warn 25 days before expiry
- requires lower + upper + digit + symbol

### 5.1 PAM password quality
Edit `/etc/security/pwquality.conf`:
```
minlen = 10
dcredit = -1
ucredit = -1
lcredit = -1
ocredit = -1
```
This forces ≥1 of each character class.

### 5.2 System-wide ageing defaults
Edit `/etc/login.defs`:
```
PASS_MAX_DAYS   30
PASS_MIN_DAYS   25
PASS_WARN_AGE   25
```

### 5.3 Apply to existing local users
```bash
for u in C1 C2; do
  sudo chage -m 25 -M 30 -W 25 $u
done
```

### 5.4 Test
```bash
sudo useradd bob
sudo passwd bob       # try 'password' → rejected
sudo passwd bob       # use 'P@ssw0rd12' → accepted
chage -l bob
```
**Expected output of `chage -l bob`:**
```
Minimum number of days between password change : 25
Maximum number of days between password change : 30
Number of days of warning before password expires : 25
```
**Marks:** [Crit A3 D72 K=0.55] complexity + ageing.

---

## Step 6 — SELinux enforcing (without breaking httpd)
```bash
sudo setenforce 1
sudo sed -i 's/^SELINUX=.*/SELINUX=enforcing/' /etc/selinux/config
sudo restorecon -Rv /var/www/manila
ls -lZ /var/www/manila/index.html   # must show httpd_sys_content_t
sudo setsebool -P httpd_can_network_connect 1
sudo systemctl restart httpd
```
**Verify** from Client1 → `curl http://www.manila.com` → returns the page.
**Marks:** [Crit A3 D76 K=0.2] sestatus enforcing; [Crit A3 D77 K=0.3] httpd context.

---

## Step 7 — HTTPS with cert from WINSRV3

### 7.1 Generate CSR for `www.manila.com`
```bash
sudo openssl req -new -newkey rsa:2048 -nodes \
  -keyout /etc/pki/tls/private/manila.key \
  -out /tmp/manila.csr \
  -subj "/CN=www.manila.com"
sudo chmod 600 /etc/pki/tls/private/manila.key
```
Copy `/tmp/manila.csr` to WINSRV3 (use WinSCP).

### 7.2 On WINSRV3 — issue the cert
```cmd
certreq -submit -attrib "CertificateTemplate:WebServer" C:\manila.csr C:\manila.cer
```
Approve via *Certification Authority → Pending Requests* if it pends. Save `manila.cer`.

Also export the CA chain to `manila-ca-chain.cer` from `certlm.msc → Trusted Root CAs → Manila-Root-CA → Export`.

### 7.3 Back on LinSRV1
```bash
sudo cp manila.cer /etc/pki/tls/certs/manila.crt
sudo cp manila-ca-chain.cer /etc/pki/ca-trust/source/anchors/
sudo update-ca-trust extract
```

### 7.4 Apache vhost (HTTPS + domain-user-only access)

**Per MA2 PDF page 13:** *"Limit the website's exposure to domain users only. A domain user should be able to access the website from the Internet through his/her domain login credential."* — so we need **HTTP basic auth backed by AD** in addition to HTTPS.

#### 7.4.1 — Install mod_authnz_ldap
```bash
sudo dnf install -y mod_ldap mod_authnz_ldap openldap-clients
```

#### 7.4.2 — Apache vhost
Edit `/etc/httpd/conf.d/manila.conf`:
```apache
<VirtualHost *:80>
  ServerName www.manila.com
  Redirect permanent / https://www.manila.com/
</VirtualHost>

<VirtualHost *:443>
  ServerName www.manila.com
  DocumentRoot /var/www/manila
  SSLEngine on
  SSLCertificateFile    /etc/pki/tls/certs/manila.crt
  SSLCertificateKeyFile /etc/pki/tls/private/manila.key
  SSLProtocol TLSv1.2 TLSv1.3
  SSLCipherSuite HIGH:!aNULL:!MD5

  <Directory /var/www/manila>
    AuthType Basic
    AuthName "Manila Domain Users Only"
    AuthBasicProvider ldap
    # Use ldaps:// if you've imported the WinSRV3 cert chain; ldap:// works for practice
    AuthLDAPURL "ldap://192.168.2.10:389/DC=manila,DC=com?sAMAccountName?sub?(objectClass=user)"
    AuthLDAPBindDN "CN=Administrator,CN=Users,DC=manila,DC=com"
    AuthLDAPBindPassword "P@ssw0rd"
    Require valid-user
  </Directory>
</VirtualHost>
```

> **Better practice for production:** use `ldaps://` (port 636) so the bind credentials don't transit cleartext. For the MA2 deliverable, `ldap://` satisfies the requirement; mention the `ldaps://` recommendation in the Table 2 GPO/risk discussion.

#### 7.4.3 — SELinux: allow httpd to LDAP
```bash
sudo setsebool -P httpd_can_connect_ldap 1
sudo apachectl configtest && sudo systemctl restart httpd
```

#### 7.4.4 — Verify
From Client1 (domain user M001 logged in):
- Browse to `https://www.manila.com` → prompted for credentials → enter `M001 / P@ssw0rd` → page loads.
- Without domain credentials → 401 Unauthorized.
**Verify** from Client1 (after Step 8 of WinSRV1 file gives Client1 the trusted root via GPO):
```
https://www.manila.com → no certificate warning, padlock shows chain Manila-Root-CA → WINSRV3 → www.manila.com
```
**Marks:** [Crit A3 D78 K=0.3] PKI cert in use.

---

## Snapshot & sanity

```bash
sudo systemctl status sshd httpd firewalld --no-pager
```
All three: `active (running)`. Take an ESXi snapshot named `LinSRV1-hardened`.

---

## Mark map for this file

| Aspect | K | Step |
|---|---|---|
| A3 D66 | 0.2 | 1 |
| A3 D67 | 0.2 | 1 |
| A3 D68 | 0.2 | 3 |
| A3 D70 | 0.2 | 4 |
| A3 D71 | 0.3 | 4 |
| A3 D72 | 0.55 | 5 |
| A3 D73 | 0.2 | 2 |
| A3 D74 | 0.2 | 4 (firewalld active again post-reboot) |
| A3 D75 | 0.3 | 4 |
| A3 D76 | 0.2 | 6 |
| A3 D77 | 0.3 | 6 |
| A3 D78 | 0.3 | 7 |
| **Total** | **~3.15** | |

Next file: **`22_Day1_MA2_WinSRV1_AD.md`**.
