# 02d — Install Realtek NIC Driver on ESXi 8.0U3e (William Lam method, Windows version)

The stock VMware ESXi 8 installer **does not include Realtek RTL8111/8125/8126/8127 NIC drivers**. If your motherboard (Gigabyte boards in particular) has only an onboard Realtek NIC, the installer fails with:

```
No Network Adapters
No network adapters were detected. Either no network adapters are physically
connected to the system, or a suitable driver could not be located.
```

This guide adapts **William Lam's official Feb 2026 walkthrough** for Windows users. It does **NOT** use PowerCLI / Python / OpenSSL / Image Builder — sidestepping all the dependency hell those introduce. We just extract the driver from the Fling, drop it into the USB installer, and edit a config file.

> 📚 **Source:** `https://williamlam.com/2026/02/installing-realtek-network-driver-fling-using-free-esxi-8-0-update-3e-iso.html` (Feb 2026)
>
> Lam's post assumes macOS/Linux. This file translates the workflow to Windows.

---

## How this method works (high-level)

The Realtek driver Fling ships as a `.vib` file (basically a Unix `ar` archive). We:

1. Extract the driver payload from the `.vib` file.
2. Rename the payload to `ifre.v00`.
3. Copy `ifre.v00` to the root of the ESXi installer USB.
4. Edit `boot.cfg` on the USB to load `ifre.v00` during boot.
5. Boot the installer — Realtek NIC now detected.
6. Complete install **but DON'T reboot yet**.
7. SSH to ESXi → copy `ifre.v00` into both `BOOTBANK1` and `BOOTBANK2`.
8. Edit `boot.cfg` in both bootbanks → append `ifre.v00`.
9. Reboot — driver loads on every boot from now on.

This is more steps than a custom ISO, but **no Python / OpenSSL / PowerCLI**.

> ⏱️ **Total time:** ~45 min.

---

## Materials needed (all verified working May 2026)

| # | Item | Source | Notes |
|---|---|---|---|
| 1 | **Stock ESXi 8.0U3e ISO** | `https://support.broadcom.com/group/ecx/free-downloads` (Broadcom account required) | The stock free installer. KB landing: `https://knowledge.broadcom.com/external/article/399823` |
| 2 | **Realtek Driver Fling v1.101.01** | `https://support.broadcom.com/group/ecx/productdownloads?subfamily=Flings&freeDownloads=true` | Same Broadcom portal, **Flings** section. Download `VMware-Re-Driver_1.101.01-...zip` |
| 3 | **Rufus** | `https://rufus.ie/` | For writing the USB |
| 4 | **7-Zip** | `https://www.7-zip.org/` | Can open the `.vib` (which is an `ar` archive) on Windows |
| 5 | **WinSCP** | `https://winscp.net/` | For copying `ifre.v00` to ESXi after install |
| 6 | **PuTTY** | `https://www.putty.org/` | For SSH-ing to ESXi to edit boot.cfg |
| 7 | A USB stick (8 GB+) | Already in use | We'll re-write it with the modified installer |

---

## Step 1 — Download stock ESXi 8.0U3e ISO from Broadcom (15 min)

