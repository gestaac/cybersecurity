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

### Step 1.3 — Rename + promote to DC (GUI)

#### 1.3.1 — Rename the computer to `DC`
**Tools:** Server Manager (opens automatically when you log in to Desktop Experience).

1. *Server Manager → Local Server* (left sidebar).
2. In the **PROPERTIES** section, find the row labeled *Computer name* — it'll show a random name like `WIN-XXXXXX`. Click that name.
3. *System Properties* dialog opens → **Change…** button.
4. *Computer name:* type `DC` → OK.
5. Prompt: *"You must restart your computer..."* → **OK**.
6. Back in System Properties → **Close** → click **Restart Now**.

**Verify after reboot:** log back in. Server Manager → Local Server → Computer name now shows `DC`.

> *Quick PowerShell alternative:* `Rename-Computer -NewName "DC" -Restart`

#### 1.3.2 — Set static IP (skip if already done in Step 1.2)
Already covered in Step 1.2 above. Confirm `ipconfig` shows `172.16.100.10` before continuing.

#### 1.3.3 — Install AD Domain Services role
**Tools:** Server Manager.

1. *Server Manager → Manage* (top-right) → **Add Roles and Features**.
2. *Before You Begin* → Next.
3. *Installation Type* → **Role-based or feature-based installation** → Next.
4. *Server Selection* → leave default (`DC` selected) → Next.
5. *Server Roles* → tick ☑ **Active Directory Domain Services**.
   - Pop-up: *"Add features that are required..."* → **Add Features** → close the pop-up → Next.
6. *Features* → leave defaults → Next.
7. *AD DS* (info screen) → Next.
8. *Confirmation* → tick ☑ **Restart the destination server automatically if required** → **Install**.
9. Wait ~3 minutes. The progress bar may say "Installation succeeded" before fully done — wait for the close button.

**Verify:** Server Manager dashboard now shows **AD DS** in the left sidebar.

> *Quick PowerShell alternative:* `Install-WindowsFeature AD-Domain-Services -IncludeManagementTools`

#### 1.3.4 — Promote to Domain Controller
After the role install, Server Manager shows a yellow flag at the top-right with a yellow triangle and the text *"Configuration required for Active Directory Domain Services at DC"*.

1. Click that yellow flag → click **Promote this server to a domain controller**.
2. *Deployment Configuration*:
   - Select **Add a new forest**.
   - Root domain name: `grimshay.local` → Next.
3. *Domain Controller Options*:
   - Forest functional level: Windows Server 2016 (default).
   - Domain functional level: Windows Server 2016 (default).
   - Tick ☑ **Domain Name System (DNS) server**.
   - Tick ☑ **Global Catalog (GC)**.
   - **Type the Directory Services Restore Mode (DSRM) password:** `P@ssw0rd` → confirm.
   - Next.
4. *DNS Options* — yellow warning *"A delegation for this DNS server cannot be created..."* — **ignore**, click Next.
5. *Additional Options*:
   - NetBIOS domain name: `GRIMSHAY` (default — confirm).
   - Next.
6. *Paths* → leave defaults (NTDS, SYSVOL, log paths) → Next.
7. *Review Options* → review → Next.
8. *Prerequisites Check* — should show *"All prerequisite checks passed successfully"*. Yellow warnings about cryptography are normal — ignore.
9. **Install** → wait ~5 min → the VM will **reboot automatically** when done.

**Verify after reboot:** at the login screen, you'll now see `GRIMSHAY\Administrator` instead of just `Administrator`. Log in.

> *Quick PowerShell alternative:* the multi-line `Install-ADDSForest` command from earlier. GUI takes ~5 min, PowerShell ~3 min.

---

### Step 1.4 — Create the `webusers` group + 3 test users (GUI)
**Why:** MA1 says only `webusers` should access the website.
**Tools:** Active Directory Users and Computers (ADUC).

#### 1.4.1 — Open ADUC
*Server Manager → Tools → Active Directory Users and Computers* (sorted alphabetically near the top).

You'll see the tree on the left:
```
grimshay.local
├── Builtin
├── Computers
├── Domain Controllers
├── ForeignSecurityPrincipals
├── Managed Service Accounts
└── Users
```

#### 1.4.2 — Create the `webusers` security group
1. Right-click the **Users** container → **New → Group**.
2. *Group name:* `webusers`.
3. *Group scope:* **Global** (default).
4. *Group type:* **Security** (default).
5. OK.

