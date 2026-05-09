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
| **SecOnion** | mgmt: PG-Servers, sniff: PG-MIRROR | Security Onion 2.4 | 8 GB | 200 GB | 192.168.2.20/24 |

> All passwords default `P@ssw0rd`. ESXi credentials per the actual MA2 PDF: `wsauser / Andres@9V4` (IP `192.168.1.1`). Workstation login: `competitor1b / Tagaytay_62&L`.

> 💡 **SecOnion (the 10th VM)** is for **Day 2 afternoon Crit B (IR/Forensics/AppSec, 25 K)**. Build per `07_Setup_SecurityOnion.md`. It needs internet during install (so-setup downloads ~10 GB Docker images + ETOPEN ruleset).

---

## Part 1 — ISP (the fake Internet)

### Step 1.1 — Create the VM on ESXi

#### 1.1.1 — VM specs (use these values in the wizard)

| Field | Value |
|---|---|
| **Name** | `ISP` |
| **Compatibility** | ESXi 8.0 virtual machine |
| **Guest OS family** | Linux |
| **Guest OS version** | CentOS 9 (64-bit) |
| **CPU** | 1 |
| **Memory** | `1024` MB (1 GB) |
| **Hard disk 1** | `10` GB, **Thin Provisioned** |
| **Network Adapter 1** | `PG-Internet` |
| **CD/DVD Drive 1** | Datastore ISO file → `CentOS-Stream-9-latest-x86_64-dvd1.iso` → ✅ Connect at power on |

#### 1.1.2 — Walk through the ESXi wizard

Follow **📘 Appendix A** at the bottom of this file (5 screens). Use the values from the table above when prompted.

After clicking Finish → power on → click **Console**.

#### 1.1.3 — Walk through the CentOS Stream 9 installer

Follow **📘 Appendix B.2** at the bottom of this file (~15 min total).

Use these values when the Anaconda installer asks:

| Anaconda spoke | What to enter |
|---|---|
| **Host Name** (Network & Host Name tile) | `isp.local` |
| **Root Password** | `P@ssw0rd` (✅ tick **Allow root SSH login with password**) |
| **User Creation** | name=`competitor`, password=`P@ssw0rd`, ✅ make administrator |
| **Software Selection** | **Minimal Install** |

📝 **Write down the interface name** shown in the **Network & Host Name** tile (likely `ens160` or `ens192` for ESXi VMware NICs). You'll need it in Step 1.2.