1. Browser → `https://support.broadcom.com/`
2. Sign in (or register a free Broadcom account).
3. **My Downloads** → search "**VMware vSphere Hypervisor 8**".
4. Pick **VMware vSphere Hypervisor 8.0 Update 3e** (free).
5. Accept the EULA, copy the free license key shown on the page.
6. Download the ISO (~640 MB).
7. Save to `D:\esxi-build\` on PC1. Filename like:
   ```
   VMware-VMvisor-Installer-8.0U3e-24585291.x86_64.iso
   ```

---

## Step 2 — Download the Realtek Driver Fling (5 min)

1. Same Broadcom portal. Navigate to: `https://support.broadcom.com/group/ecx/productdownloads?subfamily=Flings&freeDownloads=true`
2. Find **Realtek Network Driver for ESXi**.
3. Latest version: **1.101.01** (Nov 2025).
4. Download the `.zip` bundle (e.g., `VMware-Re-Driver_1.101.01-5vmw.800.1.0.20613240.zip`).
5. Save to `D:\esxi-build\`.

---

## Step 3 — Extract the `.vib` file from the offline bundle (3 min)

The Fling is a `.zip` containing the offline bundle. Inside that bundle is the actual `.vib` driver file.

1. Open `D:\esxi-build\` in File Explorer.
2. Right-click the `VMware-Re-Driver_1.101.01-...zip` → **7-Zip → Extract Here**.
3. You'll get a folder structure. Navigate into it:
   ```
   VMware-Re-Driver_1.101.01-...
   └── vib20\
       └── if-re\
           └── vmw_bootbank_if-re_1.101.01-5vmw.800.1.0.20613240.vib
   ```
4. Copy the `.vib` file (e.g., `vmw_bootbank_if-re_1.101.01-5vmw.800.1.0.20613240.vib`) up to `D:\esxi-build\` so it's easier to work with.

---

## Step 4 — Extract `ifre.v00` from the `.vib` file (2 min)

A `.vib` file is actually a Unix `ar` archive. Windows doesn't have `ar` natively. The **7-Zip GUI ("Open archive") does NOT reliably show the top-level structure** of `.vib` files — it tends to auto-drill into the payload, so you'll see folders like `usr/`, `etc/` instead of the three files we need (`descriptor.xml`, `sig.pkcs7`, `if-re`).

**Use the command-line method instead — it's actually the most reliable:**

### Method A — 7-Zip command-line ✅ recommended

Open **PowerShell** (regular, non-admin is fine) in `D:\esxi-build\`:

```powershell
cd D:\esxi-build
& "C:\Program Files\7-Zip\7z.exe" e .\vmw_bootbank_if-re_1.101.01-*.vib
```

> The `e` flag (lowercase) means "extract files, no paths" — dumps everything from the **top level** into the current directory.

You should see 7-Zip print something like:
```
Files: 3
Size: ~85000
Compressed: ~85000
Everything is Ok
```

Now `D:\esxi-build\` contains additional files. The exact set varies slightly by 7-Zip version + Fling version — there are **two common patterns** you might see:

**Pattern X (most common with Realtek Fling 1.101.01):**
- `vmw_bootbank_if-re_1.101.01-5vmw.800.1.0.20613240`  (~1.2 MB, no extension — **this is the driver payload**)
- `vmw_bootbank_if-re_1.101.01-5vmw.800.1.0.20613240.vib`  (~226 KB — original .vib, kept by 7-Zip)
- `VMware-Re-Driver_1.101.01-...zip`  (~210 KB — original Fling zip)
- `metadata.zip`  (~3 KB — metadata extracted alongside)

**Pattern Y (older 7-Zip versions / older Flings):**
- `descriptor.xml`  (~2 KB)
- `sig.pkcs7`  (~5 KB)
- `if-re`  (~50–200 KB — driver payload)

In both cases, **the file we want is the largest "if-re" or `vmw_bootbank_if-re_*` (no extension) entry** — that's the renamed Realtek driver.

Verify what 7-Zip produced:
```powershell
Get-ChildItem -File | Sort-Object Length -Descending | Select-Object Name, Length -First 6
```

Identify the largest file with **`if-re`** in its name (could be ~200 KB up to ~1.5 MB depending on Fling version — **size doesn't matter, exact name does**).

Then rename it to `ifre.v00`:
```powershell
# If you got Pattern X (long filename, no extension):
Rename-Item ".\vmw_bootbank_if-re_1.101.01-5vmw.800.1.0.20613240" "ifre.v00"

