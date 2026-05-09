# 08 — Workstations: Windows Host OS Setup + Best Practices

This file is the **canonical reference** for preparing **PC1 and PC2** — your two competition workstations. It covers Windows host prep, browser/SSH config, snapshot conventions, VM backups (on ESXi), file-share between teammates, and a recovery cookbook.

> Our setup: **2 actual PCs + 1 ESXi server**. PC1 + PC2 are thin VM viewers — they connect to the ESXi server's web UI to manage VMs, and they only host light tooling locally (browser, Burp, etc.).

> No marks earned by anything in this file — it's enabling work for everything in `10_…` through `53_…`.

---

## A. When to use this file

| Setup step | Read which file |
|---|---|
| ESXi server install | `02c_Setup_ESXi_Server.md` (+ `02d_…` if NIC driver needed) |
| Network rig (TP-Link router + switch + 3 boxes) | `02_Setup_Topology.md` |
| **Windows host prep on PC1 + PC2** | **This file** Sections B–F |
| Browser bookmarks, SSH aliases, snapshot conventions | **This file** Sections G–N |

---

## B. Hardware sanity check

PC1 and PC2 each need (minimum):
| Component | Min | Recommended |
|---|---|---|
| CPU | 4 cores, virtualization extensions enabled | 6+ cores |
| RAM | 8 GB | **16 GB** ✅ what you have |
| Disk | 256 GB SSD | **500 GB SSD** ✅ what you have |
| NIC | 1 onboard gigabit | 2 onboard or 1 + USB-Ethernet |
| Display | 1× 1080p | 2 monitors per teammate |

> Per-PC role recap: PC1 + PC2 are **VM viewers** with light local tooling (browser, Burp, optional Kali VM). The heavy VM hosting happens on the ESXi server. So 16 GB / 500 GB is plenty.

---

## C. BIOS / UEFI settings (do once per PC)

Entering BIOS varies per motherboard (typically Del or F2 at boot).

