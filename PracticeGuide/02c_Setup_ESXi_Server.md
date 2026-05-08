# 02c — Setup the ESXi Server (the 3rd PC / system unit)

End-to-end install of VMware ESXi 8 on the 3rd box. Use this when you have the dedicated ESXi server hardware and are ready to graduate from single-PC practice (`02b_…`) to the full 3-box rig (`02_…`).

> Time: ~90 minutes total. Most of it is install + first-boot updates. The actual click-through of the installer is 10 minutes.

---

## A. Pre-flight — hardware compatibility

ESXi 8 is **picky about hardware**. Before burning time on a failed install, verify:

### A.1 CPU
- **Intel:** Sandy Bridge (2011) or later, with VT-x **and** EPT enabled in BIOS.
- **AMD:** Bulldozer or later, with AMD-V **and** RVI/NPT enabled.
- ESXi 8 specifically dropped support for some older CPUs that worked in ESXi 7. If yours is pre-2014, check VMware's HCL: `https://www.vmware.com/resources/compatibility/search.php?deviceCategory=cpu`.

### A.2 RAM
- Minimum: **8 GB** (ESXi itself needs 4 GB; remaining for one tiny VM).
- Realistic for our practice: **16 GB** (your 3rd PC has this) → ~12 GB available for VMs after ESXi overhead.
- Comfortable: 32 GB+ → can run all of MA2 simultaneously.

### A.3 Storage
- **Minimum 32 GB drive** for ESXi system + scratch.
- Datastore for VMs goes on remaining space.
- **Your 240 GB SSD reality:** ESXi takes ~32 GB → ~200 GB datastore. Workable for MA2 alone with thin-prov. See `01_…` Section 0 for the per-VM disk math.

### A.4 NIC — the most common gotcha
ESXi 8's built-in driver list **dropped many cheap consumer NICs**, including:
- **Realtek RTL8111/8168/8169** (very common on consumer motherboards)
- Most USB-Ethernet adapters

If your motherboard has only a Realtek NIC, ESXi will install but **show no network adapters** and you can't manage it. Three options:

| Option | What | Cost |
|---|---|---|
| **Add an Intel I210/I225 PCIe NIC** ✅ recommended | Cheap PCIe card with a supported Intel chip | ~₱700–1,000 |
| **Build a custom ESXi ISO with the Realtek driver injected** | Use PowerCLI to merge the community Realtek VIB into the stock ISO. **Full walkthrough: `02d_Setup_ESXi_Custom_ISO.md`** | Free (~45 min effort) |
| **Skip ESXi entirely, use Workstation Pro** | Run all VMs in VMware Workstation on Windows directly (per `02b_…`). Same practice value, no driver fight | Free |

**To check before committing:** Google your motherboard model + "ESXi 8 compatibility". Or check the VMware HCL link in A.1.

> 🚨 If you boot the stock ESXi installer and see *"No Network Adapters"* error → that's the Realtek issue. **Go directly to `02d_Setup_ESXi_Custom_ISO.md`** for the custom-ISO build, OR fall back to `02b_…` for Workstation Pro.

### A.5 BIOS / UEFI features required
You'll enable these in BIOS (next section):
- **Intel VT-x** (or **AMD-V**) ← virtualization
- **VT-d** (or **AMD-Vi**) ← needed for some advanced features, harmless to leave off
- **UEFI Boot mode** preferred (Legacy works too)
- **Secure Boot OFF** (ESXi installer not signed for some Secure Boot configs)
- **Hyper-Threading** ON (gives ESXi extra logical cores)

---

## B. Create the ESXi installer USB

You need a **bootable USB** with the ESXi 8 ISO. Done from any Windows PC (e.g. PC1 of your future rig, or your single practice PC).

### B.1 Materials
- USB stick: **8 GB minimum**, will be wiped.
- ESXi 8 ISO from `https://support.broadcom.com/group/ecx/productdownloads?subfamily=VMware%20vSphere` (free login required). The ISO is ~600 MB.
- **Rufus** from `https://rufus.ie/` (portable .exe, no install needed).

