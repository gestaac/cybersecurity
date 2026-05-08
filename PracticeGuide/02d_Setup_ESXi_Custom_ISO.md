# 02d — Build a Custom ESXi 8 ISO with Realtek NIC Driver

The stock VMware ESXi 8 installer **dropped Realtek RTL8111/8168 support**. If your motherboard (Gigabyte boards in particular) uses an onboard Realtek NIC, the installer will fail with:

```
No Network Adapters
No network adapters were detected. Either no network adapters are physically
connected to the system, or a suitable driver could not be located.
```

This file walks you through **building a custom ESXi 8 ISO with the Realtek driver injected** so the install completes. ~45 minutes of work.

> ⚠️ **Honest caveats first** — read these before committing time:
>
> - **Officially unsupported by VMware/Broadcom.** Custom ISOs work but you're on your own if anything breaks.
> - **The Community Networking Driver Fling does NOT include Realtek drivers** (it covers Intel i219/i225/i226). For Realtek you need a **separate community VIB** maintained at v-front.de's vibsdepot.
> - **Performance:** Realtek NICs deliver ~700 Mbps vs ~940 Mbps Intel. Plenty for our practice lab.
> - **Easier alternative:** if you have **₱600–900 + 1–2 days delivery**, buy an Intel I210-T1 PCIe NIC card. Install proceeds with stock ISO. No driver hassle.
> - **Easiest alternative:** skip ESXi entirely, use VMware Workstation Pro on Windows (per `02b_…`). Same MA1/MA2/CTF practice, faster, no driver fight.
>
> Only proceed with this file if you specifically want bare-metal ESXi and don't want to buy a NIC card.

---

## What we're actually building

ESXi 8 has built-in support for Intel I219/I225/I226 NICs (the Community Networking Driver Fling content was merged into stock ESXi 8 in 2022). It has **no built-in Realtek support**.

We'll inject a **community-maintained Realtek VIB** (`net55-r8168` for RTL8168, or `net51-r8169` for older RTL8169 chips) into the stock ESXi 8 installer using VMware PowerCLI on a Windows machine, then write the result to a USB stick.

---

## Materials needed

| Item | Where | Confirmed working URL |
|---|---|---|
| Stock ESXi 8.0U3e ISO | Broadcom Support Portal (free, requires login) | `https://support.broadcom.com/group/ecx/free-downloads` (search "VMware vSphere Hypervisor 8") — KB landing: `https://knowledge.broadcom.com/external/article/399823` |
| Realtek community VIB (`net55-r8168`) | v-front.de's vibsdepot (community-maintained mirror) | `https://vibsdepot.v-front.de/wordpress/index-of-vibs-2/` |
| **PowerCLI 13.3.0** module | PowerShell Gallery | `https://www.powershellgallery.com/packages/VMware.PowerCLI` (install via `Install-Module`) |
| **William Lam's reference guide** (background reading) | williamlam.com | `https://williamlam.com/2022/02/how-to-create-a-customized-esxi-iso-without-vcenter-server.html` |
| **William Lam's Realtek driver post** (Nov 2025 — most current) | williamlam.com | `https://williamlam.com/2025/11/realtek-network-driver-for-esxi.html` |
| Rufus | rufus.ie | `https://rufus.ie/` |
| A Windows PC for the build | PC1 | Already in use |
| USB stick (8 GB+) | Already in use | — |

> 🔍 **About the "Community Networking Driver" Fling:** the original `flings.vmware.com` is gone. Broadcom moved Flings to `https://community.broadcom.com/vmware-cloud-foundation/viewdocument/community-network-driver-for-esxi-d`. **However, that Fling does not contain Realtek drivers** — only Intel i219/i225/i226. We'll use the Realtek-specific VIBs from v-front.de instead.

---

## Step 1 — Download ESXi 8.0U3e ISO from Broadcom (10 min)

