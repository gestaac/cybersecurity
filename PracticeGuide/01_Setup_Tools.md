# 01 — Tools & Downloads (Setup from Zero)

This file lists everything you need to download and install **before** you build any VMs. Do this on the laptop that will host VMware Workstation, and **also** prepare a USB stick with the ISOs to take to the practice site.

> Allocate **one full day** for this file. Most of it is downloads + clicking *Next*.

---

## 0. Host hardware budget (read this first)

You will run all the VMs on **one ESXi server**. Check it has enough resources before starting.

### Day-1 + CTF practice load (run together)

| VM | RAM | Disk |
|---|---|---|
| MA1 — DC.grimshay.local | 4 GB | 60 GB |
| MA1 — www.grimshay.ca (CentOS) | 2 GB | 20 GB |
| MA1 — AMClient1 (Win10) | 2 GB | 40 GB |
| MA1 — AMClient2 (Win10) | 2 GB | 40 GB |
| MA2 — ISP (CentOS) | 1 GB | 10 GB |
| MA2 — pfSense | 2 GB | 20 GB |
| MA2 — WINSRV1 (DC) | 4 GB | 80 GB |
| MA2 — WINSRV3 (CA) | 3 GB | 60 GB |
| MA2 — WINSRV4 (Root CA, off most of time) | 2 GB | 60 GB |
| MA2 — LINSRV1 (CentOS) | 2 GB | 20 GB |
| MA2 — Client1 (Win10) | 2 GB | 40 GB |
| MA2 — Client2 (Win10) | 2 GB | 40 GB |
| MA2 — Client3 (Win10) | 2 GB | 40 GB |
| Kali Linux (attacker) | 4 GB | 80 GB |
| **Sub-total (Day 1 + CTF)** | **~32 GB** *(WINSRV4 off → ~30 GB)* | **~610 GB** |

### Day-2 additional load

| VM | RAM | Disk |
|---|---|---|
| Security Onion (Eval) | 16 GB | 300 GB |
| OpenVPN service VM (CentOS) | 2 GB | 20 GB |
| **Sub-total (Day 2)** | **18 GB** | **320 GB** |

### Recommended host specs

| Practice mode | RAM | Disk | CPU |
|---|---|---|---|
| **Day-1 + CTF only** (most common; can shut down some VMs to save RAM) | **32 GB minimum, 64 GB ideal** | **800 GB SSD** | **8 cores** |
| **Day-2 too (Security Onion alongside)** | **64 GB minimum** | **1.2 TB SSD** | **8+ cores** |

> 🔑 **Memory tactics if you're tight:**
> - Power off WINSRV4 except when signing certs.
> - Power off MA1 VMs once you've moved to MA2 practice (different vSwitches anyway).
> - For Day 2 practice, shut down all MA2 VMs except the ones Security Onion is monitoring.
> - Thin-provision all VMware disks (default in ESXi) — totals above are *max*, real usage is much lower.

---

## 1. Hypervisors

### 1.1 VMware Workstation Pro 17 (laptop)
- **Download:** `https://www.vmware.com/products/workstation-pro/workstation-pro-evaluation.html`
- Newer Broadcom path: log in to `https://support.broadcom.com` → "VMware Cloud Foundation" → "My Downloads" → search "Workstation Pro 17" (free for personal use as of 2024).
- **Install:** Default options. After install, open Workstation → *Help → Enter a License Key* (use personal-use key from the same page).
- **Why we need it:** runs the local VMs that mirror what the ESXi host will run on competition day.

### 1.2 VMware ESXi 8 (server)
- **Download:** Broadcom support portal → "VMware vSphere Hypervisor 8" — `https://support.broadcom.com/group/ecx/productdownloads?subfamily=VMware%20vSphere`. Requires a free Broadcom account.
- **Install:** Boot the server from the ISO → *Install* → choose disk → set root password → reboot.
- **Post-install:**
  1. From a PC on the same wired network, browse to `https://<esxi-ip>/ui` → log in as `root`.
  2. *Manage → Licensing → Assign license* → paste the free licence key.
  3. *Networking → Virtual switches* — we'll create vSwitches in `02_Setup_Topology.md`.

