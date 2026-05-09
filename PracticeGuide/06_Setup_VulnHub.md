# 06 — Setup VulnHub VMs (the boot-to-root CTF target)

The official CTF rules doc says: *"Vulnerable machines used in the competition will be sourced from VulnHub"*. The chief confirmed they will **randomly pick** from `https://www.vulnhub.com/`.

This file gets you ready to download, import, and attack any VulnHub VM. The next two files (`41_…`, `42_…`) cover the actual hacking methodology and walkthroughs.

> Total time first run: ~2 hours (download + import + first VM).
> After that: ~10 min to spin up any new VM.

---

## Part A — What VulnHub is

- A free repository of **virtual-machine images** that are intentionally vulnerable.
- Format: `.ova` (OVF appliance) or `.zip` containing `.vmx + .vmdk`.
- Imported into ESXi (in our setup) — convert from `.vmx` to OVA first if needed (see Part E).
- Every VM has 1–6 "flags" hidden along the path from initial access to root.
- Solutions exist on each VM's download page (don't read until after you try).

---

## Part B — Lab architecture (ESXi server)

Recommended setup for our 3-PC rig:

```
┌──────────────────────────────────────────────────────┐
│   ESXi Server (the 3rd PC)                            │
│                                                        │
│   ┌────────────────┐   PG-CTF (isolated port group)   │
│   │ Kali Linux     │── 10.10.10.0/24 ──────────┐      │
│   │ (attacker)     │                            │      │
│   └────────────────┘                            │      │
│                                                  │      │
│   ┌────────────────┐                            │      │
│   │ VulnHub VM     │── 10.10.10.0/24 ──────────┘      │
│   │ (target)       │                                   │
│   └────────────────┘                                   │
└──────────────────────────────────────────────────────┘
            ▲
            │ Browser from PC1 → ESXi UI → VM consoles
```

**Why an isolated port group:**
- Vulnerable VMs sit on `PG-CTF` so they can't reach pfSense / WINSRV1 / production data on PG-Servers.
- Both Kali and the target sit on the same `10.10.10.0/24` virtual cable so Kali can scan/attack.
- No bridge to PC1's real network — vulnerable services are sealed in ESXi.

### Create PG-CTF port group on ESXi (one-time)
1. ESXi UI → **Networking** → **Port groups** → **Add port group**.
2. **Name:** `PG-CTF`. **VLAN ID:** `0`. **Virtual switch:** any (a separate vSwitch is cleanest, but reusing `vSwitch1` works).
3. Click **Add**.

> 💡 Alternative: **reuse `PG-MA1-CMS`** (built in `03_Setup_VMs_MA1.md`) for CTF practice. Both are isolated and serve the same purpose.

> **Warning:** Do **NOT** attach VulnHub VMs to PG-LAN, PG-DMZ, or PG-Servers. Their weak services would interfere with your MA2 hardening work.

---

## Part C — Kali on ESXi (already done in `03_…`)

You already imported Kali into ESXi per `03_Setup_VMs_MA1.md` Part 3. **Reuse the same Kali VM for CTF.**

To switch Kali between MA1 mode and CTF mode:
1. ESXi UI → Kali VM → **Edit** → Network adapter → change from `PG-MA1-CMS` to `PG-CTF` (or keep on the same group).
2. Update Kali's IP if needed:
   ```bash
   sudo nmcli con mod "Wired connection 1" ipv4.addresses 10.10.10.2/24 ipv4.method manual
   sudo nmcli con up "Wired connection 1"
   ```
3. Snapshot as `kali-ctf-ready`.

> 💡 If you'd rather keep them separate, **clone Kali in ESXi** (Snapshot → Clone) and use the clone for CTF only.

---

## Part D — Download a VulnHub VM (procedure)

1. Browse `https://www.vulnhub.com/`.
2. Click any VM (we cover the must-haves in Part F).
3. Each page has a **Download** section with one or more mirrors. Click → save the `.ova` (or `.zip`).
4. Verify the integrity: VulnHub publishes SHA1 / MD5. From your terminal:
   ```bash
   sha1sum <file>.ova
   # Compare with the VM page's listed hash
   ```
