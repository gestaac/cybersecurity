# 20 — Day 2 (MA2) — Firewall (pfSense + OpenVPN + Snort)

> File numbered with `Day1_MA2` historically; the actual competition gives MA2 a full Day 2.

**Target time:** 75 min.
**Owner:** Person A.
**Login to pfSense:** browse from **Client1** to `https://172.16.100.254` → user `admin` / password `pfsense` (default).

> Marks at stake (Crit A2):
> rows 48–60 → **K total ≈ 4.95**, with judgment "FW best practice" worth +1.4

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

## Step 5 — OpenVPN
**Why:** VPN Users group must dial in from Client3 with cert from WINSRV3.

### 5.1 Get the CA + server cert from WINSRV3
On WINSRV3:
1. Open `certlm.msc` → *Personal → Certificates* → find Subordinate CA cert → export with private key (PFX) → save as `winsrv3-ca.pfx` password `P@ssw0rd`.
2. Issue a new cert template "OpenVPN-Server" with EKU = "Server Authentication" → request via `certreq` → export `openvpn-server.crt` + `.key`.

### 5.2 Import in pfSense
- *System → Cert Manager → CAs → Add* → paste WINSRV3 CA chain.
- *Certificates → Add* → paste server cert + key.

### 5.3 Configure OpenVPN server
**Where:** *VPN → OpenVPN → Servers → Add*
- Server mode: `Remote Access (User Auth)` (so it auths against AD)
- Backend for authentication: create a LDAP server first under *System → User Manager → Authentication Servers* → LDAP to WINSRV1 (`ldap://192.168.2.10`, baseDN `DC=manila,DC=com`, group filter `memberOf=CN=VPNGroup,...`) — **group name is exactly `VPNGroup`** (per MA2 PDF page 10), member `VPNUser` with password `P@ssw0rd`
- Protocol: UDP/IPv4
- Interface: WAN
- Local port: 1194
- Server cert: openvpn-server (the one you imported)
- DH params: 2048
- Tunnel network: `10.8.0.0/24`
- Local network: `172.16.100.0/24, 192.168.2.0/24` (so VPN users reach LAN + Servers)
- Topology: subnet
- Save.

### 5.4 Add WAN rule for OpenVPN
*Firewall → Rules → WAN* → Add: Pass UDP any → WAN address port 1194.

### 5.5 Export client config
- Install the **OpenVPN Client Export Utility** package: *System → Package Manager → Available Packages → openvpn-client-export → Install*.
- *VPN → OpenVPN → Client Export* → choose your server → download the **Inline Configurations: Most Clients** `.ovpn` file → keep on USB for Client3.

**Marks:** [Crit A2 D56 K=0.7] OpenVPN installed; [Crit A2 D57 K=0.25] cert from CA, not self-signed.

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
