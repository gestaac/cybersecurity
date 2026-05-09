# 09 — Give a VM Temporary Internet (for installs / updates)

**What this file solves:** how to give a VM internet access just long enough to install packages (`apt install`, `dnf install`, Windows Updates, Drupal download, etc.) — then take that access away again so the VM stays isolated for practice.

**Skill level assumed:** none. We walk through every click.

**Time:** 5 min one-time setup + 2 min per VM swap.

---

## Why this file exists

Some of our practice VMs need to be on **isolated port groups** so they can't reach the internet:

| VM | Port group | Why isolated |
|---|---|---|
| CMS-Target (Day 1 MA1 pentest target) | PG-MA1-CMS | Deliberately vulnerable — must NOT phone home |
| LinSRV1 (Day 2 MA2 web server in DMZ) | PG-DMZ | Sealed DMZ for hardening practice |
| Client1, Client2 (LAN clients) | PG-LAN | Sealed LAN for AD testing |
| WINSRV1/3/4 (Servers VLAN) | PG-Servers | Internal only |
| Any VulnHub VM | PG-CTF or PG-MA1-CMS | Vulnerable, must stay sealed |

But during **initial setup**, these same VMs need internet to:
- Run `apt install` / `dnf install` for packages
- `wget` source code or installer files
- Run Windows Update
- Pull Docker images (for Security Onion)
- Install OpenVPN client packages

The trick: **temporarily move the VM's NIC to a port group that does have internet → install → move it back**.

---

## The architecture (read once, then ignore)

Your ESXi server has these vSwitches:

```
vSwitch0 (management)         ← has the physical NIC connected to TP-Link → INTERNET
   ├── Management Network     ← ESXi's own management IP (192.168.1.10)
   └── (you'll add: PG-TempInternet)

vSwitch-Internet              ← isolated, no physical uplink (just for ISP VM in MA2)
   └── PG-Internet

vSwitch-LAN                   ← isolated
   └── PG-LAN

vSwitch-DMZ                   ← isolated
   └── PG-DMZ

vSwitch-Servers               ← isolated
   └── PG-Servers

vSwitch-MA1                   ← isolated
   └── PG-MA1-CMS
```

**The only vSwitch that has internet is `vSwitch0`** — because that's the one with the physical NIC plugged into your TP-Link router.

So we add **one port group on vSwitch0 named `PG-TempInternet`**. Any VM with a NIC on `PG-TempInternet` can reach the internet via your TP-Link router.

---

## ONE-TIME SETUP — Create `PG-TempInternet` (5 min)

**Do this once. It stays in your ESXi config forever.**

### Step 1 — Open the ESXi UI

On PC1 → browser → `https://192.168.1.10/ui` → log in as `root`.

### Step 2 — Create the port group

1. Click **Networking** in the left sidebar.
2. Click the **Port groups** tab at the top.
3. Click **+ Add port group**.
4. Fill in:
   - **Name:** `PG-TempInternet`
   - **VLAN ID:** `0`
   - **Virtual switch:** `vSwitch0` ← critical — this is the one with the physical NIC
5. Click **Add**.

✅ Port group `PG-TempInternet` now appears in the list.

### Step 3 — (Optional) Confirm vSwitch0 has a physical uplink

1. Click the **Virtual switches** tab.
2. Click `vSwitch0`.
3. On the diagram, you should see at least one entry under **Physical adapters** like `vmnic0` (your physical NIC).
4. If `Physical adapters` is empty, your ESXi has no physical uplink and this whole approach won't work. (Talk to me if so.)

---

## PER-VM USE — How to give a VM internet for install (2 min per VM)

Repeat this whenever a VM needs internet temporarily.

### Stage A — Switch the VM's NIC

1. ESXi UI → **Virtual Machines** → click the VM (e.g., `CMS-Target`).
2. Click **Edit** at the top.
3. Find **Network Adapter 1** → drop-down → change from the isolated port group (e.g., `PG-MA1-CMS`) to **`PG-TempInternet`**.
4. Click **Save**.

The VM is now on the same virtual cable as your TP-Link router.

### Stage B — Make the VM use DHCP (different per OS)

The VM still has its old static IP (e.g., `192.168.2.1`) which doesn't match the TP-Link's subnet (`192.168.1.x`). We need to switch it to DHCP so it grabs a working IP from the TP-Link.