### B.2 Write the USB
1. Plug in the USB stick. Back up anything on it — it gets wiped.
2. Run `rufus-x.y.exe`.
3. *Device:* select your USB stick.
4. *Boot selection:* click **SELECT** → pick the ESXi ISO.
5. *Partition scheme:* **GPT** (for UEFI) or **MBR** (for Legacy BIOS). Match what your ESXi server will boot in.
6. *File system:* leave as default.
7. Click **START**.
8. When prompted "Write in DD Image mode?" → **Yes**.
9. Click OK to confirm wiping → wait ~3 min.
10. Eject when done.

> If Rufus throws a UEFI:NTFS prompt, accept it.

---

## C. BIOS prep on the ESXi server

Plug the USB into the 3rd PC. Don't power on yet.

### C.1 Enter BIOS
Power on → press the BIOS hotkey **immediately** (typically **F2**, **Del**, **F10**, or **Esc** — depends on motherboard brand). Spam it during the splash logo.

| Motherboard brand | Common hotkey |
|---|---|
| Gigabyte | **Del** or **F2** |
| ASUS | **Del** or **F2** |
| MSI | **Del** |
| ASRock | **F2** or **Del** |
| Dell | **F2** |
| HP | **F10** |
| Lenovo | **F1** or **F2** |

### C.2 Settings to enable

#### C.2.1 — Generic settings table (matches any vendor)

| Setting | What it does | Value |
|---|---|---|
| Intel Virtualization Technology (VT-x) / AMD-V (SVM) | Required for VMware to use hardware acceleration | **Enabled** |
| Intel VT-d / IOMMU | Lets VMs do DMA (improves disk/network perf, harmless if off) | **Enabled** |
| Hyper-Threading | Doubles logical core count | **Enabled** |
| Secure Boot | Cryptographically verifies bootloader — **blocks ESXi installer** | **Disabled** |
| Boot Mode (UEFI vs Legacy/CSM) | Disk partitioning style | **UEFI** preferred (or Legacy if you wrote USB as MBR in Rufus) |
| Fast Boot | Skips BIOS POST | **Disabled** (so F12 boot menu works reliably) |
| Boot order | First boot device | **USB first** (or use one-time F12 menu) |
| Power on after AC loss | Auto-recover from blackouts | **On** |

#### C.2.2 — Gigabyte BIOS-specific path (your motherboard)

Your Gigabyte board has these top tabs: `M.I.T. | System | BIOS | Peripherals | Chipset | Power | Save & Exit`. Here's the exact path for each setting:

