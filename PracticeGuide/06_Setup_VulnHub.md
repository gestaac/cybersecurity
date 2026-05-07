# 06 — Setup VulnHub VMs (the boot-to-root CTF target)

The official CTF rules doc says: *"Vulnerable machines used in the competition will be sourced from VulnHub"*. The chief confirmed they will **randomly pick** from `https://www.vulnhub.com/`.

This file gets you ready to download, import, and attack any VulnHub VM. The next two files (`41_…`, `42_…`) cover the actual hacking methodology and walkthroughs.

> Total time first run: ~2 hours (download + import + first VM).
> After that: ~10 min to spin up any new VM.

---

## Part A — What VulnHub is

- A free repository of **virtual-machine images** that are intentionally vulnerable.
- Format: `.ova` (OVF appliance) or `.zip` containing `.vmx + .vmdk`.
- Imported into VMware Workstation or VirtualBox.
- Every VM has 1–6 "flags" hidden along the path from initial access to root.
- Solutions exist on each VM's download page (don't read until after you try).

---

## Part B — Lab architecture

Recommended setup for practice:

```
┌──────────────────────────────────────────────┐
│   VMware Workstation Pro 17 on your laptop    │
│                                                │
│   ┌────────────────┐   VMnet1 Host-Only       │
│   │ Kali Linux     │── 192.168.56.0/24 ──┐    │
│   │ (attacker)     │                      │    │
│   └────────────────┘                      │    │
│                                            │    │
│   ┌────────────────┐                      │    │
│   │ VulnHub VM     │── 192.168.56.0/24 ──┘    │
│   │ (target)       │                           │
│   └────────────────┘                           │
└──────────────────────────────────────────────┘
```

**Why Host-Only:**
- Isolates the deliberately-vulnerable VM from your real network and the internet (some VMs phone home, none should reach anything).
- Both Kali and target sit on the same private subnet so Kali can scan/attack.

### Set up the VMware Host-Only network
1. VMware Workstation → *Edit → Virtual Network Editor*.
2. Click **Change Settings** (admin prompt).
3. Select **VMnet1** (Host-Only) → confirm enabled and check the subnet (default `192.168.x.0/24`; commonly `192.168.56.0/24` if you've used it for Vagrant).
4. Make sure **Use local DHCP service to distribute IP addresses** is ✅ ticked. This lets the target VM grab an IP automatically.
5. Apply.

> **Warning:** Do **NOT** put VulnHub VMs on Bridged or NAT — some have weak default services that will be discovered by anything on your real LAN.

---

## Part C — Install Kali Linux as the attacker VM

1. Download the **Kali Linux Pre-built VMware** image from `https://www.kali.org/get-kali/#kali-virtual-machines`. Pick the 7z or zip → extract.
2. VMware Workstation → *File → Open* → select the extracted `.vmx`.
3. After import: VM settings → Network Adapter → **Custom: VMnet1 (Host-Only)**.
4. Power on. Default credentials: `kali` / `kali`.
5. First-boot updates:
   ```bash
   sudo apt update && sudo apt -y full-upgrade
   ```
6. Verify Kali got a Host-Only IP:
   ```bash
   ip a | grep 192.168
   ```
   Should show something like `192.168.56.128/24`.
7. **Snapshot Kali clean** in VMware (so you can roll back after experiments).

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

## Part E — Import a VulnHub VM into VMware Workstation

### For `.ova` files
1. VMware Workstation → *File → Open* → select the `.ova`.
2. Name + storage path → *Import*.
3. If "OVF specification compliance" warning → click **Retry**.
4. After import: **Edit virtual machine settings → Network Adapter → Custom: VMnet1 (Host-Only)**.
5. Power on the VM.

### For `.zip` containing `.vmx`
1. Extract the `.zip` to a folder.
2. VMware Workstation → *File → Open* → select the `.vmx`.
3. When VMware asks "I copied it" vs "I moved it" → choose **I copied it** (regenerates UUID/MAC).
4. Settings → Network Adapter → **VMnet1 (Host-Only)**. Save.
5. Power on.

### Common import gotchas
| Symptom | Cause | Fix |
|---|---|---|
| "Failed to import" | OVA built for newer hardware version | Try VirtualBox import, then re-export as OVF 1.0 |
| VM boots but stuck at GRUB / kernel panic | EFI vs BIOS mismatch | VM settings → *Options → Advanced → Firmware type* — toggle BIOS ↔ UEFI |
| No IP shown on the VM console | DHCP not running on VMnet1, or VM hard-coded a different subnet | Enable DHCP on VMnet1 (Part B step 4); or check VM author notes |
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

Most VulnHub VMs come with DHCP enabled but don't show their IP on the login banner. From Kali (same Host-Only subnet):

### Method 1 — netdiscover (fastest)
```bash
sudo netdiscover -i eth0 -r 192.168.56.0/24
```
Look for an entry with a different vendor than VMware's host (e.g., your VM might appear as "PCS Systemtechnik GmbH" if VBox or "VMware Inc." with an unfamiliar MAC). Press Ctrl+C when found.

### Method 2 — arp-scan
```bash
sudo arp-scan -l --interface=eth0
```
Lists every host that responded to ARP on the local interface.

### Method 3 — nmap ping sweep
```bash
sudo nmap -sn 192.168.56.0/24
```
Slowest but most reliable.

> 🚨 If multiple VMs are running, identify the new one by elimination. Note your Kali IP first (`ip a`), then any IP you don't recognise is the target.

---

## Part H — One-time post-import test (any VM)

Repeat for each VM the first time you boot it, so you confirm everything works:

```bash
# 1. find the VM
sudo netdiscover -i eth0 -r 192.168.56.0/24

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
- [ ] All VMs from Part F (~50 GB total — use a 64 GB+ USB)
- [ ] Kali Linux VM image (10 GB)
- [ ] VMware Workstation installer
- [ ] HackTricks PDF
- [ ] PEASS-ng binaries (linpeas.sh, winPEAS.exe)
- [ ] GTFOBins offline mirror (`git clone https://github.com/GTFOBins/GTFOBins.github.io`)
- [ ] LinEnum.sh (`https://github.com/rebootuser/LinEnum`)
- [ ] Common wordlists: `/usr/share/wordlists/rockyou.txt`, SecLists clone

---

When done with this file → continue to **`51_Day3_VulnHub_BootToRoot.md`** for the boot-to-root methodology and **`52_Day3_CTF_Advanced.md`** for VM-by-VM walkthroughs.