5. Move all downloads to a single folder, e.g. `D:\VulnHub\` for easy import later.

> 📦 **Pre-flight tip for competition day:** download every popular VM (Part F list) to a USB stick **now** while you have internet. On competition day you may have no internet to grab missing files.

---

## Part E — Import a VulnHub VM into ESXi

### Step E.1 — Upload the .ova to ESXi datastore
1. ESXi UI → **Storage** → **Datastore browser** → **Upload**.
2. Select your `.ova` file. Wait for upload (5–15 min depending on file size and your network).

### Step E.2 — Deploy from OVA
1. ESXi UI → **Virtual Machines** → **Create / Register VM** → **Deploy a virtual machine from an OVF or OVA file**.
2. Name the VM (e.g., `DC-1`), browse to the uploaded `.ova` from the datastore.
3. Pick datastore → next.
4. **Network mappings:** point all networks → **PG-CTF**.
5. Disk thin → next → finish.

### Step E.3 — For `.zip` containing `.vmx` (need conversion first)
ESXi cannot directly import VMware Workstation `.vmx` files — convert them to `.ova` first.

On PC1 (with internet):
1. Download VMware OVF Tool: `https://developer.vmware.com/web/tool/4.6.0/ovf` (free).
2. Install → opens a Command Prompt where you can run `ovftool.exe`.
3. Convert:
   ```cmd
   "C:\Program Files\VMware\VMware OVF Tool\ovftool.exe" target.vmx target.ova
   ```
4. Upload the resulting `.ova` to the ESXi datastore (Step E.1) → deploy normally (Step E.2).

> 💡 **Pre-build all VulnHub OVAs locally before competition** — convert + upload them once during practice week. On competition day no internet = no conversion possible.

### Common import gotchas (ESXi)
| Symptom | Cause | Fix |
|---|---|---|
| "OVF descriptor invalid" or hardware-version error | OVA built for newer hardware than ESXi 8 supports | On PC1 use ovftool with `--lax` flag: `ovftool.exe --lax target.ovf target.ova` |
| VM boots but stuck at GRUB / kernel panic | EFI vs BIOS mismatch | Edit VM → Boot options → Firmware → toggle BIOS ↔ EFI |
| No IP shown on the VM console | The VM expects a different DHCP subnet | Manually set the VM's NIC IP to be on `10.10.10.0/24`, OR install a DHCP server on Kali |
| Slow boot | VMware Tools not present (intentional in many CTF VMs) | Ignore — doesn't affect exploitation |

---

## Part F — VMs to download and pre-stage (recommended practice list)

These are the **most-recommended training VMs**. Practising on them gives you the highest probability of having seen something similar to whatever the chief picks. Listed roughly in difficulty order.

> Each VM page on VulnHub has its own SHA-1/MD5 hash and one or more mirrors. Always verify the hash after download.

