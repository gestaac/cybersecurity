# 02d — Install ESXi 8 on Bare Metal with Realtek NIC (Fresh-Start Guide)

This guide assumes you have **fresh Windows 10 installed on your build PC (PC1)** with **nothing else installed yet**. It walks you through every download, install, command, and verification step needed to get ESXi 8 running bare-metal on your 3rd PC, even though it has an unsupported Realtek NIC.

> **Source:** William Lam, Feb 2026 — `https://williamlam.com/2026/02/installing-realtek-network-driver-fling-using-free-esxi-8-0-update-3e-iso.html`

---

## ⚠️ Read this section FIRST — hardware compatibility check

The Gigabyte B360 HD3 motherboard's onboard Realtek NIC has subsystem ID `SUBSYS_E0001458` (Gigabyte Dragon rebrand). The Broadcom Realtek Fling 1.101.01 driver does **not** automatically claim this specific subsystem variant.

**Practical implication:** even with a perfectly-built modified ESXi installer, the bare-metal install on this motherboard will fail with **"There are no supported network interfaces on this host"** because the driver loads but doesn't bind to the chip.

### Required: ONE of these to make the install succeed

You **must** acquire one of the following before starting:

| Hardware | Cost (PH) | Delivery | Why it works |
|---|---|---|---|
| **USB 3.0 → Gigabit Ethernet adapter with ASIX AX88179 or Realtek RTL8153 chip** | ₱300–500 | Same-day in Manila | ESXi has supported drivers via the USB Network Native Driver Fling |
| **Intel I210-T1 PCIe x1 NIC card** ⭐ recommended | ₱600–900 | 1–2 days | ESXi has built-in native Intel I210 support — no driver hacking needed |

**Without one of these, the install will fail.** This is a hardware compatibility issue that no software workaround fixes reliably.

### Buying recommendations (Lazada/Shopee PH)

- **USB Ethernet (cheapest, fastest):** search "**USB 3.0 Gigabit Ethernet AX88179**" — pick a seller with same-day Manila delivery.
- **Intel I210-T1 (cleanest):** search "**Intel I210-T1**" or "**Intel EXPI9301CT**" — both ESXi-native.

> 💡 **Smart move:** order BOTH. USB adapter for tomorrow's practice, Intel card for the long-term competition rig. Total ~₱1,000.

---

## What you'll do in this guide

| Phase | What | Time |
|---|---|---|
| 1 | Install build tools on Windows 10 (7-Zip, Notepad++, Rufus) | 15 min |
| 2 | Create a Broadcom account + download ESXi ISO + Realtek Fling + USB Network Fling | 30 min |
| 3 | Extract drivers using 7-Zip command line | 10 min |
| 4 | Write USB and modify it with both drivers | 15 min |
| 5 | BIOS prep on the 3rd PC (Gigabyte B360 HD3) | 5 min |
| 6 | Install ESXi from the modified USB | 15 min |
| 7 | Post-install: copy drivers to bootbanks, edit boot.cfg | 10 min |
| 8 | Verify and configure network | 10 min |
| **Total** | | **~110 min** |

---

# Phase 1 — Install build tools on Windows 10 (15 min)

You need these three free tools on the build PC. Install in this order.

## 1.1 — 7-Zip (file extraction)

1. Open **Edge** browser on the fresh Windows 10.
2. Go to: **`https://www.7-zip.org/`**
3. Download the **64-bit x64 version** (top of the page, ~1.5 MB `.exe`).
4. Run the installer. Defaults are fine. Install path: `C:\Program Files\7-Zip\`.
5. Verify: open File Explorer → right-click any file → **7-Zip** menu should appear in the context menu.

## 1.2 — Notepad++ (BOOT.CFG editing — preserves Unix line endings)

⚠️ **DO NOT use regular Notepad.** Windows Notepad converts Unix LF line endings to Windows CRLF, which breaks `BOOT.CFG`.

1. Browser → **`https://notepad-plus-plus.org/downloads/`**
2. Download the latest version (current: 8.x), **64-bit Installer** (~5 MB).
3. Run installer with defaults.

## 1.3 — Rufus (USB writer)

