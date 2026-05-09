# 09 — Give a VM Temporary Internet (when it can't reach the outside)

## What this file solves (the problem you just hit)

You're in the middle of installing your **CMS-Target** VM. You ran `apt update` or `apt install` and got this error:

```
W: Some index files failed to download. They have been ignored, or old ones used instead.
W: Failed to fetch http://archive.ubuntu.com/...
```

**Why this happened:**
- Your CMS-Target lives on the **`PG-MA1-CMS`** port group (a sealed practice subnet `192.168.2.0/24`).
- Its default gateway is `192.168.2.254` — supposed to be **pfSense**, but pfSense isn't built yet.
- So packets going to `archive.ubuntu.com` hit nothing and time out.
- Result: no internet, can't install packages.

**This file fixes that.** We'll temporarily give the VM internet, install what we need, then take that internet away again.

---

## Quick mental model — the simple analogy

Think of port groups like virtual cables:

| Port group | Where it goes |
|---|---|
| `PG-MA1-CMS` | An **isolated practice room** with no door to the outside |
| `Management Network` (on vSwitch0) | A **hallway** that connects to your TP-Link router → INTERNET |

Right now your CMS-Target is plugged into the isolated room. We'll:
1. Build a "side door" called **`PG-TempInternet`** that opens onto the hallway.
2. Temporarily move CMS-Target's cable to the side door (so it can reach the internet).
3. Install everything it needs.
4. Move the cable back to the sealed room (so it stays isolated for practice).

---

## Big picture — visual flow

```
BEFORE (where you are now):                AFTER (during install):                BACK AGAIN (after install):
                                                                                  
   CMS-Target (192.168.2.1)                CMS-Target (DHCP from TP-Link)         CMS-Target (192.168.2.1)
       │                                       │                                      │
   PG-MA1-CMS (sealed)                     PG-TempInternet (vSwitch0)             PG-MA1-CMS (sealed)
       │                                       │                                      │
   ❌ no internet                          ✅ TP-Link → INTERNET                  ❌ sealed again — perfect for practice
```

Three states. We're going from state 1 → state 2 (install phase) → state 3 (final, sealed).

---

## What you'll do — overview

1. **One-time setup** (5 min): create the `PG-TempInternet` port group on ESXi.
2. **Per VM** (when it needs internet): switch the NIC + change the VM to DHCP + verify.
3. **After install**: switch back + restore static IP + snapshot.

Skim this whole file once, then come back to do each section in order.

---

# Phase 1 — One-time setup (do this ONCE, ever)

This phase creates the "side door" port group `PG-TempInternet`. **Once it exists, it stays in your ESXi config forever.** You only do this once, then every VM that needs temp internet uses the same door.

## Step 1.1 — Open the ESXi UI

On PC1 → Chrome (or any browser) → go to:
```
https://192.168.1.10/ui
```
- Log in as `root` with the password you set during ESXi install.

## Step 1.2 — Create the port group

1. Click **Networking** in the left sidebar.
2. Click the **Port groups** tab at the top.
3. Click **+ Add port group** (button near the top).
4. A small dialog opens. Fill in:

   | Field | Value |
   |---|---|
   | Name | `PG-TempInternet` |
   | VLAN ID | `0` |
   | Virtual switch | `vSwitch0` ← **VERY IMPORTANT** — this is the only one with the physical NIC + internet |

5. Click **Add**.

✅ **Expected result:** `PG-TempInternet` now appears in the list of port groups.

## Step 1.3 — (Optional sanity check) Confirm vSwitch0 has internet

If you want to be sure vSwitch0 is actually connected to your TP-Link:

1. Click the **Virtual switches** tab.
2. Click `vSwitch0`.
3. Look at the right-side diagram. Under **Physical adapters** you should see `vmnic0` (or similar) — that's your physical NIC.

If `Physical adapters` is empty → your ESXi has no physical NIC bound to vSwitch0. Tell me — we'll fix that before continuing.

✅ **Phase 1 done. Never have to do it again.**

---

# Phase 2 — When a VM needs temp internet (do this per-VM)

This is the main flow. Repeat any time a VM needs to install packages.

## Stage A — Switch the VM's virtual cable

**Where:** ESXi UI on PC1.