After install completes → eject the ISO (per Appendix A's reminder) → reboot → log in as `root / P@ssw0rd`.

> 💡 If the interface name shown was something OTHER than `ens160`, **substitute your actual name** for `ens160` everywhere in Step 1.2.

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

### Step 2.1 — Create the VM on ESXi (4 NICs!)

#### 2.1.1 — VM specs (use these values in the wizard)

| Field | Value |
|---|---|
| **Name** | `pfSense` |
| **Compatibility** | ESXi 8.0 virtual machine |
| **Guest OS family** | Other |
| **Guest OS version** | FreeBSD 13 or later versions (64-bit) |
| **CPU** | 2 |
| **Memory** | `2048` MB (2 GB) |
| **Hard disk 1** | `20` GB, **Thin Provisioned** |
| **Network Adapter 1** | `PG-Internet` ← becomes WAN (`vmx0`) |
| **Network Adapter 2** | `PG-LAN` ← becomes LAN (`vmx1`) |
| **Network Adapter 3** | `PG-DMZ` ← becomes OPT1 (`vmx2`) |
| **Network Adapter 4** | `PG-Servers` ← becomes OPT2 (`vmx3`) |
| **CD/DVD Drive 1** | Datastore ISO file → `pfSense-CE-2.7.2-RELEASE-amd64.iso` → ✅ Connect at power on |

> ⚠️ **Order matters!** ESXi assigns FreeBSD interface names (`vmx0`, `vmx1`, ...) in the order NICs are added. The first one (PG-Internet) MUST be Network Adapter 1, second (PG-LAN) Network Adapter 2, etc. If you mix the order, pfSense's WAN/LAN/DMZ/Servers mapping will be wrong.

#### 2.1.2 — Walk through the ESXi wizard

Follow **📘 Appendix A** at the bottom of this file. Use the values from the table above.

> 💡 In **Screen 4 (Customize settings)**, click **`Add network adapter`** (top of the screen) **3 times** so you have 4 network adapters total. Then set each one to its correct port group **in order** (Adapter 1 = PG-Internet, Adapter 2 = PG-LAN, Adapter 3 = PG-DMZ, Adapter 4 = PG-Servers).

After clicking Finish → power on → click **Console**.

#### 2.1.3 — Walk through the pfSense installer

Follow **📘 Appendix B.5** at the bottom of this file (~10 min total).

The installer ends at the pfSense console main menu (Phase 10 of Appendix B.5). Continue with Step 2.2.

### Step 2.2 — Verify the interface mapping (in pfSense console)

If you followed Appendix B.5 Phase 9 correctly, you already typed the mappings. Confirm by looking at the top of the main menu:

```
WAN (wan)   -> vmx0 -> v4/DHCP4: 10.0.0.x/24    ← PG-Internet, gets IP from ISP
LAN (lan)   -> vmx1 -> v4: 192.168.1.1/24       ← PG-LAN, default IP (changes in Step 2.3)
OPT1 (opt1) -> vmx2 -> (no IP)                  ← PG-DMZ
OPT2 (opt2) -> vmx3 -> (no IP)                  ← PG-Servers
```

If the mapping is wrong (e.g., LAN went to vmx0):
1. At the menu, type `1` → **Enter** (Assign Interfaces).
2. Re-answer: VLANs → `n`, WAN → `vmx0`, LAN → `vmx1`, OPT1 → `vmx2`, OPT2 → `vmx3`, Optional 3 → empty Enter, proceed → `y`.

### Step 2.3 — Set the LAN IP from console

At the main menu prompt `Enter an option:`, type `2` → **Enter** (Set interface(s) IP address).

Walk through the prompts:

| Prompt | Answer |
|---|---|
| "Available interfaces: 1 - WAN, 2 - LAN, 3 - OPT1, 4 - OPT2. Enter the number of the interface to configure" | `2` (LAN) |
| "Configure IPv4 address LAN interface via DHCP?" | `n` |
| "Enter the new LAN IPv4 address" | `172.16.100.254` |
| "Enter the new LAN IPv4 subnet bit count" | `24` |
| "For a LAN, press Enter for none. Enter the new LAN IPv4 upstream gateway address" | (empty Enter — no upstream gateway on LAN side) |
| "Configure IPv6 address LAN interface via DHCP6?" | `n` |
| "Enter the new LAN IPv6 address" | (empty Enter) |
| "Do you want to enable the DHCP server on LAN?" | `n` (we configure DHCP later via web UI per MA2 PDF) |
| "Do you want to revert to HTTP as the webConfigurator protocol?" | `n` (keep HTTPS) |

Final confirmation: pfSense reports the LAN is now `172.16.100.254/24`.

> Repeat the same flow for OPT1 (DMZ → `192.168.1.254/24`) and OPT2 (Servers → `192.168.2.254/24`) — same `n / address / 24 / Enter / n / Enter / n / n` pattern.

### Step 2.4 — Stop here — pfSense base install is done

The pfSense base install is now **ready to be configured by competitors** (MA2 PDF page 8: *"The firewall is installed in its base configuration..."*).

**Do NOT** configure firewall rules, OpenVPN, Snort, or DHCP yet — those are the actual MA2 deliverables you'll do during practice from `20_Day1_MA2_Firewall.md`.

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

- [ ] All 10 VMs power on without errors (9 hardening VMs + Security Onion for Day 2 PM)
- [ ] ISP responds to DNS lookups from any 10.0.0.x address: `nslookup www.starcity.com.ph 10.0.0.1`
- [ ] WINSRV1 promoted to manila.com, AD users + groups created
- [ ] WINSRV3 has CertSvc service, certs issued from Manila-Root-CA chain
- [ ] LinSRV1 reachable from 192.168.2.10 (after pfSense rules), httpd serves /var/www/manila
- [ ] All snapshots taken

When all ticked → next file: **`10_Day1_MA1_Solution.md`**.

---

# 📘 Appendix A — ESXi "Create / Register VM" Wizard (Beginner Reference)

**Use this whenever a Part above says "follow Appendix A".**

The wizard has **5 screens**. Below is a generic walkthrough — each Part 1–7 gives you the **specs table** with the exact values to type for that VM.

> Beginner note: navigation here is **mouse-driven** (this is the ESXi web UI, not a console). Use **Next** at the bottom to advance, **Back** to fix mistakes.

## Screen 1 — Select creation type

- Pick **`Create a new virtual machine`**.
- Click **Next**.

> The other options ("Deploy from OVF/OVA" / "Register existing VM") are for importing pre-built VMs — not what we want when installing fresh.

## Screen 2 — Select a name and guest OS

Fill in:

| Field | What to enter |
|---|---|
| **Name** | (from the Part's specs table — e.g. `ISP`, `pfSense`, `WINSRV1`) |
| **Compatibility** | `ESXi 8.0 virtual machine` (default) |
| **Guest OS family** | `Linux` for CentOS / Ubuntu / pfSense — `Windows` for Win Server / Win 10 |
| **Guest OS version** | (from the Part's specs table) |

Click **Next**.

## Screen 3 — Select storage

- Pick your default datastore (usually `datastore1`).
- Click **Next**.

## Screen 4 — Customize settings (the big one)

This screen has many fields. Most stay at defaults. Change ONLY these (per the Part's specs table):

| Field | What to change |
|---|---|
| **CPU** | usually `1` (some VMs need 2 — check specs table) |
| **Memory** | type the MB value from the specs table (e.g. `1024`, `2048`, `4096`) |
| **Hard disk 1 → Capacity** | type the GB value (e.g. `10`, `20`, `60`, `80`) |
| **Hard disk 1 → Disk Provisioning** | `Thin Provisioned` |
| **Network Adapter 1** | drop-down → pick the port group from the specs table (e.g. `PG-Internet`) |
| **CD/DVD Drive 1** | drop-down → **`Datastore ISO file`** → browse to the ISO from the specs table → ✅ tick **`Connect at power on`** |

> 💡 **Multi-NIC VMs (like pfSense — 4 NICs):** in this screen, click **`Add network adapter`** at the top until you have the right count, then set each one to the correct port group in the order specified in the specs table. **Order matters for pfSense** — first NIC becomes WAN.

Click **Next**.

## Screen 5 — Ready to complete

- Review the summary. Confirm Name, Guest OS, Memory, Disk size, Network adapter, CD/DVD ISO.
- Click **Finish**.

## After Finish

1. The new VM appears in the **Virtual Machines** list.
2. Select it → click **Power on** at the top.
3. Click **Console** → choose **Open browser console**.
4. The OS installer boots (different per OS — see the matching Appendix B sub-section).

## ⚠️ One reminder for after OS install

When the installer says *"reboot now"* or *"please remove installation medium"*:

1. **Don't close the console.**
2. Switch to the ESXi UI tab → select the VM → click **Edit**.
3. Find **CD/DVD Drive 1** → uncheck **`Connect at power on`** AND change the dropdown to **`Host device`** (or "no media").
4. Click **Save**.
5. Switch back to the console → press Enter to reboot.

Skip this step and the VM boots the installer ISO again on every restart.

---

# 📘 Appendix B.2 — CentOS Stream 9 Installer Walkthrough (Beginner Reference)

**Use this for any CentOS Stream 9 VM (ISP, LinSRV1).**

The CentOS installer is called **Anaconda**. It uses a **hub-and-spoke** design — one main "Installation Summary" screen with several configuration tiles. You click each tile, set it up, click **Done** to return to the hub, then click **Begin Installation** when all required tiles are green/blue.

## Phase 1 — Boot menu (5–10 sec countdown)

- Highlight **`Install CentOS Stream 9`** (default).
- Press **Enter**.
- Wait ~30 sec while the installer kernel boots (lots of text scrolls).

## Phase 2 — Welcome screen — choose language

- Left list: **English**.
- Right list: **English (United States)** (or your country's English variant).
- Click **Continue** (bottom right).

## Phase 3 — Installation Summary (the hub)

You'll see a screen titled **INSTALLATION SUMMARY** with these tiles:

```
LOCALIZATION                  SOFTWARE                  USER SETTINGS
  ⚪ Keyboard                   ⚪ Installation Source    🔴 Root Password
  ⚪ Language Support           ⚪ Software Selection     🔴 User Creation
  ⚪ Time & Date

SYSTEM
  🔴 Installation Destination
  ⚪ KDUMP
  🔴 Network & Host Name
  ⚪ Security Profile
```

Tiles with a **red ❗** are **required** before you can install. Click each red one in turn.

> 💡 You don't have to configure tiles that are already green/blue. The defaults work for most.

### Tile: **Installation Destination** (required)

- Click the tile.
- You should see your 1× virtual disk (e.g., `sda 10 GiB` or `sda 20 GiB`).
- Make sure it's selected (black checkmark in the top-right corner of the disk icon).
- Storage Configuration: leave **Automatic** ticked (default).
- Click **Done** (top-left).
- Back to hub. Tile turns blue.

### Tile: **Network & Host Name** (required)

- Click the tile.
- You see your one virtual NIC (e.g., `Ethernet (ens160)` or `ens34` or `ens33`).
- **Toggle the switch in the top-right from OFF to ON** to bring the interface up. (Don't worry if no IP shows — we'll set static IP after install.)
- 📝 **Write down the interface name** (e.g., `ens160`) — you'll need it later for the `nmcli` commands in Step 1.2.
- At the bottom: **Host Name** field — type the hostname from the specs table (e.g., `isp.local` for ISP, `linsrv1.manila.com` for LinSRV1).
- Click **Apply** next to the hostname.
- Click **Done** (top-left).

### Tile: **Root Password** (required)

- Click the tile.
- **Root Password:** `P@ssw0rd`
- **Confirm:** `P@ssw0rd`
- ✅ **Tick `Allow root SSH login with password`** (very important — we'll SSH in as root from PC1 to run the Step 1.2 commands).
- Click **Done** (top-left). If it warns "weak password" → click **Done** again to confirm.

### Tile: **User Creation** (required)

- Click the tile.
- **Full name:** `competitor`
- **User name:** `competitor` (auto-fills)
- **Password:** `P@ssw0rd`
- **Confirm password:** `P@ssw0rd`
- ✅ Tick `Make this user administrator` (so they can sudo).
- Click **Done** (top-left). Click Done again if "weak password" warning.

### Tile: **Software Selection** (recommended — change default)

- Default is `Server with GUI` — that's bigger than we need.
- Click the tile.
- Left column: pick **`Minimal Install`** (smaller, faster, command-line only — perfect for ISP/LinSRV1).
- Right column (Add-ons): leave empty for ISP. For LinSRV1 you can optionally tick `Standard`.
- Click **Done** (top-left).

> 💡 Why Minimal? Less RAM use, smaller disk, faster install (~5 min instead of ~15). For our headless server VMs, no GUI is needed.

### Tile: **Time & Date** (optional)

- If you want to be precise: click the tile → pick `Asia/Manila` on the world map → Done.
- Otherwise leave default.

### Other tiles

- **Keyboard / Language Support** — leave English (US) defaults.
- **KDUMP** — leave enabled, Done.
- **Security Profile** — leave at "No profile selected", Done.

## Phase 4 — Begin Installation

- All red tiles should now be blue/green.
- Click **`Begin Installation`** (bottom right, big blue button).
- Installation progress bar appears.

## Phase 5 — Wait for install

- Takes ~5–10 min for Minimal Install.
- The progress shows: `Setting up the disk` → `Installing packages` → `Configuring boot loader`.

## Phase 6 — Reboot

- When done, button at bottom-right changes to **`Reboot System`**.

> ⚠️ **EJECT THE ISO before clicking Reboot:**
> 1. Don't close the console.
> 2. Switch to the ESXi UI tab.
> 3. Select the VM → click **Edit**.
> 4. **CD/DVD Drive 1** → uncheck "Connect at power on" → dropdown to **`Host device`**.
> 5. Click **Save**.
> 6. Switch back to console → click **`Reboot System`**.

## Phase 7 — First boot — login

After reboot you see a black text login prompt:
```
CentOS Stream 9
Kernel 5.14.x.x86_64 on an x86_64

isp login: _
```

- Type `root` → Enter.
- Password: `P@ssw0rd` → Enter (won't show as you type — that's normal).
- You see a shell prompt: `[root@isp ~]#`.

✅ **You're done with the installer.** Continue with the matching Part's Step 1.2 (or 6.2 for LinSRV1) commands.

## Common CentOS installer issues

| Problem | Fix |
|---|---|
| Mouse cursor doesn't appear or won't click | Click inside the VM console window first. Press **Ctrl+Alt** to release. |
| Tiles never go from red to blue | Some tiles need both fields filled (e.g., Root Password needs the SSH login checkbox too). Re-open the tile. |
| "No disks selected" warning | In Installation Destination, click the disk icon so a black checkmark appears in its top-right corner. |
| Begin Installation button stays grey | One required tile is still red. Hover the cursor over each red tile to see what's missing. |
| Install hangs at "Setting up the disk" >5 min | Give it more time — first format on a 20 GB disk takes a couple of minutes. |
| After reboot, boots installer again | The ISO wasn't ejected. Edit VM → CD/DVD Drive 1 → uncheck Connect at power on → reset VM. |

---

# 📘 Appendix B.5 — pfSense 2.7.2 Installer Walkthrough (Beginner Reference)

**Use this for the pfSense VM (Part 2).**

The pfSense installer is **bsdinstall** (FreeBSD-based). It uses **text/console mode only** — no mouse. Navigate with **Tab**, **arrow keys**, **Space** (toggle), **Enter** (confirm).

> The screens have a blue background with grey/white dialog boxes — old-school but very stable.

## Phase 1 — Boot menu (10-sec countdown)

- A pfSense splash with a 10-second timer.
- Just **wait** for it to auto-boot, OR press **Enter** on `1. Boot Multi User`.
- Wait ~30 sec while FreeBSD kernel loads (lots of text scrolls).

## Phase 2 — Copyright notice

- Read or skip.
- Press **Enter** (or Tab to **[ Accept ]** → Enter).

## Phase 3 — Welcome dialog

You see 3 options:
- **`Install`** ← ✅ pick this
- `Rescue Shell`
- `Recover config.xml`

Press **Enter** on `Install`.

## Phase 4 — Keymap selection

- Default: `>>> Continue with default keymap`.
- Press **Enter** to accept (US keyboard).

## Phase 5 — Partitioning

Choose how to lay out the 20 GB virtual disk. Options:
- **`Auto (UFS) BIOS`** ← ✅ simplest for our practice rig
- `Auto (UFS) UEFI`
- `Auto (ZFS)` (more features, more RAM)
- `Manual`
- `Shell`

Highlight **`Auto (UFS) BIOS`** → press **Enter**.

> 💡 **For ESXi 8 + UEFI BIOS in our specs table:** if "Auto (UFS) BIOS" fails to boot after install, retry with **`Auto (UFS) UEFI`** instead.

## Phase 6 — Install begins

- File copy log scrolls. Takes 3–5 min.
- No interaction needed.

## Phase 7 — Manual configuration prompt

After install, dialog asks: *"Would you like to manually edit any configuration files?"*
- Tab to **`No`** → press **Enter**.

## Phase 8 — Complete

Final dialog: *"Installation completed. Would you like to reboot now?"*
- ✅ **First eject the ISO** before rebooting:
  1. Don't close the console.
  2. Switch to ESXi UI → pfSense VM → **Edit**.
  3. **CD/DVD Drive 1** → uncheck "Connect at power on" → dropdown to **`Host device`**.
  4. Click **Save**.
  5. Switch back to console.
- Tab to **`Reboot`** → **Enter**.

## Phase 9 — First boot — interface assignment

After reboot, the pfSense console boots. After ~30 sec you see this prompt:

```
Should VLANs be set up now [y|n]?
```

Type `n` and press **Enter** (we don't use 802.1Q VLANs in our setup; ESXi handles network separation via port groups).

Next:
```
Enter the WAN interface name or 'a' for auto-detection
(vmx0 vmx1 vmx2 vmx3 or a):
```

You see your 4 NICs listed (`vmx0`, `vmx1`, `vmx2`, `vmx3` — these are FreeBSD's names for the VMware vmxnet3 adapters in the order you added them).

> 💡 **The order matters.** Whichever NIC was added first in ESXi (Network Adapter 1) becomes `vmx0`. That's why our specs table puts **PG-Internet first** — `vmx0` will be WAN.

Type the interface mappings as the installer asks:

| When it asks for... | Type | What this maps to |
|---|---|---|
| WAN interface name | `vmx0` | PG-Internet (gets DHCP from ISP) |
| LAN interface name | `vmx1` | PG-LAN (172.16.100.0/24) |
| Optional 1 (OPT1) interface name | `vmx2` | PG-DMZ (192.168.1.0/24) |
| Optional 2 (OPT2) interface name | `vmx3` | PG-Servers (192.168.2.0/24) |
| Optional 3 (just press Enter to skip) | (empty) | (no more interfaces) |

Final confirmation:
```
Do you want to proceed [y|n]?
```

Type `y` → Enter.

pfSense applies the assignments. After 5 sec, it tries DHCP on WAN (will succeed since ISP is running and PG-Internet has DHCP) and shows the main menu.

## Phase 10 — Main console menu

You see the pfSense console main menu:
```
*** Welcome to pfSense (amd64) on pfSense ***

  WAN (wan)        -> vmx0 -> v4/DHCP4: 10.0.0.x/24
  LAN (lan)        -> vmx1 -> v4: 192.168.1.1/24    (default; we'll change)
  OPT1 (opt1)      -> vmx2 -> (no IP)
  OPT2 (opt2)      -> vmx3 -> (no IP)

 0) Logout                              9) pfTop
 1) Assign Interfaces                  10) Filter Logs
 2) Set interface(s) IP address        11) Restart webConfigurator
 3) Reset webConfigurator password     12) PHP shell + pfSense tools
 4) Reset to factory defaults          13) Update from console
 5) Reboot system                      14) Disable Secure Shell (sshd)
 6) Halt system                        15) Restore recent configuration
 7) Ping host                          16) Restart PHP-FPM
 8) Shell

Enter an option:
```

You're now in pfSense's "command center" console. Continue with **Step 2.2** and **2.3** in Part 2 above.

## Common pfSense installer issues

| Problem | Fix |
|---|---|
| Boot menu doesn't appear / black screen | Wait 30 sec — early boot is silent. If still black after 60 sec, reset VM. |
| "vmx0" / "vmx1" not listed when assigning interfaces | The 4 NICs weren't all added in ESXi. Power off → Edit → check Network adapters 1–4 exist. |
| Wrong NIC mapping (e.g. WAN got assigned to PG-Servers) | At main menu, pick **option 1 (Assign Interfaces)** to redo. |
| Boots from ISO again after reboot | ISO not ejected. Edit VM → CD/DVD → uncheck Connect at power on → reset. |
| "DHCP failed on WAN" warning | Normal if ISP VM isn't powered on yet. Boot ISP first, then reset pfSense. |
