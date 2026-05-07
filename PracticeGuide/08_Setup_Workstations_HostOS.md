# 08 — Workstations: Windows Host OS Setup + Best Practices

This file is the **canonical reference** for preparing PC1 and PC2 (or your single practice PC). It consolidates Windows host prep, VMware Workstation Pro preferences, browser/SSH config, snapshot conventions, VM backups, file-share between teammates, and a recovery cookbook.

> Already partially in `02b_Setup_SinglePC_Practice.md` Sections C–D. This file is the deeper / more comprehensive version. Use this one once your setup is the full 3-PC rig.

> No marks earned by anything in this file — it's enabling work for everything in `10_…` through `52_…`.

---

## A. When to use this file

| Setup | Read which file |
|---|---|
| Single-PC practice (just one machine, learning) | `02b_Setup_SinglePC_Practice.md` (lighter version) |
| Two PCs + ESXi server (the actual rig) | **This file** |
| You want best-practice tips (bookmarks, SSH aliases, snapshot conventions) | **This file** Sections G–N |

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

Same as `02b_…` Section C.1 — entering BIOS varies per motherboard.

| Setting | Value | Why |
|---|---|---|
| Intel Virtualization Technology / AMD-V (SVM) | **Enabled** | Required for VMware to use hardware acceleration |
| Intel VT-d / AMD-Vi | Enabled (optional) | Future-proofs you for advanced features |
| Hyper-Threading | Enabled | Extra logical cores for VMs |
| Secure Boot | Enabled (default is fine for Windows) | OK for the workstations (unlike ESXi server which needs it OFF) |
| Boot order | NVMe / SSD first | So Windows boots fast |
| Fast Boot | Disabled | Causes weird issues with VMware kernel modules |
| Power on after AC loss | Optional | Convenient if power flickers |

---

## D. Windows host prep

### D.1 Disable Hyper-V / WSL2 / Memory Integrity (CRITICAL for VMware speed)
Run **PowerShell as Administrator**:

```powershell
Disable-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V -All -NoRestart
Disable-WindowsOptionalFeature -Online -FeatureName HypervisorPlatform -NoRestart
Disable-WindowsOptionalFeature -Online -FeatureName VirtualMachinePlatform -NoRestart
Disable-WindowsOptionalFeature -Online -FeatureName Containers-DisposableClientVM -NoRestart
bcdedit /set hypervisorlaunchtype off
Restart-Computer
```

Also: *Settings → Privacy & Security → Windows Security → Device Security → Core Isolation → Memory Integrity → Off*. Reboot.

After reboot, verify Task Manager → Performance → CPU shows *Virtualization: Enabled* and **does not** say "A hypervisor has been detected."

### D.2 Antivirus exclusions
Windows Defender real-time scan on `.vmdk` files cuts disk performance by ~70%.
Settings → Windows Security → Virus & threat protection → Manage settings → Exclusions → Add → **Folder**:
- `D:\VMware`
- `D:\ISO`
- `D:\juice-shop`
- `D:\VulnHub`
- `D:\Kali`

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
├── ISO\                    # OS installers
│   ├── pfSense-CE-2.7.2.iso
│   ├── CentOS-Stream-9.iso
│   ├── Win_Server_2022.iso
│   ├── Win10_Enterprise.iso
│   └── securityonion-2.4.iso
├── VMware\                 # local VMs (Kali, optional Juice Shop VM)
├── VulnHub\                # downloaded .ova files
├── juice-shop\             # extracted Juice Shop release
├── Kali\                   # Kali VMware image
├── PracticeGuide\          # copy of this guide
├── Tools\                  # offline tool installers
│   ├── Burp_Community.exe
│   ├── PuTTY.exe
│   ├── WinSCP.exe
│   ├── Wireshark.exe
│   ├── Nmap.exe
│   ├── Node20.msi
│   ├── Greenshot.exe
│   ├── LibreOffice.msi
│   └── linpeas.sh, winPEASany.exe
├── Wordlists\              # rockyou.txt, SecLists, PayloadsAllTheThings
├── References\             # offline reading
│   ├── Pwning_JuiceShop.pdf
│   ├── HackTricks_chapters.pdf
│   ├── GTFOBins\           # cloned from GitHub
│   └── OWASP_Top10.pdf
└── Notes\                  # YOUR running scratchpad — gitignored, personal
```

Both PCs use **the same paths** so any guide step works on either teammate's box.

---

## F. VMware Workstation Pro preferences

After install (`01_…` Section 1.1), tune these:

*Edit → Preferences*:

| Tab | Setting | Value |
|---|---|---|
| Workspace | Default location for VMs | `D:\VMware` |
| Input | Send Ctrl+Alt+Del to VM | choose your shortcut (default works) |
| Display | Autofit guest | enabled (so VMs scale to window) |
| Updates | Check for product updates | **disabled** (don't update mid-practice) |
| Devices | Default removable devices | Disconnected |
| Memory | Reserved memory | leave at default |

*VMware Workstation main menu → Edit → Virtual Network Editor*: see `02b_…` Section D.2 for VMnet creation if you're using Workstation as primary VM host (single-PC mode).

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
| Security Onion SOC | `https://192.168.1.50/` |

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
    HostName 192.168.56.128
    User kali