1. Click **Virtual Machines** in the left sidebar.
2. Click the VM you want to give internet (for you right now: **`CMS-Target`**).
3. Click **Edit** at the top.
4. Find **Network Adapter 1** (or whichever NIC) → click the drop-down next to it.
5. Change from `PG-MA1-CMS` (the isolated one) to **`PG-TempInternet`**.
6. Click **Save**.

✅ **Expected result:** the VM's "virtual cable" is now plugged into the hallway. But the VM doesn't know its IP changed — Stage B fixes that.

## Stage B — Tell the VM to ask for a new IP (DHCP)

The VM still has its old static IP (e.g., `192.168.2.1`). That IP is on the wrong subnet now — TP-Link uses `192.168.1.x`. So we tell the VM "use DHCP, ask the TP-Link for an IP".

### 🎯 Which sub-section do I use?

Pick ONE row matching the OS of the VM you're configuring:

| If your VM is... | Used by these VMs | Use sub-section |
|---|---|---|
| **Ubuntu Server 22.04** (uses `netplan`) | CMS-Target ← **YOU ARE HERE**, Security Onion | ✅ **B.1** |
| **CentOS Stream 9** (uses `nmcli`) | ISP, LinSRV1 | ✅ **B.2** |
| **Kali Linux** (Debian-based) | Kali attacker VM | ✅ **B.3** |
| **Windows Server 2022 / Windows 10 Eval** | WINSRV1/3/4, Client1/2/3, MalwareLab | ✅ **B.4** |

> 💡 **Skip the other 3 sub-sections.** They're reference for VMs you'll build later. Right now (CMS-Target = Ubuntu), only **B.1** matters.

---

### 🟢 B.1 — Ubuntu Server (CMS-Target, Security Onion)

In the VM console (the black screen with the `competitor@cms-target:~$` prompt):

#### B.1.1 — First, back up your current netplan file

This is so you can restore the static IP later without retyping the whole YAML.

```bash
sudo cp /etc/netplan/00-installer-config.yaml /etc/netplan/00-installer-config.yaml.STATIC.bak
```

✅ Backup saved as `.STATIC.bak`. Move on.

#### B.1.2 — Edit the netplan file to use DHCP

```bash
sudo nano /etc/netplan/00-installer-config.yaml
```

Nano opens the file. **Delete all the existing content** (Ctrl+K repeatedly cuts each line until the file is empty).

Then **type these 5 lines** (substitute `ens34` if your interface is named differently):

```
network:
  version: 2
  ethernets:
    ens34:
      dhcp4: true
```

Indent reminder:
- `network:` — column 1 (no spaces)
- `version:` and `ethernets:` — 2 spaces
- `ens34:` — 4 spaces
- `dhcp4: true` — 6 spaces

Save: **Ctrl+O → Enter → Ctrl+X**.

#### B.1.3 — Apply the new config

```bash
sudo chmod 600 /etc/netplan/00-installer-config.yaml
sudo netplan apply
```

✅ **Expected:** brief output (or just one harmless OVS warning), then back to the prompt.

> ⚠️ **You'll lose your old static IP `192.168.2.1`.** That's expected — you're temporarily switching to DHCP. The backup file `.STATIC.bak` will let you restore it later.

#### B.1.4 — How to restore the static IP later (after install is done)

To switch back to the static `192.168.2.1` config later, just run:

```bash
sudo cp /etc/netplan/00-installer-config.yaml.STATIC.bak /etc/netplan/00-installer-config.yaml
sudo chmod 600 /etc/netplan/00-installer-config.yaml
sudo netplan apply
```

That puts the original YAML back. Done in 3 commands.

➡️ Now jump to **Stage C** below.

---

### 🟡 B.2 — CentOS Stream 9 (ISP, LinSRV1)

> Skip this section if you're on Ubuntu. Come back here when you build ISP or LinSRV1.

CentOS uses `nmcli` (no netplan). Inside the VM:

```bash
# Switch to DHCP
sudo nmcli con mod ens160 ipv4.method auto
sudo nmcli con mod ens160 ipv4.addresses ""
sudo nmcli con mod ens160 ipv4.gateway ""
sudo nmcli con up ens160
```

To restore the static IP later (substitute the right IP for that VM):
```bash
sudo nmcli con mod ens160 ipv4.addresses 10.0.0.1/24 ipv4.method manual ipv4.gateway "" ipv4.dns 8.8.8.8
sudo nmcli con up ens160
```