**Step 1 — Enable Intel VT-x (the critical one)**
1. Top tab → **M.I.T.** (it's where your screen currently is — Motherboard Intelligent Tweaker).
2. Scroll down → highlight **Advanced CPU Core Settings** → press **Enter**.
3. Inside, scroll to find one of these (name varies by Gigabyte BIOS version):
   - **Intel Virtualization Technology** → set to **Enabled**, OR
   - **VT-x** → **Enabled**, OR
   - **SVM Mode** (if AMD CPU) → **Enabled**
4. Press **Enter** → choose **Enabled** → press **Enter** to confirm.

**Step 2 — Enable VT-d (recommended)**
- Same screen (Advanced CPU Core Settings):
- **VT-d** OR **Intel VT-d (Direct I/O)** → **Enabled**.
- If not visible there, try **Chipset** tab → **VT-d** option.

**Step 3 — Verify Hyper-Threading is on**
- Same screen:
- **Hyper-Threading Technology** OR **Intel(R) Hyper-Threading Technology** → **Enabled** (default — verify).

**Step 4 — Disable Secure Boot (only if doing bare-metal ESXi install)**
- Press **Esc** to back out to top tabs.
- Top tab → **BIOS**.
- Scroll to **Secure Boot** entry. On many Gigabyte boards you must first set:
  - **OS Type** → **Other OS** (this unlocks Secure Boot toggle)
  - Then **Secure Boot** → **Disabled**
- Or sub-menu: **Secure Boot → Secure Boot Mode → Custom → Secure Boot Enable → Disabled**.

**Step 5 — Boot mode + Fast Boot**
- Same **BIOS** tab:
- **Boot Mode**: leave as **UEFI** (default; matches GPT in Rufus).
- **CSM Support**: **Disabled** (since UEFI).
- **Fast Boot**: **Disabled**.
- **Boot Option Priorities**: put your USB first if you want it to boot automatically (or just use F12 boot menu — easier).

**Step 6 — Power on after AC loss**
- Top tab → **Power**.
- **AC BACK** OR **Power On after Power Failure** → **Always On** (or "Last State" if you want it to remember).

**Step 7 — Save and exit**
- Top tab → **Save & Exit**.
- **Save & Exit Setup** (or press **F10**) → **Yes**.
- PC reboots.

#### C.2.3 — Generic vendor-specific paths (if not Gigabyte)

| Vendor | VT-x location | Secure Boot location |
|---|---|---|
| **ASUS** | Advanced → CPU Configuration → Intel Virtualization Technology | Boot → Secure Boot → OS Type → Other OS, then Secure Boot Control → Disabled |
| **MSI** | OC → CPU Features → Intel Virtualization Tech | Settings → Advanced → Windows OS Configuration → Secure Boot → Disabled |
| **ASRock** | Advanced → CPU Configuration → Intel Virtualization Technology | Security → Secure Boot → Disabled |
| **Dell OptiPlex / Precision** | Virtualization Support → Virtualization → Enabled | Secure Boot → Secure Boot Enable → Disabled |
| **HP** | Advanced → System Options → Virtualization Technology (VTx) | Advanced → Secure Boot Configuration → Disabled |
| **Lenovo ThinkCentre** | Advanced → CPU Setup → Intel Virtualization Technology | Security → Secure Boot → Disabled |

> 🔑 **Universal trick:** if you can't find a setting, use the BIOS search feature (most modern UEFI BIOSes — usually F-key or `/` shortcut) and type "virtualization" or "secure boot".

#### C.2.4 — Which BIOS changes you actually need (depends on your install path)

| Install path | BIOS changes required |
|---|---|
| **Bare-metal ESXi** (replaces Windows on the 3rd PC) | All of Steps 1–7 above |
| **Nested ESXi** (ESXi as a VM inside VMware Workstation on the 3rd PC) | Only Steps 1 + 2 (VT-x + VT-d). Skip Secure Boot, boot mode, USB priority. |
| **Workstation directly** (no ESXi at all — single-PC mode per `02b_…`) | Only Step 1 (VT-x). |

After applying, **Save & Exit (F10)** and reboot.

### C.3 Boot from USB (only for bare-metal ESXi install)

At the Gigabyte splash logo on next reboot:
- Press **F12** repeatedly → boot menu appears → arrow-key to your USB → Enter.
- (If F12 doesn't work, try **F11**. Some older Gigabyte boards use F12 only in UEFI mode, F11 in Legacy.)

> Skip this step entirely if you're using nested ESXi or single-PC Workstation mode.

---

## D. ESXi installer wizard (click-by-click)

Boots into a yellow/black VMware splash, then the installer.

### D.1 Welcome screen
*Welcome to the VMware ESXi 8.x.x Installation* → **Enter** to continue.

### D.2 EULA
*Esc* skips, **F11** accepts. Press **F11**.

### D.3 Select disk
List of detected drives. Pick your **240 GB SSD**. **Enter**.

> ⚠️ If the installer says "Contains an existing VMFS / VMware datastore" → ESXi will preserve it (Upgrade option) or wipe it (Install option). For a fresh install, choose **Install** → *Overwrite VMFS datastore*.

### D.4 Keyboard layout
**US Default** (or your preference). **Enter**.

### D.5 Set root password
Pick a strong one. **Write it down.** Suggested: `Cyber@Manila2025!` or whatever you'll remember (must be 7+ chars, mix of cases/numbers/symbols).
Re-enter to confirm. **Enter**.

### D.6 CPU compatibility scan
Installer checks your CPU. If it says *"This CPU support has been deprecated and will not be supported in a future release"* — that's a warning, not an error. **F11** to proceed (it works for ESXi 8).
If it says *"This CPU is not supported"* — your CPU is too old. Either swap hardware or fall back to Proxmox VE.

### D.7 Confirm install
Final confirmation: *"The installer will erase the disk you selected"*. **F11** to install.

### D.8 Wait
Progress bar. ~5 min. When done: *"Installation complete. Remove installation USB. Press Enter to reboot."*

### D.9 First boot
Yank the USB *before* it reboots. Press **Enter**.
After reboot: yellow/grey ESXi splash with version + IP info.
- If DHCP found a server, you'll see an IP at the top: `https://<IP>/`.
- If not, IP shows as `0.0.0.0` — that's fine, we'll set static next.

---

## E. Post-install network configuration

Already partially documented in `02_…` Section A — repeated here for completeness.

### E.1 Set the management network static IP
At the ESXi console (yellow/grey screen):
1. **F2** → log in as `root` / your password.
2. *Configure Management Network → Network Adapters*. Confirm at least one vmnic is selected (the physical NIC).
3. *VLAN (optional)* → leave blank.
4. *IPv4 Configuration* → *Set static IPv4 address and network configuration*:
   - IPv4 Address: `192.168.1.10`
   - Subnet Mask: `255.255.255.0`
   - Default Gateway: `192.168.1.1` (your TP-Link router)
5. *DNS Configuration*:
   - Primary DNS: `8.8.8.8`
   - Alternate DNS: `1.1.1.1`
   - Hostname: `esxi-team1`
6. *Custom DNS Suffixes*: leave blank.
7. **Esc** → **Y** to apply and restart management network.
8. Verify: console top shows `https://192.168.1.10/` (or whatever IP you set).

### E.2 Test from PC1
From PC1 (on the same TP-Link switch + DHCP from same router):
```cmd
ping 192.168.1.10
```
Should reply. If not, check the cable and switch link lights.

---

## F. Web UI activation

### F.1 First login
From PC1 browser → `https://192.168.1.10/ui` (note: **NOT** `/`, it's `/ui`).
- Cert warning is expected (self-signed) — accept and proceed.
- Log in: `root` / your password.

### F.2 Apply the licence
Out of the box, ESXi runs in **60-day evaluation** with full features.

To get the free hypervisor licence (extends past 60 days, but limits some features):
1. *Manage → Licensing → Assign license*.
2. Paste the licence key from the Broadcom portal (or skip — eval mode covers your competition prep window).

> 💡 Honestly: for 2 weeks of practice + competition week, **skip the licence — eval mode is fine and has more features**.

### F.3 Verify the datastore
*Storage → Datastores* tab. You should see one entry like `datastore1` with ~200 GB free (the unused SSD space).

If no datastore appears: *New datastore → New VMFS datastore* → name `datastore1` → pick the SSD → use full disk → Finish.

### F.4 Networking — create the port groups
This is the key step that links to the rest of the guide.

Follow `02_…` **Section B steps 1–5** to create:
- vSwitch + port group `PG-Internet`
- vSwitch + port group `PG-LAN`
- vSwitch + port group `PG-DMZ`
- vSwitch + port group `PG-Servers`
- vSwitch + port group `PG-MA1-LAN`

**Promiscuous mode (optional):** if you ever want to capture traffic with Wireshark from a VM that needs to see other VMs' packets:
- *Port Group → Edit settings → Security* → set **Promiscuous mode = Accept**.
- Not required for the MA1/MA2/CTF deliverables, but useful for troubleshooting.

---

## G. ESXi specific tips for the 240 GB SSD

Your storage is tight. Apply these rules:

### G.1 Always thin-provision
When creating any VM: *Disk Provisioning → Thin*. Default is "Same as source" or "Thick lazy zeroed" depending on path — change to thin every time.

### G.2 Delete snapshots aggressively
Each snapshot can grow to the size of the VM disk. Snapshots are great for "before risky change" but **delete them within hours**, not days. *VM → Snapshots → Delete All*.

### G.3 Power off and **un-register** old VMs
After a session, if you don't need the VM for a while:
- Power off, then *Right-click → Unregister* (NOT Delete from disk).
- This removes it from inventory but keeps the files.
- Frees no disk but reduces memory overhead during boot.

### G.4 Monitor datastore usage
*Storage → datastore1 → Datastore browser* — watch the **Used** vs **Capacity**. When **Used > 80%**, the VMs will start mis-behaving on writes.

If you fill up:
- Delete unused snapshots first.
- Delete unused VMs (right-click → Delete from disk).
- Move heaviest VMs to PC1's local Workstation install (export OVA → import on PC1).

---

## H. Common installation issues + fixes

| Symptom | Cause | Fix |
|---|---|---|
| Installer boots but no disks listed | NVMe driver missing for your motherboard | Try ESXi 8.0 U2 (newer drivers) or U3; if still no, build community ISO with the missing driver |
| Installer boots but no NICs | Realtek consumer NIC not supported | See A.4 — install Intel PCIe NIC or use community ISO |
| "This CPU has been deprecated" warning | Your CPU is borderline; works on ESXi 8 but may not on future versions | Proceed anyway; revisit when ESXi 9 ships |
| "This CPU is not supported" hard error | CPU too old | Swap hardware or use Proxmox VE 8 |
| After install, no IP on console | DHCP server didn't reply | Set static IP per Section E.1 |
| Web UI returns "ERR_SSL_PROTOCOL_ERROR" | Browser too strict on the self-signed cert | Try Firefox; or Chrome → *Advanced → Proceed (unsafe)* |
| Web UI very slow / times out | Memory pressure (host has too little RAM for VMs running) | Power off some VMs |
| Can't ping ESXi from PC1 | Subnet mismatch, or PC1's NIC is on a different VMnet | Match subnets; verify with `ipconfig` on PC1 vs ESXi static IP |
| Booting takes 5+ minutes | Failed disk in the system; ESXi sweeps then times out | Remove the bad disk, or *boot* options → exclude it |

---

## I. Verification checklist

Before declaring the ESXi server "ready", confirm all:

- [ ] Console shows `https://192.168.1.10/` (or whatever IP you set)
- [ ] PC1 can `ping 192.168.1.10` — replies
- [ ] PC1 browser opens `https://192.168.1.10/ui` and you can log in as `root`
- [ ] *Storage → Datastores* shows at least one datastore with ~200 GB free
- [ ] *Networking → Port groups* shows the 5 port groups (PG-Internet, PG-LAN, PG-DMZ, PG-Servers, PG-MA1-LAN)
- [ ] (Optional) Promiscuous mode set to **Accept** on any port group you'll capture from with Wireshark
- [ ] You took a note of the root password in the team's password file
- [ ] You set "Power on after AC loss" in BIOS so the server auto-recovers
- [ ] Time on ESXi is correct: *Manage → System → Time & date* — set NTP server `pool.ntp.org` and sync

When all ticked → return to `02_Setup_Topology.md` Section B (creating vSwitches/port groups) if you haven't already, then move on to `03_Setup_VMs_MA1.md` to build the first VMs on this datastore.

---

## J. Backup the ESXi config (do once, after setup is done)

Lose the ESXi config and you re-do everything. To export:

```
ssh root@192.168.1.10
vim-cmd hostsvc/firmware/sync_config
vim-cmd hostsvc/firmware/backup_config
```
The output gives you a URL like `http://192.168.1.10/downloads/.../configBundle-esxi-team1.tgz`. Open in browser, save the `.tgz` to your USB.

To restore on a re-install: enter ESXi maintenance mode, then `vim-cmd hostsvc/firmware/restore_config /tmp/configBundle.tgz` (after copying the file in via scp).

> Take this backup **after** you've created the port groups + datastore, **before** you build the practice VMs. That snapshot of the ESXi config is your insurance.

Next file: **`08_Setup_Workstations_HostOS.md`** — the PC1/PC2 host prep + best-practice addendum.
