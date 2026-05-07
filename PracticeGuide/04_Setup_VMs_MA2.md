# 04 — Build the MA2 Practice Environment (manila.com)

In MA2 you receive a **half-built** network: pfSense is racked but unconfigured, WINSRV3 is partially CA, no GPOs, no hardening on LinSRV1, no VPN. Your job is to finish all of it.

For practice, build it to the **same starting point**. Then practice the actual implementation work via files `20_…` to `23_…`.

> **Total build time:** ~5 hours first time, then 45 min from snapshots.

---

## VM inventory (per MA2 Table 1)

| VM | Port group | OS | RAM | Disk | IP |
|---|---|---|---|---|---|
| ISP | PG-Internet | CentOS 9 | 1 GB | 10 GB | 10.0.0.1/24 |
| pfSense | WAN: PG-Internet, LAN: PG-LAN, DMZ: PG-DMZ, Servers: PG-Servers | pfSense 2.7.2 | 2 GB | 20 GB | per Table 1 |
| WINSRV1 | PG-Servers | Win Server 2022 | 4 GB | 80 GB | 192.168.2.10/24 |
| WINSRV3 | PG-Servers | Win Server 2022 | 3 GB | 60 GB | 192.168.2.30/24 |
| WINSRV4 | PG-Servers | Win Server 2022 | 2 GB | 60 GB | 192.168.2.50/24 (off) |
| LINSRV1 | PG-DMZ | CentOS 9 | 2 GB | 20 GB | 192.168.1.10/24 |
| Client1 | PG-LAN | Win 10 Eval | 2 GB | 40 GB | DHCP |
| Client2 | PG-LAN | Win 10 Eval | 2 GB | 40 GB | DHCP |
| Client3 | PG-Internet | Win 10 Eval | 2 GB | 40 GB | DHCP from ISP |

> All passwords default `P@ssw0rd`. ESXi credentials per the actual MA2 PDF: `wsauser / Andres@9V4` (IP `192.168.1.1`). Workstation login: `competitor1b / Tagaytay_62&L`.

---

## Part 1 — ISP (the fake Internet)

### Step 1.1 — Install CentOS Stream 9
1 GB RAM, 1 NIC on `PG-Internet`, hostname `isp.local`.

### Step 1.2 — Static IP + DNS + DHCP + websites
```bash
nmcli con mod ens160 ipv4.addresses 10.0.0.1/24 ipv4.method manual
nmcli con up ens160
sudo dnf install -y dnsmasq httpd mod_ssl
```

`/etc/dnsmasq.conf`:
```
interface=ens160
dhcp-range=10.0.0.50,10.0.0.150,12h
dhcp-option=3,10.0.0.1
dhcp-option=6,10.0.0.1

# fake "internet" hostnames
address=/www.nationalmuseum.gov.ph/10.0.0.10
address=/www.starcity.com.ph/10.0.0.20
```

```bash
sudo systemctl enable --now dnsmasq
```

Add IP aliases for the two test sites:
```bash
sudo nmcli con mod ens160 +ipv4.addresses 10.0.0.10/24 +ipv4.addresses 10.0.0.20/24
sudo nmcli con up ens160
```

Set up two trivial vhosts:
```bash
sudo mkdir -p /var/www/nm /var/www/sc
echo "<h1>National Museum (test)</h1>" | sudo tee /var/www/nm/index.html
echo "<h1>Star City (test)</h1>"      | sudo tee /var/www/sc/index.html
```

`/etc/httpd/conf.d/nm.conf`:
```apache
Listen 10.0.0.10:443
<VirtualHost 10.0.0.10:443>
  ServerName www.nationalmuseum.gov.ph
  DocumentRoot /var/www/nm
  SSLEngine on
  SSLCertificateFile    /etc/pki/tls/certs/nm.crt
  SSLCertificateKeyFile /etc/pki/tls/private/nm.key
</VirtualHost>
```
Same for `sc.conf` with 10.0.0.20 / `www.starcity.com.ph`. Generate self-signed certs (per MA2 PDF page 8 — both test sites are intentionally self-signed):
```bash
for h in nm sc; do sudo openssl req -x509 -nodes -newkey rsa:2048 -days 365 \
   -keyout /etc/pki/tls/private/$h.key -out /etc/pki/tls/certs/$h.crt \
   -subj "/CN=$h.test"; done
sudo systemctl enable --now httpd
sudo firewall-cmd --add-service=http --add-service=https --add-service=dns --add-service=dhcp --permanent
sudo firewall-cmd --reload
```

