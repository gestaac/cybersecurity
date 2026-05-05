# 03 — Build the MA1 Practice Environment (grimshay.local)

In MA1 you're handed a pre-built grimshay.local environment with an Apache/website you must **assess** for security flaws. For practice you build the same environment yourself **with the deliberate flaws baked in**, so on competition day you'll already recognise them.

> **Total build time:** ~3 hours first time, then 30 min from snapshots.

---

## VMs to create on `vSwitch-MA1` / `PG-MA1-LAN`

| VM | OS | RAM | Disk | IP |
|---|---|---|---|---|
| DC.grimshay.local | Win Server 2022 Eval | 4 GB | 60 GB | 172.16.100.10 |
| www.grimshay.ca | CentOS Stream 9 | 2 GB | 20 GB | 172.16.100.13 |
| AMClient1 | Win 10 Enterprise Eval | 2 GB | 40 GB | 172.16.100.1 |
| AMClient2 | Win 10 Enterprise Eval | 2 GB | 40 GB | 172.16.100.2 |

> All accounts use password `P@ssw0rd` per MA1 line 45.

---

## Part 1 — Domain Controller (DC.grimshay.local)

### Step 1.1 — Install Windows Server 2022
**Tools:** ESXi *Create VM* wizard, Windows Server 2022 ISO.
**Clicks:** New VM → Guest OS Windows 2022 → 4 GB RAM → 2 vCPU → 60 GB disk → Network adapter on `PG-MA1-LAN` → mount ISO → Power on → standard install (*Standard Eval (Desktop Experience)*) → set Administrator password to `P@ssw0rd` → log in.
**Expected:** Server Manager opens.
**If it fails:** make sure VM has at least 4 GB RAM; install will hang otherwise.

### Step 1.2 — Set static IP
**Where:** DC → *Settings → Network & Internet → Ethernet → Edit IP*
**Settings:**
- IPv4 manual: 172.16.100.10 / 24 / no gateway
- DNS: 127.0.0.1
**Verify:** `ipconfig` shows the address.

### Step 1.3 — Rename + promote to DC
**Tools:** PowerShell as Administrator.
```powershell
Rename-Computer -NewName "DC" -Restart
```
After reboot, log back in:
```powershell
Install-WindowsFeature AD-Domain-Services -IncludeManagementTools
Import-Module ADDSDeployment
Install-ADDSForest `
    -DomainName "grimshay.local" `
    -DomainNetbiosName "GRIMSHAY" `
    -SafeModeAdministratorPassword (ConvertTo-SecureString "P@ssw0rd" -AsPlainText -Force) `
    -InstallDns -Force
```
**Expected:** machine reboots into the new domain.

### Step 1.4 — Create the `webusers` group + a few test users
**Why:** MA1 says only `webusers` should access the website.
**Tools:** PowerShell as Domain Admin.
```powershell
New-ADGroup -Name "webusers" -GroupScope Global -GroupCategory Security
"alice","bob","carol" | % { New-ADUser -Name $_ -AccountPassword (ConvertTo-SecureString "P@ssw0rd" -AsPlainText -Force) -Enabled $true; Add-ADGroupMember -Identity webusers -Members $_ }
```
**Expected:** *Active Directory Users & Computers → grimshay.local → Users* shows alice, bob, carol and a `webusers` group.

### Step 1.5 — Add DNS A record for the webserver
```powershell
Add-DnsServerResourceRecordA -Name "www" -ZoneName "grimshay.local" -IPv4Address "172.16.100.13"
Add-DnsServerResourceRecordA -Name "www" -ZoneName "grimshay.ca" -IPv4Address "172.16.100.13"  # if you create the .ca zone
```
For grimshay.ca you'll need to add it as a primary zone:
```powershell
Add-DnsServerPrimaryZone -Name "grimshay.ca" -ReplicationScope "Forest"
Add-DnsServerResourceRecordA -Name "www" -ZoneName "grimshay.ca" -IPv4Address "172.16.100.13"
```

### Step 1.6 — Snapshot
ESXi → DC → *Take snapshot* → name `01-DC-clean`.

---

## Part 2 — Linux webserver (www.grimshay.ca, **deliberately weak**)

This is the box you will assess in MA1. Build it with the **5 deliberate vulnerabilities** the marking-scheme judge notes call out (rows 32 G-column).

### Step 2.1 — Install CentOS Stream 9
**Clicks:** New VM → Linux → CentOS 9 → 2 GB RAM → 1 vCPU → 20 GB disk → NIC on `PG-MA1-LAN` → mount ISO → standard install (*Server*), password `P@ssw0rd` for root, create user `competitor / P@ssw0rd`.
**After install:**
```bash
nmcli con mod ens160 ipv4.addresses 172.16.100.13/24 ipv4.method manual ipv4.dns 172.16.100.10
nmcli con up ens160
hostnamectl set-hostname www.grimshay.ca
```

### Step 2.2 — Install Apache + LDAP modules
```bash
sudo dnf install -y httpd mod_ldap mod_ssl openldap-clients
sudo systemctl enable --now httpd
```

### Step 2.3 — Drop a tiny site so there's something to assess
```bash
sudo mkdir -p /var/www/grimshay
echo "<h1>grimshay.ca - members area</h1>" | sudo tee /var/www/grimshay/index.html
```

