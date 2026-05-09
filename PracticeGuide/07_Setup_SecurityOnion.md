# 07 — Setup Security Onion (Stage A — one-time build)

**What this file covers:** how to install **Security Onion 2.4** on the ESXi server as your **threat-hunting / SOC platform** for Day 2.

**Why we need it:** the marking scheme has a whole **Criterion B = "Cyber Security Incident Response, Digital Forensics, Application Security" worth 25 marks on Day 2**. Security Onion is the standard FREE platform for that work. Marlon explicitly mentioned it (*"may mga setup doon na need internet yung sa OpenVPN and Security Onion"*).

**Skill level assumed:** none. We'll explain every term.

**Time:** ~3 hours total (60 min download + 30 min install + 90 min `so-setup`).

**Internet required during this whole file** — Security Onion downloads ~10 GB during initial setup.

---

## What is Security Onion? (read this once, in plain English)

Imagine your network is a city, and every car driving through is a packet. Security Onion is a **CCTV system + alarm panel + investigations desk** for that city, all in one Linux VM:

- **CCTV (sensor):** captures every packet on the wire (full PCAP).
- **Alarm panel (Suricata IDS + community rules):** flags suspicious traffic (e.g., known malware C2 IPs).
- **Investigations desk (Hunt + Cases + Kibana web UI):** lets an analyst search alerts, pivot to PCAP, write notes, mark cases solved.

Day 2 work in this competition = **be the analyst. Find the suspicious thing. Write the report.** Security Onion is what you use to do that.

### Three things Security Onion gives you on Day 2

| Tool inside SO | What it shows you | Day 2 use |
|---|---|---|
| **Hunt / Alerts** | List of "something weird happened" events ranked by severity | Triage: which alert is real, which is noise |
| **Dashboards / Kibana** | Charts (top talkers, top DNS queries, top URLs, etc.) | Spot anomalies (e.g., one host suddenly making 10000 DNS queries) |
| **PCAP** | Raw packets for any flow | Open in Wireshark, see exactly what happened |

You'll use these to find evidence of attacks (the "HO Flags") and write reports.

---

## Where Security Onion fits in your network

Looks like this on the ESXi server:

```
                      INTERNET (PG-Internet)
                          │
                       pfSense (firewall)
                          │
            ┌─────────────┼─────────────┐
            │             │             │
         PG-LAN        PG-DMZ       PG-Servers
       (clients)     (LinSRV1)     (WINSRV1/3/4)
            │             │             │
   ┌────────┴───┐         │             │
   │            │         │             │
 Client1     Client2      │             │
                          │             │
                  ╔═══════╧═══════╗     │
                  ║ SECURITY ONION ║◄═══╪══════ promiscuous (mirror) port
                  ║ 192.168.2.20   ║     │      sniffs all traffic on 
                  ║ on PG-Servers  ║     │      PG-LAN + PG-DMZ + PG-Servers
                  ╚════════════════╝     │
```

- **eth0 (management):** static IP `192.168.2.20` on PG-Servers VLAN. This is how you log into the web UI from Client1 or PC1's browser.
- **eth1 (sniffing):** **no IP**. Connects to a special "all-can-see-all" port group called **PG-MIRROR**. Picks up everything.

You'll create PG-MIRROR in this file too. Don't worry — we'll walk through it.

---

## Phase 1 — Plan the resources (5 min)

| Resource | Min for practice | Recommended |
|---|---|---|
| RAM | 8 GB | 12 GB |
| Disk | 200 GB | 250 GB |
| vCPU | 4 | 4 |
| NICs | 2 (mgmt + sniff) | 2 |

> 💡 If your ESXi box has only 16 GB total RAM, give SO **8 GB** and during practice power off WINSRV4 (Offline Root CA — only needed once for cert signing) to free up its 2 GB. Net effect: SO has memory it needs, and your stack still runs.

---

## Phase 2 — Download the Security Onion ISO (15–60 min depending on your internet)

### Step 2.1 — Download

**Where:** PC1 (your competition workstation) — needs internet.

**What to download:** the Security Onion 2.4 ISO. It's about **10 GB**.

1. Open Chrome on PC1.
2. Go to: `https://github.com/Security-Onion-Solutions/securityonion/blob/2.4/main/VERIFY_ISO.md`
3. On that page, find the most-recent ISO download link (it will be on the official Security Onion download mirror — typically `securityonionsolutions.com/software`).
4. Click the latest stable `.iso` file. It will start downloading.

> ⚠️ **Don't use a "minimal" or "lite" version.** Get the full Security Onion 2.4 ISO. The difference is whether the rule sets and Docker images come pre-bundled.

5. While it's downloading, also download the **SHA256 hash file** from the same page. We'll use this to verify the ISO isn't corrupted.