> 💡 ISP uses `10.0.0.1/24`, LinSRV1 uses `192.168.1.10/24`. Use whichever matches the VM you're working on.

➡️ Jump to **Stage C**.

---

### 🟡 B.3 — Kali Linux

> Skip this section if you're on Ubuntu. Come back here when you build Kali.

Kali is Debian-based and uses NetworkManager (with a GUI applet on the desktop).

#### B.3 GUI (recommended — easiest)

1. Top-right of Kali's desktop → click the **network icon** (looks like 2 stacked arrows, or a plug).
2. Click **Edit Connections...**
3. Highlight **`Wired connection 1`** → click the **gear icon** (Edit) at the bottom.
4. Click the **IPv4 Settings** tab.
5. Method drop-down → change to **Automatic (DHCP)**.
6. Click **Save**.
7. Top-right network icon → click your connection → **Disconnect**, then click again to **Connect**. (This reapplies the new method.)

#### B.3 CLI (alternative)

```bash
sudo nmcli con mod "Wired connection 1" ipv4.method auto ipv4.addresses "" ipv4.gateway "" ipv4.dns ""
sudo nmcli con up "Wired connection 1"
```

To restore Kali's static IP later:
```bash
sudo nmcli con mod "Wired connection 1" \
    ipv4.addresses 192.168.2.2/24 \
    ipv4.gateway 192.168.2.254 \
    ipv4.dns 8.8.8.8 \
    ipv4.method manual
sudo nmcli con up "Wired connection 1"
```

➡️ Jump to **Stage C**.

---

### 🟡 B.4 — Windows Server 2022 / Windows 10 Eval

> Skip this section if you're on Ubuntu. Come back here when you build any Windows VM.

1. Inside the VM, right-click the **Start menu** → **Network Connections**.
2. Click **Change adapter options**.
3. Right-click your ethernet adapter → **Properties**.
4. Scroll the list → click **Internet Protocol Version 4 (TCP/IPv4)** → click **Properties**.
5. Tick **Obtain an IP address automatically** + **Obtain DNS server address automatically**.
6. Click **OK** twice → close the windows.
7. Verify in PowerShell:
   ```powershell
   ipconfig
   ```
   You should see an IP in the `192.168.1.x` range from your TP-Link.

To restore the original static IP: same flow, but pick **Use the following IP address** and re-enter the values from the VM's specs table.

➡️ Jump to **Stage C**.

---

## Stage C — Verify the VM has internet

The most important step. Don't proceed until this works.

In the VM console (Linux):
```bash
ping -c 3 8.8.8.8
ping -c 3 archive.ubuntu.com
```

In the VM (Windows PowerShell):
```powershell
ping 8.8.8.8
ping www.microsoft.com
```

✅ **Expected output:**
```
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=117 time=15.2 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=117 time=14.9 ms
64 bytes from 8.8.8.8: icmp_seq=3 ttl=117 time=15.4 ms
```

3 replies → ✅ you have internet. Proceed to Stage D.

❌ **If pings fail** — see the troubleshooting table at the end of this file.

---

## Stage D — Run your install commands

Now run whatever install commands the original guide asked for. Examples:

| If you're building... | Run these (per the original guide) |
|---|---|
| CMS-Target (Step 2.3 of `03_…`) | `sudo apt update`, `sudo apt install -y apache2 mariadb-server php ...`, `wget https://ftp.drupal.org/files/projects/drupal-7.57.tar.gz` |
| ISP (Part 1 of `04_…`) | `sudo dnf install -y dnsmasq httpd mod_ssl` |
| LinSRV1 (Part 6 of `04_…`) | `sudo dnf install -y httpd mod_ssl realmd sssd ...` |
| Kali (Step 3.3 of `03_…`) | `sudo apt update`, `sudo apt install -y exploitdb metasploit-framework hashcat ...` |
| Security Onion (Phase 6 of `07_…`) | `sudo so-setup` (downloads ~10 GB of Docker images) |
| MalwareLab (in `41_…`) | Download PEStudio, CFF Explorer, etc. via Chrome |

This phase takes however long the install needs (5 min for ISP, 30 min for SecOnion).

---

## Stage E — Snapshot before reverting (safety net)

Before switching back to the isolated port group, take a snapshot. This is your "everything installed but still has internet" save point — useful if something goes wrong in Stage F.

