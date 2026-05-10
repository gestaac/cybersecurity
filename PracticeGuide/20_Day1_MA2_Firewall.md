# 20 — Day 2 (MA2) — Firewall (pfSense + OpenVPN + Snort)

> File numbered with `Day1_MA2` historically; the actual competition gives MA2 a full Day 2.

**Target time:** 75 min.
**Owner:** Person A.
**Login to pfSense:** browse from **Client1** to `https://172.16.100.254` → user `admin` / password `pfsense` (default).

> Marks at stake (Crit A2):
> rows 48–60 → **K total ≈ 4.95**, with judgment "FW best practice" worth +1.4

---

## 📖 Beginner orientation — the pfSense WebUI

pfSense is configured entirely through a web browser interface (called the **WebConfigurator** or **WebUI**). Think of it like a router admin page on steroids.

### How to access pfSense

1. Power on **Client1** in VMware (it's on the LAN side, can reach pfSense)
2. Login as any domain user (`MANILA\M001` / `P@ssw0rd` — but local admin works too for first-time setup)
3. Open **Chrome** (taskbar icon)
4. URL bar → type: `https://172.16.100.254` → Enter
5. Browser warning "Your connection is not private" → click **Advanced** → **Proceed to 172.16.100.254 (unsafe)** (this is normal — pfSense uses self-signed cert until you replace it)
6. Login screen:
   - Username: `admin`
   - Password: `pfsense` (default — we'll change this in Step 1)
7. Click **SIGN IN**

You're now in the pfSense **Dashboard**.

### Top menu bar — what each section does

| Menu | What's inside | Used in step |
|---|---|---|
| **System** | User Manager, Cert Manager, Package Manager, Backup | Step 1, 5, 6 |
| **Interfaces** | LAN, WAN, DMZ, Servers — interface IPs + settings | (already configured) |
| **Firewall** | Rules, NAT, Aliases, Schedules | Steps 3, 4 |
| **Services** | DHCP Server, Snort, OpenVPN, DNS Resolver | Steps 2, 5, 6 |
| **VPN** | OpenVPN client/server config | Step 5 |
| **Status** | Dashboard, System logs, Services status | Verification |
| **Diagnostics** | Backup/Restore, Tools, Logs | Step 8 |

### Common actions you'll repeat

- **Save** any change → click the **Save** button at the bottom of the form
- **Apply changes** → after saving, a **green Apply Changes** banner appears at top → click it
- **Add a new entry** to a list → look for **+ Add** button (usually top-right of a table)
- **Edit existing entry** → click the pencil icon ✏️ next to a row
- **Delete entry** → click the trash icon 🗑️ next to a row

> 🧠 **MEMORIZE:** every change in pfSense takes 2 clicks — first **Save**, then **Apply Changes** banner. If you only Save without Apply, the change isn't live yet.

### The "Add firewall rule" walkthrough (you'll do this many times)

Every firewall rule in Step 4 follows this pattern:
1. Top menu → **Firewall** → **Rules**
2. Click the **interface tab** (LAN / DMZ / Servers / WAN)
3. Click **+ Add** (top-right) — there are TWO Add buttons (up arrow ⬆️ and down arrow ⬇️) — for now use the **down arrow** (adds at bottom)
4. Form opens. Fill in:
   - **Action:** `Pass` (allow) or `Block` (deny)
   - **Protocol:** TCP, UDP, or any
   - **Source:** Single host, network, or alias name
   - **Destination:** same options
   - **Destination Port Range:** From + To (or use an alias for multi-port)
   - **Description:** anything (free text — judges look at this for "FW best practice" judgment)
5. Click **Save**
6. Click the green **Apply Changes** banner

Repeat for each rule. **The order of rules MATTERS** — pfSense processes top-to-bottom, first match wins. Default rule at the bottom is always block.

---

## Step 1 — Change admin password
**Why:** default credentials are auto-marked as a fail.
**Where:** pfSense WebUI → *System → User Manager → admin → Edit*
**Action:** set password `P@ssw0rd` → Save.
**Marks:** [Crit A2 D48 K=0.2] admin password set.
**If it fails:** can't log in → reset via pfSense console option *3 — Reset webConfigurator password*.

---

## Step 2 — Set up DHCP for LAN clients
**Why:** the marking scheme says pfSense must hand out LAN DHCP, with WINSRV1 as DNS.
**Where:** *Services → DHCP Server → LAN tab*
- Enable DHCP server on LAN interface ☑
- Range: `172.16.100.50` – `172.16.100.150`
- DNS servers: `192.168.2.10` (WINSRV1)
- Gateway: `172.16.100.254`
- Domain name: `manila.com`
- Save → Apply.
**Verify from Client1:** open cmd → `ipconfig /release && ipconfig /renew && ipconfig /all` → see address from 172.16.100.50–150 with the right DNS/Gateway/Domain.
**Marks:** [Crit A2 D49 K=0.2] DHCP handled by firewall.

---

## Step 3 — Add interface aliases for clarity (best-practice points)
**Why:** named aliases earn the "FW best practice" judgment marks.
**Where:** *Firewall → Aliases → IP*
Create:
| Name | Type | Content |
|---|---|---|
| `LAN_NET` | Network | 172.16.100.0/24 |
| `DMZ_NET` | Network | 192.168.1.0/24 |
| `SERVERS_NET` | Network | 192.168.2.0/24 |
| `LINSRV1` | Host | 192.168.1.10 |
| `WINSRV1` | Host | 192.168.2.10 |
**Where (Ports):** *Aliases → Ports*
| Name | Content |
|---|---|
| `AD_PORTS` | 53 88 135 137 138 389 443 445 464 636 3268 3269 |
| `WEB_PORTS` | 80 443 |
| `SSH_HTTP_HTTPS_DNS` | 22 53 80 443 |
| `PKI_PORT` | 9389 |
**Marks:** indirect — boosts judgment row D60 (best practice).

---

## Step 4 — Firewall rules (the meat)
**Where:** *Firewall → Rules*
**Order matters.** Each tab below = one interface.

### 4.1 LAN tab
| # | Action | Proto | Source | Destination | Port | Description |
|---|---|---|---|---|---|---|
| 1 | Pass | TCP/UDP | LAN_NET | LINSRV1 | SSH_HTTP_HTTPS_DNS | LAN→DMZ to LinSRV1 |
| 2 | Pass | TCP/UDP | LAN_NET | SERVERS_NET | AD_PORTS | LAN→Servers AD |
| 3 | Pass | TCP | LAN_NET | SERVERS_NET | PKI_PORT | LAN→Servers PKI |
| 4 | Pass | any | LAN_NET | WAN net | * | LAN→Internet |
| 5 | Block | any | * | * | * | Default deny |
**Marks:** [Crit A2 D51/D52 K=0.6/0.3]

### 4.2 DMZ tab
| # | Action | Proto | Source | Destination | Port | Description |
|---|---|---|---|---|---|---|
| 1 | Pass | TCP/UDP | DMZ_NET | SERVERS_NET | AD_PORTS | DMZ→Servers AD (LinSRV1 needs to join) |
| 2 | Block | any | * | * | * | Default deny |
**Marks:** [Crit A2 D54 K=0.4]

### 4.3 SERVERS tab
| # | Action | Proto | Source | Destination | Port | Description |
|---|---|---|---|---|---|---|
| 1 | Pass | any | SERVERS_NET | WAN net | * | servers can reach Internet for updates |
| 2 | Block | any | * | * | * |  |
**Marks:** [Crit A2 D53 K=0.1]

### 4.4 WAN tab + NAT port forwards
**Where:** *Firewall → NAT → Port Forward*
- WAN any → 192.168.1.10 TCP/80 → "Public web HTTP"
- WAN any → 192.168.1.10 TCP/443 → "Public web HTTPS"
- WAN any → 192.168.1.10 UDP/53 → "Public DNS"
- WAN any → 192.168.1.10 TCP/53 → "Public DNS TCP"
After saving NAT, pfSense auto-creates the matching WAN rules. **Verify the WAN tab shows them.**
**Marks:** [Crit A2 D55 K=0.4]

### 4.5 Block starcity.com.ph from LAN (URL filter via DNS-resolver override)
**Easiest method:** *Services → DNS Resolver → Host Overrides → Add*
- Host: `www`, Domain: `starcity.com.ph`, IP: `127.0.0.1`
This sinkholes the domain so LAN clients get an unreachable IP.
Alternative (cleaner): use pfBlockerNG package — add `www.starcity.com.ph` to a custom blocklist.
**Marks:** indirectly satisfies the LAN→Internet test for "blocked starcity" in A6.

---

## Step 5 — OpenVPN (the most complex step — go slowly)

**Why:** Client3 (external) must dial into the LAN through OpenVPN, authenticated by AD users in the `VPNGroup` group, using a cert from WINSRV3 CA.

> 🧠 **OpenVPN setup has 3 parts working together:**
> 1. **CA + server cert** (from WINSRV3) → tells OpenVPN clients pfSense is legitimate
> 2. **LDAP backend** (queries WINSRV1) → who is allowed to login (VPNGroup members)
> 3. **OpenVPN server config** (on pfSense) → glues 1 and 2 together
>
> Do them in this order: CA → LDAP → Server.

### 5.1 Get the CA + server cert from WINSRV3

#### 5.1a — Create the AD group `VPNGroup` and user `VPNUser`

On WINSRV1 (do this first — LDAP backend needs the group to exist):

1. Open **Active Directory Users and Computers** (Win+R → `dsa.msc` → Enter)
2. Right-click on a suitable OU (e.g. `Manila`) → **New → Group**
   - Group name: `VPNGroup`
   - Group scope: Global
   - Group type: Security
   - OK
3. Right-click `VPNGroup` → **Properties** → **Members** tab → **Add**
4. Type `VPNUser` → click Check Names → if not found, create it first:
   - Right-click OU → New → User
   - First name: `VPN`, Last name: `User`, User logon name: `VPNUser`
   - Password: `P@ssw0rd`, untick "User must change password at next logon", tick "Password never expires"
   - Finish
5. Add VPNUser to VPNGroup

> 🧠 **MEMORIZE:** group name is **exactly `VPNGroup`** (per MA2 PDF page 10) — case-sensitive in some configs.

#### 5.1b — Export root CA chain from WINSRV3

On WINSRV3 (this is the issuing CA — `23_…` Step 2 should be done):

1. Win+R → `certlm.msc` → Enter (this is the **Local Machine** cert store)
2. Left tree → **Personal → Certificates**
3. Find your subordinate CA cert (issuer = Manila-Root-CA)
4. Right-click → **All Tasks → Export**
5. Wizard:
   - "Export private key" → Yes (private key needed for pfSense as a CA)
   - File format: `.pfx` (Personal Information Exchange)
   - Password: `P@ssw0rd` (encryption password for the PFX file)
   - Save as: `C:\winsrv3-ca.pfx`

#### 5.1c — Issue a server cert for OpenVPN

```powershell
# On WINSRV3 PowerShell as admin
certreq -enroll -machine -q "Web-Server-Manila"
# Or use IIS Manager → Server Certificates → Create Domain Certificate
```

Export the new cert + private key as PFX:
1. `certlm.msc` → Personal → Certificates
2. Find the new cert (Subject = WINSRV3.manila.com)
3. Right-click → All Tasks → Export → with private key → PFX → password `P@ssw0rd` → save as `C:\openvpn-server.pfx`

Copy both PFX files to a USB or network share where pfSense can grab them. Or just copy/paste the PEM contents into pfSense's web UI.

### 5.2 Import certs into pfSense

#### 5.2a — Import CA
1. pfSense WebUI → top menu **System → Cert Manager**
2. Top tab **CAs** → click **+ Add**
3. **Descriptive name:** `Manila-Sub-CA`
4. **Method:** Import an existing Certificate Authority
5. **Certificate data:** paste the WINSRV3 sub-CA cert (PEM format — extract from the PFX using openssl or Windows export wizard "Base-64 encoded X.509 .CER")
6. **Certificate Private Key:** paste the private key (also PEM)
7. **Save**

#### 5.2b — Import server cert
1. pfSense → System → Cert Manager → tab **Certificates**
2. **+ Add/Sign**
3. **Method:** Import an existing Certificate
4. **Descriptive name:** `openvpn-server`
5. Paste cert + private key
6. **Save**

> ⚠️ **If "Save" errors with "x509 verification failed":** the CA cert and server cert don't form a valid chain. Make sure the CA cert (5.2a) is imported FIRST and is the one that signed the server cert.

### 5.3 Configure LDAP backend (so VPN users can authenticate against AD)

1. pfSense → **System → User Manager**
2. Top tab **Authentication Servers** → click **+ Add**
3. Fill in:
   - **Descriptive name:** `manila-ldap`
   - **Type:** LDAP
   - **Hostname or IP address:** `192.168.2.10` (WINSRV1)
   - **Port value:** `389`
   - **Transport:** TCP - Standard
   - **Protocol version:** 3
   - **Server Timeout:** 25
   - **Search scope — Level:** Entire Subtree
   - **Search scope — Base DN:** `DC=manila,DC=com`
   - **Authentication containers:** `CN=Users,DC=manila,DC=com`
   - **Bind anonymous:** ☐ **uncheck**
   - **Bind credentials user DN:** `CN=Administrator,CN=Users,DC=manila,DC=com`
   - **Bind credentials password:** `P@ssw0rd`
   - **Initial Template:** Microsoft AD
   - **User naming attribute:** `samAccountName`
   - **Group naming attribute:** `cn`
   - **Group member attribute:** `memberOf`
4. Click **Save**

#### 5.3a — Test LDAP

1. **System → User Manager → Authentication Servers**
2. Top tab **Diagnostics → Authentication**
3. Authentication Server: `manila-ldap`
4. Username: `VPNUser`, Password: `P@ssw0rd`
5. Click **Test**

**✅ Success:** "User VPNUser authenticated successfully" + group membership shows `VPNGroup`.
**❌ Failure:** check Bind credentials, BaseDN, network connectivity (LAN→Servers AD ports rule must be in place from Step 4.1).

### 5.4 Configure OpenVPN server

1. pfSense → **VPN → OpenVPN**
2. Top tab **Servers** → **+ Add**
3. Fill in (long form — go slowly):

| Section | Field | Value |
|---|---|---|
| General | **Server mode** | `Remote Access ( User Auth )` |
| General | **Backend for authentication** | `manila-ldap` (selected from Step 5.3) |
| General | **Protocol** | UDP on IPv4 only |
| General | **Device mode** | tun - Layer 3 Tunnel Mode |
| General | **Interface** | WAN |
| General | **Local port** | `1194` |
| General | **Description** | `OpenVPN-Manila` |
| Crypto | **TLS Configuration** | ☑ Use a TLS Key |
| Crypto | **Generate TLS Key** | ☑ Automatically generate a TLS Key |
| Crypto | **Peer Certificate Authority** | `Manila-Sub-CA` (from Step 5.2a) |
| Crypto | **Server certificate** | `openvpn-server` (from Step 5.2b) |
| Crypto | **DH Parameters Length** | 2048 |
| Tunnel | **Tunnel Network** | `10.8.0.0/24` |
| Tunnel | **Local Network/s** | `172.16.100.0/24, 192.168.2.0/24, 192.168.1.0/24` |
| Tunnel | **Concurrent connections** | `10` |
| Tunnel | **Topology** | Subnet |
| Client Settings | **DNS Server 1** | `192.168.2.10` (WINSRV1) |
| Client Settings | **Force DNS cache update** | ☑ check |
| Advanced | **Custom options** | `push "redirect-gateway def1"` |

4. **Save** at the bottom

### 5.5 Allow OpenVPN traffic on WAN

1. **Firewall → Rules → WAN** tab
2. **+ Add (down arrow ⬇️)**
3. Action: `Pass`, Protocol: `UDP`, Source: any, Destination: `WAN address`, Destination port: `1194`, Description: `Allow OpenVPN`
4. **Save → Apply Changes**

### 5.6 Allow tunneled traffic on the OpenVPN interface

After step 5.4 saves, pfSense creates a new interface called **OpenVPN**. Need a rule on it to allow tunneled traffic to LAN/Servers:

1. **Firewall → Rules → OpenVPN** tab
2. **+ Add**
3. Action: `Pass`, Protocol: any, Source: `Network 10.8.0.0/24`, Destination: any, Description: `Allow VPN clients to LAN/Servers`
4. **Save → Apply Changes**

### 5.7 Install + use the Client Export package

1. **System → Package Manager → Available Packages**
2. Search box: `openvpn-client-export`
3. Click **Install** → Confirm
4. Wait ~1 min for install
5. **VPN → OpenVPN → Client Export** tab
6. Settings:
   - **Remote Access Server:** select `OpenVPN-Manila`
   - **Host Name Resolution:** Other → enter pfSense WAN IP (e.g. the public WAN IP from ISP)
   - **Verify Server CN:** Automatic
7. Scroll down to **OpenVPN Clients** section → click **Inline Configurations → Most Clients**
8. A `.ovpn` file downloads to Client1's Downloads folder

> 🧠 **Save the .ovpn file to a USB stick** — you'll need to copy it to Client3 for the A8.1 verification test.

**Marks:** [Crit A2 D56 K=0.7] OpenVPN installed; [Crit A2 D57 K=0.25] cert from CA (NOT self-signed).

---

## Step 6 — Snort (IDS)

### 6.1 Install Snort package
- *System → Package Manager → Available → snort → Install* (pre-downloaded per MA2 PDF page 10).

### 6.2 Global settings
- *Services → Snort → Global Settings* → enable "Install Snort VRT rules" if available; otherwise just Emerging Threats free rules.
- Update rules.

### 6.3 Add WAN interface
- *Snort → Snort Interfaces → Add*
- Interface: WAN
- Enabled
- Logging: log to system log
- Save.

### 6.4 Add the custom FIN scan rule
**Source: MA2 PDF page 10** — the rule is **FIN scan**, NOT XMAS scan (the older docx had XMAS — superseded).

- *Snort → WAN tab → Rules → Category: custom.rules*
- Add (paste exactly):
```
alert tcp any any <> $HOME_NET any (flags: F; msg: "Possible FIN scan"; sid: 100001;)
```

> The MA2 PDF rule has placeholder `a.b.c.d/24` — replace with `$HOME_NET` (Snort variable that auto-resolves to the WAN subnet you set in the Snort interface settings, e.g. `10.0.0.0/24`).
> Action is `alert` per the PDF (NOT `drop`).
> Flags `F` = FIN flag only.

### 6.5 Start Snort on WAN
- *Snort Interfaces → WAN → Start* (▶).
**Marks:** [Crit A2 D58 K=0.5] Snort installed; [Crit A2 D59 K=0.3] custom rule active.

---

## Step 7 — Best-practice cleanup (judgment marks)
**Where:** every interface tab.
- Remove the default "allow LAN to any" rule and replace with the explicit ones above.
- Confirm no `*` source on inbound WAN rules.
- Add **descriptions** to every rule (e.g. "LAN→DMZ web", "block default") — judges look for this.
- Reorder so the most-specific rules are at top, generic blocks at bottom.
**Marks:** [Crit A2 D60 K=1.0–1.4] FW best practice judgment.

---

## Step 8 — Save / backup config (so you don't lose your work)
*Diagnostics → Backup & Restore → Download configuration as XML* → save to USB.

---

## Smoke tests before moving on
From **Client1**:
- `nslookup www.manila.com` → 192.168.1.10
- `nslookup www.nationalmuseum.gov.ph` → 10.0.0.10 (via pfSense forwarder)
- `nslookup www.starcity.com.ph` → 127.0.0.1 (sinkholed)
- `curl -k https://www.manila.com` → returns the LinSRV1 page (after rules + LinSRV1 https config later)

From **Client3** (after Client3 receives DHCP from ISP):
- `ping 10.0.0.1` → ISP responds
- (later) connect via OpenVPN and re-test the above

---

## Mark map for this file

| Aspect | K | Step |
|---|---|---|
| A2 D48 admin password | 0.2 | 1 |
| A2 D49 DHCP from FW | 0.2 | 2 |
| A2 D51 LAN rules | 0.6 | 4.1 |
| A2 D52 LAN→DMZ ssh | 0.3 | 4.1 |
| A2 D53 Servers iface | 0.1 | 4.3 |
| A2 D54 DMZ iface | 0.4 | 4.2 |
| A2 D55 WAN/NAT | 0.4 | 4.4 |
| A2 D56 OpenVPN installed | 0.7 | 5 |
| A2 D57 OpenVPN cert from CA | 0.25 | 5.1–5.3 |
| A2 D58 Snort installed | 0.5 | 6.1–6.3 |
| A2 D59 Snort tracking | 0.3 | 6.4 |
| A2 D60 FW best practice (Judg) | 1.0 (max 1.4) | 7 |
| **Total** | **~4.95** | |

Next file: **`21_Day1_MA2_LinSRV1.md`**.