**Download time:** 30–60 min on a typical office connection. Have a coffee.

### Step 2.2 — Verify the ISO is not corrupted

**Why we do this:** a corrupted ISO will fail mid-install after 90 minutes of waiting. 1 minute of verification = saves an hour later.

**Where:** PC1 PowerShell.

```powershell
cd $env:USERPROFILE\Downloads
Get-FileHash securityonion-2.4.*.iso -Algorithm SHA256 | Format-List
```

**Expected output:** a long hex string like `A1B2C3D4...`.

Compare it line-by-line to the SHA256 value on the download page. **If it matches → safe. If it doesn't → re-download.**

### Step 2.3 — Upload the ISO to ESXi datastore

**Where:** ESXi web UI on PC1's browser → `https://192.168.1.1` → login `wsauser / Andres@9V4`.

1. Click **Storage** in the left menu.
2. Click your datastore (typically `datastore1`).
3. Click **Datastore browser**.
4. Click **Upload**.
5. Select the `securityonion-2.4.*.iso` file from your Downloads folder.
6. Wait for upload to finish (10 GB will take 10–30 min depending on your network).

**Expected result:** the ISO file appears in the datastore listing.

---

## Phase 3 — Create PG-MIRROR port group with promiscuous mode (10 min)

**What's a port group?** Think of it as a virtual network cable. Right now you have PG-LAN, PG-DMZ, etc. PG-MIRROR is special — any VM connected to it can **see all traffic** going through it.

**Why we need it:** Security Onion's sniffing NIC needs to "see" what's happening on the network without participating. It's like a security guard with a camera, not a delivery driver.

### Step 3.1 — Create PG-MIRROR port group

**Where:** ESXi web UI → **Networking** in the left menu.

1. Click the **Port groups** tab.
2. Click **Add port group**.
3. Fill in:
   - **Name:** `PG-MIRROR`
   - **VLAN ID:** `0` (or the same VLAN as PG-Servers — `2` if you tagged it)
   - **Virtual switch:** the same vSwitch hosting PG-Servers (typically `vSwitch1`).
4. Click **Security** section to expand it.
5. Set **all three** of these to **Accept** (override the inherited "Reject"):
   - **Promiscuous mode:** Accept
   - **MAC address changes:** Accept
   - **Forged transmits:** Accept
6. Click **Add**.

**What you just did:** you created a port group where any connected VM can see all packets passing through.

### Step 3.2 — (Optional) Span port traffic from other VLANs

If you want SO to also see PG-LAN and PG-DMZ traffic (recommended for a real SOC view), the simplest approach is to put SO's sniffing NIC on the **same vSwitch** as the main traffic.

In the next phase, when we add the sniffing NIC, we'll use **PG-MIRROR**, but ESXi vSwitches don't natively span between virtual switches. Workaround: **connect SO to the vSwitch that has all your VLANs**, and use promisc mode to see everything on that vSwitch.

For our 4-VLAN setup, the cleanest layout:

| Day-2 NIC role | Connect to |
|---|---|
| Management (eth0, has IP) | PG-Servers (192.168.2.20) |
| Sniffing (eth1, no IP) | PG-MIRROR (promisc enabled) |

This sees all traffic on the same vSwitch as PG-Servers (which carries WINSRV1/3/4 traffic). For full visibility into pfSense packet flows, you can later add a 3rd NIC on PG-LAN with promisc to also catch client-side traffic. **Not required for marks** — just nice-to-have.

---

## Phase 4 — Create the Security Onion VM (10 min)

**Where:** ESXi web UI → **Virtual Machines** → **Create / Register VM**.

### Step 4.1 — Wizard

1. Select **Create a new virtual machine** → Next.
2. Fill in:
   - **Name:** `SecOnion`
   - **Compatibility:** ESXi 8.0
   - **Guest OS family:** Linux
   - **Guest OS version:** Oracle Linux 9 (or Other Linux 6.x or later 64-bit)
3. Click Next → select your datastore → Next.
4. Customize the hardware:
   - **CPU:** 4 vCPU (cores per socket = 2)
   - **Memory:** `8192` MB (8 GB)
   - **Hard disk 1:** `200` GB, thin-provisioned
   - **Network adapter 1 (eth0 — mgmt):** PG-Servers
   - Click **Add network adapter**:
     - **Network adapter 2 (eth1 — sniff):** PG-MIRROR
   - **CD/DVD Drive 1:** Datastore ISO file → browse to `securityonion-2.4.*.iso` → tick **Connect at power on**
5. Review → Finish.

### Step 4.2 — Boot from ISO

