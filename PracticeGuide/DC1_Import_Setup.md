# DC-1 — Import to ESXi + Network Setup (Beginner Guide)

> Use this **once** to get DC-1 running. After it's set up + snapshotted, you can practise the v2 MA1 walkthrough as many times as you want.
>
> **Total time:** 20-30 min.

---

## Before you start — what you have

- File: `DC-1.zip` (downloaded from vulnhub.com — about 700 MB)
- Inside the zip: `DC-1.ova` (the actual VM file)
- ESXi server already running with your existing Kali VM

## What you'll do

1. Extract the zip → get `DC-1.ova`
2. Upload `DC-1.ova` to ESXi
3. Put it on the **same network as Kali** so they can talk
4. Power it on, find its IP
5. Test from Kali (`ping`, `nmap`)
6. Snapshot it (so you can reset to a clean state for repeat practice)

---

## Step 1 — Extract DC-1.zip (1 min)

On your PC:
1. Right-click `DC-1.zip` → **Extract All** → choose a folder (e.g. `D:\OVA\DC-1\`)
2. Inside, you should see `DC-1.ova` (about 700 MB)
3. Done.

> 🧠 **Don't double-click DC-1.ova** — that opens VMware Workstation, which we don't want. We want it on **ESXi**.

---

## Step 2 — Upload + import to ESXi (10 min)

### 2a. Open ESXi web UI
1. Browser → `https://<your-esxi-ip>` (probably `https://192.168.10.10/` per your topology)
2. Accept the self-signed cert warning
3. Login: `wsauser / Andres@9V4` (per topology) or whatever you set
4. You're in the ESXi web console

### 2b. Start the import wizard
1. Left sidebar → **Virtual Machines**
2. Top button bar → **Create / Register VM**
3. A wizard opens

### 2c. Walk through the wizard

| Step | Setting |
|---|---|
| 1. Select creation type | **Deploy a virtual machine from an OVF or OVA file** → Next |
| 2. Select OVF and VMDK files | **Name:** `DC-1` <br>Click the file box → browse → select `DC-1.ova` <br>Wait for upload bar (~1-2 min) → Next |
| 3. Select storage | Pick your local datastore (usually only one) → Next |
| 4. Deployment options | **Network mappings:** ⭐ pick **the same port group your Kali VM uses** (likely `VM Network` or `PG-MA1-CMS`). This is critical — see Step 3 below. <br>**Disk provisioning:** Thin <br>**Power on automatically:** ☐ unchecked (we'll power on manually after checking settings) <br>Next |
| 5. Ready to complete | Review → **Finish** |

ESXi imports the VM. This takes 1-3 min.

### 2d. If you get an error like "Unsupported hardware version"

DC-1 was exported from VirtualBox. ESXi sometimes complains. **Don't panic.** Try this fix:

```cmd
"C:\Program Files\VMware\VMware OVF Tool\ovftool.exe" --lax --allowAllExtraConfig "D:\OVA\DC-1\DC-1.ova" "D:\OVA\DC-1\DC-1-fixed.ovf"
```

This converts to a clean OVF. Re-upload `DC-1-fixed.ovf` (and the `.vmdk` it generates) via the same wizard.

### 2e. Verify it imported
- ESXi → Virtual Machines → you should see `DC-1` in the list with status "powered off"

---

## Step 3 — Confirm network port group (CRITICAL)

DC-1 must be on the **same network as your Kali VM** so they can talk to each other. If they're on different port groups, ping won't work and you'll be stuck.

### 3a. Find Kali's port group
1. ESXi → Virtual Machines → click on `kali` VM
2. Look at **Networking** section in the right pane
3. Note the port group name (e.g. `VM Network`, `PG-MA1-CMS`, etc.)

### 3b. Set DC-1 to the same port group
1. ESXi → Virtual Machines → right-click `DC-1` → **Edit settings**
2. Find **Network adapter 1**
3. Change the dropdown to **the same port group as Kali**
4. Click **Save**

> 🧠 **MEMORIZE THIS:** when adding ANY new VM in ESXi, always check its network port group matches the network you want it to talk to. This is the #1 cause of "VM is up but I can't reach it."

---

## Step 4 — Power on DC-1 (1 min)

1. ESXi → Virtual Machines → right-click `DC-1` → **Power → Power On**
2. Click on `DC-1` → top right click **Console** icon (or right-click → Open console)
3. You'll see DC-1 boot. Wait for the login prompt:

```
DC-1 login: _
```

> You don't need to log in here — DC-1 isn't running anything you need to configure interactively. It's a target box.

---

## Step 5 — Find DC-1's IP

DC-1 is configured for **DHCP** by default. The IP it gets depends on whether your network has a DHCP server.

### 5a. Try from Kali first (easiest)

From Kali (SSH into Kali from your PC, or use ESXi console for Kali):
```bash
sudo arp-scan -l
# OR if arp-scan isn't installed:
sudo nmap -sn 192.168.2.0/24
```

You'll see something like:
```
192.168.2.1   00:0c:29:aa:bb:cc    VMware, Inc.   ← old Drupal target (if still up)
192.168.2.2   00:0c:29:dd:ee:ff    VMware, Inc.   ← Kali itself
192.168.2.3   00:0c:29:11:22:33    VMware, Inc.   ← DC-1 (new!)
```

The new VMware MAC = DC-1. **Write its IP down.** Probably `192.168.2.3` or similar.

### 5b. If DC-1 has no IP (DHCP failed)

Some isolated port groups don't have a DHCP server. In that case, DC-1 sits there with no network. We'll set a static IP.

**On the DC-1 console**, login as: `root` / no-password-needed?

Actually, DC-1 doesn't give us root login. The trick: boot DC-1 into single-user mode to set IP, OR use Kali to brute-force its way in (which is the whole point of the box). For setup purposes, just **let DC-1 sit at the login prompt** and configure Kali's NIC to send DHCP requests on its segment.

**Easier solution if no DHCP**: ESXi → DC-1 → Edit Settings → power off → make sure it's on the same port group as your existing Drupal target (which already had network working). Power on. Should get an IP via... actually no, neither of them have DHCP.

**Cleanest fix**: run a tiny DHCP server on Kali for the isolated network:
```bash
# On Kali (only if DC-1 won't get an IP):
sudo apt install -y isc-dhcp-server

sudo nano /etc/dhcp/dhcpd.conf
# Add:
subnet 192.168.2.0 netmask 255.255.255.0 {
  range 192.168.2.50 192.168.2.150;
  option routers 192.168.2.2;
  option domain-name-servers 8.8.8.8;
}

sudo nano /etc/default/isc-dhcp-server
# Set: INTERFACESv4="eth0"   (or whatever NIC is on 192.168.2.0/24)

sudo systemctl restart isc-dhcp-server
```

Then **reboot DC-1** — it'll grab an IP from the new DHCP pool.

### 5c. Alternative — boot DC-1 into recovery mode and set static IP

If you prefer not to run DHCP on Kali:

1. Power off DC-1
2. Power on, hold **Shift** during GRUB to see boot menu
3. Choose **Advanced options** → **rescue mode**
4. At root prompt, edit network:
   ```bash
   ip addr add 192.168.2.3/24 dev eth0
   ip link set eth0 up
   echo "auto eth0" >> /etc/network/interfaces
   echo "iface eth0 inet static" >> /etc/network/interfaces
   echo "  address 192.168.2.3" >> /etc/network/interfaces
   echo "  netmask 255.255.255.0" >> /etc/network/interfaces
   reboot
   ```

Now DC-1 always boots with `192.168.2.3`.

---

## Step 6 — Test from Kali (2 min)

```bash
# Replace the IP with whatever you found in Step 5
export DC1=192.168.2.3

# Ping check
ping -c 4 $DC1
# Expect: 4 packets received

# Port scan
nmap -sV $DC1
# Expect: 22/ssh, 80/http (Drupal 7), 111/rpcbind
```

If you see Drupal 7 on port 80 — **success!** DC-1 is ready for practice.

If ping times out → check Step 3 (port group) and Step 5 (IP).

---

## Step 7 — Snapshot DC-1 (1 min — DO NOT SKIP)

Snapshots let you reset DC-1 to a clean state in 30 seconds for repeat practice runs.

1. ESXi → Virtual Machines → right-click `DC-1` → **Snapshots → Take snapshot**
2. **Name:** `DC-1-clean-baseline`
3. **Description:** `Pre-pentest, before any modification`
4. ☐ Snapshot the virtual machine's memory (uncheck — saves disk)
5. **Take snapshot**

Now whenever you want to redo MA1 practice from scratch:
- Right-click `DC-1` → **Snapshots → Manage Snapshots** → select `DC-1-clean-baseline` → **Restore**
- Power on → identical clean state in 30 sec

> 🧠 **MEMORIZE this habit:** snapshot every target VM before each practice run. It's the difference between 30 seconds of reset and 30 minutes of re-importing.

---

## Troubleshooting checklist

| Symptom | Likely cause | Fix |
|---|---|---|
| Upload fails midway | Browser timeout / large file | Use Firefox not Chrome; or use ovftool from CLI (Step 2d) |
| "Unsupported hardware version" | OVA from VirtualBox, ESXi picky | Run ovftool with `--lax` (Step 2d) |
| DC-1 boots but no IP | No DHCP on the port group | Run isc-dhcp-server on Kali (Step 5b) or set static IP (Step 5c) |
| Ping from Kali times out | DC-1 on different port group than Kali | Edit DC-1 settings, change network adapter to Kali's port group (Step 3) |
| nmap shows host but no services | DC-1 still booting, services not started | Wait 60 sec, retry |
| Web page shows "Service Unavailable" | Drupal/Apache not started | Wait — sometimes the box takes 90 sec to fully boot. If still broken, reboot DC-1. |

---

## What to remember for tomorrow

If chief gives you a different VulnHub VM tomorrow, the import flow is **identical** — only Step 5 (find IP) and Step 6 (test) might vary. Drill these steps once tonight on DC-1, and tomorrow's import takes 10 min flat.

---

## Next file

Once DC-1 is reachable from Kali at its IP (e.g. `192.168.2.3`):

→ **`10_Day1_MA1_Solution_v2_DC1.md`** — full v2 walkthrough.
