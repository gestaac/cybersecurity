# 02d — Install ESXi 8 on Bare Metal with Realtek Driver Injection

This guide walks you through installing ESXi 8.0 Update 3 directly on bare-metal hardware (a PC with onboard Realtek NIC), by injecting the Realtek driver Fling into a stock ESXi installer USB.

The standard ESXi 8 installer **doesn't include** a Realtek driver. The free Broadcom **Realtek Network Driver Fling** adds support for RTL8111/8125/8126/8127 chipsets. We weld it onto the installer USB so the installer can see the NIC during install, then persist the driver to the installed system after.

> **Source:** William Lam, Feb 2026 — `https://williamlam.com/2026/02/installing-realtek-network-driver-fling-using-free-esxi-8-0-update-3e-iso.html`

> Assumes: a fresh Windows 10/11 build PC (PC1) with nothing yet installed, and a separate target PC (3rd PC) where ESXi will run bare-metal.

---

## ⚠️ Hardware compatibility — read first

The Realtek Fling supports specific chips: **RTL8111 / RTL8125 / RTL8126 / RTL8127**.

**Critical caveat for some Gigabyte motherboards:** boards with Realtek subsystem ID `SUBSYS_E0001458` (Gigabyte "Dragon" rebrand of RTL8111H — common on B360 HD3) have an issue where the Fling driver loads but **doesn't bind to that specific subsystem variant**. If you hit this after install (`No compatible network adapter found` on first boot from disk), the only reliable fix is to add a different supported NIC:

- **USB 3.0 → Gigabit Ethernet adapter** with **ASIX AX88179** or **Realtek RTL8153** chip (~₱300–500 same-day in Manila) — works via the *USB Network Native Driver Fling*. See **Appendix A** at the end of this guide.
- **Intel I210-T1 PCIe x1 NIC card** (~₱600–900, 1–2 days) — ESXi 8 has built-in Intel I210 support, no driver hacking needed.

**Try the Realtek-only path first.** If it works on your specific motherboard revision, great. If not, fall back to one of the two hardware options above.

---

## What you'll do in this guide

| Phase | What | Time |
|---|---|---|
| 1 | Install build tools on Windows (7-Zip, Notepad++, Rufus) | 15 min |
| 2 | Create Broadcom account + download ESXi ISO + Realtek Fling | 25 min |
| 3 | Extract `ifre.v00` from the Fling using 7-Zip command-line | 5 min |
| 4 | Write USB with Rufus, then drop `ifre.v00` on it and edit `boot.cfg` | 15 min |
| 5 | BIOS prep on the 3rd PC | 5 min |
| 6 | Install ESXi from the modified USB | 15 min |
| 7 | Post-install: copy `ifre.v00` to bootbanks, edit `boot.cfg` (do NOT reboot until done) | 10 min |
| 8 | Verify and configure network | 10 min |
| **Total** | | **~100 min** |

---

# Phase 1 — Install build tools on Windows (15 min)

Three free tools on the build PC (PC1). Install in this order.

## 1.1 — 7-Zip

1. Browser → **`https://www.7-zip.org/`**
2. Download **64-bit x64 version** (~1.5 MB `.exe`).
3. Run installer with defaults. Install path: `C:\Program Files\7-Zip\`.

## 1.2 — Notepad++ (preserves Unix line endings — required for boot.cfg)

⚠️ **Do not use regular Windows Notepad.** It converts Unix LF to Windows CRLF and breaks `boot.cfg`.

1. Browser → **`https://notepad-plus-plus.org/downloads/`**
2. Download latest 64-bit installer (~5 MB).
3. Install with defaults.

## 1.3 — Rufus (USB writer)

1. Browser → **`https://rufus.ie/`**
2. Click **Rufus 4.x** (Standard, ~1.5 MB portable .exe).
3. Save to `Downloads\Rufus.exe` — no install needed.

## 1.4 — Working folder

```
D:\esxi-build\
```