**Verify:** the right pane now lists `webusers` as type *Security Group - Global*.

#### 1.4.3 — Create users alice, bob, carol
For **each** of `alice`, `bob`, `carol`:
1. Right-click **Users** → **New → User**.
2. *First name:* alice (or bob, carol).
3. *User logon name:* alice (lowercase, matches first name).
4. Next.
5. *Password:* `P@ssw0rd` → confirm.
6. **UNTICK** *"User must change password at next logon"*.
7. **TICK** *"Password never expires"*.
8. Next → Finish.

Repeat for bob, then carol.

**Verify:** *Users* container now lists alice, bob, carol as *User* objects.

#### 1.4.4 — Add alice, bob, carol to the `webusers` group
**Method 1** — from the user side:
1. Right-click **alice** → **Properties → Member Of** tab → **Add**.
2. Type `webusers` → **Check Names** (auto-completes) → **OK** → OK.
3. Repeat for bob and carol.

**Method 2** — from the group side (faster for many users):
1. Right-click **webusers** group → **Properties → Members** tab → **Add**.
2. Type `alice; bob; carol` (semicolon-separated) → **Check Names** → OK → OK.

**Verify:** double-click `webusers` → *Members* tab shows all three users.

> *Quick PowerShell alternative:*
> ```powershell
> New-ADGroup -Name "webusers" -GroupScope Global -GroupCategory Security
> "alice","bob","carol" | % {
>   New-ADUser -Name $_ -AccountPassword (ConvertTo-SecureString "P@ssw0rd" -AsPlainText -Force) -Enabled $true
>   Add-ADGroupMember -Identity webusers -Members $_
> }
> ```

---

### Step 1.5 — Add DNS records for the webserver (GUI)
**Why:** AMClient1 needs to resolve `www.grimshay.ca` and `www.grimshay.local` to LinSRV1's IP `172.16.100.13`.
**Tools:** DNS Manager.

#### 1.5.1 — Open DNS Manager
*Server Manager → Tools → DNS*.

Tree on the left:
```
DNS
└── DC
    ├── Forward Lookup Zones
    │   ├── _msdcs.grimshay.local
    │   └── grimshay.local            ← already exists
    ├── Reverse Lookup Zones
    └── ...
```

#### 1.5.2 — Add A record for `www` in `grimshay.local`
1. Expand *Forward Lookup Zones*.
2. Right-click **grimshay.local** → **New Host (A or AAAA)…**.
3. *Name (uses parent domain name if blank):* `www`.
4. *IP address:* `172.16.100.13`.
5. UNTICK *"Create associated pointer (PTR) record"* (no reverse zone yet).
6. **Add Host** → success message → **OK** → **Done**.

**Verify:** double-click `grimshay.local` → see `www` row with type *Host (A)* and data `172.16.100.13`.

#### 1.5.3 — Create the `grimshay.ca` primary zone
The website hostname is actually `www.grimshay.ca` (per MA1), so we need a separate zone for the `.ca` TLD too.

1. Right-click **Forward Lookup Zones** → **New Zone…**.
2. Wizard:
   - *Zone Type* → **Primary zone** (leave *"Store the zone in Active Directory"* ticked).
   - *Replication Scope* → **To all DNS servers running on domain controllers in this forest** → Next.
   - *Zone Name* → `grimshay.ca` → Next.
   - *Dynamic Update* → **Allow only secure dynamic updates (recommended for AD)** → Next.
   - **Finish**.

**Verify:** *Forward Lookup Zones* now shows both `grimshay.local` and `grimshay.ca`.

#### 1.5.4 — Add A record for `www` in `grimshay.ca`
1. Right-click **grimshay.ca** → **New Host (A or AAAA)…**.
2. *Name:* `www`.
3. *IP address:* `172.16.100.13`.
4. **Add Host** → OK → Done.

**Verify:** from a command prompt on DC:
```cmd
nslookup www.grimshay.ca
nslookup www.grimshay.local
```
Both should return `172.16.100.13`.

> *Quick PowerShell alternative:*
> ```powershell
> Add-DnsServerResourceRecordA -Name "www" -ZoneName "grimshay.local" -IPv4Address "172.16.100.13"
> Add-DnsServerPrimaryZone -Name "grimshay.ca" -ReplicationScope "Forest"
> Add-DnsServerResourceRecordA -Name "www" -ZoneName "grimshay.ca" -IPv4Address "172.16.100.13"
> ```

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