# If you got Pattern Y (short filename "if-re"):
Rename-Item ".\if-re" "ifre.v00"
```

> ⚠️ **7-Zip naming quirk:** when 7-Zip's command-line extracts an `ar` archive, it sometimes preserves the `.vib`'s long internal filename as the extracted file's name (Pattern X above). Other times it uses the short member name (`if-re` — Pattern Y). Both are the **same file content** — just different naming conventions. Either way, you want to rename it to `ifre.v00`.

Verify the rename:
```powershell
Get-Item .\ifre.v00 | Format-List Name, Length
```

Expected:
```
Name   : ifre.v00
Length : 1229357   (or anywhere from ~50,000 to ~1,500,000 depending on Fling version)
```

> 📏 **About the file size:** the older Realtek community VIBs were ~50–250 KB. The newer Broadcom Realtek Fling 1.101.01 is **~1.2 MB** because it includes drivers for 4 chip families (RTL8111/8125/8126/8127). **Don't worry if your size is "too big" vs older sizes documented elsewhere on the internet — 1.2 MB is correct for v1.101.01.**

#### Optional cleanup
You can delete the leftover scaffolding files (the original .vib, the metadata.zip, the original Fling .zip) — they're not used from here on. But it's safer to **keep them** in case you need to re-extract later. Disk space is cheap.

```powershell
# Optional cleanup of just the small leftovers (NOT the .vib or .zip — keep those):
Remove-Item .\metadata.zip -ErrorAction SilentlyContinue
```

### Method B — WSL with `ar` (fallback if 7-Zip command-line fails)

If 7-Zip refuses to extract the `.vib` (e.g., "cannot open as archive"), the cleanest fallback is **Windows Subsystem for Linux** which has the actual `ar` tool that William Lam used on macOS.

#### One-time WSL install (~10 min, only if needed)
1. Open **PowerShell as Administrator** → run:
   ```powershell
   wsl --install
   ```
2. Reboot when prompted.
3. After reboot, the Ubuntu installer auto-launches. Set a username + password (anything you'll remember).
4. Done.

#### Extract the `.vib` with `ar`
Open the **Ubuntu** terminal (Start menu → Ubuntu) and run:

```bash
cd /mnt/d/esxi-build/
ar -x vmw_bootbank_if-re_1.101.01-5vmw.800.1.0.20613240.vib
ls -la
# You should see: descriptor.xml  sig.pkcs7  if-re
mv if-re ifre.v00
ls -la ifre.v00
```

The file `ifre.v00` is now in `D:\esxi-build\` (visible from Windows too).

### Method C — Why 7-Zip GUI ("Open archive") doesn't work for this

If you tried the 7-Zip GUI ("Open archive") and saw a folder structure like:
```
usr\lib\vmware\vmkmod\if_re\
```

That's because 7-Zip auto-decompressed the `if-re` payload (it recognized the inner gzip+cpio format) and showed you the **post-install file layout**. We don't want that — we want the raw outer payload.

**Don't use 7-Zip GUI for this step.** Use Method A (command-line) or Method B (WSL).

If you already extracted via the GUI and see `usr\` and `etc\` folders — just delete them and start over with Method A.

---

## Step 5 — Write the stock ESXi ISO to USB with Rufus (3 min)

1. Plug in the USB stick (it'll be wiped).
2. Open Rufus.
3. **Device:** select your USB.
4. **Boot selection** → **SELECT** → pick the **stock** `VMware-VMvisor-Installer-8.0U3e-...iso` (NOT a custom one).
5. **Partition scheme:** **GPT** (for UEFI).
6. **File system:** leave default.
7. Click **START**.
8. Prompt: *"Write in DD Image mode?"* → **Yes**.
9. Wait ~3 min. **Don't eject** — we'll edit files on it next.

---

## Step 6 — Modify the USB to load the Realtek driver during install (5 min)

The USB is now a bootable ESXi installer. We add `ifre.v00` and tell `boot.cfg` to load it.

### 6.1 — Open the USB in File Explorer
The USB will show up under "This PC" with a label like `ESXI-X.X.X` (multiple partitions). The one we want is the **EFI partition** that contains a `BOOT` folder.

> ⚠️ Windows sometimes shows a "Format disk" prompt for unfamiliar partitions. **Click Cancel** — DON'T format. Use a tool like **EaseUS Partition Master Free** or **Rufus's mounted partitions** if Windows can't natively browse all partitions.

If you have trouble browsing the USB partitions:
- **Easier path:** use **DiskGenius Free** (`https://www.diskgenius.com/free.php`) → browse all partitions → find the one with `BOOT.CFG`.

