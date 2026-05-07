# 02b — Single-PC Practice Setup (no ESXi server, no second PC)

This file replaces `02_Setup_Topology.md` for the situation where you only have **one PC** and want to start practising before getting the full 3-box rig (2 PCs + ESXi server). When you later get that hardware, switch to `02_Setup_Topology.md` and migrate VMs by exporting/importing them.

> Goal: get you practising MA1 + MA2 + CTF on a single machine, this week.

---

## A. The simplification

The official competition uses:
```
[ PC1 ]──┐
         ├──[ Switch ]──[ ESXi server with all VMs ]
[ PC2 ]──┘
```

For solo practice on one PC, we **collapse all three boxes into the host**:
```
┌──────────────────────────────────────────────┐
│  Your single PC — Windows host                │
│                                                │
│  ┌────────────────────────────────────────┐   │
│  │  VMware Workstation Pro 17             │   │
│  │                                          │   │
│  │  Runs ALL practice VMs directly         │   │
│  │  (no nested ESXi)                        │   │
│  │                                          │   │
│  │  Network isolation via VMnets            │   │
│  │  (Workstation's equivalent of ESXi       │   │
│  │   port groups)                           │   │
│  └────────────────────────────────────────┘   │
└──────────────────────────────────────────────┘
```

**Why no nested ESXi?**
- Performance: nested virt is 30–50% slower than running VMs directly.
- Memory: ESXi itself eats 6–8 GB before any VM runs, leaving little for the practice VMs.
- Complexity: another layer to debug.
- Equivalence: Workstation has the same network primitives (VMnets ≈ port groups), so the lab logically matches the ESXi setup.

**Trade-off:** you won't see the ESXi web UI screens shown in the rest of the guide. When the guide says "*ESXi UI → Networking → Virtual switches*", you do the equivalent in Workstation's *Edit → Virtual Network Editor*. Mapping table in section D below.

---

## B. Host hardware reality check

If your PC is **16 GB RAM / 500 GB SSD** (typical for what you described):

| Item | RAM cost |
|---|---|
| Windows 11 host idle | ~4 GB |
| VMware Workstation Pro running | ~1 GB |
| Browser + tools (Chrome, Burp etc.) | ~2 GB |
| **Available for VMs at any moment** | **~9 GB** |

That's enough to run **one phase at a time**, never all of them. Concrete plan in section F.

If you have **32 GB+** → comfortable, can run all of MA2 simultaneously.
If you have **8 GB or less** → you'll struggle. Practice individual VMs only (e.g. just Juice Shop, just Kali). Wait for proper hardware before attempting MA2.

Disk: 500 GB on Windows = 350 GB available after OS/programs. With **thin-provisioning** (VMware default), the practice VMs use ~150 GB real storage even though nominal totals are ~600 GB. Workable.

---

## C. Windows host preparation (do once)

These steps make the difference between "VMs run smoothly" and "VMs lag and crash."

### C.1 BIOS — enable virtualization extensions
1. Restart PC → press the BIOS hotkey (usually **F2**, **F10**, **Del**, or **Esc** — depends on motherboard).
2. Find one of these settings (different BIOS vendors use different names):
   - **Intel:** *Advanced → CPU Configuration → Intel Virtualization Technology* → **Enabled**.
   - **AMD:** *Advanced → CPU Configuration → SVM Mode* → **Enabled**.
3. Save and exit.

> Verify after Windows boot: open Task Manager → Performance → CPU → look for *Virtualization: Enabled*. If it says Disabled, re-check BIOS.

### C.2 Disable Hyper-V and WSL2 (CRITICAL for VMware)
If Hyper-V is enabled on Windows 11 (which is the default on many systems), VMware Workstation runs in a slow "WHP" mode that's 3–10× slower.