```

> Adjust IPs to match your environment. The `kali` IP comes from netdiscover on VMnet1 host-only.

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
*VMware Workstation: File → Export to OVF* OR *ESXi: Right-click VM → Export*.
Save the `.ova` to:
- A **different drive** than the VM lives on, or
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
Pick one OVA backup. *File → Open → import the OVA → boot it → confirm it works.* If you've never tested a restore, you don't have a backup.

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
| VM won't boot | Right-click → *Snapshot → Revert to last good*. If no snapshot → boot from install ISO + recovery mode |
| VM boots but no network | *VM Settings → Network Adapter*: confirm correct VMnet/port group. Inside VM: `nmcli con up <conn>` (Linux) or `netsh int ip reset` (Windows) |
| pfSense forgot rules | Restore from `Diagnostics → Backup & Restore → Restore configuration` |
| WinSRV1 AD broken | If you have an OVA backup, restore. Otherwise: `dcpromo /forceremoval` then reinstall — **slow** |
| LinSRV1 SELinux blocking httpd | `setenforce 0` temporarily, fix context with `restorecon -Rv /var/www/manila`, `setenforce 1` |
| Burp showing cert errors on Juice Shop | Re-import Burp CA cert (`05_…` Step 5.2). Restart Firefox |
| Juice Shop won't start | `cd D:\juice-shop\juice-shop_<ver> && rd /s /q node_modules && npm install && npm start` |
| Kali VM stuck at boot | F12 menu → boot recovery mode → log in single-user → fix `/etc/fstab` if disk error |
| ESXi web UI hangs | SSH in (`ssh esxi`) → `/etc/init.d/hostd restart && /etc/init.d/vpxa restart` |
| Workstation says "VMware Authorization Service not running" | Run as admin: `net start "VMware Authorization Service"`. If still failing → reinstall Workstation |
| Snapshot tree got bloated, low disk | Delete snapshots top-down (oldest first). Workstation: *VM → Snapshot → Snapshot Manager → Delete All* |

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

- [ ] Pass Task Manager virtualization-enabled check
- [ ] Have all `D:\...` folders created and AV-excluded
- [ ] VMware Workstation Pro 17 installed + licensed
- [ ] Browser bookmarks loaded
- [ ] OpenSSH client config aliases working (`ssh esxi` connects)
- [ ] Both PCs can reach `https://192.168.1.10/ui`
- [ ] SMB share between PC1 ↔ PC2 working
- [ ] Greenshot, LibreOffice, Notepad++ installed

When all ticked → return to your day plan in `90_Practice_Schedule.md`.

> The next files (`24_Day2_…` Wazuh + OpenVPN VM build, then the CTF playbooks) will be expanded next.