### Step 2.4 — Configure the **vulnerable** Apache vhost
**Why deliberately weak:** these are the flaws competitors must identify.

Create `/etc/httpd/conf.d/grimshay.conf`:
```apache
<VirtualHost *:80>
    ServerName www.grimshay.ca
    DocumentRoot /var/www/grimshay
    Redirect permanent / https://www.grimshay.ca/
</VirtualHost>

<VirtualHost *:443>
    ServerName www.grimshay.ca
    DocumentRoot /var/www/grimshay
    SSLEngine on
    SSLCertificateFile    /etc/pki/tls/certs/grimshay.crt
    SSLCertificateKeyFile /etc/pki/tls/private/grimshay.key
    # ⚠️ DELIBERATE FLAW 1: SSLProtocol not pinned (defaults allow TLS 1.0/1.1)
    # ⚠️ DELIBERATE FLAW 2: SSLCipherSuite not restricted

    <Location />
        AuthType Basic
        AuthName "Members Area"
        AuthBasicProvider ldap
        # ⚠️ DELIBERATE FLAW 3: ldap:// not ldaps:// — credentials sent in clear
        AuthLDAPURL "ldap://172.16.100.10/DC=grimshay,DC=local?sAMAccountName?sub?(objectClass=user)"
        AuthLDAPBindDN "CN=Administrator,CN=Users,DC=grimshay,DC=local"
        AuthLDAPBindPassword "P@ssw0rd"
        Require ldap-group CN=webusers,CN=Users,DC=grimshay,DC=local
    </Location>
</VirtualHost>
```

Generate the self-signed cert:
```bash
sudo openssl req -x509 -nodes -newkey rsa:2048 -days 365 \
    -keyout /etc/pki/tls/private/grimshay.key \
    -out    /etc/pki/tls/certs/grimshay.crt \
    -subj "/CN=www.grimshay.ca"
sudo chmod 644 /etc/pki/tls/certs/grimshay.crt
sudo chmod 644 /etc/pki/tls/private/grimshay.key   # ⚠️ DELIBERATE FLAW 4: world-readable private key
```

### Step 2.5 — More deliberate flaws
```bash
# ⚠️ DELIBERATE FLAW 5: SELinux off
sudo setenforce 0
sudo sed -i 's/^SELINUX=.*/SELINUX=disabled/' /etc/selinux/config

# ⚠️ DELIBERATE FLAW 6: default apache user/group used (no service-specific account)
grep -E "^User|^Group" /etc/httpd/conf/httpd.conf
# (should print "User apache" / "Group apache" — leave as is)

# ⚠️ DELIBERATE FLAW 7: open file permissions on web root
sudo chmod -R 777 /var/www/grimshay
```

Start Apache:
```bash
sudo systemctl restart httpd
sudo firewall-cmd --add-service=http  --permanent
sudo firewall-cmd --add-service=https --permanent
sudo firewall-cmd --reload
```

### Step 2.6 — Sanity test from a browser on the DC
Open Edge/Chrome → `https://www.grimshay.ca` → accept cert warning → log in as `alice / P@ssw0rd` → expect the "members area" page.

### Step 2.7 — Snapshot
ESXi → www.grimshay.ca → snapshot `02-web-vulnerable`.

> **Why we baked these flaws in:** These are exactly the vulnerabilities listed in the marking-scheme judge's notes for A1 row 32 (executive summary). On competition day, when you assess the supplied VM, you will be looking for these same patterns — practice spotting them.

---

## Part 3 — AMClient1 + AMClient2

Repeat for both:

### Step 3.1 — Install Windows 10 Enterprise Eval
4 GB RAM ideal (2 GB minimum). NIC on `PG-MA1-LAN`. Local user `Competitor0 / CharterDressing` (per MA1 line 20).

### Step 3.2 — Static IPs
- AMClient1 → 172.16.100.1 / 24 / DNS 172.16.100.10
- AMClient2 → 172.16.100.2 / 24 / DNS 172.16.100.10

### Step 3.3 — Domain join
```powershell
Add-Computer -DomainName grimshay.local -Credential (Get-Credential) -Restart
```
Use Administrator / `P@ssw0rd` when prompted.

### Step 3.4 — Install client tools
- Chrome, PuTTY, Wireshark (per MA1 line 50).

### Step 3.5 — Snapshot
Each client → snapshot `03-amclient-clean`.

---

## Part 4 — Final integration check

From AMClient1, log in as `alice@grimshay.local / P@ssw0rd`:

1. Open Chrome → `https://www.grimshay.ca` → accept cert → log in alice/P@ssw0rd → expect "members area".
2. Open PuTTY → `ssh competitor@172.16.100.13` → password `P@ssw0rd`.
3. Open Wireshark → start capture → re-load the website → look at the LDAP frames going from 172.16.100.13 → 172.16.100.10 on TCP/389 — you should see `bindRequest` and credentials in cleartext. **This is one of the vulnerabilities you will report tomorrow morning.**

If all three work — environment is ready.

---

## Snapshots checklist

- [ ] DC: `01-DC-clean`
- [ ] www.grimshay.ca: `02-web-vulnerable`
- [ ] AMClient1: `03-amclient-clean`
- [ ] AMClient2: `03-amclient-clean`

Next file: **`04_Setup_VMs_MA2.md`**.