Run **PowerShell as Administrator** and execute:
```powershell
# Turn off Hyper-V Windows feature
Disable-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V -All -NoRestart

# Turn off Windows Hypervisor Platform
Disable-WindowsOptionalFeature -Online -FeatureName HypervisorPlatform -NoRestart

# Turn off Virtual Machine Platform (used by WSL2)
Disable-WindowsOptionalFeature -Online -FeatureName VirtualMachinePlatform -NoRestart

# Disable hypervisor at boot
bcdedit /set hypervisorlaunchtype off

# Disable Memory Integrity (Core Isolation) — interferes with VMware
# Settings → Privacy & Security → Windows Security → Device Security → Core Isolation → Off

# Reboot
Restart-Computer
```

After reboot, verify Task Manager → Performance → CPU shows *Virtualization: Enabled* but no longer says "A hypervisor has been detected".

> ⚠️ If you actively use Docker Desktop with WSL2 backend, the above breaks Docker. We're not using Docker (we removed it from Juice Shop install — see `05_…`), so this is fine.

### C.3 Antivirus exclusions
Windows Defender real-time scan on `.vmdk` files cuts VM disk performance by ~70%.

Settings → Privacy & Security → Windows Security → Virus & threat protection → *Manage settings* → *Add or remove exclusions* → **Add exclusion → Folder** and add:
- `D:\VMware` (your VM storage folder — see C.5)
- `D:\ISO`
- `D:\juice-shop`
- `D:\VulnHub`

Add the same paths under *Process exclusions* if available.

### C.4 Power plan — never sleep during practice
Settings → System → Power & battery → Screen and sleep:
- Screen turns off: **Never** (or 30 min)
- PC goes to sleep: **Never**

Also: Settings → System → Power → Power mode → **Best performance**.

Disable Fast Startup (interferes with VMware kernel modules):
Control Panel → Power Options → *Choose what the power buttons do* → *Change settings that are currently unavailable* → uncheck *Turn on fast startup* → Save.

### C.5 Standardise the folder layout

Create on a dedicated partition or your D: drive:
```
D:\
├── ISO\                        (OS installers — pfSense, CentOS, Win Server, Win 10)
├── VMware\                     (where Workstation stores VM files)
│   ├── MA1-DC\
│   ├── MA1-Web\
│   ├── MA2-pfSense\
│   ├── ... (one folder per VM)
├── VulnHub\                    (downloaded .ova files)
├── juice-shop\                 (extracted Juice Shop ZIP)
├── Kali\                       (Kali VMware image)
└── PracticeGuide\              (copy of this guide)
```

Set VMware Workstation default location: *Edit → Preferences → Workspace → Default location for virtual machines* = `D:\VMware\`.

---

## D. VMware Workstation Pro install + network setup

### D.1 Install
Download from `https://www.vmware.com/products/workstation-pro/workstation-pro-evaluation.html` (or the Broadcom portal).
Install with all defaults. After install, *Help → Enter a License Key* (use the personal-use key from the same Broadcom page, or run for 30 days in eval mode).

### D.2 Create the network segments (VMnets ≈ ESXi port groups)

VMware Workstation has **VMnet0–VMnet19**. We will create 5 isolated VMnets that mirror the ESXi port groups described in `02_Setup_Topology.md`.

**Why:** the MA1 environment uses one isolated subnet, and MA2 uses 4 subnets that pfSense routes between. We need the same isolation here.

**Open** *Edit → Virtual Network Editor → Change Settings* (admin prompt).

Create these custom VMnets (click *Add Network*):