### 6.2 — Copy `ifre.v00` to the USB root + EFI/BOOT folder
1. Copy `D:\esxi-build\ifre.v00` to:
   - The **root** of the EFI partition (alongside `BOOT.CFG`)
   - Also copy to `EFI\BOOT\` folder (some systems read from there)
2. Two copies of the same file — paranoia for both legacy and UEFI paths.

### 6.3 — Edit `BOOT.CFG` to load the new module
1. Find `BOOT.CFG` in the EFI partition root (and in `EFI\BOOT\` — there are usually two copies).
2. Open with Notepad++ or any text editor (NOT regular Notepad — line endings matter).
3. You'll see a line like:
   ```
   modules=b.b00 --- jumpstrt.gz --- useropts.gz --- features.gz --- k.b00 --- uc_intel.b00 --- uc_amd.b00 --- uc_hygon.b00 --- procfs.b00 --- vmx.v00 --- vim.v00 --- ...
   ```
   It's one **very long single line** with many `.b00`/`.v00` files separated by ` --- `.
4. **Append `--- ifre.v00`** at the end of the modules list:
   ```
   modules=...(everything)... --- vsanmgmt.v00 --- xorg.v00 --- ifre.v00
   ```
5. Save the file. Repeat for both copies of `BOOT.CFG` if there are two.

> ⚠️ **Critical:** must be `--- ifre.v00` (three dashes, single space, the filename). One missing dash and the boot fails.

### 6.4 — Safely eject the USB
Right-click the USB drive in File Explorer → **Eject**.

---

## Step 7 — Boot the 3rd PC from the USB (5 min)

1. Plug the USB into the 3rd PC.
2. Power on → press **F12** repeatedly during the Gigabyte splash logo → boot menu.
3. Select your USB → Enter.
4. ESXi installer loads. **Watch the boot text** — you should see:
   ```
   loading /ifre.v00
   ```
   Right before the installer launches. That's the Realtek driver loading.
5. The installer prompt now shows **"1 NIC(s) found"** instead of the previous error.

---

## Step 8 — Run the ESXi installer normally (10 min)

Follow `02c_Setup_ESXi_Server.md` Section D click-by-click:
- Welcome → F11 to accept EULA
- Pick disk → keyboard → set root password
- F11 to install → wait

> ⚠️ **CRITICAL:** when installation finishes and prompts to **press Enter to reboot — DO IT. But the FIRST reboot will lose the driver** because we haven't yet copied `ifre.v00` into the persistent boot banks. ESXi's first boot will fail to find the NIC.

You have two options:

### Option A (William Lam's method) — Skip the reboot, drop to ESXi shell now
1. **Don't press Enter** when prompted to reboot.
2. Press **Alt+F1** → log in as `root` / your password.
3. Continue to Step 9 below from this shell.

### Option B (more practical) — Let it reboot, fix from another machine
The first boot fails (no NIC). To recover:
1. Yank USB during reboot, but boot ESXi from disk (it's installed).
2. Console will show `0.0.0.0` for IP (no NIC).
3. Press F2 at console → log in as root → drop to shell with **F1** then `unsupported` (or use the troubleshoot menu).
4. From the shell, manually mount the USB or use a second USB with `ifre.v00` on it → copy to bootbanks (Step 9).

**Option A is cleaner.** Do it.

---

## Step 9 — Copy `ifre.v00` to both bootbanks (5 min)

You're at the ESXi console shell (Option A above) or you've SSH'd in.

```sh
# William Lam's exact commands — run as root:

cp /tardisks/ifre.v00 /vmfs/volumes/BOOTBANK1/ifre.v00
cp /tardisks/ifre.v00 /vmfs/volumes/BOOTBANK2/ifre.v00
```

Verify:
```sh
ls -la /vmfs/volumes/BOOTBANK1/ifre.v00
ls -la /vmfs/volumes/BOOTBANK2/ifre.v00
```

Both should show file size matching what was on the USB.

---

## Step 10 — Edit `boot.cfg` in BOTH bootbanks (5 min)

Both BOOTBANK1 and BOOTBANK2 have their own `boot.cfg`. Edit both.

```sh
vi /vmfs/volumes/BOOTBANK1/boot.cfg
```

Find the `modules=` line. Append ` --- ifre.v00` at the end (just like you did on the USB in Step 6.3). Save (`:wq` in vi).

Repeat for BOOTBANK2:
```sh
vi /vmfs/volumes/BOOTBANK2/boot.cfg
```

> ⚠️ **vi quick reference if you're not familiar:**
> - Press `i` to enter insert mode → make edits.
> - Press `Esc` to exit insert mode.
> - Type `:wq` then Enter to save and quit.
> - If you mess up: `Esc` → `:q!` → Enter to quit without saving, then start over.

---

## Step 11 — Reboot

```sh
reboot
```

PC reboots. This time it loads `ifre.v00` from the bootbank → Realtek NIC initializes → DHCP picks up an IP → the console shows `https://192.168.x.x/` at the top.