> **Snapshot:** `ISP-ready`.

---

## Part 2 — pfSense (4 NICs, base install)

### Step 2.1 — Create the VM with 4 NICs
ESXi → New VM → *Other → FreeBSD 13 64-bit*, 2 GB RAM, 20 GB disk, then **add 4 network adapters** in this order:
1. `PG-Internet`  (will become WAN)
2. `PG-LAN`
3. `PG-DMZ`
4. `PG-Servers`

Mount the pfSense ISO → Power on → standard install (defaults).

### Step 2.2 — Assign interfaces in console
After first boot pfSense asks "Should VLANs be set up?" → **n**.
Then "WAN interface name" → `vmx0` (the first NIC). LAN → `vmx1`. Then it asks for OPT1, OPT2 — answer `vmx2` (DMZ), `vmx3` (Servers). Confirm.

### Step 2.3 — Set the LAN IP from console
Pick option **2 (Set interface(s) IP address)** → LAN → IPv4 manual → `172.16.100.254 / 24` → no DHCP yet (we'll do via web UI) → no HTTP redirect.

### Step 2.4 — Open the pfSense web UI from a Client1 VM later
For now, the pfSense base install is **ready to be configured by competitors** (MA2 PDF page 8: *"The firewall is installed in its base configuration..."*). Snapshot.

> **Snapshot:** `pfSense-base`.

---

## Part 3 — Active Directory (WINSRV1)

### Step 3.1 — Install Win Server 2022
Same approach as MA1 DC. Static IP `192.168.2.10/24`, gateway `192.168.2.254`, DNS `127.0.0.1`. Rename to `WINSRV1`.

### Step 3.2 — Promote to DC for `manila.com` (GUI)

Same pattern as MA1 (`03_…` Step 1.3). Quick recap of the click path:

#### 3.2.1 — Install AD DS + DNS + DHCP roles
1. *Server Manager → Manage → Add Roles and Features*.
2. Wizard: Next → Next → leave server `WINSRV1` selected → Next.
3. *Server Roles* tick:
   - ☑ **Active Directory Domain Services** (add features when prompted).
   - ☑ **DNS Server** (add features when prompted; ignore the "static IP recommended" warning — we already set one).
   - ☑ **DHCP Server** (add features when prompted).
4. Next → Next → leave defaults on info pages → tick *Restart automatically* → **Install**.
5. Wait ~5 min for installation.

#### 3.2.2 — Promote to Domain Controller
1. Server Manager top-right yellow flag → **Promote this server to a domain controller**.
2. *Deployment Configuration* → **Add a new forest** → Root domain name `manila.com` → Next.
3. *Domain Controller Options*:
   - DSRM password: `P@ssw0rd` (confirm).
   - Tick DNS server + Global Catalog.
   - Next.
4. *DNS Options* → ignore the delegation warning → Next.
5. *Additional Options* → NetBIOS name `MANILA` → Next.
6. *Paths* → defaults → Next.
7. *Review* → Next.
8. *Prerequisites* → Install. **VM reboots automatically.**

After reboot, log in as `MANILA\Administrator / P@ssw0rd`.

#### 3.2.3 — Create the OUs (Organisational Units)
Per MA2 PDF Table 3, users are split into **Manila** and **Singapore** OUs.

**Tools:** *Server Manager → Tools → Active Directory Users and Computers*.

Right-click `manila.com` → **New → Organizational Unit**. Create two OUs:
- `Manila`
- `Singapore`

#### 3.2.4 — Create the AD groups
Right-click `manila.com` (or the Users container) → **New → Group**. Scope **Global**, type **Security**. Create:

- `Marketing`
- `Customer Service`
- `Sales`
- `Executive`
- `IT`
- `VPNGroup` (per MA2 PDF page 10 — must be named exactly `VPNGroup`)

(6 groups total.)

#### 3.2.5 — Create the AD users
Per MA2 PDF Table 3 (page 14). For each user, place them in the matching OU and group.

**Inside the Manila OU:**
| Username | Display Name | Job Title | Department | Group |
|---|---|---|---|---|
| M001 | Brand Marketing Specialist | Brand Marketing Specialist | Marketing | Marketing |
| M002 | Customer Experience Rep | Customer Experience Representative | Customer Service | Customer Service |
| M003 | Store Sales | Store Sales | Sales | Sales |
| M004 | Operations Manager | Operations Manager | Operations and Logistics | Executive |
| C1 | Technical Support (MNL) | Technical Support | IT | IT |

**Inside the Singapore OU:**
| Username | Display Name | Job Title | Department | Group |
|---|---|---|---|---|
| S001 | Marketing Manager | Marketing Manager | Marketing | Executive |
| C2 | Technical Support (SG) | Technical Support | IT | IT |

**Plus VPN test user (in Manila OU or Users):**
| Username | Group |
|---|---|
| VPNUser | VPNGroup |

For each user:
- *First name:* the display name (or just M001/M002/etc.).
- *User logon name:* M001 / M002 / M003 / M004 / S001 / C1 / C2 / VPNUser.
- Password: `P@ssw0rd` (and re-confirm — note the project says "do not modify the passwords unless required").
- UNTICK *"User must change password at next logon"*.
- TICK *"Password never expires"*.
- After creation: right-click user → **Properties → Organization** tab → fill in Job Title, Department, City (Manila or Singapore).

#### 3.2.6 — Add users to groups
Right-click each group → **Properties → Members → Add**:

| Group | Members |
|---|---|
| Marketing | M001 |
| Customer Service | M002 |
| Sales | M003 |
| Executive | M004, S001 |
| IT | C1, C2 |
| VPNGroup | VPNUser |

> *Quick PowerShell alternative for all of Step 3.2:*
> ```powershell
> Install-WindowsFeature AD-Domain-Services,DNS,DHCP -IncludeManagementTools
> Install-ADDSForest -DomainName "manila.com" -DomainNetbiosName "MANILA" `
>   -SafeModeAdministratorPassword (ConvertTo-SecureString "P@ssw0rd" -AsPlainText -Force) `
>   -InstallDns -Force
> # ... after reboot:
> "Marketing","Customer Service","Sales","Executive","IT","VPNGroup" |
>   % { New-ADGroup -Name $_ -GroupScope Global -GroupCategory Security }
> @("M001","M002","M003","M004","S001","C1","C2","VPNUser") |
>   % { New-ADUser -Name $_ -SamAccountName $_ -AccountPassword (ConvertTo-SecureString "P@ssw0rd" -AsPlainText -Force) -Enabled $true -ChangePasswordAtLogon $false }
> Add-ADGroupMember "Marketing" M001
> Add-ADGroupMember "Customer Service" M002
> Add-ADGroupMember "Sales" M003
> Add-ADGroupMember "Executive" M004,S001
> Add-ADGroupMember "IT" C1,C2
> Add-ADGroupMember "VPNGroup" VPNUser
> ```

### Step 3.3 — DNS records (GUI)
**Tools:** *Server Manager → Tools → DNS*.

Expand **DC** (or **WINSRV1**) → **Forward Lookup Zones**. The zone `manila.com` already exists from the promotion.

Right-click **manila.com** → **New Host (A or AAAA)…**. Add three records (one at a time):

| Name | IP | What it points to |
|---|---|---|
| `www` | `192.168.1.10` | LinSRV1 in DMZ (per MA2 PDF page 11) |
| `webtest` | `192.168.2.30` | WINSRV3 IIS (per MA2 PDF page 11) |

For each: type the Name, type the IP, click **Add Host** → OK → Done.

**Verify:** double-click `manila.com` → see `www`, `webtest`, `w3` each with type *Host (A)*.

> *Quick PowerShell alternative:*
> ```powershell
> Add-DnsServerPrimaryZone -Name "manila.com" -ReplicationScope Forest  # only if zone doesn't exist
> Add-DnsServerResourceRecordA -ZoneName manila.com -Name "www"     -IPv4Address 192.168.1.10
> Add-DnsServerResourceRecordA -ZoneName manila.com -Name "webtest" -IPv4Address 192.168.2.30
> ```

<!-- Step 3.4 removed: the new MA2 PDF does not require the Chrome Enterprise Bundle / google GPO. Skip this step. -->

### Step 3.5 — Pre-create the `pictures` share folder (empty)
**Per MA2 PDF page 12** — the share is at `C:\shares\pictures` and contains a file called **`park.jpg`**.

```powershell
mkdir C:\shares\pictures
# Drop a placeholder so the MA2 PDF "park.jpg audit" step (page 12) has something to audit:
Set-Content C:\shares\pictures\park.jpg "fake-jpeg-bytes"
```

> **Do NOT** create the share, GPOs, password policy, or audit yet — those are the actual MA2 deliverables you'll do during practice from `22_Day1_MA2_WinSRV1_AD.md`.

> **Snapshot:** `WINSRV1-base`.

---

## Part 4 — Issuing CA (WINSRV3, partially configured)

MA2 PDF page 11: *"WinSRV3 will be the issuing CA. This machine is already configured as the subordinate (issuing) CA for the domain. Complete the following tasks…"*

### Step 4.1 — Install Win Server 2022, static IP, domain-join `manila.com` (GUI)

#### 4.1.1 — Install OS + set static IP
Same as MA1 / WINSRV1 base install. Static IP `192.168.2.30/24`, gateway `192.168.2.254`, DNS `192.168.2.10` (points at WINSRV1).

#### 4.1.2 — Rename to `WINSRV3`
*Server Manager → Local Server → click computer name → Change → name `WINSRV3` → OK → restart.*

#### 4.1.3 — Join the manila.com domain
1. After reboot, log in as local Administrator.
2. *Server Manager → Local Server → click the link "WORKGROUP"* (right of *Workgroup:*).
3. *System Properties → Change…* button.
4. Member of: **Domain** → type `manila.com` → OK.
5. Prompt for credentials: `MANILA\Administrator` / `P@ssw0rd` → OK.
6. *"Welcome to the manila.com domain"* → OK.
7. Restart prompt → OK → Close → **Restart Now**.

#### 4.1.4 — Verify after reboot
Login screen should show *"Sign in to: MANILA"* option. Log in as `MANILA\Administrator / P@ssw0rd`.
Server Manager → Local Server → Domain shows `manila.com`.

> *Quick PowerShell alternative:*
> ```powershell
> Rename-Computer -NewName WINSRV3 -Restart
> # after reboot
> Add-Computer -DomainName manila.com -Credential (Get-Credential) -Restart
> ```

### Step 4.2 — Install AD CS as Enterprise Subordinate CA (GUI)

#### 4.2.1 — Add the AD CS role
1. *Server Manager → Manage → Add Roles and Features*.
2. Wizard: Next → role-based → server selection (`WINSRV3`) → Next.
3. *Server Roles* tick:
   - ☑ **Active Directory Certificate Services** (add features when prompted).
   - ☑ **Web Server (IIS)** under *"Web Server (IIS)"* (add features when prompted) — needed for web enrollment.
4. *AD CS* role services screen — tick:
   - ☑ **Certification Authority**
   - ☑ **Certification Authority Web Enrollment**
5. Next → Next → leave IIS defaults → Next → Install.
6. Wait. Close when done.

#### 4.2.2 — Configure AD CS
After role install, Server Manager top-right shows yellow flag *"Configuration required for Active Directory Certificate Services at WINSRV3"*.

1. Click yellow flag → **Configure Active Directory Certificate Services on the destination server**.
2. *Credentials* → leave `MANILA\Administrator` → Next.
3. *Role Services* → tick:
   - ☑ Certification Authority
   - ☑ Certification Authority Web Enrollment
   - Next.
4. *Setup Type* → **Enterprise CA** → Next.
5. *CA Type* → **Subordinate CA** → Next.
6. *Private Key* → **Create a new private key** → Next.
7. *Cryptography for CA*:
   - Cryptographic provider: `RSA#Microsoft Software Key Storage Provider`
   - Key length: `2048`
   - Hash: `SHA256`
   - Next.
8. *CA Name*:
   - Common name: `manila-WINSRV3-CA` (default).
   - Distinguished name suffix: leave default.
   - Next.
9. *Certificate Request*:
   - Select **Save a certificate request to file on the target machine**.
   - File name: `C:\winsrv3.req` → Next.
10. *Certificate Database* → leave defaults (paths) → Next.
11. *Confirmation* → **Configure** → wait → success → **Close**.

The CA service is now **installed but not running** (waiting on the signed CSR from WINSRV4 root). That's the half-built state.

> *Quick PowerShell alternative:*
> ```powershell
> Install-WindowsFeature AD-Certificate, ADCS-Cert-Authority, ADCS-Web-Enrollment, Web-Server -IncludeManagementTools
> Install-AdcsCertificationAuthority -CAType EnterpriseSubordinateCA `
>    -HashAlgorithm SHA256 -KeyLength 2048 `
>    -CryptoProviderName "RSA#Microsoft Software Key Storage Provider" `
>    -OutputCertRequestFile C:\winsrv3.req -Force
> ```

### Step 4.3 — Snapshot
> **Snapshot:** `WINSRV3-half`.

---

## Part 5 — Offline Root CA (WINSRV4)

### Step 5.1 — Install Win Server 2022 standalone (NOT joined to domain)
Static `192.168.2.50` only for cert issuance; can be powered off during normal practice.

### Step 5.2 — Install Standalone Root CA
```powershell
Install-WindowsFeature AD-Certificate -IncludeManagementTools
Install-AdcsCertificationAuthority -CAType StandaloneRootCA `
   -CACommonName "Manila-Root-CA" -ValidityPeriod Years -ValidityPeriodUnits 10 `
   -HashAlgorithm SHA256 -KeyLength 4096 -Force
```

### Step 5.3 — Sign the WINSRV3 subordinate CSR (one-time, then power off)
1. Copy `C:\winsrv3.req` from WINSRV3 to WINSRV4 (use a shared folder or USB).
2. On WINSRV4: `certreq -submit C:\winsrv3.req` → choose Manila-Root-CA → save the issued `.cer`.
3. Copy the `.cer` back to WINSRV3 → `certutil -installCert C:\winsrv3.cer` → `Start-Service CertSvc`.
4. Power off WINSRV4.

> **Snapshot WINSRV4:** `WINSRV4-RootCA-signed-once`.

---

## Part 6 — LinSRV1 (CentOS, in DMZ, **un-hardened**)

### Step 6.1 — Install CentOS Stream 9, NIC on PG-DMZ
Static `192.168.1.10/24`, gateway `192.168.1.254`, DNS `192.168.2.10`.

### Step 6.2 — Install required packages
```bash
sudo dnf install -y httpd mod_ssl realmd sssd oddjob oddjob-mkhomedir adcli \
                    samba-common-tools krb5-workstation chrony \
                    libpwquality pam_pwquality \
                    firewalld sudo openssl wireshark putty
sudo systemctl enable --now chronyd firewalld httpd
```

### Step 6.3 — Drop the placeholder website
```bash
sudo mkdir -p /var/www/manila
echo "<h1>Manila intranet site</h1>" | sudo tee /var/www/manila/index.html
sudo cp /etc/httpd/conf.d/welcome.conf /etc/httpd/conf.d/welcome.conf.bak
sudo bash -c 'cat > /etc/httpd/conf.d/manila.conf <<EOF
<VirtualHost *:80>
  DocumentRoot /var/www/manila
  ServerName www.manila.com
</VirtualHost>
EOF'
sudo systemctl restart httpd
```

### Step 6.4 — Leave it un-hardened
- Do NOT enable SELinux yet (`sudo setenforce 0`).
- Do NOT change ssh port.
- Do NOT add password complexity.
- Do NOT join domain.

That's the starting point competitors receive.

> **Snapshot:** `LinSRV1-base-unhardened`.

---

## Part 7 — Clients (Client1, Client2, Client3)

Three Win 10 Enterprise Eval VMs.

| VM | Port group | NIC | Static or DHCP |
|---|---|---|---|
| Client1 | PG-LAN | 1 | DHCP (waits for pfSense DHCP later) |
| Client2 | PG-LAN | 1 | DHCP |
| Client3 | PG-Internet | 1 | DHCP from ISP |

For each:
- Local user `Competitor / P@ssw0rd` (per Table 1 line 71).
- Install Chrome, PuTTY, Wireshark.
- For Client3 also install **OpenVPN Connect** + **Nmap/Zenmap**.
- Do **not** domain-join (per Table 1 they're not domain members initially; the GPO test will join Client1/2 to manila.com once pfSense routing is up).

> **Snapshot each:** `Client1-base`, `Client2-base`, `Client3-base`.

---

## Final integration check before practice

- [ ] All 9 VMs power on without errors
- [ ] ISP responds to DNS lookups from any 10.0.0.x address: `nslookup www.starcity.com.ph 10.0.0.1`
- [ ] WINSRV1 promoted to manila.com, AD users + groups created
- [ ] WINSRV3 has CertSvc service, certs issued from Manila-Root-CA chain
- [ ] LinSRV1 reachable from 192.168.2.10 (after pfSense rules), httpd serves /var/www/manila
- [ ] All snapshots taken

When all ticked → next file: **`10_Day1_MA1_Solution.md`**.