> ⚠️ **2024 licensing change.** Broadcom acquired VMware in late 2023 and changed the free-ESXi tier:
> - The classic *VMware vSphere Hypervisor* free licence is now **personal-use, 60-day eval extended** — apply on the Broadcom portal after install.
> - If you can't get a key in time, ESXi runs in **60-day evaluation mode** with full features — perfectly fine for the 2 weeks of practice + competition week.
> - **Fallback if Broadcom registration is a problem:** install **Proxmox VE 8** instead — `https://www.proxmox.com/en/downloads`. Free, no licence, runs the same VMs (import OVAs the same way). Some screens differ from this guide but functionally equivalent.

---

## 2. Operating-system ISOs (download all to `D:\ISO\`)

| File | Source | Approx size |
|---|---|---|
| `pfSense-CE-2.7.2-RELEASE-amd64.iso.gz` | `https://www.pfsense.org/download/` (Community Edition, AMD64, ISO Installer) | 700 MB |
| `CentOS-Stream-9-latest-x86_64-dvd1.iso` | `https://www.centos.org/download/` | 9 GB |
| `Windows_Server_2022_eval.iso` | `https://www.microsoft.com/en-us/evalcenter/download-windows-server-2022` (180-day eval) | 5 GB |
| `Windows10_Enterprise_eval.iso` | `https://www.microsoft.com/en-us/evalcenter/download-windows-10-enterprise` (90-day eval) | 6 GB |

> Decompress the pfSense `.gz` with **7-Zip** (free: `https://www.7-zip.org/`).

---

## 3. Workstation tools (install on your competitor PCs)

### 3.1 Web + remote-access
| Tool | Why | Source |
|---|---|---|
| **Google Chrome** | Required for "Chrome enterprise" GPO test | `https://www.google.com/chrome/` |
| **Mozilla Firefox** | Backup browser for cert inspection | `https://www.mozilla.org/firefox/` |
| **PuTTY 0.79+** | SSH to LinSRV1 over port 2022 | `https://www.putty.org/` |
| **WinSCP** | File copy to LinSRV1 | `https://winscp.net/` |
| **OpenVPN Connect** | VPN client on Client3 | `https://openvpn.net/client/` |
| **VMware Workstation Player** (if not Pro) | View VMs locally | same page as Pro |

### 3.2 Network + analysis
| Tool | Why | Source |
|---|---|---|
| **Wireshark 4.x** | Packet capture for troubleshooting LDAP/HTTPS | `https://www.wireshark.org/` |
| **Nmap + Zenmap** | XMAS scan from Client3 to test Snort | `https://nmap.org/download.html` |

### 3.3 Microsoft management bundles
| Tool | Why | Source |
|---|---|---|
| **RSAT (Remote Server Administration Tools)** | If you ever manage AD from a Win10 client | `Settings → Apps → Optional Features → Add → RSAT` |
| **Google Chrome Enterprise Bundle 64** | Provides ADMX/ADML files for the "google" GPO | `https://chromeenterprise.google/intl/en_us/browser/download/` (choose **Chrome Enterprise Bundle**, .zip with admx) |

> The googleChromeEnterpriseBundle64.zip is also pre-staged in the official competition's domain administrator's `Documents` folder per MA2 line 204 — but it's safer to have your own copy.

---

## 4. Inside-Linux tools (we install via `dnf` on LinSRV1; pre-cache only if no internet)

For LinSRV1 (CentOS Stream 9). When LinSRV1 has internet (during practice setup):

```bash
sudo dnf install -y httpd mod_ssl realmd sssd oddjob oddjob-mkhomedir adcli \
                    samba-common-tools krb5-workstation chrony \
                    policycoreutils-python-utils setools-console \
                    libpwquality cracklib pam_pwquality \
                    firewalld sudo openssl
sudo systemctl enable --now chronyd firewalld httpd
```

Take a snapshot **before** doing the security work so you can re-run drills.

---

## 5. CTF tooling — install on a Kali Linux VM (or Parrot)

> 🎯 **Confirmed by chief Marlon:** The CTF target is **OWASP Juice Shop**. See `05_Setup_JuiceShop.md` for full Juice Shop install. The list below covers the *supporting* tools.

Easiest path: download **Kali Linux 2025.x VMware image** from `https://www.kali.org/get-kali/#kali-virtual-machines`. It comes pre-loaded with most of what you need.

### 5.1 Critical (install or verify these first — Juice Shop needs them)