1. From PC1 browser, visit: `https://support.broadcom.com/`
2. Create a free Broadcom account (if you don't have one). The acquisition moved everything off `customerconnect.vmware.com`.
3. Sign in → **My Downloads** → search "**VMware vSphere Hypervisor**" or "**ESXi 8**".
4. Choose **VMware vSphere Hypervisor 8.0 Update 3e** (free version, re-released as free in April 2025).
5. Accept EULA → download the ISO (~640 MB).
6. Save to `D:\esxi-build\` on PC1. Filename will be something like `VMware-VMvisor-Installer-8.0U3e-...iso`.

If Broadcom's portal gives you trouble (it's notoriously slow):
- KB article with direct guidance: `https://knowledge.broadcom.com/external/article/399823`
- The license key (free) shows on the same download page — copy it.

---

## Step 2 — Download the Realtek VIB (5 min)

1. From PC1 browser, visit: `https://vibsdepot.v-front.de/wordpress/index-of-vibs-2/`
2. The page is alphabetical. Search for one of:
   - `net55-r8168` (covers RTL8168/8111 — most common Gigabyte chipset, including RTL8111B/C/D/E/F/G/H)
   - `net51-r8169` (older RTL8169 chips — only if 8168 doesn't work)
3. Click the version compatible with **ESXi 8.0** (the file metadata shows targetESXi version).
4. Download the `.vib` or `.zip` file → save to `D:\esxi-build\`.

> 📝 **Identifying which VIB you need:** on Windows (PC1 or your 3rd PC), open Device Manager → Network adapters → right-click the Realtek entry → Properties → Details tab → set Property to "Hardware Ids". You'll see something like `PCI\VEN_10EC&DEV_8168...`. Device ID:
>
> | Device ID | Use VIB |
> |---|---|
> | 8168 | `net55-r8168` |
> | 8169 | `net51-r8169` |
> | 8125 | RTL8125 — not covered by either; you may need to fall back to USB Ethernet adapter or buy Intel NIC |

> ⚠️ **If v-front.de is down** (the maintainer is one person), alternative archived mirrors:
> - Internet Archive snapshot: search `archive.org` for `vibsdepot.v-front.de`
> - GitHub community mirrors: search `github.com` for `r8168 esxi vib`

---

## Step 3 — Install PowerCLI on PC1 (5 min)

Open **PowerShell as Administrator** on PC1:

```powershell
# Allow scripts (one-time per machine)
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser -Force

# Install PowerCLI from PowerShell Gallery
Install-Module -Name VMware.PowerCLI -Scope CurrentUser -Force -AllowClobber

# Suppress noisy CEIP/cert warnings
Set-PowerCLIConfiguration -InvalidCertificateAction Ignore -ParticipateInCEIP $false -Confirm:$false
```

PowerCLI is ~150 MB across many sub-modules — install takes ~3 min.

Verify version:
```powershell
Get-Module -ListAvailable VMware.PowerCLI | Select-Object Name, Version
```
Should print **13.3.0.24145083** or newer (current as of mid-2025).

> ⚠️ Broadcom recently flagged `VMware.PowerCLI` as **deprecated** in favor of `VCF.PowerCLI`. For ISO building, the old module still works perfectly. Don't worry about it.

---

## Step 4 — Build the custom ISO (10–15 min)

Stay in PowerShell as Admin. `cd` to your build folder:

```powershell
cd D:\esxi-build
ls   # should show: stock ESXi ISO + Realtek VIB/zip
```

### 4.1 — Open the stock ISO as a software depot

```powershell
Add-EsxSoftwareDepot .\VMware-VMvisor-Installer-8.0U3e-*.iso
```

### 4.2 — Add the Realtek VIB as a second depot

If you downloaded a `.zip`:
```powershell
Add-EsxSoftwareDepot .\net55-r8168_*.zip
```

If you downloaded a single `.vib`, you may need to wrap it (PowerCLI prefers depot zips). Easiest: keep the .zip from v-front.de.

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

Copy the `-standard` name (we want Tools included).

### 4.4 — Clone the standard profile

```powershell
$base = "ESXi-8.0U3e-24585291-standard"   # adjust to match what Get-EsxImageProfile printed
New-EsxImageProfile -CloneProfile $base -Name "ESXi-8.0U3e-with-Realtek" -Vendor "Custom" -AcceptanceLevel CommunitySupported
```

> ⚠️ The `-AcceptanceLevel CommunitySupported` flag is mandatory because the Realtek VIB is community-signed, not VMware-signed.

### 4.5 — List packages from the Realtek depot

```powershell
Get-EsxSoftwarePackage | Where-Object { $_.Name -like "*r8168*" -or $_.Name -like "*r8169*" }
```

Note the exact `Name` value (e.g., `net55-r8168`).

### 4.6 — Inject the package into your custom profile

```powershell
Add-EsxSoftwarePackage -ImageProfile "ESXi-8.0U3e-with-Realtek" -SoftwarePackage "net55-r8168"
```

If it errors with "Acceptance level too low", re-run with the bypass:
```powershell
Add-EsxSoftwarePackage -ImageProfile "ESXi-8.0U3e-with-Realtek" -SoftwarePackage "net55-r8168" -Force
```

### 4.7 — Export the new profile to a bootable ISO

```powershell
Export-EsxImageProfile -ImageProfile "ESXi-8.0U3e-with-Realtek" `
  -ExportToIso -FilePath "D:\esxi-build\ESXi-8.0U3e-with-Realtek.iso" -Force
```

5–10 min later you'll have:
```
D:\esxi-build\ESXi-8.0U3e-with-Realtek.iso
```

This is your custom installer.

---

## Step 5 — Write the custom ISO to USB with Rufus (3 min)

1. Plug the USB stick back into PC1.
2. Open **Rufus** (`https://rufus.ie/`).
3. **Device:** select your USB.
4. **Boot selection** → click **SELECT** → pick `ESXi-8.0U3e-with-Realtek.iso`.
5. **Partition scheme:** **GPT** (for UEFI boot) — match what the 3rd PC's BIOS is set to.
6. **File system:** leave default.
7. Click **START**.
8. Prompt: *"Write in DD Image mode?"* → **Yes**.
9. Confirm wipe → wait ~3 min → eject when done.

> ⚠️ **DD mode is required** for ESXi installers. If you accidentally pick ISO mode, the USB will boot but show errors during install.

---

## Step 6 — Boot the 3rd PC from the custom USB (5 min)

1. Plug the USB into the 3rd PC.
2. Power on → press **F12** repeatedly during the Gigabyte splash logo to open the boot menu.
3. Select your USB → **Enter**.
4. ESXi installer loads (yellow/black VMware splash).
5. **Critical check:** at the early boot stage, before the installer prompts, the screen briefly shows kernel module load messages. Look for:
   ```
   Loading /vmfs/.../net55-r8168.v00
   ```
   That confirms your driver is being loaded. If you see it, the install will succeed.
6. The installer prompt should now show **"1 NIC(s) found"** instead of the previous "No Network Adapters" error.

---

## Step 7 — Continue the install per `02c_…` Section D

From here, the install is identical to a stock ESXi install. Pick up at:

- **`02c_Setup_ESXi_Server.md` Section D** — wizard click-by-click (welcome → EULA → disk → keyboard → root pwd → install)
- Then **Section E** — set static management IP per `02_Setup_Topology.md` Section A

---

## Verification after install

Once ESXi is up and running:

### V1 — Console shows IP
The yellow/grey ESXi splash should show `https://192.168.x.x/` at the top. If it shows `0.0.0.0` → set static IP via F2 → Configure Management Network.

### V2 — From PC1 browser
Browse to `https://<ESXi-IP>/ui` → log in as `root` → confirm the web UI loads.

### V3 — Confirm Realtek NIC is present
ESXi web UI → **Networking → Physical NICs**. You should see `vmnic0` with driver `r8168` listed.

If you see `vmnic0` but the speed shows as `0 Mbps`:
- SSH into ESXi (enable SSH first via *Manage → Services → TSM-SSH → Start*)
- Run `esxcfg-nics -l` → check link state
- Try a different ethernet cable / switch port

---

## Troubleshooting

| Error | Fix |
|---|---|
| `Add-EsxSoftwareDepot: Could not load file or assembly` | PowerCLI install incomplete. `Update-Module VMware.PowerCLI -Force` |
| `Add-EsxSoftwarePackage: The package 'net55-r8168' could not be located` | Check the exact name from Step 4.5. Some VIBs are named `net-r8168` instead of `net55-r8168` depending on author. |
| `Insufficient acceptance level` | Add `-Force` flag to `Add-EsxSoftwarePackage`. Confirm clone profile was created with `-AcceptanceLevel CommunitySupported`. |
| ISO builds successfully but still says "No NIC" at install | Wrong VIB for your chipset. Re-check Device ID per Step 2. RTL8125 chips need a different driver (or won't work). |
| Custom ISO boots but kernel panics at "PSOD" | Driver incompatibility with ESXi 8. Try the older `net51-r8169` driver instead. If still fails: this NIC is unsupported, fall back to Intel PCIe card or Workstation. |
| Rufus shows no "DD mode" prompt | Re-download Rufus latest. Or: re-export the ISO with `Export-EsxImageProfile -CreateImage` flag. |

---

## If this still fails

You've hit a NIC chipset that simply has no community VIB working with ESXi 8. Three remaining options, in order of effort:

1. **Buy Intel I210-T1 PCIe NIC** (~₱700, 1–2 day delivery on Lazada/Shopee). Plug in, retry stock installer. Done.
2. **Try a different ESXi version**: ESXi 7.0 U3 has broader Realtek support. You can run ESXi 7 → it's fine for our practice lab; the MA2 deliverables don't care about the ESXi version.
3. **Skip ESXi, use VMware Workstation Pro** per `02b_Setup_SinglePC_Practice.md`. Same MA1/MA2/CTF practice, no driver fight, ready in 30 min.

Honest take: if the custom ISO route fails twice, **switch to Workstation Pro and don't look back**. You'll save 4+ hours that should go into actual practice.

---

## Updating to a new ESXi version later

If Broadcom releases ESXi 8.0U4 etc., you'll need to rebuild the custom ISO:
1. Download new stock ISO.
2. Repeat Steps 4.1–4.7 with the new ISO + same Realtek VIB.
3. The Realtek VIB (`net55-r8168`) should keep working across ESXi 8.x updates because the kernel ABI is stable within a major version.

For ESXi 9.0 (when released): the VIB will likely need a new community version. Watch v-front.de or William Lam's blog.

---

## References (verified May 2026)

- **William Lam — Custom ESXi ISO without vCenter** (PowerCLI walkthrough): `https://williamlam.com/2022/02/how-to-create-a-customized-esxi-iso-without-vcenter-server.html`
- **William Lam — Realtek Network Driver for ESXi** (most current Realtek-specific guide, Nov 2025): `https://williamlam.com/2025/11/realtek-network-driver-for-esxi.html`
- **v-front.de vibsdepot** (community VIB host for Realtek + other consumer NICs): `https://vibsdepot.v-front.de/wordpress/index-of-vibs-2/`
- **ESXi-Customizer-PS GitHub** (alternative tool, last release v2.9.0 Nov 2022, still works): `https://github.com/VFrontDe-Org/ESXi-Customizer-PS`
- **PowerCLI on PSGallery**: `https://www.powershellgallery.com/packages/VMware.PowerCLI`
- **Broadcom ESXi 8.0U3e free download KB**: `https://knowledge.broadcom.com/external/article/399823`
- **Broadcom Community — Community Networking Driver page** (Intel-only, FYI): `https://community.broadcom.com/vmware-cloud-foundation/viewdocument/community-network-driver-for-esxi-d`

When done with the install → return to **`02c_Setup_ESXi_Server.md` Section E** for the post-install network config, then **`02_Setup_Topology.md` Section B** for port group creation.