#### B.1 — Linux (Ubuntu / netplan)

Inside the VM console, edit the netplan file:

```bash
sudo nano /etc/netplan/00-installer-config.yaml
```

Replace contents with this simple DHCP version (adjust `ens34` to YOUR interface name):

```yaml
network:
  version: 2
  ethernets:
    ens34:
      dhcp4: true
```

Save: **Ctrl+O → Enter → Ctrl+X**.

Apply:
```bash
sudo chmod 600 /etc/netplan/00-installer-config.yaml
sudo netplan apply
```

📝 **Important: write down your ORIGINAL netplan content somewhere** so you can restore it later. Better yet, before editing, back it up:

```bash
sudo cp /etc/netplan/00-installer-config.yaml /etc/netplan/00-installer-config.yaml.STATIC.bak
```

Then to restore later:
```bash
sudo mv /etc/netplan/00-installer-config.yaml.STATIC.bak /etc/netplan/00-installer-config.yaml
```

#### B.2 — Linux (CentOS Stream / nmcli)

Inside the VM:
```bash
sudo nmcli con mod ens160 ipv4.method auto
sudo nmcli con mod ens160 ipv4.addresses ""
sudo nmcli con mod ens160 ipv4.gateway ""
sudo nmcli con up ens160
```

To restore the static IP later:
```bash
sudo nmcli con mod ens160 ipv4.addresses 192.168.2.1/24 ipv4.method manual ipv4.gateway 192.168.2.254 ipv4.dns 8.8.8.8
sudo nmcli con up ens160
```