---

## Verification

### V1 — Console shows IP
ESXi splash should show `https://192.168.x.x/` (not `0.0.0.0`).

### V2 — Web UI from PC1
Browser → `https://<ESXi-IP>/ui` → log in as `root` → confirm web UI loads.

### V3 — NIC visible in web UI
Web UI → **Networking → Physical NICs** → should see `vmnic0` with driver `if-re`.

---

## Troubleshooting

| Error | Fix |
|---|---|
| 7-Zip can't open the `.vib` file | Try `7z e file.vib` from command line. Or use WSL with `ar -x`. Or copy `.vib` to any Linux machine and run `ar -x` there. |
| `BOOT.CFG` doesn't have a `modules=` line visible | You opened the wrong partition. The EFI partition has it; the data partition doesn't. Check all partitions on the USB. |
| `loading /ifre.v00` not seen at boot | `BOOT.CFG` edit didn't take — re-check `--- ifre.v00` syntax (three dashes + space + filename). |
| Still "No Network Adapters" after boot | Wrong NIC chipset (e.g., RTL8169 — older, not in this Fling). Cross-check Device Manager hardware ID. |
| Can't drop to shell after install (Alt+F1 doesn't work) | Use Option B from Step 8 — boot another USB with `ifre.v00` and recover from the troubleshooting menu. |
| `cp` to BOOTBANK fails with "permission denied" | You need to be `root`, not the install-user. `whoami` to check. |
| After reboot, NIC works but is slow | Realtek throughput tops out at ~700 Mbps. Normal. Not a fix needed. |

---

## If this still fails after 2 attempts

Stop sinking time into ESXi. Two pragmatic alternatives:

1. **Buy an Intel I210-T1 PCIe NIC card** (~₱700, 1–2 days delivery on Lazada/Shopee). Stock ESXi installer detects it instantly.
2. **Switch to VMware Workstation Pro** per `02b_Setup_SinglePC_Practice.md`. Same MA1/MA2/CTF practice, no driver fight, ready in 30 min.

The actual graded work (MA1 pentest, MA2 hardening, CTF) doesn't care whether the lab runs on ESXi or Workstation. Don't burn 4 hours of practice time fighting drivers.

---

## References (verified May 2026)

- **William Lam — Installing Realtek Driver Fling on free ESXi 8.0U3e** ⭐ source for this guide:
  `https://williamlam.com/2026/02/installing-realtek-network-driver-fling-using-free-esxi-8-0-update-3e-iso.html`
- **William Lam — Realtek Network Driver for ESXi (Nov 2025 background)**:
  `https://williamlam.com/2025/11/realtek-network-driver-for-esxi.html`
- **CoSci.de — RTL8111/8125/8126/8127 walkthrough (Dec 2025)**:
  `https://cosci.de/en/server-en/install-realtek-network-driver-on-vmware-esxi-8-0-3-how-to-enable-rtl8125-rtl8111-rtl8126-and-rtl8127/`
- **Broadcom Realtek Fling page**:
  `https://community.broadcom.com/vmware-cloud-foundation/discussion/realtek-network-card-drivers-for-esxi-v7-and-later`
- **Broadcom ESXi 8.0U3e free download KB**:
  `https://knowledge.broadcom.com/external/article/399823`
- **7-Zip (extracts `.vib` files)**: `https://www.7-zip.org/`

---

When the install is done and the NIC works → return to **`02c_Setup_ESXi_Server.md` Section E** (network config), then **`02_Setup_Topology.md` Section B** (port group creation).