(If your PC has only C: drive, use `C:\esxi-build\` and adjust paths.)

---

# Phase 2 — Downloads (25 min)

## 2.1 — Create a Broadcom account

1. Browser → **`https://support.broadcom.com/`**
2. Click **Register** (top right). Use any valid email.
3. Verify via email confirmation link.
4. Sign in.

## 2.2 — Download ESXi 8.0 Update 3 ISO (~640 MB)

1. From Broadcom portal → search "**VMware vSphere Hypervisor 8**".
2. Pick **VMware vSphere Hypervisor (Free) 8.0 Update 3**.
3. Accept EULA → download the installer ISO. Filename like:
   ```
   VMware-VMvisor-Installer-8.0U3-24677879.x86_64.iso
   ```
4. Save to `D:\esxi-build\`.
5. **Copy the free license key** from the same page → save to `D:\esxi-build\license.txt`.

## 2.3 — Download the Realtek Driver Fling

1. Browser → **`https://support.broadcom.com/group/ecx/productdownloads?subfamily=Flings&freeDownloads=true`**
2. Find **Realtek Network Driver for ESXi**.
3. Latest version: **1.101.01** (Nov 2025).
4. Download the ZIP. Filename like:
   ```
   VMware-Re-Driver_1.101.01-5vmw.800.1.0.20613240.zip
   ```
5. Save to `D:\esxi-build\`.

After Phase 2, `D:\esxi-build\` should contain:
```
VMware-VMvisor-Installer-8.0U3-...iso          (~640 MB)
VMware-Re-Driver_1.101.01-...zip                (~210 KB)
license.txt
```

---

# Phase 3 — Extract `ifre.v00` (5 min)

Open **PowerShell** in `D:\esxi-build\`:

```powershell
cd D:\esxi-build
```

## 3.1 — Extract the `.vib` from the Fling ZIP

```powershell
& "C:\Program Files\7-Zip\7z.exe" e .\VMware-Re-Driver_1.101.01-*.zip
```

Result: a `.vib` file appears (~226 KB) plus a small `metadata.zip`.

## 3.2 — Extract the driver payload from the `.vib`

```powershell
& "C:\Program Files\7-Zip\7z.exe" e .\vmw_bootbank_if-re_1.101.01-*.vib
```

Result: a file with the long name `vmw_bootbank_if-re_1.101.01-5vmw.800.1.0.20613240` (~1.2 MB, no extension) — that's the driver payload.

## 3.3 — Rename to `ifre.v00`

```powershell
Rename-Item ".\vmw_bootbank_if-re_1.101.01-5vmw.800.1.0.20613240" "ifre.v00"
```

## 3.4 — Verify

```powershell
Get-Item .\ifre.v00 | Format-List Name, Length
```

Expected:
```
Name   : ifre.v00
Length : 1229357
```

> 📏 The Fling 1.101.01 payload is ~1.2 MB because it includes drivers for 4 chip families. Older internet docs that mention "50–250 KB" refer to older single-chip community VIBs — not relevant here.

---

# Phase 4 — Create the modified USB installer (15 min)

## 4.1 — Plug in your USB stick

8 GB or larger. Everything on it gets wiped.

## 4.2 — Write the stock ESXi ISO with Rufus

1. Run `Rufus.exe`.
2. **Device:** select your USB stick.
3. **Boot selection** → **SELECT** → pick `VMware-VMvisor-Installer-8.0U3-...iso`.
4. **Partition scheme:** **GPT**.
5. **Target system:** **UEFI (non CSM)**.
6. Click **START**.
7. Prompt: *"Write in DD Image mode?"* → click **Yes**.
8. Confirm wipe → wait ~3 min.
9. **Don't eject yet.**

## 4.3 — Find the USB's boot partition

After Rufus finishes, Windows will assign at least one drive letter to the USB's partitions (e.g., `E:\` labeled `TESTING-ESXI` or `ESXI-X.X.X`). The relevant partition is the **EFI/FAT boot partition** containing:

- Root files: `b.b00`, `k.b00`, `boot.cfg`, `*.v00`, etc.
- A folder `efi\boot\` with `boot.cfg`, `bootx64.efi`, etc.

If multiple partitions appear, pick the one with `boot.cfg` at the root.

> ⚠️ Windows may show "Format disk" prompts for unfamiliar partitions. **Always click Cancel** — don't format anything.

## 4.4 — Copy `ifre.v00` to TWO locations on the USB

Adjust `E:\` to whatever drive letter Windows assigned:

```powershell
$usb = "E:\"

# Copy to USB root (next to b.b00, etc.)
Copy-Item D:\esxi-build\ifre.v00 "$usb\ifre.v00"

# Copy also to EFI\BOOT folder (for UEFI boot path)
Copy-Item D:\esxi-build\ifre.v00 "$usb\efi\boot\ifre.v00"
```

## 4.5 — Edit BOTH `boot.cfg` files

Two `boot.cfg` files exist on the USB:
- `E:\boot.cfg` (root — Legacy BIOS path)
- `E:\efi\boot\boot.cfg` (UEFI path — used by modern Gigabyte boards)

**Edit BOTH** with **Notepad++** (NOT regular Notepad).

Find the line beginning with `modules=`. It's a single very long line with module names separated by ` --- `. Append at the very end:

```
 --- /ifre.v00
```

So the line ends with:
```
... --- /imgpayld.tgz --- /ifre.v00
```

Save each file (Notepad++ preserves Unix LF line endings automatically).

## 4.6 — Safely eject the USB

In File Explorer → right-click the USB drive → **Eject**.

---

# Phase 5 — BIOS prep on the 3rd PC (5 min)

Plug the USB into the 3rd PC. Power on, spam **Del** at the Gigabyte splash.

## 5.1 — Settings to apply

| Setting | Path on Gigabyte BIOS | Value |
|---|---|---|
| Intel Virtualization Technology (VT-x) | M.I.T. → Advanced CPU Core Settings | **Enabled** |
| Intel VT-d | M.I.T. → Advanced CPU Core Settings | **Enabled** |
| Hyper-Threading | M.I.T. → Advanced CPU Core Settings | **Enabled** |
| Secure Boot | BIOS → Secure Boot (set OS Type → Other OS first if greyed out) | **Disabled** |
| Boot Mode | BIOS | **UEFI** |
| Fast Boot | BIOS | **Disabled** |
| AC BACK / Restore on AC Loss | Power | **Always On** (optional) |

Press **F10** → **Yes** to save and exit. PC reboots.

---

# Phase 6 — Install ESXi from the USB (15 min)

## 6.1 — Boot from USB

At the next Gigabyte splash, spam **F12** → boot menu → select your USB → Enter.

## 6.2 — Watch for the driver load

Boot text scrolls fast. Among the lines, you should see:
```
Loading /ifre.v00
```

If you see it without "module not found" → success, the driver loaded.

## 6.3 — Run the installer wizard

1. Welcome screen → **Enter**.
2. EULA → **F11** to accept.
3. **Disk selection:** pick your SSD/HDD → **Enter**.
4. **Keyboard:** US Default → **Enter**.
5. **Root password:** strong password — write it down.
6. Confirm install → **F11**.
7. Wait ~5 min while it installs.

## 6.4 — ⚠️ DO NOT REBOOT YET

When the screen prompts **"Press Enter to reboot"** — **STOP**.

If you reboot now, you'll see **"No compatible network adapter found"** because the driver `ifre.v00` is on the USB but not yet in the persistent bootbanks on disk.

Press **Alt+F1** to drop to the console shell.
- Login: `root` / your password.

You're now at the ESXi shell. Continue to Phase 7.

> 💡 **If you already rebooted by mistake** and see "No compatible network adapter found" — see **Phase 7.0 Recovery** below.

---

# Phase 7 — Post-install: persist the driver (10 min)

> Background: when ESXi boots from a USB whose `boot.cfg` lists `ifre.v00`, the kernel auto-mounts each loaded module under `/tardisks/`. So `/tardisks/ifre.v00` exists in RAM during this session. We copy it from RAM to the on-disk bootbanks so it persists across reboots.

## 7.0 Recovery — only if you already rebooted and got "no compatible network adapter found"

1. Force power off (hold power button 10 sec).
2. Plug the modified USB stick back in.
3. Power on, spam F12 → boot menu → select USB.
4. Wait for the yellow **ESXi installer welcome screen**.
5. Press **Alt+F1** at the welcome screen.
6. Login: `root` / no password (the installer-stage shell has no password).
7. Verify:
   ```sh
   ls -la /tardisks/ifre.v00
   ls /vmfs/volumes/
   ```
   You should see `ifre.v00` in `/tardisks/`, AND `BOOTBANK1` + `BOOTBANK2` in `/vmfs/volumes/`.
8. Continue to Phase 7.1 below.

If `BOOTBANK1` and `BOOTBANK2` are missing → the install was wiped. Re-run Phase 6, this time press Alt+F1 instead of rebooting.

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

Both should show file size ~1,229,357 bytes (~1.2 MB).

> ❌ If `cp` says **"No such file or directory"** for `/tardisks/ifre.v00` → you're not booted from the modified USB. Go back to Phase 7.0 Recovery.

## 7.2 — Add `ifre.v00` to `boot.cfg` in both bootbanks

The `modules=` line is one extremely long line (~1,500 chars). Don't try to edit it manually with `vi` — use one of the methods below.

> ⚠️ **Use Method A first.** Method B (`sed -i`) often fails silently on ESXi's busybox shell — the command appears to run but the file doesn't change. If you suspect that happened, run the verification step (7.3) — if `grep -c` returns `0`, switch to Method A.

### Method A — `awk` ✅ recommended (most reliable on ESXi)

Copy and paste this whole block into the shell:

```sh
awk '/^modules=/ {print $0 " --- /ifre.v00"; next} {print}' /vmfs/volumes/BOOTBANK1/boot.cfg > /tmp/bc1.new
cp /tmp/bc1.new /vmfs/volumes/BOOTBANK1/boot.cfg

awk '/^modules=/ {print $0 " --- /ifre.v00"; next} {print}' /vmfs/volumes/BOOTBANK2/boot.cfg > /tmp/bc2.new
cp /tmp/bc2.new /vmfs/volumes/BOOTBANK2/boot.cfg
```

**What this does:** `awk` reads each line of `boot.cfg`. For the `modules=` line, it appends ` --- /ifre.v00` and prints. All other lines pass through unchanged. Output is redirected to a temp file in `/tmp/`, then `cp` overwrites the original. This bypasses the in-place-edit limitation.

If both commands return without error → continue to 7.3 verify.

### Method B — `sed -i` (try this only if Method A doesn't work)

ESXi's `sed -i` (in-place edit) is unreliable on the busybox shell — it often **silently fails** without an error. Try only if Method A had an issue:

```sh
sed -i 's|imgpayld.tgz|imgpayld.tgz --- /ifre.v00|' /vmfs/volumes/BOOTBANK1/boot.cfg
sed -i 's|imgpayld.tgz|imgpayld.tgz --- /ifre.v00|' /vmfs/volumes/BOOTBANK2/boot.cfg
```

If `grep -c` (in 7.3 below) returns `0` after this → sed didn't take. Use Method A or C.

### Method C — Manual `vi` (last resort)

> ⚠️ vi displays the long `modules=` line wrapped with `@` continuation markers and possibly with syntax highlighting colors. These look like corruption but aren't — the file is fine. Don't be alarmed.

```sh
vi /vmfs/volumes/BOOTBANK1/boot.cfg
```

In `vi`:
1. Press **Esc** to ensure you're in command mode.
2. Press **`7G`** to jump directly to line 7 (the `modules=` line).
3. Press **`$`** to move cursor to END of that line (even if it's visually wrapped).
4. Press **`a`** to enter append-after-cursor mode (status line shows `-- INSERT --`).
5. Type exactly: ` --- /ifre.v00` (note: leading space, then three dashes, then space, then `/ifre.v00`).
6. Press **Esc** to leave insert mode.
7. Type `:wq` and press **Enter** to save and quit.

If you make a mistake: press **Esc**, type `:q!`, press Enter to quit without saving. Re-run the `vi` command.

Repeat for BOOTBANK2:
```sh
vi /vmfs/volumes/BOOTBANK2/boot.cfg
```

## 7.3 — Verify the edits

Whichever method you used (A / B / C), always verify before rebooting:

### 7.3.1 — Quickest check: count occurrences

```sh
grep -c "ifre.v00" /vmfs/volumes/BOOTBANK1/boot.cfg
grep -c "ifre.v00" /vmfs/volumes/BOOTBANK2/boot.cfg
```

| Output | Meaning |
|---|---|
| Both print **`1`** | ✅ Edit succeeded — proceed to 7.4 reboot |
| Either prints **`0`** | ❌ Edit didn't take — try a different method (A → C) and re-verify |
| Either prints **`2`** or more | ⚠️ You ran the edit twice and `ifre.v00` is duplicated. ESXi tolerates this at boot but cleanup recommended (see "If duplicated" below) |

### 7.3.2 — Confirm `ifre.v00` is the LAST entry (visual)

```sh
grep "modules=" /vmfs/volumes/BOOTBANK1/boot.cfg | awk -F' --- ' '{print $NF}'
grep "modules=" /vmfs/volumes/BOOTBANK2/boot.cfg | awk -F' --- ' '{print $NF}'
```

This splits the modules line by ` --- ` and prints only the last field. **Expected output: each prints `/ifre.v00`** — meaning `ifre.v00` is correctly appended at the end of the modules list.

### 7.3.3 — See the end of the long modules line

```sh
grep "modules=" /vmfs/volumes/BOOTBANK1/boot.cfg | tail -c 100
```

Should end with: `... --- /imgpayld.tgz --- /ifre.v00`

### 7.3.4 — All-in-one sanity check (paste this block)

```sh
echo "=== Count check (expect 1 for each) ==="
grep -c "ifre.v00" /vmfs/volumes/BOOTBANK1/boot.cfg
grep -c "ifre.v00" /vmfs/volumes/BOOTBANK2/boot.cfg
echo ""
echo "=== Last entry (expect /ifre.v00) ==="
grep "modules=" /vmfs/volumes/BOOTBANK1/boot.cfg | awk -F' --- ' '{print $NF}'
grep "modules=" /vmfs/volumes/BOOTBANK2/boot.cfg | awk -F' --- ' '{print $NF}'
```

If both `1`s are printed AND both last-entry lines say `/ifre.v00` → **edit confirmed, proceed to 7.4**.

### If duplicated (count returns 2 or more)

You ran the edit twice. To clean up:

```sh
# Remove all instances of ' --- /ifre.v00' first, then re-add once
sed -i 's| --- /ifre.v00||g' /vmfs/volumes/BOOTBANK1/boot.cfg
awk '/^modules=/ {print $0 " --- /ifre.v00"; next} {print}' /vmfs/volumes/BOOTBANK1/boot.cfg > /tmp/bc1.new
cp /tmp/bc1.new /vmfs/volumes/BOOTBANK1/boot.cfg

sed -i 's| --- /ifre.v00||g' /vmfs/volumes/BOOTBANK2/boot.cfg
awk '/^modules=/ {print $0 " --- /ifre.v00"; next} {print}' /vmfs/volumes/BOOTBANK2/boot.cfg > /tmp/bc2.new
cp /tmp/bc2.new /vmfs/volumes/BOOTBANK2/boot.cfg
```

Then re-run the count check. Should be `1` each.

## 7.4 — Yank the USB, then reboot

⚠️ **Physically pull the USB stick out** of the 3rd PC before rebooting. If you leave it plugged in, the BIOS may try to boot from USB again.

After the USB is unplugged:
```sh
reboot
```

The 3rd PC reboots from the SSD. This time `BOOTBANK1/boot.cfg` lists `ifre.v00`, so it loads at boot, and the Realtek driver initializes the onboard NIC.

---

# Phase 8 — Verify and configure network (10 min)

## 8.1 — Check the splash

After reboot, the yellow/grey ESXi splash should show:
```
https://192.168.x.x/
```

at the top.

| Result | Meaning |
|---|---|
| `https://192.168.x.x/` shown | ✅ Driver bound to your NIC. Proceed to 8.2. |
| `0.0.0.0` shown | NIC didn't get DHCP. Check cable + router. Try F2 → Configure Management Network → Restart Network. |
| `No compatible network adapter found` | ❌ Driver loaded but didn't bind to your NIC's specific subsystem ID (the SUBSYS_E0001458 issue). Move to **Appendix A** to add a USB-Ethernet adapter, OR plan to buy an Intel I210-T1 PCIe NIC. |

## 8.2 — Set static IP (recommended)

Press **F2** at the ESXi splash → log in as `root`.

1. **Configure Management Network → Network Adapters** → confirm a vmnic is selected.
2. **IPv4 Configuration** → Set static IPv4 address:
   - IP: `192.168.1.1` (or whatever the MA2 PDF specified)
   - Mask: `255.255.255.0`
   - Gateway: your router's IP
3. **DNS Configuration:** primary `8.8.8.8`, hostname `esxi-team1`.
4. **Esc** → **Y** to apply.

## 8.3 — Test from PC1

From PC1's browser:
- `https://<ESXi-IP>/ui`
- Cert warning expected → **Advanced → Proceed (unsafe)**.
- Login: `root` / your install password.
- ESXi web UI loads. ✅ **Bare-metal ESXi is up.**

---

# Troubleshooting

| Symptom | Fix |
|---|---|
| Boot text shows "Unable to load module: ifre.v00" | File not on USB. Re-do Phase 4.4. |
| ESXi installer says "No Network Adapters" | Driver loaded but didn't bind to your NIC. Likely SUBSYS mismatch. See **Appendix A** or buy Intel NIC. |
| `cp /tardisks/ifre.v00 ...` says "No such file or directory" | Not booted from modified USB. See Phase 7.0 Recovery. |
| `sed -i` errors out | Use the `awk` fallback in 7.2. |
| `grep -c` returns 0 | Edit didn't take. Try the `awk` fallback. |
| First reboot from disk: "no compatible network adapter found" | Driver was loaded at install but is missing from the bootbanks. See Phase 7.0 Recovery. |
| ESXi web UI not reachable from PC1 | Confirm both are on same subnet. Disable Windows Firewall on PC1 temporarily to rule it out. |

---

# Next steps

After ESXi is up:
1. Return to **`02_Setup_Topology.md` Section B** to create the 5 port groups (PG-Internet, PG-LAN, PG-DMZ, PG-Servers, PG-MA1-CMS).
2. Then **`03_Setup_VMs_MA1.md`** to build the CMS pentest target + Kali.
3. Then **`04_Setup_VMs_MA2.md`** to build the manila.com environment.

---

---

# Appendix A — Optional: USB-Ethernet adapter fallback

**Use this only if the Realtek-only path failed** (Phase 8.1 showed "No compatible network adapter found"). This appendix adds the **USB Network Native Driver Fling** alongside `ifre.v00` so a USB-Ethernet adapter can serve as the working NIC.

## A.1 — Buy a supported USB-Ethernet adapter

Required chip: **ASIX AX88179** or **Realtek RTL8153**. Both work with the USB Network Native Driver Fling. ~₱300–500 from Lazada/Shopee, often same-day Manila delivery.

## A.2 — Download the USB Network Native Driver Fling

1. Broadcom Flings: `https://support.broadcom.com/group/ecx/productdownloads?subfamily=Flings&freeDownloads=true`
2. Find **USB Network Native Driver for ESXi** (compatible with ESXi 8.0).
3. Download the offline bundle ZIP. Filename like:
   ```
   ESXi800-VMKUSB-NIC-FLING-xxxxx-component.zip
   ```
4. Save to `D:\esxi-build\`.

## A.3 — Extract `vmkusb_nic_fling.v00`

```powershell
cd D:\esxi-build
& "C:\Program Files\7-Zip\7z.exe" e .\ESXi800-VMKUSB-NIC-FLING-*.zip
& "C:\Program Files\7-Zip\7z.exe" e .\vmw_bootbank_vmkusb-nic-fling_*.vib

# Find the driver payload (largest non-.vib non-.zip file)
Get-ChildItem -File | Where-Object { $_.Name -notmatch '\.(vib|zip|iso|xml|pkcs7|txt)$' } | Sort-Object Length -Descending | Select-Object Name, Length -First 5

# Rename whatever the largest payload is — substitute its actual filename:
Rename-Item ".\<long-filename>" "vmkusb_nic_fling.v00"
```

Expected size: ~600 KB to 1.5 MB.

## A.4 — Add to USB

Plug the same USB stick into PC1. Then:

```powershell
$usb = "E:\"   # adjust to your USB drive letter
Copy-Item D:\esxi-build\vmkusb_nic_fling.v00 "$usb\vmkusb_nic_fling.v00"
Copy-Item D:\esxi-build\vmkusb_nic_fling.v00 "$usb\efi\boot\vmkusb_nic_fling.v00"
```

Edit BOTH `boot.cfg` files on the USB (root and `efi\boot\`) — append a second module:
```
... --- /imgpayld.tgz --- /ifre.v00 --- /vmkusb_nic_fling.v00
```

Save with Notepad++.

## A.5 — Plug in the USB-Ethernet adapter

Plug it into a USB 3.0 port (blue inside) on the 3rd PC.

## A.6 — Reinstall ESXi from the updated USB

Repeat Phases 5–7, but with both drivers active. In Phase 7.1, also copy the second driver:

```sh
cp /tardisks/vmkusb_nic_fling.v00 /vmfs/volumes/BOOTBANK1/vmkusb_nic_fling.v00
cp /tardisks/vmkusb_nic_fling.v00 /vmfs/volumes/BOOTBANK2/vmkusb_nic_fling.v00
```

In Phase 7.2, use the combined sed:
```sh
sed -i 's|imgpayld.tgz|imgpayld.tgz --- /ifre.v00 --- /vmkusb_nic_fling.v00|' /vmfs/volumes/BOOTBANK1/boot.cfg
sed -i 's|imgpayld.tgz|imgpayld.tgz --- /ifre.v00 --- /vmkusb_nic_fling.v00|' /vmfs/volumes/BOOTBANK2/boot.cfg
```

In Phase 7.3, also verify `vmkusb_nic_fling.v00`:
```sh
grep -c "vmkusb_nic_fling.v00" /vmfs/volumes/BOOTBANK1/boot.cfg
grep -c "vmkusb_nic_fling.v00" /vmfs/volumes/BOOTBANK2/boot.cfg
```

After reboot, ESXi will use the USB-Ethernet adapter as its primary NIC (typically `vmnic32` — USB NICs get high vmnic numbers).

---

# References (verified May 2026)

- William Lam — Realtek Driver on free ESXi 8.0U3e (Feb 2026): `https://williamlam.com/2026/02/installing-realtek-network-driver-fling-using-free-esxi-8-0-update-3e-iso.html`
- William Lam — Realtek Network Driver background (Nov 2025): `https://williamlam.com/2025/11/realtek-network-driver-for-esxi.html`
- Broadcom Flings portal: `https://support.broadcom.com/group/ecx/productdownloads?subfamily=Flings&freeDownloads=true`
- Broadcom ESXi 8 free download KB: `https://knowledge.broadcom.com/external/article/399823`
- 7-Zip: `https://www.7-zip.org/`
- Notepad++: `https://notepad-plus-plus.org/downloads/`
- Rufus: `https://rufus.ie/`