1. Select the **SecOnion** VM → **Power on**.
2. Click **Console** to open the live console window.
3. The Security Onion installer boots → press Enter at the install prompt.

---

## Phase 5 — Install Security Onion to disk (~30 min unattended)

**What happens:** the installer formats the 200 GB disk, copies the OS, and reboots.

### Step 5.1 — Choose the install path

You'll see a menu. Pick:

```
1) Install Security Onion 2.4
```

Press Enter.

### Step 5.2 — Disk wipe confirmation

It asks: *"This will erase all data on the disk. Continue?"*

Type `yes` and press Enter.

### Step 5.3 — Set the install user password

The installer creates a Linux user that you'll log in as later (this is **NOT** the SOC analyst account — that comes later).

Set:
- **Username:** `socadmin`
- **Password:** `P@ssw0rdSO!` (write this down — you can't recover it)
- Confirm password.

### Step 5.4 — Wait for the install to finish

The installer copies files (~25 min). When done, it reboots.

**At reboot, EJECT the ISO:** from ESXi UI → SecOnion VM → Edit → CD/DVD Drive 1 → uncheck "Connect at power on" → Save. (If you forget, the VM will boot the ISO again instead of from disk.)

---

## Phase 6 — First boot + run `so-setup` (~90 min — needs internet)

**Where:** ESXi Console for SecOnion.

### Step 6.1 — Login and run setup

1. After reboot, you see a login prompt: `socadmin@securityonion login:`.
2. Login with `socadmin` / `P@ssw0rdSO!`.
3. The setup wizard auto-launches. If it doesn't, run:
   ```bash
   sudo so-setup
   ```

### Step 6.2 — Choose the deployment type

Menu shows several options. For a small single-team practice lab, choose:

```
EVAL
```

(Stands for "Evaluation" — single-machine deployment with everything bundled. Good for our 1-host lab.)

> 📝 **In a real SOC,** you'd pick `STANDALONE` or `DISTRIBUTED`, but `EVAL` is right for practice — it uses less RAM and skips some advanced clustering.

### Step 6.3 — Set hostname + management IP

Fill in:
- **Hostname:** `seconion`
- **Domain:** `manila.com` (matches your AD domain)
- **Management interface:** `eth0`
- **IP method:** `static`
- **IP address:** `192.168.2.20`
- **Netmask:** `255.255.255.0`
- **Gateway:** `192.168.2.254` (your pfSense)
- **DNS server:** `192.168.2.10` (your WINSRV1)

### Step 6.4 — Set the monitoring interface

- **Sniffing interface:** `eth1`
- **Confirm:** yes (this brings eth1 up in promiscuous mode, no IP).

### Step 6.5 — Set the analyst web account

This is the account you'll use to log into the **Security Onion web UI**:
- **Email:** `analyst@manila.com`
- **Password:** `P@ssw0rdAnalyst!` (write this down)

### Step 6.6 — Choose ruleset

You'll see options:
```
1) ETOPEN (free, recommended)
2) ETPRO (paid)
3) Talos (paid)
```

Pick **`1) ETOPEN`** — Emerging Threats Open ruleset. Free, ~30000 detection rules. Good enough for practice.

### Step 6.7 — Confirm and let it run

The wizard summarizes settings. Type `yes` to start setup.

**What happens for the next 60–90 min:**
- Pulls Docker images (~3 GB).
- Pulls ETOPEN ruleset.
- Configures Suricata, Zeek, Stenographer, Elasticsearch, Kibana.
- Creates analyst account.
- Starts all services.

**Stay on the console** — it logs progress. If it asks anything, answer. Otherwise wait.

When done you see: `Setup complete. Browse to https://192.168.2.20`

### Step 6.8 — First login to the web UI

**Where:** Client1 (Win10 in PG-LAN) or PC1 — either browser.

1. Open Chrome → `https://192.168.2.20`
2. Click "Advanced" → "Proceed" past the cert warning (it's self-signed for now).
3. Log in: `analyst@manila.com` / `P@ssw0rdAnalyst!`
4. You should see the Security Onion landing page with tiles for **Hunt**, **Dashboards**, **Cases**, **Kibana**, **CyberChef**.

> ✅ **Success:** if you see this dashboard, Security Onion is alive and you're done with Stage A install.

---

## Phase 7 — Verify SO is actually capturing traffic (10 min)

**Why:** install can succeed but eth1 might not be sniffing properly. Let's confirm before we walk away.

### Step 7.1 — Generate test traffic

**On Client1:**
```cmd
ping 192.168.2.10
ping 192.168.2.30
```

(Pings WINSRV1 + WINSRV3 to put traffic on PG-Servers.)

### Step 7.2 — Look for it in Hunt

**On the SO web UI:**
1. Click **Hunt** in the top menu.
2. In the search bar paste: `event.dataset:icmp`
3. Click the time range and set "Last 15 minutes".
4. Click **Hunt**.

**Expected:** rows showing your ICMP pings.

If nothing → eth1 isn't sniffing. Troubleshoot:
- ESXi → SecOnion VM → confirm Network adapter 2 is on **PG-MIRROR**.
- ESXi → Networking → PG-MIRROR → confirm **Promiscuous: Accept**.
- SSH to SO: `sudo so-status` — confirm `OK` for all services.

### Step 7.3 — Check alerts panel

1. Click **Alerts** in the top menu.
2. Set time range "Last 1 hour".

You may already see some alerts (background scanning, DNS queries flagged as "potentially unwanted", etc.). That's normal — Security Onion is doing its job.

---

## Phase 8 — Snapshot the VM (1 min — IMPORTANT)

**Why:** so you can revert to "clean SOC, no alerts yet" before each Day 2 practice run.

**Where:** ESXi UI → SecOnion VM → Actions → Snapshots → **Take snapshot**.
- **Name:** `Stage-A-Complete`
- **Description:** `Fresh SO install, no alerts, ready for Day 2 practice`

Click **Take snapshot**.

> 💡 You'll create a second snapshot later named `pre-day-2` to capture "right before the practice run starts".

---

## Phase 9 — Stop & note credentials (2 min — IMPORTANT)

Write these down somewhere safe (you and your teammate need them):

| What | Where | Username | Password |
|---|---|---|---|
| ESXi UI | `https://192.168.1.1` | `wsauser` | `Andres@9V4` |
| Linux user inside SO VM | Console / SSH to `192.168.2.20:22` | `socadmin` | `P@ssw0rdSO!` |
| SO web UI / analyst | `https://192.168.2.20` | `analyst@manila.com` | `P@ssw0rdAnalyst!` |

---

## Common problems and fixes

### Problem: install hangs at "Pulling Docker images"
**Cause:** internet is slow or blocked by firewall.
**Fix:** verify SO can reach the internet:
```bash
curl -I https://hub.docker.com
```
Should return HTTP 200/301. If not, check pfSense allows SO (192.168.2.20) → WAN HTTPS.

### Problem: "No alerts ever appear"
**Cause:** sniffing NIC not in promiscuous mode.
**Fix:** ESXi UI → Networking → PG-MIRROR → Edit → Security → Promiscuous: **Accept**. Then on SO console: `sudo so-suricata-restart`.

### Problem: web UI shows "Bad Gateway" or hangs at login
**Cause:** Elasticsearch isn't fully up. It takes 5–10 min after `so-setup` finishes.
**Fix:** wait 5 more min. If still broken: `sudo so-status` — anything not OK? `sudo so-elastic-restart`.

### Problem: SO VM uses 100% CPU and is slow
**Cause:** under-resourced. Default is 4 vCPU, 8 GB.
**Fix:** ESXi → Edit VM settings → bump RAM to 12 GB. Or: power off WINSRV4 (Offline Root CA, rarely needed) to free RAM for SO.

### Problem: forgot the analyst password
**Fix:** SSH to SO as `socadmin` → run:
```bash
sudo so-user reset-password analyst@manila.com
```
Set a new password.

---

## What you've built

✅ **Security Onion 2.4** running at `192.168.2.20`
✅ **Sniffing eth1** in promiscuous mode catching traffic on PG-Servers
✅ **ETOPEN ruleset** loaded (~30000 detection rules)
✅ **Snapshot saved** — restore in 30 seconds for any practice run
✅ **Analyst web UI** ready at `https://192.168.2.20`

You now have what every modern SOC analyst uses to hunt threats. **Day 2's IR/Forensics work happens entirely inside this web UI.** See `40_Day2_IR_Forensics.md` for the playbook.

---

## What goes on this VM later

Don't install anything else here. Treat Security Onion as a sealed appliance. **Any tool you'd use against the network (nmap, tcpdump, Wireshark) you run from Client1, Kali, or PC1 — Security Onion just watches.**

---

## Time-budget recap

| Step | Time |
|---|---|
| Download ISO | 30–60 min |
| Verify hash | 1 min |
| Upload ISO to ESXi | 10–30 min |
| Create PG-MIRROR | 10 min |
| Create + boot VM | 10 min |
| Disk install | 30 min unattended |
| `so-setup` wizard | 60–90 min unattended |
| Verify capture | 10 min |
| Snapshot | 1 min |
| **Total** | **~3 hours** (mostly unattended) |

You only do this **once**. Afterwards: 30-second snapshot revert before every Day 2 practice.

---

End of file. Next: open `40_Day2_IR_Forensics.md` to start using Security Onion for marks.