ESXi UI → VM → **Actions** → **Snapshots** → **Take snapshot**:
- Name: `<VM>-after-install` (e.g., `CMS-Target-after-install`)
- Description: `Packages installed, internet still on`

Click **Take snapshot**.

✅ Snapshot saved. Move on.

---

## Stage F — Switch BACK to the sealed port group

Now we reverse the temp-internet setup so the VM is sealed again.

### F.1 — Inside the VM: restore the original static IP

| OS | What to do |
|---|---|
| **Ubuntu** (CMS-Target, SecOnion) | `sudo cp /etc/netplan/00-installer-config.yaml.STATIC.bak /etc/netplan/00-installer-config.yaml` then `sudo netplan apply` |
| **CentOS** (ISP, LinSRV1) | Re-run the static `nmcli` command (see B.2 above) |
| **Kali** | Re-run the static `nmcli` command (see B.3 above) |
| **Windows** | Network Connections → adapter → IPv4 → "Use the following IP address" → re-enter values |

### F.2 — Power off the VM

Cleanest method (Linux):
```bash
sudo shutdown now
```

Windows: Start → Power → Shut down.

### F.3 — Switch the NIC back in ESXi UI

1. ESXi UI → Virtual Machines → click the VM.
2. Click **Edit**.
3. Find **Network Adapter 1** → drop-down → switch from `PG-TempInternet` back to the isolated group (e.g., `PG-MA1-CMS`).
4. Click **Save**.

### F.4 — Power the VM back on

ESXi UI → VM → **Power on**.

Log in.

### F.5 — Verify the static IP is back

Linux:
```bash
ip a show ens34
```
Should show: `inet 192.168.2.1/24` (or whatever the original static was).

Windows:
```powershell
ipconfig
```

✅ **Expected:** the VM is now back on the isolated port group with its original static IP.

---

## Stage G — Snapshot the FINAL clean state

This is the snapshot you'll revert to before every practice run.

ESXi UI → VM → **Actions** → **Snapshots** → **Take snapshot**:
- Name: `<VM>-base` or `<VM>-vulnerable` (matches what `03_…` Step 2.8 asks for: `cms-target-vulnerable`)
- Description: `Sealed in PG-MA1-CMS, all setup complete, ready for practice`

✅ **You're done.** This VM is now permanently sealed in its practice port group, fully installed.

---

# Worked example — full walkthrough for CMS-Target

Putting it all together, here's the start-to-finish flow you're doing **right now**:

```
1. ESXi UI → Networking → Port groups → Add → PG-TempInternet on vSwitch0     [ONE-TIME]
                                                                                  
2. ESXi UI → CMS-Target → Edit → Network Adapter 1 → PG-TempInternet → Save
                                                                                  
3. In CMS-Target console:
     sudo cp /etc/netplan/00-installer-config.yaml /etc/netplan/00-installer-config.yaml.STATIC.bak
     sudo nano /etc/netplan/00-installer-config.yaml
       (replace contents with:)
       network:
         version: 2
         ethernets:
           ens34:
             dhcp4: true
     Ctrl+O → Enter → Ctrl+X
     sudo chmod 600 /etc/netplan/00-installer-config.yaml
     sudo netplan apply
                                                                                  
4. Verify:  ping -c 3 8.8.8.8   ← should reply
                                                                                  
5. Run Step 2.3 of 03_…  (LAMP install + Drupal download)
   Run Step 2.4 of 03_…  (Drupal web install)
   Run Step 2.5 of 03_…  (john user)
   Run Step 2.6 of 03_…  (privesc path)
   Run Step 2.7 of 03_…  (sanity test in browser)
                                                                                  
6. Take snapshot:  CMS-Target-after-install
                                                                                  
7. Restore static IP:
     sudo cp /etc/netplan/00-installer-config.yaml.STATIC.bak /etc/netplan/00-installer-config.yaml
     sudo netplan apply
   Power off:  sudo shutdown now
                                                                                  
8. ESXi UI → CMS-Target → Edit → Network Adapter 1 → PG-MA1-CMS → Save
   Power on
                                                                                  
9. Verify:  ip a show ens34   ← should show 192.168.2.1
                                                                                  
10. Take snapshot:  cms-target-vulnerable   ← matches 03_… Step 2.8
```

That's the whole loop. Steps 2–9 take about 60 min (most of it is the LAMP+Drupal install in step 5).

---

# Reference tables

## Which VMs need temp internet?