| VMnet | Type | Subnet (we'll set) | Used by | Equivalent in ESXi guide |
|---|---|---|---|---|
| **VMnet10** | Host-only | `10.0.0.0/24` | MA2 ISP + Client3 + pfSense WAN | `PG-Internet` |
| **VMnet11** | Host-only | `172.16.100.0/24` | MA2 LAN: Client1, Client2, pfSense LAN | `PG-LAN` |
| **VMnet12** | Host-only | `192.168.1.0/24` | MA2 DMZ: LinSRV1, pfSense DMZ | `PG-DMZ` |
| **VMnet13** | Host-only | `192.168.2.0/24` | MA2 Servers: WinSRV1/3/4, pfSense Servers | `PG-Servers` |
| **VMnet15** | Host-only | `192.168.2.0/24` | **MA1: CMS target + Kali** (per MA1 PDF Table 1) | `PG-MA1-CMS` |

> ⚠️ **Two VMnets share `192.168.2.0/24` (VMnet13 + VMnet15) — this is fine.** VMware host-only VMnets are physically isolated from each other. VMs on VMnet13 cannot see VMs on VMnet15, even though they share a subnet number. **Just don't power on MA1 + MA2 simultaneously** (a single VM can't be on two VMnets at once anyway).
>
> 💡 **Why VMnet15 instead of VMnet14?** VMware reserves VMnet0/1/8 for default networks and VMnet2/14 are sometimes pre-mapped on Workstation Pro. Using VMnet15 avoids any pre-existing mapping. If your install has a free VMnet14, that works too — just keep it consistent in the network mapping table below.

