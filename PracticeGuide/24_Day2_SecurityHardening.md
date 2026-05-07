# 24 — Day 2 Security Hardening (speculative — verify with chief)

**Time:** 6 hours.
**Owner:** Both teammates split as below.

> ⚠️ **Important context.** As of writing, the official MA-doc for Day 2 has **not been released**. The chief (Marlon) confirmed it's a "Security Hardening from scratch" module that names **Security Onion** and **OpenVPN** as components. The marking scheme calls this Criterion B = *"Cyber Security Incident Response, Digital Forensics, Application Security"* — 25 marks.
>
> This file's structure is the **best educated guess** based on those hints. **Treat every step as a hypothesis** — when the official doc lands, walk it through and update this file. Some sections may be moot, others may be missing.

---

## Likely day plan (4 phases, 6 hours)

| Phase | Time | Theme | Owner |
|---|---|---|---|
| 1 | 90 min | Deploy Security Onion + onboard log sources | Person A |
| 2 | 60 min | Stand up OpenVPN service (separate from MA2's) | Person B |
| 3 | 90 min | Incident Response — investigate provided alerts | Both |
| 4 | 90 min | Digital Forensics — analyse provided artefact | Both |
| 5 | 30 min | Application Security — harden a deployed web app | Person A |
| 6 | 30 min | Documentation + report | Both |

---

## Phase 1 — Security Onion deployment (Person A)

Pre-req: `07_Setup_SecurityOnion.md` complete; SO Eval VM running and reachable.

### Step 1.1 — Confirm SO is monitoring
**Where:** SOC web UI (`https://<so-ip>/`).
**Commands:**
```bash
sudo so-status
```
**Expected:** every component `OK`.
**Marks:** [Crit B placeholder] SOC platform deployed.

### Step 1.2 — Onboard a Linux host as a Wazuh agent (full procedure)
**Why:** Wazuh = HIDS = file-integrity + log shipping from servers.
**Where:** SSH to LinSRV1 from PC1 — `ssh C1@manila.com@192.168.1.10 -p 2022` (port 2022 from Day 1's hardening).

#### 1.2.1 — Get the agent installer
The simplest path is to copy the RPM you pre-staged on USB (per `01_…` Section 5.6) into LinSRV1, e.g. via WinSCP:
```bash
# On LinSRV1
ls /tmp/wazuh-agent-4.x.x-1.x86_64.rpm    # confirm it's there
```
Or if internet is available during build (per chief's note that Day 2 packages may be pre-staged, but practice has internet):
```bash
sudo rpm --import https://packages.wazuh.com/key/GPG-KEY-WAZUH
sudo dnf install -y https://packages.wazuh.com/4.x/yum/wazuh-agent-4.x.x-1.x86_64.rpm
```

#### 1.2.2 — Configure the manager IP in `/var/ossec/etc/ossec.conf`
**Note:** the SO box's IP. Substitute in the placeholder below.
```bash
sudo sed -i 's|<address>MANAGER_IP</address>|<address>192.168.1.50</address>|' \
    /var/ossec/etc/ossec.conf
```
Verify the change:
```bash
sudo grep -A1 "<server>" /var/ossec/etc/ossec.conf | head -5
# Should print:
#   <server>
#       <address>192.168.1.50</address>
```

#### 1.2.3 — Register the agent with the SO manager
On Security Onion:
```bash
sudo so-wazuh-agent-list                  # show currently registered agents
sudo /var/ossec/bin/manage_agents
# Pick (A)dd → name 'linsrv1' → IP 192.168.1.10
# Note the **agent ID** (e.g. 002) printed back
# Pick (E)xtract → enter agent ID → it prints a one-time KEY (long string)
# Save the KEY to clipboard or paste-buffer.
```

Back on LinSRV1, register with the key:
```bash
sudo /var/ossec/bin/manage_agents
# Pick (I)mport → paste the KEY → confirm
# Pick (Q)uit
```

#### 1.2.4 — Start + enable the agent
```bash
sudo systemctl enable --now wazuh-agent
sudo systemctl status wazuh-agent --no-pager | head -20
# Expected: 'active (running)' + connection to manager IP shown
```

#### 1.2.5 — Verify in SOC UI
1. Browse to SO web UI → log in → *Wazuh → Agents*.
2. Look for `linsrv1` → status: **Active** (green).
3. Click the agent → *Inventory* → see the host's OS/users/ports.

**Common failures:**
| Symptom | Fix |
|---|---|
| Agent shows "Disconnected" | Firewalld blocking outbound — `sudo firewall-cmd --add-port=1514/udp --add-port=1515/tcp --permanent && sudo firewall-cmd --reload` |
| "ERROR: Unable to connect to manager" | Wrong manager IP in ossec.conf, or SO firewall blocking — check `sudo so-allow` on SO |
| Agent never registers | Agent's name already in use — pick a different one |

**Marks:** [Crit B placeholder] Linux Wazuh agent onboarded.

### Step 1.3 — Onboard Windows (WINSRV1) as Wazuh agent
**Where:** WINSRV1 console as `MANILA\Administrator`. Open **PowerShell as Administrator**.

#### 1.3.1 — Get the MSI
Either copy `wazuh-agent-4.x.x-1.msi` from your USB into `C:\Temp\` via SMB, or download:
```powershell
$msi = "C:\Temp\wazuh-agent.msi"
Invoke-WebRequest -Uri "https://packages.wazuh.com/4.x/windows/wazuh-agent-4.x.x-1.msi" -OutFile $msi
```

#### 1.3.2 — Install with manager IP set inline
```powershell
$so = "192.168.1.50"
msiexec.exe /i C:\Temp\wazuh-agent.msi /q WAZUH_MANAGER="$so" WAZUH_REGISTRATION_SERVER="$so" WAZUH_AGENT_NAME="winsrv1"
```
Wait ~30 sec. The MSI runs `agent-auth.exe` automatically against the SO manager.

#### 1.3.3 — Start + verify
```powershell
Start-Service WazuhSvc
Get-Service WazuhSvc
# Expected: Status = Running
```
Tail the log for the registration line:
```powershell
Get-Content "C:\Program Files (x86)\ossec-agent\ossec.log" -Tail 30
# Look for: "Connected to enrollment service" and "Connection accepted"
```

#### 1.3.4 — Verify in SOC UI
SO web UI → Wazuh → Agents → see `winsrv1` Active.
Trigger a quick test: on WINSRV1, edit any file in `C:\Windows\System32\drivers\etc\hosts` → save → Wazuh should fire **rule 550** (file integrity changed). Check SOC UI → Alerts within 1 min.

**Marks:** [Crit B placeholder] Windows Wazuh agent onboarded.

### Step 1.4 — Forward pfSense logs (syslog → SO)
**Where:** pfSense web UI.
**Action:** *Status → System Logs → Settings → Remote Logging Options*:
- Enable
- Source IP: WAN
- Remote syslog server: `<so-ip>:514`
- Remote Syslog Contents: ☑ Firewall events, ☑ DHCP, ☑ System events
- Save.
**Verify:** SOC UI → *Hunt* → search `dataset:"firewall"` → see pfSense entries.
**Marks:** [Crit B placeholder] FW logs ingested.

---

## Phase 2 — OpenVPN service (Person B)

This is an OpenVPN install **on a Linux server**, separate from the pfSense one in MA2. Likely the chief will provide a fresh CentOS VM. For practice we build it ourselves.

### Step 2.0 — Create the OpenVPN service VM (practice prep)

#### Option A — On ESXi
1. ESXi web UI → Virtual Machines → **Create / Register VM** → *Create a new virtual machine*.
2. Name: `openvpn-srv`, Compatibility: ESXi 8.0, Guest family: Linux, Version: CentOS 9 (64-bit).
3. Datastore: `datastore1`.
4. Customize:
   - CPU: 1 core
   - RAM: 2 GB
   - Disk: 20 GB, **Thin Provision**
   - Network adapter: choose `PG-Servers` (so it can talk to AD if needed) or whichever subnet the chief specifies; for our practice put it on `PG-Servers` 192.168.2.x with a static `.20`.
   - CD/DVD: mount the CentOS Stream 9 ISO from your datastore (upload via *Datastores → datastore1 → Upload*)
5. Power on → finish CentOS install (Server with GUI is fine; Minimal is leaner).
6. Set hostname `openvpn-srv` and static IP:
   ```bash
   sudo hostnamectl set-hostname openvpn-srv
   sudo nmcli con mod ens160 ipv4.addresses 192.168.2.20/24 ipv4.gateway 192.168.2.254 ipv4.dns 192.168.2.10 ipv4.method manual
   sudo nmcli con up ens160
   ```
7. Snapshot the VM as `openvpn-base`.

#### Option B — On VMware Workstation (single-PC practice)
Same specs (1 vCPU, 2 GB RAM, 20 GB thin disk) but:
- Network: VMnet13 (PG-Servers equivalent)
- Static IP: 192.168.2.20 (gateway 192.168.2.254 if pfSense is also on VMnet13, or 192.168.1.1 if pointing back at TP-Link router during Day 2 standalone practice).
Snapshot.

> **Quickest path:** clone your Day 1 LinSRV1 snapshot, change hostname + IP, and you have an OpenVPN base in 2 min.

### Step 2.1 — Install OpenVPN + Easy-RSA
**Pre-req:** internet to the VM (during practice you have it; on competition day organisers pre-stage the RPMs).
**Where:** SSH to the OpenVPN VM as root: `ssh root@192.168.2.20`.

```bash
sudo dnf install -y epel-release
sudo dnf install -y openvpn easy-rsa
```

**If no internet (offline competition mode):** copy the staged RPMs to the VM, then:
```bash
sudo dnf localinstall -y /tmp/wheels/openvpn*.rpm /tmp/wheels/easy-rsa*.rpm
```

### Step 2.2 — Generate the PKI
```bash
mkdir -p /etc/openvpn/easy-rsa
cp -r /usr/share/easy-rsa/3/* /etc/openvpn/easy-rsa/
cd /etc/openvpn/easy-rsa
./easyrsa init-pki
./easyrsa build-ca nopass             # set CN: OpenVPN-CA
./easyrsa gen-req server nopass       # CN: server
./easyrsa sign-req server server
./easyrsa gen-dh
openvpn --genkey --secret pki/ta.key
```

### Step 2.3 — Server config
`/etc/openvpn/server/server.conf`:
```
port 1194
proto udp
dev tun
ca   /etc/openvpn/easy-rsa/pki/ca.crt
cert /etc/openvpn/easy-rsa/pki/issued/server.crt
key  /etc/openvpn/easy-rsa/pki/private/server.key
dh   /etc/openvpn/easy-rsa/pki/dh.pem
tls-auth /etc/openvpn/easy-rsa/pki/ta.key 0
server 10.9.0.0 255.255.255.0
push "redirect-gateway def1"
push "dhcp-option DNS 192.168.2.10"
keepalive 10 120
cipher AES-256-CBC
auth SHA256
user nobody
group nobody
persist-key
persist-tun
status /var/log/openvpn-status.log
verb 3
```

### Step 2.4 — Enable + firewall + sysctl
```bash
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
sudo firewall-cmd --add-service=openvpn --permanent
sudo firewall-cmd --add-masquerade --permanent
sudo firewall-cmd --reload
sudo systemctl enable --now openvpn-server@server
sudo systemctl status openvpn-server@server
```
**Expected:** `active (running)` listening on UDP/1194.
**Marks:** [Crit B placeholder] OpenVPN service up.

### Step 2.5 — Issue a client cert + .ovpn file
```bash
cd /etc/openvpn/easy-rsa
./easyrsa gen-req client1 nopass
./easyrsa sign-req client client1
```
Build `client1.ovpn` (template in `/usr/share/doc/openvpn/sample/sample-config-files/client.conf`):
```
client
dev tun
proto udp
remote <openvpn-server-ip> 1194
resolv-retry infinite
nobind
persist-key
persist-tun
remote-cert-tls server
cipher AES-256-CBC
auth SHA256
key-direction 1
verb 3
<ca>
... ca.crt contents ...
</ca>
<cert>
... client1.crt contents ...
</cert>
<key>
... client1.key contents ...
</key>
<tls-auth>
... ta.key contents ...
</tls-auth>
```

### Step 2.6 — Test connection
From Client3 (or any external client):
- Install OpenVPN Connect.
- Import `client1.ovpn`.
- Connect. Tunnel up → ping the server's internal IP.
**Marks:** [Crit B placeholder] VPN client connection works.

---

## Phase 3 — Incident Response (both teammates)

Speculative — but standard in any SOC module.

The chief likely pre-loads the SO instance with a few alerts (or you trigger them mid-day with a fake attack). For each alert:

### Step 3.1 — Investigate methodology
**Where:** SOC UI → *Alerts*.
1. Click an alert → **right-click** the source IP → *Hunt → src.ip*.
2. **Right-click** the destination IP → *Hunt → dst.ip*.
3. Open Stenographer → grab the PCAP for the time window.
4. Open in Wireshark → look at the suspicious flow.
5. Document:
   - What happened (1 sentence).
   - When (UTC timestamp).
   - Source & destination.
   - Indicator of Compromise (IOC) — IP, domain, file hash.
   - Severity (Low/Med/High).
   - Recommended response action.
**Marks:** [Crit B placeholder] IR investigation quality.

### Step 3.2 — Create a Case
**Where:** SOC UI → *Cases → New Case*.
- Title: e.g. *"SQL Injection attempt against www.manila.com"*
- Severity: High.
- Add the alert as evidence.
- Tag observables.
- Comment: a 1-paragraph summary + recommendation.

### Step 3.3 — Tune a noisy rule
If a rule fires constantly with false positives:
```bash
sudo so-rule disable <sid>
# OR threshold it:
sudo nano /opt/so/saltstack/local/pillar/global.sls
# add a 'thresholding' section
sudo so-restart
```
**Marks:** [Crit B placeholder] rule tuning.

---

## Phase 4 — Digital Forensics (both)

Speculative — but Criterion B title explicitly includes "Digital Forensics."

### Likely artefact types (prepare for any)

| Artefact | Likely tool | Marks placeholder |
|---|---|---|
| **PCAP file** | Wireshark, NetworkMiner, `so-import-pcap` | `[Crit B placeholder] PCAP analysis` |
| **Memory dump** (.raw / .lime / .vmem) | Volatility 3 (already installed in Kali) | `[Crit B placeholder] Memory analysis` |
| **Disk image** (.dd / .img / .e01) | Autopsy / sleuthkit | `[Crit B placeholder] Disk forensics` |
| **Compromised log file** | grep / awk / Kibana ingest | `[Crit B placeholder] Log triage` |
| **Suspicious binary** | strings, file, Ghidra | `[Crit B placeholder] RE basics` |

### Step 4.1 — PCAP forensics workflow
```bash
# 1. Replay through SO so you get instant Suricata + Zeek output
sudo so-import-pcap /tmp/evidence.pcap
# 2. Wait, then go to SOC UI → Hunt → see the new logs

# Or analyse directly in Wireshark
wireshark /tmp/evidence.pcap
# - File → Export Objects → HTTP/SMB/FTP — extract files
# - Statistics → Conversations → top talkers
# - tshark CLI: tshark -r evidence.pcap -Y "http.request" -T fields -e http.host -e http.request.uri
```

### Step 4.2 — Memory forensics workflow
```bash
vol -f mem.raw windows.info
vol -f mem.raw windows.pstree
vol -f mem.raw windows.cmdline
vol -f mem.raw windows.netscan
vol -f mem.raw windows.malfind
vol -f mem.raw windows.hashdump
```
Cross-reference timestamps with the alert in SOC UI.

### Step 4.3 — Disk image workflow
```bash
file disk.img
mmls disk.img
sudo mount -o loop,ro,offset=$((512*PARTSTART)) disk.img /mnt/forensic
# Then:
ls -la /mnt/forensic/home/*/.bash_history
cat /mnt/forensic/var/log/auth.log
find /mnt/forensic -newer reference_date -type f
```

### Step 4.4 — Document findings
Each finding gets a short paragraph in the report:
```
Finding: Attacker (10.0.0.42) attempted SQL injection against www.manila.com
between 14:32 and 14:38 UTC. The attack used time-based blind SQLi via the
search field. Suricata fired SID 2027235 ("SQL Injection time-based"). No
data was exfiltrated based on Stenographer review of the response sizes.

Severity: High
Recommendation: Patch the search endpoint; review WAF rules; rotate any
credentials accessible from the manila DB.
```
**Marks:** [Crit B placeholder] forensic report quality.

---

## Phase 5 — Application Security (Person A)

The "Application Security" piece of Criterion B's title. Speculative — but the most likely deliverable is to **harden a provided web application**.

### Likely deliverable shape
You'll receive a vulnerable web app (DVWA, OWASP Juice Shop, or a custom one) and asked to deploy it **securely**. Common hardening checklist:

| Layer | Hardening action |
|---|---|
| TLS | Enforce HTTPS only, HSTS header, TLS 1.2/1.3 only, strong ciphers |
| Headers | Add CSP, X-Frame-Options DENY, X-Content-Type-Options nosniff, Referrer-Policy strict-origin-when-cross-origin |
| Input | Parameterised queries / ORM, input length limits, allow-list for special chars |
| Session | HttpOnly + Secure + SameSite=Lax cookies, short timeout, regenerate on auth |
| Auth | Strong password policy, lockout, MFA if available, brute-force throttle |
| File upload | Allow-list extensions, AV scan (ClamAV), random filename, no exec from upload dir |
| Logging | Log all auth attempts to syslog → forward to Security Onion |
| WAF | ModSecurity with OWASP CRS in front of the web server |

### Quick example: ModSecurity in front of httpd
```bash
sudo dnf install -y mod_security mod_security_crs
sudo systemctl restart httpd
# Verify:
curl -i "http://localhost/?q=<script>alert(1)</script>"
# Should return 403 if CRS is blocking
```

**Marks:** [Crit B placeholder] AppSec hardening.

---

## Phase 6 — Documentation

Same pattern as Day 1: combine all reports into one PDF on the desktop:
```
Filename: PHL_Team1_Day2_Report.pdf
Sections:
  1. Security Onion deployment summary
  2. OpenVPN configuration & test result
  3. Incident Response — case summaries (table of alerts investigated)
  4. Forensic findings (one section per artefact)
  5. AppSec hardening — before/after diff
```

---

## Marking-scheme placeholder map

Until the chief releases the official Criterion B aspect rows, track every step you complete in this template:

| Phase | Step | Likely K-value | Solved? |
|---|---|---|---|
| 1 | SO deployed + monitoring | TBD | ☐ |
| 1 | Wazuh on Linux + Windows agents | TBD | ☐ |
| 1 | pfSense logs forwarded | TBD | ☐ |
| 2 | OpenVPN service running | TBD | ☐ |
| 2 | Client cert issued + connects | TBD | ☐ |
| 3 | Alerts investigated + cases created | TBD | ☐ |
| 3 | At least one rule tuned | TBD | ☐ |
| 4 | Forensic finding documented | TBD | ☐ |
| 5 | Web app hardened with HTTPS + headers + WAF | TBD | ☐ |
| 6 | Report PDF on desktop with country code | TBD | ☐ |

When the official doc lands → replace **TBD** with the actual K-values, and add/remove rows.

---

## What to ask Marlon (concrete questions)

To make this file accurate, ping the chief with:

1. *"Sir, sa Day 2 — meron na po bang official MA-doc na ire-release? Like ng MA1 + MA2?"*
2. *"Aside from Security Onion + OpenVPN, ano pa pong major tools sa Day 2? May Wazuh standalone? May SIEM dashboard build? May forensic artefact?"*
3. *"Ang AppSec part po, secure-deploy ba ng ina-provide na web app, or harden lang ng existing Day 1 services?"*
4. *"Day 2 build niyo po sa same VM topology ng Day 1 (manila.com), or fresh new VM set?"*

Update this file with answers — that closes our biggest unknown.

Next file: **`50_Day3_CTF_Playbook.md`** (Juice Shop — was 40_…).
