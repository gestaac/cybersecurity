# 02d — Build a Custom ESXi 8 ISO with Realtek NIC Driver

The stock VMware ESXi 8 installer **does not include Realtek RTL8111/8125/8126/8127 NIC drivers**. If your motherboard (especially Gigabyte boards) uses an onboard Realtek NIC, the installer fails with:

```
No Network Adapters
No network adapters were detected. Either no network adapters are physically
connected to the system, or a suitable driver could not be located.
```

**Good news (as of November 2025):** Broadcom released an **official Realtek Network Driver Fling** for ESXi 8.0 U3 and later. You no longer need community-maintained VIBs from defunct mirrors. This file walks you through using the official Broadcom Fling to build a working installer.

> ✅ **Verified May 2026:** Broadcom-maintained Fling, not community-hacked. Andreas Peetz's old `v-front.de/vibsdepot` is dead — don't try to use it.

---

## What we're building

A custom ESXi 8.0U3e installer ISO with the **official Broadcom Realtek driver** baked in, so the installer detects your Realtek NIC during install.

**Time:** ~45 minutes if everything works first try.
**Cost:** Free.
**Difficulty:** Moderate (requires PowerShell + PowerCLI on a Windows PC).

---

## Honest caveats first

| Caveat | Reality |
|---|---|
| Officially supported by Broadcom | ✅ Yes — released Nov 2025, updated to 1.101.01 shortly after |
| ESXi 8.0 U3 / ESXi 9.x compatibility | ✅ Confirmed |
| ESXi 7.x compatibility | ❌ Not supported (vmklinux drivers were removed in 7) |
| RTL8111 / RTL8125 / RTL8126 / RTL8127 chips | ✅ Supported (covers 8168 family — what's on most Gigabyte boards) |
| Hardware offload (TSO/LRO/WOL) | ❌ Not supported by the Fling — basic connectivity only. Fine for our practice lab. |
| Performance | Realtek ~700 Mbps vs Intel ~940 Mbps. Plenty for the lab. |
| Will it break on ESXi 8 updates | Low risk — Broadcom is shipping it themselves now |

> **Easier alternatives if you don't want to do this:**
> - **Buy an Intel I210-T1 PCIe NIC** (~₱700, 1–2 day shipping). Stock ESXi installer detects it instantly.
> - **Skip ESXi entirely**, use VMware Workstation Pro per `02b_Setup_SinglePC_Practice.md`. Same MA1/MA2/CTF practice, no driver fight, ready in 30 min.
>
> Only proceed with this guide if you specifically want bare-metal ESXi.

---

## Materials needed (all verified working May 2026)

| # | Item | Source URL | Notes |
|---|---|---|---|
| 1 | **Stock ESXi 8.0U3e ISO** | `https://support.broadcom.com/group/ecx/free-downloads` | Free; requires Broadcom account. Search "VMware vSphere Hypervisor 8". KB landing: `https://knowledge.broadcom.com/external/article/399823` |
| 2 | **Realtek Driver Fling** (`VMware-Re-Driver_1.101.01-...zip`) | `https://support.broadcom.com/group/ecx/productdownloads?subfamily=Flings&freeDownloads=true` | Same Broadcom portal, **Flings** section. Same login. |
| 3 | **PowerCLI 13.3.0** module | `https://www.powershellgallery.com/packages/VMware.PowerCLI` | Install via `Install-Module` (covered below) |
| 4 | **Rufus** | `https://rufus.ie/` | Already in use |
| 5 | A Windows PC with internet | PC1 | For the build |
| 6 | USB stick (8 GB+) | Already in use | We'll re-write it |

### Reference reading (verified working)

- **William Lam — Realtek Network Driver for ESXi (Nov 2025)**: `https://williamlam.com/2025/11/realtek-network-driver-for-esxi.html`
- **William Lam — Installing on free ESXi 8.0U3e (Feb 2026)** ⭐: `https://williamlam.com/2026/02/installing-realtek-network-driver-fling-using-free-esxi-8-0-update-3e-iso.html`
- **CoSci.de — How-to for RTL8111/8125/8126/8127 on ESXi 8.0U3 (Dec 2025)**: `https://cosci.de/en/server-en/install-realtek-network-driver-on-vmware-esxi-8-0-3-how-to-enable-rtl8125-rtl8111-rtl8126-and-rtl8127/`

These three blog posts are the authoritative external references. If anything in this guide is unclear, cross-reference there.

---

## Step 1 — Download the stock ESXi 8.0U3e ISO from Broadcom (15 min)

1. From PC1 browser, visit: `https://support.broadcom.com/`
2. Sign in (or register a free Broadcom account — needed for both downloads).
3. Navigate to **My Downloads** → search "**VMware vSphere Hypervisor 8**".
4. Pick **VMware vSphere Hypervisor 8.0 Update 3e** (the current free version, re-released April 2025).
5. Accept the EULA, copy the free license key shown on the same page (you'll need it later).
6. Download the ISO (~640 MB).
7. Save to `D:\esxi-build\` on PC1. The filename will be similar to:
   ```
   VMware-VMvisor-Installer-8.0U3e-24585291.x86_64.iso
   ```

> ⚠️ Broadcom's download portal is slow and confusing. If you can't find it via search, try the KB direct link: `https://knowledge.broadcom.com/external/article/399823`.

---

## Step 2 — Download the Realtek Driver Fling from Broadcom (5 min)

Same Broadcom portal, different section.

1. From PC1 browser: `https://support.broadcom.com/group/ecx/productdownloads?subfamily=Flings&freeDownloads=true`
2. Find **Realtek Network Driver for ESXi** in the Flings list.
3. Latest version as of writing: **1.101.01** (Nov 2025 update — fixes early 1.x bugs).
4. Download: `VMware-Re-Driver_1.101.01-5vmw.800.1.0.20613240.zip` (or newer build number).
5. Save to `D:\esxi-build\` alongside the stock ISO.

> 💡 **Verify your chipset is supported** before downloading. The Fling covers:
> - RTL8111 (most Gigabyte boards — including yours)
> - RTL8125 (newer 2.5 GbE chips)
> - RTL8126 (5 GbE)
> - RTL8127 (10 GbE)
>
> All "8168" devices use the RTL8111 family driver — supported.

To confirm your chipset on Windows: **Device Manager → Network adapters → Realtek entry → Properties → Details → Hardware IDs** → look for `PCI\VEN_10EC&DEV_8168` (RTL8111 family) or `&DEV_8125` (RTL8125).

---

## Step 3 — Install PowerCLI on PC1 (5 min)

Open **PowerShell as Administrator** on PC1:

```powershell
# Allow scripts (one-time per machine)
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser -Force

# Install PowerCLI from PowerShell Gallery
Install-Module -Name VMware.PowerCLI -Scope CurrentUser -Force -AllowClobber

# Suppress noisy warnings (CEIP, SSL)
Set-PowerCLIConfiguration -InvalidCertificateAction Ignore -ParticipateInCEIP $false -Confirm:$false
```

PowerCLI is ~150 MB across many sub-modules — install takes ~3 min.

Verify:
```powershell
Get-Module -ListAvailable VMware.PowerCLI | Select-Object Name, Version
```
Should print **13.3.0.24145083** or newer.

> ⚠️ Broadcom flagged `VMware.PowerCLI` as deprecated in favor of `VCF.PowerCLI`. For ISO-building, the old module still works perfectly. Don't worry about it.

---

## Step 4 — Build the custom ISO with PowerCLI (10–15 min)

Stay in PowerShell as Administrator. Switch to your build folder:

```powershell
cd D:\esxi-build
ls   # should show the stock ISO + the Re-Driver zip
```

### 4.1 — Open the stock ISO as a software depot

```powershell
Add-EsxSoftwareDepot .\VMware-VMvisor-Installer-8.0U3e-*.iso
```

### 4.2 — Add the Realtek Fling as a second depot

```powershell
Add-EsxSoftwareDepot .\VMware-Re-Driver_1.101.01-*.zip
```

(Adjust filename to whatever you actually downloaded.)

### 4.3 — Find the stock image profile name

```powershell
Get-EsxImageProfile | Sort-Object CreationTime -Descending | Select-Object Name, CreationTime -First 5
```

You'll see something like:
```
Name                                           CreationTime
----                                           ------------
ESXi-8.0U3e-24585291-standard                  2025-04-15
ESXi-8.0U3e-24585291-no-tools                  2025-04-15
```

Copy the `-standard` name (the one with VMware Tools — recommended).

### 4.4 — Clone the standard profile

```powershell
$base = "ESXi-8.0U3e-24585291-standard"   # adjust to match what you saw above
New-EsxImageProfile -CloneProfile $base -Name "ESXi-8.0U3e-with-Realtek" -Vendor "Custom" -AcceptanceLevel PartnerSupported
```

> The Broadcom Fling is **PartnerSupported** acceptance level (not Community), so we set the clone profile to match.

### 4.5 — List the Realtek packages from the Fling depot

```powershell
Get-EsxSoftwarePackage | Where-Object { $_.Name -like "*re-*" -or $_.Name -like "*realtek*" }
```

You should see the package name. Most likely: `VMware-Re-Driver` or just `re`. Note the exact `Name` value.

### 4.6 — Inject the package into your custom profile

Replace `<package-name>` with what Step 4.5 returned:

```powershell
Add-EsxSoftwarePackage -ImageProfile "ESXi-8.0U3e-with-Realtek" -SoftwarePackage "<package-name>"
```

If acceptance-level issues:
```powershell
Add-EsxSoftwarePackage -ImageProfile "ESXi-8.0U3e-with-Realtek" -SoftwarePackage "<package-name>" -Force
```

### 4.7 — Export the new profile to a bootable ISO

```powershell
Export-EsxImageProfile -ImageProfile "ESXi-8.0U3e-with-Realtek" `
  -ExportToIso -FilePath "D:\esxi-build\ESXi-8.0U3e-with-Realtek.iso" -Force
```

5–10 min later you have:
```
D:\esxi-build\ESXi-8.0U3e-with-Realtek.iso
```

That's your custom installer.

---

## Step 5 — Write the custom ISO to USB with Rufus (3 min)

1. Plug the USB stick back into PC1.
2. Open **Rufus** (`https://rufus.ie/`).
3. **Device:** select your USB.
4. **Boot selection** → **SELECT** → pick `ESXi-8.0U3e-with-Realtek.iso`.
5. **Partition scheme:** **GPT** (for UEFI boot).
6. **File system:** leave default.
7. Click **START**.
8. Prompt: *"Write in DD Image mode?"* → **Yes**.
9. Confirm wipe → wait ~3 min → eject.

> ⚠️ **DD mode is mandatory** for ESXi installers. ISO mode will boot but error mid-install.

---

## Step 6 — Boot the 3rd PC from the custom USB

1. Plug the USB into the 3rd PC.
2. Power on → press **F12** repeatedly during the Gigabyte splash logo to open the boot menu.
3. Select your USB → **Enter**.
4. ESXi installer loads (yellow/black VMware splash).
5. **Watch the early boot text** — you should see kernel module load lines like:
   ```
   re               loaded successfully
   ```
   (or similar Realtek confirmation).
6. The installer prompt should now show **"1 NIC(s) found"** instead of the previous "No Network Adapters" error.

---

## Step 7 — Continue the install per `02c_…` Section D

From here, the install is identical to a stock ESXi install:

- **`02c_Setup_ESXi_Server.md` Section D** — wizard click-by-click (welcome → EULA → disk → keyboard → root pwd → install).
- Then **Section E** — set static management IP per `02_Setup_Topology.md` Section A.

---

## Verification after install

### V1 — Console shows IP
The yellow/grey ESXi splash should show `https://192.168.x.x/` at the top. If `0.0.0.0` → set static IP via F2 → Configure Management Network.

### V2 — From PC1 browser
Browse to `https://<ESXi-IP>/ui` → log in as `root` → confirm web UI loads.

### V3 — Confirm Realtek NIC is detected
ESXi web UI → **Networking → Physical NICs**. You should see `vmnic0` with driver `re` (or `r8168`, depending on driver version).

If `vmnic0` is up but speed shows `0 Mbps`:
- SSH to ESXi (enable via *Manage → Services → TSM-SSH → Start*)
- Run `esxcfg-nics -l` → check link state
- Verify cable + switch port good

---

## Alternative: post-install driver injection (if custom-ISO approach fails)

If for some reason the custom ISO won't build, you can **install ESXi via temporary USB-Ethernet adapter or different machine, then add the driver afterward**:

```bash
# SSH into running ESXi first
esxcli software component apply -d /vmfs/volumes/datastore1/VMware-Re-Driver_1.101.01-*.zip
reboot
```

After reboot the Realtek NIC will be detected. CoSci.de's blog post (linked above in References) walks this through.

---

## Troubleshooting

| Error | Fix |
|---|---|
| `Add-EsxSoftwareDepot: Could not load file or assembly` | PowerCLI install incomplete. `Update-Module VMware.PowerCLI -Force` |
| `Add-EsxSoftwarePackage: package could not be located` | The package name is different. Re-run Step 4.5 with broader filter: `Get-EsxSoftwarePackage \| Format-Table Name, Vendor` |
| `Insufficient acceptance level` | Add `-Force` flag. The Fling is PartnerSupported, profile clone must match. |
| ISO builds but still "No NIC" at install | Re-check chipset against supported list (RTL8111/8125/8126/8127). RTL8169 (older) is **not** in this Fling — that one needs a community VIB or a different approach. |
| Custom ISO boots but PSOD (purple screen of death) | Driver incompatibility. Try fresh build with the latest Re-Driver version. If still fails, switch strategy (Intel NIC or Workstation). |
| Rufus has no DD mode prompt | Update Rufus. Or re-export the ISO; PowerCLI sometimes produces a non-bootable ISO if it ran low on disk. |

---

## If this fails twice — fallback strategy

You've spent ~1.5 hours and the custom-ISO approach isn't working. Don't sink more time. In order of preference:

1. **Buy Intel I210-T1 PCIe NIC** (~₱700, 1–2 days). Stock ESXi installer detects it instantly.
2. **Use ESXi 7.0 U3** instead of 8.x. ESXi 7 has broader NIC support natively. The lab works identically — none of the MA1/MA2/CTF deliverables care about the ESXi version.
3. **Switch to VMware Workstation Pro** per `02b_Setup_SinglePC_Practice.md`. 30 min to running, no driver issues.

**Be honest about cost vs benefit:** the ESXi web UI experience is worth ~5 minutes of unfamiliarity at competition. If the custom-ISO route is eating multiple hours of your prep window, it's the wrong investment.

---

## Updating to a future ESXi version

If Broadcom releases ESXi 8.0 U4+ or ESXi 9.x:

1. Download the new stock ISO.
2. Repeat Steps 4.1–4.7 with the new ISO + the **same Re-Driver Fling** (Broadcom maintains it across versions).
3. The driver should keep working — Broadcom has committed to maintaining it.

For ESXi 9.0 specifically: the William Lam Feb 2026 post confirms compatibility.

---

## References (verified working May 2026)

- **William Lam — Realtek Network Driver for ESXi (Nov 2025)**: `https://williamlam.com/2025/11/realtek-network-driver-for-esxi.html`
- **William Lam — Installing on free ESXi 8.0U3e (Feb 2026)** ⭐ best step-by-step: `https://williamlam.com/2026/02/installing-realtek-network-driver-fling-using-free-esxi-8-0-update-3e-iso.html`
- **CoSci.de — RTL8111/8125/8126/8127 install guide (Dec 2025)**: `https://cosci.de/en/server-en/install-realtek-network-driver-on-vmware-esxi-8-0-3-how-to-enable-rtl8125-rtl8111-rtl8126-and-rtl8127/`
- **andysworld.org.uk — Minisforum Realtek install (Jan 2026)**: `https://andysworld.org.uk/2026/01/11/minisforum-ms-a2-how-to-install-the-new-realtek-driver-on-esxi-8-0/`
- **Broadcom Community — Realtek discussion thread**: `https://community.broadcom.com/vmware-cloud-foundation/discussion/realtek-network-card-drivers-for-esxi-v7-and-later`
- **Broadcom ESXi 8.0U3e free download KB**: `https://knowledge.broadcom.com/external/article/399823`
- **PowerCLI on PSGallery**: `https://www.powershellgallery.com/packages/VMware.PowerCLI`

---

When done with the install → return to **`02c_Setup_ESXi_Server.md` Section E** for post-install network config, then **`02_Setup_Topology.md` Section B** for port group creation.