(Substitute your VM's actual interface name and target IP.)

#### B.3 — Windows (Win Server / Win 10)

1. Inside the VM, right-click the **Start menu → Network Connections**.
2. Click **Change adapter options**.
3. Right-click your ethernet adapter → **Properties**.
4. Scroll → **Internet Protocol Version 4 (TCP/IPv4)** → **Properties**.
5. Tick **Obtain an IP address automatically** + **Obtain DNS server address automatically**.
6. Click **OK** twice.
7. The adapter renews from DHCP. Verify in PowerShell:
   ```powershell
   ipconfig
   ```

   You should see an IP in the `192.168.1.x` range from your TP-Link.

To restore later: same steps, but pick **Use the following IP address** with your original static settings.

### Stage C — Verify internet works

Inside the VM:

**Linux:**
```bash
ping -c 3 8.8.8.8
ping -c 3 archive.ubuntu.com    # or mirror.centos.org
```

**Windows:**
```powershell
ping 8.8.8.8
ping www.microsoft.com
```

If all 3 replies → ✅ you have internet. Proceed to Stage D.

If pings fail:
- Confirm you switched the NIC in ESXi (Stage A).
- Confirm the VM took DHCP (Stage B).
- Confirm your TP-Link router is on and PC1 has internet too (test on PC1).
- Try `sudo netplan apply` again or `ipconfig /renew` on Windows.

### Stage D — Run your install

Now you can run all the install commands the original guide asks for:
- `apt update && apt install -y ...` (Ubuntu)
- `dnf install -y ...` (CentOS)
- `wget <url>` for source code or installers
- Windows Update
- `npm install`, `pip install`, etc.

Take however long you need. The VM has full internet.

### Stage E — Snapshot before reverting

**Important:** Before switching back to the isolated port group, **snapshot the VM**.

ESXi UI → VM → **Actions → Snapshots → Take snapshot** → name it `<VM>-after-install`.

This saves the "fully installed but still on internet" state so you can roll back if Stage F goes wrong.

### Stage F — Switch BACK to the isolated port group

1. **Inside the VM:** restore the original static IP config.
   - Linux Ubuntu: restore the netplan backup (`sudo mv .../*.STATIC.bak .../...yaml`) OR re-edit nano with the static YAML.
   - Linux CentOS: re-run the `nmcli` static commands from B.2.
   - Windows: switch the IPv4 adapter back to "Use the following IP address" with your static settings.

2. **Power off the VM** (cleanest):
   ```bash
   sudo shutdown now
   ```
   (or Windows: Start → Power → Shut down)

3. **ESXi UI:** select the VM → Edit → **Network Adapter 1** → drop-down back to the isolated port group (e.g., `PG-MA1-CMS`) → Save.

4. **Power the VM on** → log in.

5. **Verify the static IP is back:**
   - Linux: `ip a show ens34` → should show your original static IP (e.g., `192.168.2.1/24`)
   - Windows: `ipconfig`

✅ The VM is now back on the isolated port group with its original IP.

### Stage G — Snapshot the FINAL state

ESXi UI → VM → Actions → Snapshots → Take snapshot → name it `<VM>-base` or `<VM>-ready`.

This is your **clean restore point** before competition-style practice begins.

---

## Quick reference table — when each VM needs this

| VM | When you need this | What you install during temp internet |
|---|---|---|
| **CMS-Target** | Once during initial build | LAMP stack (Apache, MariaDB, PHP) + Drupal 7 tarball |
| **Kali** | During initial ISO install (recommended) + occasional `apt update` afterwards | Mirror config, security updates, sqlmap/wpscan updates |
| **ISP** (MA2) | Once during initial build | dnsmasq, httpd, mod_ssl |
| **LinSRV1** (MA2) | Once during initial build | httpd, realmd, sssd, krb5, libpwquality |
| **WINSRV1/3/4** (MA2) | Optional during initial install (for Windows Updates) | Windows Updates, RSAT tools |
| **Client1/2/3** (MA2) | Optional | Chrome, PuTTY, Wireshark, Nmap, OpenVPN Connect |
| **SecOnion** | During `so-setup` | ~10 GB Docker images + ETOPEN ruleset (already documented in `07_…` Phase 6) |
| **MalwareLab** | Once during initial install | PEStudio, CFF Explorer, DiE, Process Hacker, Wireshark, 7-Zip |
| **VulnHub VMs** | Never (deliberately offline) | — |

---

## Common issues

| Problem | Fix |
|---|---|
| `PG-TempInternet` doesn't appear in the NIC drop-down | You created it on the wrong vSwitch. Delete + recreate on **vSwitch0**. |
| VM gets DHCP but still can't reach internet | TP-Link router internet is down. Test on PC1 first. |
| `ping 8.8.8.8` works but `ping archive.ubuntu.com` doesn't | DNS issue. On Ubuntu run `sudo systemd-resolve --status` and check the DNS server is `192.168.1.1`. |
| Forgot to back up the original netplan and can't remember the static IP | Look at `02_Setup_Topology.md` Section C / D for the canonical IP plan, or `04_Setup_VMs_MA2.md` Table 1. |
| VM still on `PG-TempInternet` after install — forgot to switch back | Power off → Edit VM → switch NIC → power on. Or use snapshot to revert if you took one. |
| Network restart broke SSH session | Expected — your IP changed. Reconnect with the new IP, OR work directly in the VM console. |

---

## How this fits in the bigger picture

| File | When it references this | Why |
|---|---|---|
| `03_Setup_VMs_MA1.md` | Step 2.2.5 (between netplan and LAMP install) | CMS-Target needs internet for Drupal install |
| `04_Setup_VMs_MA2.md` | Part 1 (ISP) Step 1.2 | ISP needs internet for dnsmasq + httpd packages |
| `04_Setup_VMs_MA2.md` | Part 6 (LinSRV1) Step 6.2 | LinSRV1 needs internet for httpd + realmd packages |
| `07_Setup_SecurityOnion.md` | Phase 6 (so-setup) | Security Onion needs internet for Docker images |
| `41_Day2_MalwareIR.md` | MalwareLab build | Win 10 needs internet for security tools |

---

## TL;DR — the 7-step recipe

For any VM that needs to install packages during initial setup:

1. **Stage A:** ESXi UI → VM → Edit → NIC → switch to `PG-TempInternet` → Save
2. **Stage B:** Inside VM → switch network config to DHCP (`netplan` or `nmcli` or Windows GUI)
3. **Stage C:** Verify with `ping 8.8.8.8`
4. **Stage D:** Run your install commands
5. **Stage E:** Take a snapshot (`<VM>-after-install`)
6. **Stage F:** Restore static IP → power off → switch NIC back to isolated port group → power on
7. **Stage G:** Take a snapshot of the final state (`<VM>-base`)

Done. Move on to the next file.