### Beginner (do these first)
| VM | Page URL | Why valuable |
|---|---|---|
| **Basic Pentesting: 1** | `https://www.vulnhub.com/entry/basic-pentesting-1,216/` | Multiple paths (FTP, web, SSH brute) — great first VM |
| **Kioptrix: Level 1 (#1)** | `https://www.vulnhub.com/entry/kioptrix-level-1-1,22/` | Classic; teaches old-Apache/Samba enumeration |
| **DC-1** | `https://www.vulnhub.com/entry/dc-1,292/` | Drupal 7 → Drupalgeddon → SUID `find` privesc |
| **Mr. Robot: 1** | `https://www.vulnhub.com/entry/mr-robot-1,151/` | Themed, popular, very well-documented |
| **OWASP Broken Web Apps** (alt to Juice Shop VM) | `https://www.vulnhub.com/entry/owasp-broken-web-applications-project-12,46/` | Stack of vulnerable web apps incl. WebGoat, DVWA, Mutillidae |

### Intermediate
| VM | Page URL | Why valuable |
|---|---|---|
| **DC-2** | `https://www.vulnhub.com/entry/dc-2,311/` | wpscan + restricted shell escape + sudo git privesc |
| **DC-3** | `https://www.vulnhub.com/entry/dc-32,312/` | Single-flag, harder web exploit |
| **DC-4** | `https://www.vulnhub.com/entry/dc-4,313/` | Brute-force + restricted shell + binary privesc |
| **Kioptrix: Level 2** | `https://www.vulnhub.com/entry/kioptrix-level-11-2,23/` | LAMP + command injection + kernel exploit |
| **Kioptrix: Level 3** | `https://www.vulnhub.com/entry/kioptrix-level-12-3,24/` | LotusCMS + sudo abuse |
| **Basic Pentesting: 2** | `https://www.vulnhub.com/entry/basic-pentesting-2,241/` | More realistic enum + OSINT |

### Advanced
| VM | Page URL | Why valuable |
|---|---|---|
| **DC-5** | `https://www.vulnhub.com/entry/dc-5,314/` | Local file inclusion + log poisoning |
| **DC-6** | `https://www.vulnhub.com/entry/dc-6,315/` | wpscan + nmap NSE privesc |
| **DC-7** | `https://www.vulnhub.com/entry/dc-7,356/` | OSINT-driven foothold |
| **DC-8** | `https://www.vulnhub.com/entry/dc-8,367/` | SQLi + Drupal + exim4 |
| **DC-9** | `https://www.vulnhub.com/entry/dc-9,412/` | knockd + LFI + scheduled exec |
| **Sunset: Decoy** | `https://www.vulnhub.com/entry/sunset-decoy,505/` | Hash cracking + suid path-hijack |
| **Sunset: Dawn** | `https://www.vulnhub.com/entry/sunset-dawn,341/` | SMB + sudo wildcards |
| **HackInOS** | `https://www.vulnhub.com/entry/hackinos-1,295/` | Container escape + WordPress |

> Realistic study target: 8–12 VMs over Week 2. Don't try to do all of them — **understand the methodology** instead.

---

## Part G — Find the VulnHub VM's IP (no DHCP info given!)

Most VulnHub VMs come with DHCP enabled but don't show their IP on the login banner. From Kali (same `PG-CTF` subnet):

### Method 1 — netdiscover (fastest)
```bash
sudo netdiscover -i eth0 -r 10.10.10.0/24
```
Look for an entry with an unfamiliar MAC. Press Ctrl+C when found.

### Method 2 — arp-scan
```bash
sudo arp-scan -l --interface=eth0
```
Lists every host that responded to ARP on the local interface.

### Method 3 — nmap ping sweep
```bash
sudo nmap -sn 10.10.10.0/24
```
Slowest but most reliable.

> 💡 If the VulnHub VM **doesn't** get a DHCP lease (no DHCP server on `PG-CTF`), you have two options:
>   1. Install a DHCP server on Kali: `sudo apt install isc-dhcp-server` then configure `/etc/dhcp/dhcpd.conf` with subnet `10.10.10.0/24`.
>   2. Find the VM's static IP from console boot logs (boot the VM, watch console messages for `eth0: ...`).

> 🚨 If multiple VMs are running, identify the new one by elimination. Note your Kali IP first (`ip a`), then any IP you don't recognise is the target.

---

## Part H — One-time post-import test (any VM)

Repeat for each VM the first time you boot it, so you confirm everything works:

```bash
# 1. find the VM
sudo netdiscover -i eth0 -r 10.10.10.0/24

# 2. quick port scan to confirm reachable
nmap -p 22,80,443 -T4 <ip>

# 3. open whatever http port responded
firefox http://<ip>
```

If you see a web page or services responding — you're ready. **Snapshot the VM in clean state** before attacking, so you can revert.

---

## Part I — Practice routine

- **Don't read the official write-up first.** Spend at least 60 minutes trying.
- After each VM, **write your own walkthrough** in a notebook — this builds muscle memory.
- After failing at a step for >30 min, look at the official write-up only for that step.
- Track in spreadsheet: VM name, finish time, what techniques were used, what you got stuck on.

> 💡 **Tip:** the techniques repeat across VMs — once you've done 5 VMs you'll have seen most of the playbook in `41_…`.

---

## Part J — Build your USB

For competition day, copy these onto your USB:
- [ ] All VMs from Part F **converted to .ova format** (~50 GB total — use a 64 GB+ USB)
- [ ] Kali Linux Installer ISO (~4 GB) — backup if you ever need to rebuild Kali
- [ ] VMware OVF Tool installer (in case you need to re-convert anything)
- [ ] HackTricks PDF
- [ ] PEASS-ng binaries (linpeas.sh, winPEAS.exe)
- [ ] GTFOBins offline mirror (`git clone https://github.com/GTFOBins/GTFOBins.github.io`)
- [ ] LinEnum.sh (`https://github.com/rebootuser/LinEnum`)
- [ ] Common wordlists: `/usr/share/wordlists/rockyou.txt`, SecLists clone

---

When done with this file → continue to **`51_Day3_VulnHub_BootToRoot.md`** for the boot-to-root methodology and **`52_Day3_CTF_Advanced.md`** for VM-by-VM walkthroughs.
