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
| **VMnet14** | Host-only | `172.16.100.0/24` | MA1: DC.grimshay, www, AMClient1/2 | `PG-MA1-LAN` |

For each one:
1. Click *Add Network* → choose VMnet number → OK.
2. Select *Host-only* (not Bridged, not NAT — we want isolation).
3. Set *Subnet IP* + *Subnet mask* per the table above.
4. **UNTICK** *"Use local DHCP service"* — we want the lab itself to control DHCP (pfSense will be DHCP for MA2 LAN, ISP for MA2 Internet, etc.). Exception: tick DHCP only for VMnet14 (MA1 LAN) so MA1 clients can boot before AD is configured.
5. UNTICK *"Connect a host virtual adapter to this network"* (we don't want Windows to have an IP on these subnets — the labs are isolated).

Apply when done.

> 💡 You can keep the default **VMnet8 (NAT)** as is — that's the network you'll use for any VM that needs internet (Kali during package installs, the practice host running Juice Shop's `npm start`).

### D.3 Network mapping (when reading the rest of the guide)

When the guide says... | Do this in Workstation
---|---
*ESXi UI → Networking → Port Groups → PG-Internet* | *VM settings → Network Adapter → Custom: VMnet10*
*ESXi UI → Port Groups → PG-LAN* | *Custom: VMnet11*
*ESXi UI → Port Groups → PG-DMZ* | *Custom: VMnet12*
*ESXi UI → Port Groups → PG-Servers* | *Custom: VMnet13*
*ESXi UI → Port Groups → PG-MA1-LAN* | *Custom: VMnet14*
*Power on a VM in ESXi web UI* | Right-click the VM tab in Workstation → Power On
*ESXi snapshot* | VM menu → *Snapshot → Take Snapshot…*

The MA1/MA2 build files (`03_…` and `04_…`) reference port groups by name — substitute VMnet numbers as above.

---

## E. Internet for the practice host

Your one PC connects to your home router (WiFi or ethernet) for internet — this is how you download all the installers, pull packages during VM setup, and grab Juice Shop / VulnHub VMs.

**No special networking needed for the host itself** — just whatever you normally use to get online. The practice VMs are isolated on their VMnets and only reach the internet if you explicitly attach them to VMnet8 (NAT).

> 🔒 Privacy: the lab VMs (especially the deliberately-vulnerable ones like www.grimshay.ca) **must not** be on Bridged networking — that exposes them to your home network. Always use Host-only VMnets, or NAT only when you specifically need internet for a VM.

---

## F. Memory budget — what to power on for each session

With ~9 GB available for VMs on a 16 GB host, run only one phase at a time. Take VM snapshots before powering off so you can resume.

### Session 1 — MA1 morning (assess Apache vulns)
Power on:
- DC.grimshay.local (4 GB)
- www.grimshay.ca (2 GB)
- AMClient1 (2 GB)

Total: **8 GB** ✅
Leave AMClient2 off — only needed for parallel-team checks.

### Session 2 — MA2 firewall + LinSRV1 (Person A's morning role)
Power on:
- pfSense (2 GB)
- LinSRV1 (2 GB)
- Client1 (2 GB)
- ISP (1 GB) — only when testing Internet rules

Total: **7 GB** ✅
Skip WINSRV1/3/4 in this session if memory is tight; pfSense and LinSRV1 work without AD initially.

### Session 3 — MA2 AD + PKI (Person B's afternoon role)
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

### ❌ Session you CANNOT do on 16 GB single PC
**Day 2 Security Onion** needs 16 GB itself just for the SO VM, plus host overhead = won't fit. Options:
1. Skip SO during single-PC practice; come back to it once you have a 32 GB+ machine.
2. Reduce SO to 12 GB (`Edit VM settings → Memory`) — sluggish but functional.
3. Run SO in *Import-only* mode (4 GB minimum, no live monitoring) — useful for forensic PCAP analysis but not for live IR.

---

## G. What you can fully practise on a single PC

| Practice activity | On 16 GB single PC? |
|---|---|
| MA1 morning (Day 1) | ✅ Fully |
| MA2 firewall + LinSRV1 | ✅ Fully (skip WINSRV4) |
| MA2 AD + GPOs | ✅ Fully |
| MA2 PKI | ✅ Fully (boot WINSRV4 briefly) |
| MA2 verification from clients | ✅ Mostly — pick 1 client at a time |
| CTF Juice Shop (★1 → ★6) | ✅ Fully |
| CTF VulnHub (any VM) | ✅ Fully (one VM at a time) |
| Day 2 Security Onion (live monitoring) | ❌ No (needs 16 GB just for SO) |
| Day 2 Security Onion (PCAP import only) | ⚠️ Yes if you reduce SO to 4 GB |
| Day 2 OpenVPN service install | ✅ Fully |

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
- [ ] VMnet10–14 created in Virtual Network Editor with the correct subnets
- [ ] All 4 OS ISOs downloaded to `D:\ISO\`
- [ ] You understand which VMnet each VM should attach to (table in section D.3)

When all ticked → start `03_Setup_VMs_MA1.md`. Whenever it says "PG-MA1-LAN", attach the VM's network adapter to **VMnet14** instead.

---

## J. Honest expectations for solo-PC practice

You can comfortably learn:
- The **MA1 + MA2 deliverables top-to-bottom** (estimate: 80% of what the actual competition tests).
- Every **Juice Shop challenge** ★1 → ★6.
- **Boot-to-root methodology** on any VulnHub VM.

You will NOT be able to fully simulate:
- **Day 2 live Security Onion monitoring** (need 32 GB+).
- **Two teammates working in parallel on different VMs** (need a second PC).
- **The exact 3-box topology** with eSXi web UI workflow.

That's fine — once you have the proper hardware, your skills transfer 100%. The lab topology is just a vehicle for the security work; the security work itself is identical.

Now go to **`03_Setup_VMs_MA1.md`** and start building MA1. Remember: when it says `PG-MA1-LAN`, you select **VMnet14**.