| Tool | Why | Install |
|---|---|---|
| **Burp Suite Community** | Intercept/modify every request. The single most important tool. | Pre-installed in Kali; or `https://portswigger.net/burp/communitydownload` |
| **Firefox** + **FoxyProxy** addon | Browser of choice for CTF; FoxyProxy toggles Burp on/off | `apt install firefox-esr` |
| **jwt_tool** | Forge JWTs for Juice Shop hard challenges | `pip install jwt-tool` or `git clone https://github.com/ticarpi/jwt_tool` |
| **CyberChef offline** | Encode/decode/encrypt anywhere | `git clone https://github.com/gchq/CyberChef && cd CyberChef && npm install && npm run build` → use `dist/index.html` |
| **Node.js 20 LTS** | Required to run Juice Shop from ZIP/source (we don't use Docker) | `https://nodejs.org/` — pick the LTS download |
| **sqlmap** | Some Juice Shop endpoints (e.g. `/rest/products/search`) crack faster with sqlmap | `apt install sqlmap` |
| **Postman** or **Hoppscotch** | Replay REST calls cleanly | `https://hoppscotch.io/download` (offline-installable) |

### 5.2 Useful (install if you have time/space)

```bash
sudo apt update && sudo apt install -y \
  zaproxy gobuster ffuf wfuzz hashcat john \
  hydra binwalk foremost exiftool \
  ghidra radare2 \
  zeek wireshark
```

### 5.3 Offline references — STAGE BEFORE COMPETITION

| Reference | Why | How |
|---|---|---|
| **Pwning OWASP Juice Shop** (free PDF/EPUB) | Walks every Juice Shop challenge | Leanpub `https://leanpub.com/juice-shop` set price = $0; or clone `https://github.com/juice-shop/pwning-juice-shop` |
| **HackTricks book** PDF | General web/AD/OSINT cheat-sheet | `https://book.hacktricks.wiki/` → save chapters to PDF |
| **GTFOBins** HTML | Linux post-exploitation cheats | clone `https://github.com/GTFOBins/GTFOBins.github.io` and open `index.md` locally |
| **HackTricks Cloud** | Cloud-meta SSRF references | `https://cloud.hacktricks.wiki/` |
| **OWASP Top 10** PDF | Map every challenge to a category | `https://owasp.org/Top10/` → save the PDF |

### 5.4 Lower priority (only if you're doing other CTF formats too)

```bash
sudo apt install -y crackmapexec impacket-scripts smbclient enum4linux \
  steghide stegoveritas volatility3 autopsy sleuthkit \
  python3-pwntools gdb gdb-peda
```

### 5.5 Wordlists + post-exploitation binaries (need the URLs)

| Item | Why | Source |
|---|---|---|
| **rockyou.txt** | Default password list for hashcat/hydra | Pre-installed in Kali at `/usr/share/wordlists/rockyou.txt.gz` — `gunzip` to use |
| **SecLists** | The richest wordlist collection (web paths, subdomains, passwords, payloads) | `git clone https://github.com/danielmiessler/SecLists.git` (~1 GB) |
| **LinPEAS / WinPEAS** | Linux + Windows post-exploitation scripts | `https://github.com/peass-ng/PEASS-ng/releases` (download `linpeas.sh`, `winPEASany.exe`, `winPEASx64.exe`) |
| **LinEnum.sh** | Older but still useful Linux enum | `https://github.com/rebootuser/LinEnum/raw/master/LinEnum.sh` |
| **PayloadsAllTheThings** | Reference wordbook for every web vuln | `git clone https://github.com/swisskyrepo/PayloadsAllTheThings.git` |
| **GTFOBins offline** | Linux SUID/sudo bypass cheats | `git clone https://github.com/GTFOBins/GTFOBins.github.io.git` |

> **CTF rule reminder:** during the actual CTF you are **not allowed** to access internet write-ups, ChatGPT/Gemini/Copilot, or external help. Everything must be on the **competitor PC's local disk** before the CTF starts.

---

## 5.6 Day-2 (Security Hardening) tooling

| Item | Why | Source | Approx size |
|---|---|---|---|
| **Security Onion 2.4 ISO** | Day 2 SOC platform (Suricata + Zeek + Wazuh + ELK + SOC web UI) | `https://securityonionsolutions.com/software` → links to GitHub releases at `https://github.com/Security-Onion-Solutions/securityonion/releases` | ~9 GB |
| **Wazuh Agent (Linux RPM)** | HIDS endpoint on CentOS hosts | `https://documentation.wazuh.com/current/installation-guide/wazuh-agent/wazuh-agent-package-linux.html` (RPM for RHEL/CentOS) | ~50 MB |
| **Wazuh Agent (Windows MSI)** | HIDS endpoint on Windows hosts | `https://documentation.wazuh.com/current/installation-guide/wazuh-agent/wazuh-agent-package-windows.html` | ~25 MB |
| **OpenVPN server packages (CentOS)** | Day 2 standalone OpenVPN install | EPEL repo (already pulled when you `dnf install -y epel-release openvpn easy-rsa`) | included |
| **NetworkMiner** (free) | Forensic GUI for PCAP — bonus for Day 2 forensics phase | `https://www.netresec.com/?page=NetworkMiner` | ~50 MB |

### Day-2 minimum download list (USB)
- [ ] Security Onion ISO (~9 GB)
- [ ] Wazuh agent RPM (Linux)
- [ ] Wazuh agent MSI (Windows)
- [ ] NetworkMiner ZIP (optional)

---

## 6. Documentation tools

| Tool | Why |
|---|---|
| **LibreOffice** or **MS Office** | Editing the appendix / executive summary in MA1 |
| **Greenshot / Snip & Sketch** | Screenshots for the CTF report |
| **Obsidian / Notepad++** | Personal cheat-sheets you can open offline |
| **Print-to-PDF (built into Windows)** | Final submission format |

---

## 7. Pre-flight checklist (tick before moving to `02_Setup_Topology.md`)

### Day-1 (MA1/MA2) prep
- [ ] VMware Workstation 17 installed + licensed on team laptop
- [ ] ESXi 8 installed on team server, web UI reachable on `https://<esxi-ip>/ui`
- [ ] All 4 OS ISOs downloaded to `D:\ISO\`
- [ ] Chrome, Firefox, PuTTY, WinSCP, Wireshark, Nmap on competitor PCs
- [ ] Chrome Enterprise Bundle .zip saved (for `googleChromeEnterpriseBundle64`)
- [ ] OpenVPN Connect installed on Client3

### CTF (Juice Shop) prep — see `05_Setup_JuiceShop.md`
- [ ] Kali Linux VM downloaded
- [ ] Node.js 20 LTS installed (`node --version` returns `v20.x.x`)
- [ ] OWASP Juice Shop running locally on `http://localhost:3000`
- [ ] Burp Suite Community installed and CA cert imported into Firefox
- [ ] jwt_tool installed
- [ ] CyberChef offline build extracted
- [ ] Pwning OWASP Juice Shop PDF saved offline
- [ ] HackTricks PDF saved offline
- [ ] (Optional) Local CTFd running on `http://localhost:8000` with Juice Shop challenges imported

### CTF (VulnHub boot-to-root) prep — see `06_Setup_VulnHub.md`
- [ ] VMware Workstation Host-Only network (VMnet1) configured
- [ ] Kali Linux running on VMnet1, can ping itself
- [ ] At least 8 VulnHub VMs downloaded to D:\VulnHub\ and verified by hash
  - [ ] Basic Pentesting: 1
  - [ ] Mr. Robot: 1
  - [ ] DC-1, DC-2, DC-3
  - [ ] Kioptrix: Level 1, Level 2
  - [ ] OWASP Broken Web Apps
- [ ] LinPEAS, WinPEAS binaries on USB
- [ ] GTFOBins offline mirror cloned
- [ ] LinEnum.sh on USB
- [ ] SecLists wordlist collection cloned
- [ ] PayloadsAllTheThings cloned
- [ ] /usr/share/wordlists/rockyou.txt extracted (it's gzipped by default)

### Day-2 (Security Hardening) prep — see `07_Setup_SecurityOnion.md`
- [ ] Host has ≥ 64 GB RAM if you'll run Day 2 alongside MA2 (else 32 GB and shut down MA2 VMs first)
- [ ] Security Onion 2.4 ISO downloaded (~9 GB) + SHA-256 verified
- [ ] Wazuh agent RPM (Linux) on USB
- [ ] Wazuh agent MSI (Windows) on USB
- [ ] CentOS Stream 9 ISO already on hand (you have it from Day 1) — used for the OpenVPN service VM
- [ ] NetworkMiner ZIP on USB (optional, for forensics phase)

When all ticked, go to **`02_Setup_Topology.md`**.