1. Browser → **`https://rufus.ie/`**
2. Click **Rufus 4.x** (Standard version, ~1.5 MB portable `.exe`).
3. Save to `Downloads\Rufus.exe`. **No install needed** — it's portable.

## 1.4 — Create a working folder

Open File Explorer. Create:
```
D:\esxi-build\
```

(If your build PC has only a C: drive, use `C:\esxi-build\` everywhere instead. Adjust paths as you go.)

---

# Phase 2 — Downloads (30 min, mostly waiting)

You need three files — all from Broadcom's portal (free with registration).

## 2.1 — Create a Broadcom account (one-time, 5 min)

1. Browser → **`https://support.broadcom.com/`**
2. Click **Register** (top right).
3. Fill in email, name, phone (any valid info).
4. Verify email via the confirmation link Broadcom emails you.
5. Sign in with the new account.

## 2.2 — Download ESXi 8.0 Update 3 ISO (~640 MB, 10 min)

1. From the Broadcom portal home → search bar → type "**VMware vSphere Hypervisor 8**".
2. Click the result for **VMware vSphere Hypervisor (Free)**.
3. Pick the latest **8.0 Update 3** version (current: build 24677879 or 24585291 = U3e).
4. Accept the EULA → confirm export compliance → click the download link for the **ESXi installer ISO**.
5. Filename will be something like:
   ```
   VMware-VMvisor-Installer-8.0U3-24677879.x86_64.iso
   ```
6. Save to `D:\esxi-build\`.
7. **Copy the free license key** from the same download page — paste it into a text file `D:\esxi-build\license.txt` so you don't lose it.

## 2.3 — Download the Realtek Driver Fling (~250 KB, 2 min)

Same Broadcom portal.

1. Navigate to: **`https://support.broadcom.com/group/ecx/productdownloads?subfamily=Flings&freeDownloads=true`**
2. Find **Realtek Network Driver for ESXi**.
3. Latest version: **1.101.01** (Nov 2025).
4. Download the ZIP bundle. Filename like:
   ```
   VMware-Re-Driver_1.101.01-5vmw.800.1.0.20613240.zip
   ```
5. Save to `D:\esxi-build\`.

## 2.4 — Download the USB Network Native Driver Fling (~200 KB, 2 min)

This is what makes USB-Ethernet adapters work in ESXi.

1. Same Broadcom Flings page: **`https://support.broadcom.com/group/ecx/productdownloads?subfamily=Flings&freeDownloads=true`**
2. Find **USB Network Native Driver for ESXi**.
3. Pick the version compatible with **ESXi 8.0** (latest is usually 1.13 or newer).
4. Download the offline bundle ZIP. Filename like:
   ```
   ESXi800-VMKUSB-NIC-FLING-xxxxx-component.zip
   ```
5. Save to `D:\esxi-build\`.

After Phase 2, your `D:\esxi-build\` folder should contain:
```
D:\esxi-build\
├── VMware-VMvisor-Installer-8.0U3-...iso        (~640 MB)
├── VMware-Re-Driver_1.101.01-...zip             (~210 KB)
├── ESXi800-VMKUSB-NIC-FLING-...zip              (~200 KB)
└── license.txt                                   (your free ESXi license key)
```

Verify with PowerShell:
```powershell
cd D:\esxi-build
Get-ChildItem -File | Format-Table Name, Length
```

---

# Phase 3 — Extract drivers (10 min)

We'll extract `ifre.v00` (Realtek) and `vmkusb_nic_fling.v00` (USB-Ethernet).

## 3.1 — Extract the Realtek `.vib` from its ZIP

Open **PowerShell** (regular, non-admin is fine). Type:

```powershell
cd D:\esxi-build
& "C:\Program Files\7-Zip\7z.exe" e .\VMware-Re-Driver_1.101.01-*.zip
```

After extraction, your folder has:
```
vmw_bootbank_if-re_1.101.01-5vmw.800.1.0.20613240.vib    (~226 KB)
metadata.zip                                              (~3 KB)
```

## 3.2 — Extract the Realtek driver payload from the `.vib`

A `.vib` file is a Unix `ar` archive. 7-Zip handles it. Run:

```powershell
& "C:\Program Files\7-Zip\7z.exe" e .\vmw_bootbank_if-re_1.101.01-*.vib
```

After extraction:
```
vmw_bootbank_if-re_1.101.01-5vmw.800.1.0.20613240    (~1.2 MB, no extension — driver payload)
```

⚠️ **7-Zip naming quirk:** the extracted file inherits the long `.vib` filename without an extension. **This IS the driver — just renamed weirdly.**

Rename to `ifre.v00`:
```powershell
Rename-Item ".\vmw_bootbank_if-re_1.101.01-5vmw.800.1.0.20613240" "ifre.v00"
```

Verify:
```powershell
Get-Item .\ifre.v00 | Format-List Name, Length
```

Expected:
```
Name   : ifre.v00
Length : 1229357
```

## 3.3 — Extract the USB Network driver from its ZIP

```powershell
& "C:\Program Files\7-Zip\7z.exe" e .\ESXi800-VMKUSB-NIC-FLING-*.zip
```

After extraction, look for a file like:
```
vmw_bootbank_vmkusb-nic-fling_1.13-1vmw.x.x.x.vib    (~150 KB)
```

## 3.4 — Extract the USB Network driver payload

```powershell
& "C:\Program Files\7-Zip\7z.exe" e .\vmw_bootbank_vmkusb-nic-fling_*.vib
```

You'll get a payload file with a long name. Rename it:
```powershell
# Find the largest non-.vib file
Get-ChildItem -File | Where-Object { $_.Name -notmatch '\.(vib|zip|iso|xml|pkcs7|txt)$' } | Sort-Object Length -Descending | Select-Object Name, Length -First 5

# Rename whatever the largest payload is — substitute the actual filename:
Rename-Item ".\<long-filename>" "vmkusb_nic_fling.v00"
```

Verify:
```powershell
Get-Item .\vmkusb_nic_fling.v00 | Format-List Name, Length
```

Expected: ~600 KB to 1.5 MB.

## 3.5 — Cleanup leftover files (optional)

Your folder now has many extra files (descriptor.xml, sig.pkcs7, metadata.zip, etc.). The two we need are `ifre.v00` and `vmkusb_nic_fling.v00`. The rest can be deleted, but keep the two original `.zip` and `.vib` files as backups:

```powershell
Remove-Item .\descriptor.xml -ErrorAction SilentlyContinue
Remove-Item .\sig.pkcs7 -ErrorAction SilentlyContinue
Remove-Item .\metadata.zip -ErrorAction SilentlyContinue
```

After Phase 3, `D:\esxi-build\` has:
```
VMware-VMvisor-Installer-8.0U3-...iso          (~640 MB — stock ESXi installer)
ifre.v00                                        (~1.2 MB — Realtek driver)
vmkusb_nic_fling.v00                            (~700 KB — USB-Ethernet driver)
VMware-Re-Driver_1.101.01-...zip                (backup)
ESXi800-VMKUSB-NIC-FLING-...zip                 (backup)
vmw_bootbank_if-re_*.vib                        (backup)
vmw_bootbank_vmkusb-nic-fling_*.vib             (backup)
license.txt                                     (license key)
```

---

# Phase 4 — Create the modified USB installer (15 min)

## 4.1 — Plug in your USB stick

Use a USB stick **8 GB or larger**. Everything on it gets wiped.

## 4.2 — Write the stock ESXi ISO with Rufus

1. Run `Rufus.exe`.
2. **Device:** select your USB stick.
3. **Boot selection:** click **SELECT** → choose `VMware-VMvisor-Installer-8.0U3-...iso`.
4. **Partition scheme:** **GPT**.
5. **Target system:** **UEFI (non CSM)**.
6. **File system:** leave default (Rufus picks the right one).
7. Click **START**.
8. Prompt: *"Write in DD Image mode?"* → click **Yes**.
9. Confirm wipe → wait ~3 min.
10. **Don't eject yet** — we still need to add files.

## 4.3 — Browse the USB partitions

The DD-mode write creates multiple partitions on the USB. Windows will see one as a drive letter (e.g., `D:\` labelled `TESTING-ESXI`). The relevant partition is the **EFI boot partition** (FAT16/FAT32) which contains:

- Root files: `b.b00`, `k.b00`, `boot.cfg`, `*.v00`, etc.
- A folder `efi\boot\` with `boot.cfg`, `bootx64.efi`, etc.

If you can't see the partition:
- Check Disk Management (Win+X → Disk Management) — confirm the USB has at least 2 partitions.
- The boot partition is the small (~ a few hundred MB) FAT one.
- If it's not assigned a drive letter, right-click → Change Drive Letter and Paths → Add → assign one.

## 4.4 — Copy the two drivers to the USB

Suppose Windows assigned **`E:\`** to the boot partition (adjust to your actual letter).

Copy `ifre.v00` and `vmkusb_nic_fling.v00` to TWO locations on the USB:

```powershell
# Adjust E:\ to your actual USB drive letter
$usb = "E:\"

# Copy to USB root (alongside b.b00, etc.)
Copy-Item D:\esxi-build\ifre.v00 "$usb\ifre.v00"
Copy-Item D:\esxi-build\vmkusb_nic_fling.v00 "$usb\vmkusb_nic_fling.v00"

# Copy also to EFI\BOOT folder (for UEFI boot path)
Copy-Item D:\esxi-build\ifre.v00 "$usb\efi\boot\ifre.v00"
Copy-Item D:\esxi-build\vmkusb_nic_fling.v00 "$usb\efi\boot\vmkusb_nic_fling.v00"
```

## 4.5 — Edit BOTH `boot.cfg` files on the USB

There are two `boot.cfg` files on the USB:
- `E:\boot.cfg` (root)
- `E:\efi\boot\boot.cfg` (EFI boot path — used by modern UEFI BIOS)

**Edit BOTH.** Open each with **Notepad++** (NOT regular Notepad).

In each file, find the line beginning with `modules=`. It's a **single very long line** with module names separated by ` --- `. Append at the very end:

```
 --- /ifre.v00 --- /vmkusb_nic_fling.v00
```

So the line ends with:
```
... --- /imgpayld.tgz --- /ifre.v00 --- /vmkusb_nic_fling.v00
```

Save each file. Notepad++ auto-preserves Unix line endings.

## 4.6 — Safely eject the USB

In File Explorer, right-click the USB drive → **Eject**.

---

# Phase 5 — BIOS prep on the 3rd PC (5 min)

## 5.1 — Plug in your USB-Ethernet adapter (or confirm Intel NIC is installed)

If you bought a **USB Ethernet adapter:** plug it into a USB 3.0 port (blue inside) on the 3rd PC.

If you bought an **Intel I210-T1 PCIe card:** power off the 3rd PC, install the card in any free PCIe x1 slot, close the case.

## 5.2 — Enter BIOS

Power on the 3rd PC → spam **Del** key during the Gigabyte splash logo.

## 5.3 — Settings to enable

Navigate using arrow keys. On Gigabyte B360 HD3:

| Setting | Path | Value |
|---|---|---|
| Intel Virtualization Technology (VT-x) | **M.I.T. → Advanced CPU Core Settings** | **Enabled** |
| Intel VT-d | **M.I.T. → Advanced CPU Core Settings** | **Enabled** |
| Hyper-Threading | **M.I.T. → Advanced CPU Core Settings** | **Enabled** |
| Secure Boot | **BIOS → Secure Boot** (set OS Type → Other OS first if needed) | **Disabled** |
| Boot Mode | **BIOS** | **UEFI** |
| Fast Boot | **BIOS** | **Disabled** |
| AC BACK / Restore on AC Loss | **Power** | **Always On** (optional but recommended) |

Press **F10** → **Yes** to save and exit. PC reboots.

---

# Phase 6 — Install ESXi (15 min)

## 6.1 — Boot from the USB

When the Gigabyte splash logo appears after reboot, spam **F12** to open the boot menu. Select your USB stick → **Enter**.

## 6.2 — Watch for driver loads

The boot text scrolls fast. Look for these lines:
```
Loading /ifre.v00
Loading /vmkusb_nic_fling.v00
```

Both should appear without errors. If you see a "module not found" error → check that you copied the files to the USB root (Phase 4.4) and edited `boot.cfg` correctly (Phase 4.5).

## 6.3 — ESXi installer launches

You should see **"VMware ESXi 8.0.x Installer"** with a blue welcome screen.

If you still see **"No Network Adapters"**:
- The Realtek onboard NIC didn't bind (expected — we knew this).
- The USB-Ethernet adapter wasn't recognized.
  - Check: USB plugged into a USB 3.0 port (blue), not USB 2.0.
  - Check: USB adapter is on the supported chip list (AX88179, RTL8153, etc.).
  - Try a different USB port.

If both NICs are unrecognized → the USB adapter isn't supported by the Fling. Try a different USB adapter chip, or use the Intel I210-T1 PCIe card path instead.

## 6.4 — Run the installer

1. Welcome screen → press **Enter** to continue.
2. EULA → press **F11** to accept.
3. **Disk selection:** the installer shows local disks. Select your SSD/HDD. **Press Enter.**
4. **Keyboard layout:** **US Default** → Enter.
5. **Root password:** set a strong password (you'll need this for ESXi web UI). Write it down.
6. Confirm install → **F11**.
7. Wait ~5 min while it installs.

## 6.5 — DON'T REBOOT YET

When it finishes, the screen prompts to **press Enter to reboot**. **Stop here.**

If you reboot now, the system will boot from the installed ESXi on disk — but **the bootbanks don't have `ifre.v00` and `vmkusb_nic_fling.v00`** yet. Drivers need to be copied first.

Press **Alt+F1** to drop to a console login.
- Username: `root`
- Password: (whatever you just set)

You're now at the ESXi shell.

---

# Phase 7 — Post-install: copy drivers + edit boot.cfg (10 min)

You're at the ESXi shell prompt (looks like `[root@localhost:~]` or similar).

## 7.1 — Copy `ifre.v00` to both bootbanks

```sh
cp /tardisks/ifre.v00 /vmfs/volumes/BOOTBANK1/ifre.v00
cp /tardisks/ifre.v00 /vmfs/volumes/BOOTBANK2/ifre.v00
```

Verify:
```sh
ls -la /vmfs/volumes/BOOTBANK1/ifre.v00
ls -la /vmfs/volumes/BOOTBANK2/ifre.v00
```

Both should show file sizes around 1,229,357 bytes.

## 7.2 — Copy `vmkusb_nic_fling.v00` to both bootbanks

```sh
cp /tardisks/vmkusb_nic_fling.v00 /vmfs/volumes/BOOTBANK1/vmkusb_nic_fling.v00
cp /tardisks/vmkusb_nic_fling.v00 /vmfs/volumes/BOOTBANK2/vmkusb_nic_fling.v00
```

## 7.3 — Edit `/vmfs/volumes/BOOTBANK1/boot.cfg`

```sh
vi /vmfs/volumes/BOOTBANK1/boot.cfg
```

In `vi`:
1. Press `i` to enter Insert mode.
2. Use arrow keys to navigate to the end of the `modules=` line (it's one very long line).
3. Append: ` --- /ifre.v00 --- /vmkusb_nic_fling.v00`
4. Press **Esc** to exit Insert mode.
5. Type `:wq` then press **Enter** to save and quit.

## 7.4 — Edit `/vmfs/volumes/BOOTBANK2/boot.cfg`

Repeat:
```sh
vi /vmfs/volumes/BOOTBANK2/boot.cfg
```

Same edit. Save with `:wq`.

## 7.5 — Reboot

```sh
reboot
```

---

# Phase 8 — Verify and configure network (10 min)

## 8.1 — First boot from disk

ESXi boots from the local SSD (no USB needed anymore — you can yank it during the BIOS splash).

The yellow/grey ESXi splash should show:
```
https://192.168.x.x/
```

at the top. If you see `0.0.0.0` → the NIC didn't initialize. Check:
- USB-Ethernet adapter still plugged in to the USB 3.0 port?
- Network cable connected on the other side?
- Router/switch powered on and providing DHCP?

## 8.2 — Set static IP (recommended)

Press **F2** at the ESXi splash → log in as `root` / your password.

Navigate:
1. **Configure Management Network → Network Adapters** → confirm one vmnic is selected (it'll be the USB adapter, e.g., `vmnic32`).
2. **IPv4 Configuration** → Set static IPv4 address:
   - IP: `192.168.1.1` (or whatever the MA2 PDF specified)
   - Mask: `255.255.255.0`
   - Gateway: your router's IP
3. **DNS Configuration:** primary `8.8.8.8`, hostname `esxi-team1`.
4. Press **Esc** → **Y** to apply.

## 8.3 — Test from PC1 browser

From PC1 (your build PC):
- Browser → `https://192.168.1.1/` (or whatever IP)
- Cert warning expected → **Advanced → Proceed (unsafe)**.
- Login: `root` / your password.
- ESXi web UI loads. ✅ **bare-metal ESXi installation complete.**

## 8.4 — (Optional) Add the onboard Realtek NIC's SUBSYS to the driver claim list

Now that ESXi is running, you can manually tell the Realtek driver to claim your onboard NIC's specific subsystem ID. From the web UI → enable SSH (Manage → Services → TSM-SSH → Start), then SSH from PC1:

```sh
ssh root@192.168.1.1
```

Run:
```sh
esxcli system module parameters set -m if-re -p "PCI_DEV_TBL_OVR=10ec:8168:1458:e000"
/sbin/auto-backup.sh
reboot
```

After reboot, the onboard Realtek may now also be detected as a second `vmnic`. Bonus — but not required for ESXi to function.

---

# Troubleshooting

| Symptom | Fix |
|---|---|
| Boot text shows "Unable to load module: ifre.v00" | The file isn't on the USB root. Re-do Phase 4.4. |
| "No network adapters" with both drivers loaded | USB-Ethernet adapter is using an unsupported chip. Try a different one (AX88179 or RTL8153 are the safest). |
| `cp` to BOOTBANK fails: "permission denied" | You're not root. Type `whoami` to confirm. Re-login as root. |
| `vi` edit doesn't save | You forgot to press Esc before `:wq`. Re-open file. |
| ESXi boots but NIC is `vmnic32` instead of `vmnic0` | This is **normal** for USB NICs — they always get high vmnic numbers. Web UI works fine. |
| Web UI not reachable from PC1 | Confirm ESXi static IP is on the same subnet as PC1. Check cables. Disable Windows firewall on PC1 temporarily to rule it out. |
| Realtek SUBSYS injection (Step 8.4) doesn't bind onboard NIC | Some Gigabyte Dragon variants need a different parameter. Skip — keep using USB adapter. The Intel I210-T1 NIC is the cleanest fix when it arrives. |

---

# Summary — what you have when this is done

✅ Bare-metal ESXi 8.0U3 installed on the 3rd PC (Gigabyte B360 HD3)
✅ Network connectivity via supported USB-Ethernet adapter (or Intel PCIe card)
✅ ESXi web UI accessible from PC1 browser at `https://192.168.1.1/`
✅ Ready to create port groups (Phase B of `02_Setup_Topology.md`) and build practice VMs

---

# Next steps

Once ESXi is up:
1. Return to **`02_Setup_Topology.md` Section B** to create the 5 port groups (PG-Internet, PG-LAN, PG-DMZ, PG-Servers, PG-MA1-CMS).
2. Then **`03_Setup_VMs_MA1.md`** to build the CMS pentest target + Kali.
3. Then **`04_Setup_VMs_MA2.md`** to build the manila.com environment.

You're back on track for practice.

---

# References (verified May 2026)

- William Lam — Realtek Driver on free ESXi 8.0U3e (Feb 2026): `https://williamlam.com/2026/02/installing-realtek-network-driver-fling-using-free-esxi-8-0-update-3e-iso.html`
- William Lam — Realtek Network Driver background (Nov 2025): `https://williamlam.com/2025/11/realtek-network-driver-for-esxi.html`
- Broadcom Flings portal: `https://support.broadcom.com/group/ecx/productdownloads?subfamily=Flings&freeDownloads=true`
- Broadcom ESXi 8 free download KB: `https://knowledge.broadcom.com/external/article/399823`
- 7-Zip: `https://www.7-zip.org/`
- Notepad++: `https://notepad-plus-plus.org/downloads/`
- Rufus: `https://rufus.ie/`