| VM | When to use this | What it installs |
|---|---|---|
| **CMS-Target** | During initial build (Steps 2.3–2.6 of `03_…`) | Apache, MariaDB, PHP, Drupal 7 |
| **Kali** | During first boot post-install (optional, for `apt update`) | Mirror config + tool updates |
| **ISP** (MA2) | During initial build (Step 1.2 of `04_…`) | dnsmasq, httpd, mod_ssl |
| **LinSRV1** (MA2) | During initial build (Step 6.2 of `04_…`) | httpd, realmd, sssd, krb5 |
| **WINSRV1/3/4** (MA2) | Optional — only if you want Windows Updates | Windows Updates |
| **Client1/2/3** (MA2) | Optional — to install Chrome/PuTTY/Wireshark | Chrome, PuTTY, Wireshark, Nmap, OpenVPN |
| **SecOnion** | During `so-setup` (Phase 6 of `07_…`) | ~10 GB Docker images + ETOPEN ruleset |
| **MalwareLab** | During initial build | PEStudio, CFF Explorer, Process Hacker, Wireshark |
| **VulnHub VMs** | Never (deliberately offline) | — |

## Where this file is referenced

| File | Where | Why |
|---|---|---|
| `03_Setup_VMs_MA1.md` | Step 2.2.5 (between netplan and LAMP install) | CMS-Target needs internet for Drupal |
| `04_Setup_VMs_MA2.md` | Part 1 (ISP) Step 1.2 | ISP needs internet for dnsmasq + httpd |
| `04_Setup_VMs_MA2.md` | Part 6 (LinSRV1) Step 6.2 | LinSRV1 needs internet for httpd + realmd |
| `07_Setup_SecurityOnion.md` | Phase 6 (so-setup) | SO needs internet for Docker images |
| `41_Day2_MalwareIR.md` | MalwareLab build | Win 10 needs internet for security tools |

---

# Troubleshooting

| Problem | What's wrong | Fix |
|---|---|---|
| `PG-TempInternet` doesn't appear in the NIC drop-down when editing a VM | You created the port group on the wrong vSwitch | Delete it → create again on **vSwitch0** (the one with the physical NIC) |
| VM gets a DHCP IP but `ping 8.8.8.8` fails | Your TP-Link router doesn't have internet, OR vSwitch0 has no physical uplink | Test on PC1 first — does PC1 have internet? If yes, check ESXi → Networking → Virtual switches → vSwitch0 → "Physical adapters" |
| `ping 8.8.8.8` works but `ping archive.ubuntu.com` fails | DNS not configured | On Ubuntu run `cat /etc/resolv.conf` — should list `127.0.0.53` (systemd-resolved). If empty, run `sudo systemctl restart systemd-resolved` |
| Forgot to back up the original netplan, can't remember the static IP | You're stuck in DHCP mode | Look at this guide's worked example for `192.168.2.1`. Or check the VM specs in `03_…` (CMS-Target = `192.168.2.1`) or `04_…` (ISP, LinSRV1, etc.) |
| VM still on `PG-TempInternet` after install | Forgot Stage F | ESXi UI → VM → Edit → NIC → switch back to the original port group → Save |
| SSH session disconnected during Stage B | Expected — your IP changed when DHCP took over | Reconnect with the new IP, or use the VM console window directly |
| Took a snapshot earlier (Stage E) and want to go back | You can revert to that point | ESXi UI → VM → Actions → Snapshots → Manage → revert to `<VM>-after-install` |

---

# TL;DR — the 7-stage flow

For any VM that needs temp internet:

| # | Stage | Action |
|---|---|---|
| 1 | Stage A | ESXi UI → VM → Edit → NIC → switch to `PG-TempInternet` → Save |
| 2 | Stage B | Inside VM → switch to DHCP (B.1 Ubuntu / B.2 CentOS / B.3 Kali / B.4 Windows) |
| 3 | Stage C | Verify with `ping 8.8.8.8` (must get 3 replies) |
| 4 | Stage D | Run your install commands |
| 5 | Stage E | Snapshot: `<VM>-after-install` |
| 6 | Stage F | Restore static IP → power off → switch NIC back to sealed port group → power on |
| 7 | Stage G | Snapshot final clean state: `<VM>-base` or `<VM>-vulnerable` |

Done. The VM is fully installed AND sealed in its isolated practice port group.