For each one:
1. Click *Add Network* → choose VMnet number → OK.
2. Select *Host-only* (not Bridged, not NAT — we want isolation).
3. Set *Subnet IP* + *Subnet mask* per the table above.
4. **UNTICK** *"Use local DHCP service"* — we want the lab itself to control DHCP (pfSense will be DHCP for MA2 LAN, ISP for MA2 Internet, etc.). For **VMnet15 (MA1)** you can leave DHCP unticked too — MA1 PDF expects static IPs on the CMS target and Kali (192.168.2.1 and 192.168.2.2).
5. **TICK** *"Connect a host virtual adapter to this network"* — yes, leave it ticked. ⚠️ **This is required** so the VMnet appears in the VM Settings → Network Adapter → Custom dropdown. (If unticked, VMware Workstation hides the VMnet from the dropdown and you can't assign VMs to it.) The downside is your Windows host gets an IP on each lab subnet — fine for solo practice; it actually helps with troubleshooting (you can `ping` lab VMs directly from Windows).

Apply when done.

> 🔧 **If you already created the VMnets with the host adapter UNticked** and now the dropdown is empty when you go to VM Settings → Network Adapter:
> 1. Go back to *Edit → Virtual Network Editor → Change Settings* (admin).
> 2. For each VMnet (10, 11, 12, 13, 15), click it → **tick** *"Connect a host virtual adapter to this network"* → Apply.
> 3. Now reopen VM Settings → the VMnets appear in the dropdown.

> 💡 You can keep the default **VMnet8 (NAT)** as is — that's the network you'll use for any VM that needs internet (Kali during package installs / exploit-db updates, or the host running Juice Shop's `npm start`).

### D.3 Network mapping (when reading the rest of the guide)

When the guide says... | Do this in Workstation
---|---
*ESXi UI → Networking → Port Groups → PG-Internet* | *VM settings → Network Adapter → Custom: VMnet10*
*ESXi UI → Port Groups → PG-LAN* | *Custom: VMnet11*
*ESXi UI → Port Groups → PG-DMZ* | *Custom: VMnet12*
*ESXi UI → Port Groups → PG-Servers* | *Custom: VMnet13*
*ESXi UI → Port Groups → PG-MA1-CMS* (or `PG-MA1-LAN` in older docs) | *Custom: VMnet15* |
*Power on a VM in ESXi web UI* | Right-click the VM tab in Workstation → Power On
*ESXi snapshot* | VM menu → *Snapshot → Take Snapshot…*

The MA1/MA2 build files (`03_…` and `04_…`) reference port groups by name — substitute VMnet numbers as above.

### D.4 MA1 IPs match the PDF — no overrides needed

Per **MA1 PDF Table 1**: CMS target = `192.168.2.1`, Kali = `192.168.2.2`. Use these IPs as-is when building the practice MA1 VMs from `03_Setup_VMs_MA1.md`.

| VM | IP (per MA1 PDF) | VMnet |
|---|---|---|
| Linux Server with CMS (e.g. Drupal 7 target) | **192.168.2.1** | VMnet15 |
| Kali Linux | **192.168.2.2** | VMnet15 |

> Both VMs sit on **VMnet15** which we set to `192.168.2.0/24`. They reach each other directly. **VMnet13 (MA2 Servers) also has `192.168.2.0/24`** — but VMnet13 and VMnet15 are isolated, so MA1 VMs can't see MA2 VMs and vice versa. No collision.
>
> Practice rule: only power on MA1 VMs OR MA2 VMs at a time. Single-PC RAM (16 GB) won't fit both anyway.

---

## E. Internet for the practice host

Your one PC connects to your home router (WiFi or ethernet) for internet — this is how you download all the installers, pull packages during VM setup, and grab Juice Shop / VulnHub VMs.

**No special networking needed for the host itself** — just whatever you normally use to get online. The practice VMs are isolated on their VMnets and only reach the internet if you explicitly attach them to VMnet8 (NAT).

> 🔒 Privacy: the deliberately-vulnerable lab VMs (especially the CMS pentest target) **must not** be on Bridged networking — that exposes them to your home network. Always use Host-only VMnets, or NAT only when you specifically need internet for a VM.

---

## F. Memory budget — what to power on for each session

With ~9 GB available for VMs on a 16 GB host, run only one phase at a time. Take VM snapshots before powering off so you can resume.

### Session 1 — Day 1: MA1 CMS pentest (per MA1 PDF)
Power on:
- CMS target VM (Drupal 7 / vulnerable Linux box) — 2 GB
- Kali Linux — 4 GB

Total: **6 GB** ✅ very comfortable on 16 GB host.

> The MA1 PDF (Table 1) only requires 2 VMs (CMS target + Kali). Single-PC practice for MA1 is comfortable on 16 GB.

### Session 2 — Day 2 part A: MA2 firewall + LinSRV1 (Person A's role)
Power on:
- pfSense (2 GB)
- LinSRV1 (2 GB)
- Client1 (2 GB)
- ISP (1 GB) — only when testing Internet rules

Total: **7 GB** ✅
Skip WINSRV1/3/4 in this session if memory is tight; pfSense and LinSRV1 work without AD initially.

### Session 3 — Day 2 part B: MA2 AD + PKI (Person B's role)
Power on:
- pfSense (2 GB)
- WINSRV1 (4 GB)
- WINSRV3 (3 GB)

Total: **9 GB** ⚠️ at the limit — close all browser tabs first.
WINSRV4 stays off; only power on briefly when you need to sign the WINSRV3 CSR (~5 min).

### Session 4 — MA2 verification (clients reach servers)
Power on:
- pfSense + WINSRV1 + LinSRV1 + Client1 (no WINSRV3 — just pre-applied policies)

Total: **10 GB** ⚠️ borderline. Reduce LinSRV1 RAM to 1 GB if needed.

### Session 5 — CTF Juice Shop only
Run Juice Shop on Windows host (`npm start`) — no VM at all. **Cost: 1 GB host RAM.**
Open Burp + Firefox. Practise ★1–★4 challenges.

Total: **~3 GB host overhead**, no VMs needed for this. ✅ very comfortable.

### Session 6 — CTF VulnHub VM
Power on:
- Kali (4 GB)
- One VulnHub VM (typically 1–2 GB)

Total: **6 GB** ✅

### All sessions fit on 16 GB single PC ✅

The confirmed 3-day structure (MA1 + MA2 + CTF) doesn't require Security Onion or any heavyweight SOC platform. Earlier speculation about a separate Day 2 SOC module turned out to be unnecessary — MA2 PDF IS the entire Day 2 hardening module. Single-PC practice is now end-to-end viable on 16 GB.

---

## G. What you can fully practise on a single PC

| Practice activity | On 16 GB single PC? |
|---|---|
| MA1 (Day 1) — CMS pentest | ✅ Fully |
| MA2 firewall + LinSRV1 | ✅ Fully (skip WINSRV4) |
| MA2 AD + GPOs | ✅ Fully |
| MA2 PKI | ✅ Fully (boot WINSRV4 briefly) |
| MA2 verification from clients | ✅ Mostly — pick 1 client at a time |
| CTF Juice Shop (★1 → ★6) | ✅ Fully |
| CTF VulnHub (any VM) | ✅ Fully (one VM at a time) |
| Day 2 (MA2 — Security Hardening): pfSense + AD GPOs + LinSRV1 hardening + PKI | ✅ Fully |
| Day 2 OpenVPN setup (the one in MA2 PDF) | ✅ Fully |

---

## H. Migration — when you get the 3-PC setup later

When you finally have the full hardware (PC1 + PC2 + ESXi server), you don't lose your work:

### H.1 Export each Workstation VM as OVA
For each VM you've built and snapshotted:
- VMware Workstation → File → Export to OVF → save as `<vmname>.ova` to your USB.

### H.2 Import into ESXi
- ESXi web UI → Virtual Machines → Create / Register VM → Deploy a virtual machine from an OVF or OVA file → upload your `.ova` → assign to the matching port group (e.g. `PG-LAN` instead of `VMnet11`).
- Power on. The VM keeps all its config, snapshots may need re-taking.

### H.3 Switch to the proper guide
- Bookmark `02_Setup_Topology.md` — that becomes the wiring + ESXi config reference.
- This file (`02b`) is no longer needed; archive or delete.

---

## I. Verification before moving to `03_Setup_VMs_MA1.md`

- [ ] BIOS: virtualization enabled (Task Manager confirms)
- [ ] Hyper-V/WSL2 disabled, Memory Integrity off, fast startup off
- [ ] Antivirus exclusions added for `D:\VMware`, `D:\ISO`, `D:\VulnHub`, `D:\juice-shop`
- [ ] Power plan: Best performance, never sleep
- [ ] VMware Workstation Pro 17 installed and licensed
- [ ] VMnet10, 11, 12, 13, 15 created in Virtual Network Editor with the correct subnets
- [ ] All 4 OS ISOs downloaded to `D:\ISO\` (pfSense, CentOS, Win Server 2022, Win 10)
- [ ] Kali Linux VMware image extracted to `D:\Kali\` (used by both MA1 + CTF)
- [ ] You understand which VMnet each VM should attach to (table in section D.3)

When all ticked → start `03_Setup_VMs_MA1.md`.

**Quick reference for the new MA1 (per PDF):**
- `03_…` builds 2 VMs only: CMS target on VMnet15 with IP `192.168.2.1`, Kali on VMnet15 with IP `192.168.2.2`.
- Workstation login during MA1: `competitor1a / Boracay@14!`.
- Workstation login during MA2: `competitor1b / Tagaytay_62&L`.

For MA2 references in `04_…` and `20_…`+:
- `PG-Internet` → VMnet10
- `PG-LAN` → VMnet11
- `PG-DMZ` → VMnet12
- `PG-Servers` → VMnet13
- ESXi credentials (the `wsauser/Andres@9V4` mention in the PDF) — ignore on single-PC mode (no ESXi).

---

## J. Honest expectations for solo-PC practice

You can comfortably learn:
- **MA1 deliverables top-to-bottom** — the new 4-task PDF is a 2-VM pentest, runs perfectly on a single 16 GB PC.
- **MA2 deliverables top-to-bottom** — all 11 GPOs, firewall, PKI, AD, LinSRV1, verification (estimate: 95% of what competition tests).
- Every **Juice Shop challenge** ★1 → ★6.
- **Boot-to-root methodology** on any VulnHub VM (which is what MA1 also tests).

You will NOT be able to fully simulate:
- **Two teammates working in parallel on different VMs at the same time** (need a second PC).
- **The exact 3-box topology** with the ESXi web UI workflow.

That's fine — once you have the proper hardware, your skills transfer 100%. The lab topology is just a vehicle for the security work; the security work itself is identical.

Now go to **`03_Setup_VMs_MA1.md`** and start building MA1.

When `03_…` references `VMnet15` (or `PG-MA1-CMS` in older versions), use **VMnet15** in your Network Adapter dropdown. The CMS target gets `192.168.2.1` and Kali gets `192.168.2.2`, exactly as the MA1 PDF specifies.
