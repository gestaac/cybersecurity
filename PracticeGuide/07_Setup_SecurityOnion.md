# 07 — Setup Security Onion 2 (Day 2 SOC platform)

The chief confirmed Day 2 = "Security Hardening from scratch" with **Security Onion** + **OpenVPN** as named tools. This file gets Security Onion installed and verified on your practice rig so you know it cold before competition.

> Security Onion is the heaviest VM in this guide — **16 GB RAM minimum, 200 GB disk recommended**. Don't try to install it alongside MA2's full ESXi load; either give it its own host or shut down most MA2 VMs while practising.

> Time: ~3 hours first install (mostly waiting for Setup Wizard).

---

## Part A — What Security Onion is

A free, open-source Linux distribution built for **Network Security Monitoring (NSM)** + **SIEM** + **threat hunting**. One install bundles:

| Component | Purpose |
|---|---|
| **Suricata** | Network IDS — generates alerts from traffic (signature-based) |
| **Zeek** (formerly Bro) | Network metadata logger — records every connection, DNS, HTTP, SSL, etc. |
| **Stenographer** | Full-packet capture (rotating PCAP buffer) |
| **Wazuh** | Host-based IDS / endpoint agent (file integrity, log analysis, syscall monitoring) |
| **Elastic Stack** (Elasticsearch + Logstash + Kibana) | Log storage, indexing, dashboards |
| **Strelka** | File extraction + malware scanning |
| **Playbook** | Detection-as-code — alert playbooks |
| **SOC** (web UI) | Single pane of glass for analysts |

**Source:** `https://securityonionsolutions.com/` — Documentation: `https://docs.securityonion.net/`

Current major version (as of writing): **Security Onion 2.4.x**. Always grab the latest 2.4 ISO before competition.

---

## Part B — Hardware sizing (Eval install)

Per official docs:

| Mode | Min RAM | Min Disk | Min CPU | Use |
|---|---|---|---|---|
| Import only (forensics on PCAPs) | 4 GB | 200 GB | 2 cores | Just analyse files; no live monitoring |
| **Eval (recommended for practice)** | **12–16 GB** | **200 GB** | **4 cores** | All-in-one box; live monitoring works |
| Standalone (production) | 16+ GB | 1 TB+ | 4+ cores | Single-box deployment |
| Distributed (multi-host) | varies | varies | varies | Manager + sensors |

For competition + practice: **Eval mode**. It runs Manager + Search + Sensor on one VM.

---

## Part C — Download

1. Visit `https://securityonionsolutions.com/software`.
2. Click *Download Security Onion 2.x ISO* → land on the GitHub releases page (the ISO is hosted on GitHub).
3. Download `securityonion-2.4.x-YYYYMMDD.iso` (≈ 9 GB).
4. Verify SHA-256 against the hash on the release page:
   ```bash
   sha256sum securityonion-*.iso
   ```