| Setting | Value | Why |
|---|---|---|
| Intel Virtualization Technology / AMD-V (SVM) | **Enabled** | Required if you ever boot a small local VM (Burp's embedded browser, etc.) |
| Intel VT-d / AMD-Vi | Enabled (optional) | Future-proofs you for advanced features |
| Hyper-Threading | Enabled | Extra logical cores |
| Secure Boot | Enabled (default is fine for Windows) | OK for the workstations (unlike ESXi server which needs it OFF) |
| Boot order | NVMe / SSD first | So Windows boots fast |
| Fast Boot | Disabled | Cleaner boot, fewer driver headaches |
| Power on after AC loss | Optional | Convenient if power flickers |

---

## D. Windows host prep

### D.1 Disable Hyper-V / WSL2 / Memory Integrity (only if you'll boot local VMs)

PC1 + PC2 are thin clients — most VMs run on ESXi, not locally. **You can SKIP this step** unless you plan to run a Kali / Juice Shop VM locally on PC1 or PC2.

If you DO need local VM hosting, run **PowerShell as Administrator**:

```powershell
Disable-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V -All -NoRestart
Disable-WindowsOptionalFeature -Online -FeatureName HypervisorPlatform -NoRestart
Disable-WindowsOptionalFeature -Online -FeatureName VirtualMachinePlatform -NoRestart
Disable-WindowsOptionalFeature -Online -FeatureName Containers-DisposableClientVM -NoRestart
bcdedit /set hypervisorlaunchtype off
Restart-Computer
```

Also: *Settings → Privacy & Security → Windows Security → Device Security → Core Isolation → Memory Integrity → Off*. Reboot.

### D.2 Antivirus exclusions (recommended)

Helps if you store ISOs / OVAs locally on PC1/PC2 before uploading to ESXi.
Settings → Windows Security → Virus & threat protection → Manage settings → Exclusions → Add → **Folder**:
- `D:\ISO` (where you stage OS ISOs before uploading to ESXi datastore)
- `D:\OVA` (where you stage VulnHub OVAs before uploading)
- `D:\Tools`
- `D:\Notes`

### D.3 Power plan + sleep
Settings → System → Power → Power mode = **Best performance**.
Settings → System → Power & battery → Screen and sleep:
- Screen turns off: **Never** (or 30 min)
- PC goes to sleep: **Never**

Disable Fast Startup (Control Panel → Power Options → *Choose what the power buttons do* → uncheck *Turn on fast startup*).

### D.4 Pagefile sizing
Default Windows-managed pagefile is fine **unless** your D: drive is also where VMs live. If so, move pagefile to C: only:
*System Properties → Advanced → Performance → Settings → Advanced → Virtual memory → Change* → uncheck "Automatically manage" → set custom on C: only, "No paging file" on D:.

---

## E. Folder layout (standardised across both PCs)

```
D:\
├── ISO\                    # OS installers (staged before upload to ESXi datastore)
│   ├── pfSense-CE-2.7.2.iso
│   ├── CentOS-Stream-9.iso
│   ├── Win_Server_2022.iso
│   ├── Win10_Enterprise.iso
│   └── securityonion-2.4.iso
├── OVA\                    # VulnHub + Kali OVA files (staged for ESXi upload)
├── PracticeGuide\          # copy of this guide
├── Tools\                  # local tool installers (run on PC1/PC2)
│   ├── Burp_Community.exe
│   ├── PuTTY.exe
│   ├── WinSCP.exe
│   ├── Wireshark.exe
│   ├── Nmap.exe
│   ├── Node20.msi
│   ├── OVF-Tool.msi        # convert .vmx → .ova for ESXi
│   ├── Greenshot.exe
│   ├── LibreOffice.msi
│   └── linpeas.sh, winPEASany.exe
├── Wordlists\              # rockyou.txt, SecLists, PayloadsAllTheThings
├── References\             # offline reading
│   ├── Pwning_JuiceShop.pdf
│   ├── HackTricks_chapters.pdf
│   ├── GTFOBins\           # cloned from GitHub
│   └── OWASP_Top10.pdf
└── Notes\                  # YOUR running scratchpad — personal
```

Both PCs use **the same paths** so any guide step works on either teammate's box.

---

## F. VMware OVF Tool (for converting OVAs)

In our setup VMware Workstation Pro is **NOT** the primary VM host — that's the ESXi server. But you'll occasionally need to **convert `.vmx` files to `.ova`** before uploading to ESXi (e.g., the Kali Linux pre-built image, some VulnHub VMs).

Install OVF Tool on **both PC1 and PC2**:
1. Download (free): `https://developer.vmware.com/web/tool/4.6.0/ovf`.
2. Run the installer.
3. Test:
   ```cmd
   "C:\Program Files\VMware\VMware OVF Tool\ovftool.exe" --version
   ```
4. Typical use:
   ```cmd
   ovftool.exe target.vmx target.ova
   ```
5. Upload the resulting `.ova` to ESXi datastore via the **Storage → Datastore browser → Upload** UI.

> 💡 You don't need the full VMware Workstation Pro installed. OVF Tool alone is enough for this conversion task.

---

## G. Browser bookmarks (paste into Firefox / Chrome)

Pre-populate these on **both PCs** so during practice you don't waste time typing URLs.

### G.1 Cybersecurity practice folder
| Name | URL |
|---|---|
| ESXi web UI | `https://192.168.1.10/ui` |
| pfSense web UI (LAN side) | `https://172.16.100.254` |
| pfSense web UI (alt) | `https://192.168.1.254` |
| Juice Shop | `http://localhost:3000` |
| Juice Shop Score Board | `http://localhost:3000/#/score-board` |
| WinSRV1 IIS test | `https://webtest.manila.com` |
| LinSRV1 web | `https://www.manila.com` |
| Burp's CA cert page | `http://burp` |

### G.2 Reference (offline-cached PDFs)
| Name | URL or local path |
|---|---|
| Pwning Juice Shop (PDF) | `file:///D:/References/Pwning_JuiceShop.pdf` |
| HackTricks (PDF) | `file:///D:/References/HackTricks_chapters.pdf` |
| GTFOBins (offline) | `file:///D:/References/GTFOBins/index.html` |

To import as a single block: bookmark this page in Firefox, then *Bookmarks → Manage Bookmarks → Import and Backup → Import Bookmarks from HTML* if you save the above as an HTML file.

---

## H. SSH config aliases (Windows OpenSSH client)

Windows 10/11 has OpenSSH built in (`ssh.exe`). Create aliases so you type `ssh esxi` instead of `ssh root@192.168.1.10`.

Edit (or create) `C:\Users\<YourName>\.ssh\config`:

```
Host esxi
    HostName 192.168.1.10
    User root
    StrictHostKeyChecking no
    UserKnownHostsFile /dev/null

Host pfsense
    HostName 172.16.100.254
    User admin
    Port 22

Host linsrv1
    HostName 192.168.1.10
    User C1@manila.com
    Port 2022

Host winsrv1
    HostName 192.168.2.10
    User Administrator
    Port 22

Host kali
    HostName 192.168.2.2
    User kali
```

> Adjust IPs to match your ESXi environment. The `kali` IP is its static IP on `PG-MA1-CMS` set in `03_Setup_VMs_MA1.md` Step 3.2.

Test: open Command Prompt → `ssh esxi` → password prompt → done.

---

## I. Snapshot naming convention (USE THIS)

Currently the guide says "take a snapshot" 12 times without a system. Adopt this:

```
<vm-name>-<state>-YYYY-MM-DD
```

Examples:
- `LinSRV1-base-2026-05-07` — fresh install, before hardening
- `LinSRV1-hardened-2026-05-07` — after `21_…` complete
- `WINSRV1-policies-applied-2026-05-08`
- `pfSense-firewall-rules-done-2026-05-08`
- `MA2-VM-set-baseline-2026-05-08` (if multi-VM consolidated)

**Rules:**
1. **One snapshot per logical state**, not one per minor change.
2. **Delete old snapshots** within a week — they consume disk linearly.
3. **Take a snapshot** before any irreversible operation (e.g. CSR signing, GPO link, firewall rule import).
4. **Don't snapshot a powered-on VM with active work** — snapshot only when idle / paused / off.

---

## J. VM backup strategy

Snapshots are **not backups** — they live on the same disk.

### J.1 Periodic OVA exports (do weekly during practice)
*ESXi UI: Right-click VM → Export* — saves the VM as `.ovf` + `.vmdk` files to your local PC's download folder.

Save the exported files to:
- A **different drive** than the ESXi datastore (e.g., download to PC1 then copy to external SSD), or
- An **external SSD** dedicated to backups.

Naming: `<vm>-<state>-YYYYMMDD.ova` e.g. `LinSRV1-hardened-20260507.ova`.

### J.2 Critical VMs — back up always
- WINSRV1 (DC) — losing it = lose AD = lose half your practice work
- WINSRV3 (CA) — losing it = re-issue every cert
- pfSense — losing it = re-do all firewall rules

### J.3 Disposable VMs — don't bother backing up
- Client1/2/3 — rebuild in 30 min from ISO
- VulnHub VMs — re-download
- Juice Shop — `npm start` again

### J.4 Restore test (do once)
Pick one OVA backup. ESXi UI → Create / Register VM → Deploy from OVA → import → boot it → confirm it works. If you've never tested a restore, you don't have a backup.

---

## K. File-share between PC1 and PC2 during practice

You'll need to swap configs / screenshots / OVAs between teammates. Three options:

### K.1 Windows SMB share (recommended)
On PC1:
1. Create folder `D:\Share\` → right-click → Properties → Sharing → *Advanced Sharing* → tick "Share this folder" → Permissions → Everyone Read/Write.
2. From PC2: open File Explorer → type `\\PC1` in the address bar → enter PC1's user/password → see the Share folder.

### K.2 USB stick passed back and forth
Lowest tech, always works. Slow for large files (OVAs).

### K.3 Single user on both PCs (advanced)
Use the same Microsoft account on both PCs and let OneDrive sync `D:\Share\`. Avoid putting OVAs in OneDrive — they're huge.

---

## L. Recovery cookbook — when things go wrong

| Symptom | First thing to try |
|---|---|
| VM won't boot | ESXi UI → VM → Snapshots → *Revert to last good*. If no snapshot → boot from install ISO + recovery mode |
| VM boots but no network | ESXi UI → VM → Edit → Network Adapter: confirm correct port group. Inside VM: `nmcli con up <conn>` (Linux) or `netsh int ip reset` (Windows) |
| pfSense forgot rules | Restore from `Diagnostics → Backup & Restore → Restore configuration` |
| WinSRV1 AD broken | If you have an OVA backup, restore. Otherwise: `dcpromo /forceremoval` then reinstall — **slow** |
| LinSRV1 SELinux blocking httpd | `setenforce 0` temporarily, fix context with `restorecon -Rv /var/www/manila`, `setenforce 1` |
| Burp showing cert errors on Juice Shop | Re-import Burp CA cert (`05_…` Step 5.2). Restart Firefox |
| Juice Shop won't start | On Kali (per `05_…`): `cd ~/juice-shop_<ver> && rm -rf node_modules && npm install && npm start` |
| Kali VM stuck at boot | ESXi UI → Console → press `e` at GRUB → boot single-user → fix `/etc/fstab` if disk error |
| ESXi web UI hangs | SSH in (`ssh esxi`) → `/etc/init.d/hostd restart && /etc/init.d/vpxa restart` |
| Snapshot tree got bloated on ESXi datastore | ESXi UI → VM → Snapshots → Delete oldest first |

---

## M. Documentation toolset (install on both PCs)

For writing the MA1 appendix, MA2 GPO recommendations, Day 2 incident report, CTF report.

| Tool | Source | Purpose |
|---|---|---|
| **LibreOffice** | `https://www.libreoffice.org/download/download/` | Free Word-compatible editor for the appendix doc |
| **Greenshot** | `https://getgreenshot.org/downloads/` | Screen capture with annotations (better than Win+Shift+S) |
| **Notepad++** | `https://notepad-plus-plus.org/downloads/` | Light editor for `*.md` files + configs |
| **MS Print to PDF** | Built into Windows | Final submission as PDF |

---

## N. Pre-practice 90-second checklist (every session)

Before starting any practice session, run through this:

1. [ ] PC1 + PC2 both powered on, both can `ping 192.168.1.10` (ESXi)
2. [ ] Browser tabs preloaded: ESXi UI, pfSense (if doing MA2), Juice Shop (if doing CTF)
3. [ ] Today's session noted in your `Notes/session-log.md` (date, what you'll practise, target end time)
4. [ ] Stopwatch / phone timer set for the time-box (e.g. 90 min for MA1)
5. [ ] If memory tight: shut down unused VMs first
6. [ ] Snapshot taken on each VM you're about to change

After the session:
1. [ ] Tally what you completed against `99_Marking_Map.md`
2. [ ] Note any failed steps in `Notes/issues-to-fix.md`
3. [ ] Take "end-of-session" snapshots so next session resumes here
4. [ ] Power off non-critical VMs

---

## O. Verification

After this file's setup is done, both PCs should:

- [ ] Have all `D:\...` folders created and AV-excluded
- [ ] VMware OVF Tool installed (for OVA conversion)
- [ ] Browser bookmarks loaded
- [ ] OpenSSH client config aliases working (`ssh esxi` connects)
- [ ] Both PCs can reach `https://192.168.1.1/ui` (ESXi web UI)
- [ ] SMB share between PC1 ↔ PC2 working
- [ ] Greenshot, LibreOffice, Notepad++ installed

When all ticked → return to your day plan in `90_Practice_Schedule.md`.

> Next files: the day-by-day deliverables (`10_…` for Day 1 MA1, `20_…` to `30_…` for Day 2 MA2, `50_…` to `52_…` for Day 3 CTF).