5. Save to `D:\ISO\` and copy onto your USB stick for offline competition use.

---

## Part D — Create the VM (in VMware Workstation or ESXi)

### D.1 VM specs
- Guest OS: Linux → **Other Linux 5.x kernel 64-bit** (or Oracle Linux 9 if in the dropdown — Security Onion 2.4 is built on OL9).
- RAM: **16 GB**.
- vCPU: **4 cores**.
- Disk: **300 GB** (thin-provisioned is fine).
- **Two NICs** required:
  - **NIC 1 (Management)** — connect to your normal practice network (`PG-MA1-LAN` or your home network) so you can SSH/web-UI in.
  - **NIC 2 (Monitor)** — connect to a port group that sees traffic you want to monitor. For practice, attach to `PG-LAN` (the MA2 LAN VLAN) so you can watch Client1/Client2 traffic.
- Mount the Security Onion ISO.

### D.2 ⚠️ Promiscuous Mode for the monitor NIC
The monitor NIC needs to see **all** traffic on its port group, not just packets addressed to its MAC. In ESXi:

1. *Networking → Port Groups → PG-LAN → Edit settings*.
2. *Security* section → set the following to **Accept**:
   - Promiscuous mode → **Accept**
   - MAC address changes → Accept (default)
   - Forged transmits → Accept (default)
3. Save. Same for any port group you'll mirror.

In VMware Workstation Pro: Edit → *Virtual Network Editor* → select VMnetX → tick "Allow promiscuous mode" (requires Linux host with vmnet group; on Windows host the option may be hidden — workaround is using a span/mirror at the physical switch level).

### D.3 Install steps
1. Power on → boots to installer.
2. **Network detection** — wait for it to complete.
3. Choose **Install Security Onion 2** (not Eval-Install — the on-disk option is "Install" + then config wizard picks Eval mode).
4. User: `socadmin` (or any non-`admin`) / strong password — write it down.
5. Disk partitioning: **Use entire disk** (you gave the VM 300 GB).
6. Reboot when done. Login as `socadmin`.

---

## Part E — First-boot Setup wizard

After login, the SO setup wizard auto-launches. If not:
```bash
sudo so-setup
```

Walk through:

| Prompt | Answer |
|---|---|
| Type of installation | **Eval** |
| Hostname | `securityonion` (or whatever) |
| Management interface | **eth0** (NIC1 — the management one) |
| Set IP for mgmt? | **Static** → e.g. `192.168.1.50/24`, gateway your router |
| DNS server | `8.8.8.8` (or your home router) |
| Sniffing/monitor interface | **eth1** (NIC2 — the promiscuous one) |
| Email for admin | any (e.g. `team1@manila.local`) |
| Password for SOC web UI admin | strong password — write it down |
| Network ranges to consider "HOME_NET" | enter your monitored subnets, e.g. `172.16.100.0/24,192.168.1.0/24,192.168.2.0/24` |
| OS patches now? | **Yes** (during practice you have internet) |
| Pull rule updates now? | **Yes** |

> 🕒 The wizard takes **30–60 minutes** the first time (downloading containers, updating rules). Make tea.

---

## Part F — Verify the install

After the wizard finishes:
```bash
sudo so-status
```
Every component should show `OK`. If anything is `FAIL` → `sudo so-restart`.

Browse from a laptop on the same management subnet:
```
https://192.168.1.50/
```
Login with the admin email + password you set in Part E.

You should land on the **SOC web UI** with tabs:
- **Alerts** — Suricata + Wazuh alerts (empty initially)
- **Hunt** — search Zeek/Suricata/Stenographer logs
- **Cases** — incident tickets
- **Dashboards** — Kibana
- **Downloads** — agent installers (Wazuh, etc.)

---

## Part G — Generate test traffic so you see something

By default the box is silent. To verify monitoring works:

1. From Client1 (on PG-LAN that NIC2 is monitoring), browse some test sites:
   ```
   curl -k https://www.manila.com
   ping 8.8.8.8
   nslookup google.com
   ```
2. Wait 30–60 sec → SOC UI → **Hunt** → search `event.dataset:"zeek.conn"` → you should see connections logged.
3. Trigger a real alert with a test signature — from Client1:
   ```
   curl http://testmyids.com/uid/index.html
   ```
   This URL triggers the canonical Suricata test rule. Wait, then SOC UI → **Alerts** → see *"GPL ATTACK_RESPONSE id check returned root"* (or similar).

If both work: Security Onion is monitoring correctly.

---

## Part H — Stage offline rules (for competition)

Security Onion's first-boot pulls Emerging Threats Suricata rules + Wazuh rulesets from the internet. For competition the chief will pre-stage these on the VM, but you should verify your practice install also has rules:

```bash
sudo so-rule list   # Suricata rules count
sudo so-rule disable <sid>   # disable noisy rules
sudo so-rule enable <sid>    # enable specific
```

Most rules sit under `/opt/so/rules/` and `/etc/wazuh/rules/`.

---

## Part I — Snapshot

Take a snapshot named `SecurityOnion-fresh-eval` *after* the wizard completes successfully and Hunt shows traffic. You'll restore to this between practice runs so the data store is clean.

---

## Part J — Common operations cheat-sheet

```bash
sudo so-status                   # health check
sudo so-restart                  # restart all services
sudo so-stop                     # stop all
sudo so-start                    # start all
sudo so-log-tail                 # live tail of SO logs
sudo so-import-pcap file.pcap    # replay a PCAP through Suricata + Zeek
sudo so-allow                    # add an analyst's IP to firewall (so SOC UI is reachable)
sudo so-test                     # generate a known alert (uses curl http://testmyids.com)
sudo so-elastic-clear            # wipe all indexed data (fresh start)
```

---

## Part K — What to expect on Day 2

Speculative based on Marlon's hint + Criterion B title (*"Cyber Security Incident Response, Digital Forensics, Application Security"*). The chief hasn't released the official Day 2 deliverables doc yet, but expect:

1. **Deploy Security Onion** (Eval mode) and confirm it's monitoring a target subnet. — what this file taught.
2. **Configure log forwarding** from Day 1 servers (WINSRV1/LinSRV1) — Wazuh agent install, syslog forwarding.
3. **Investigate provided alerts** in the SOC UI (incident response).
4. **Forensic analysis** of a provided PCAP / memory dump / disk image.
5. **Secure-deploy a web app** (application security part) — possibly DVWA, Juice Shop, or a hardened webapp.
6. Possibly: **OpenVPN as a bastion** — separate from MA2's pfSense VPN, an OpenVPN service on a Linux box.

The actual walkthrough (with steps mapped to marking-scheme rows) is in `24_Day2_SecurityHardening.md`.

---

## USB checklist (for competition)

- [ ] Security Onion 2.4 ISO (~9 GB)
- [ ] SHA-256 hash file
- [ ] Snapshot of clean SO Eval VM (so you can re-import if needed)
- [ ] This guide (`07_…` + `24_…`)
- [ ] Wazuh agent installers for Win10 + CentOS (download from `https://documentation.wazuh.com/current/installation-guide/index.html`)

Next file: **`24_Day2_SecurityHardening.md`** — the Day 2 walkthrough.
