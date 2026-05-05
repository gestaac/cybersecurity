# 01 — Tools & Downloads (Setup from Zero)

This file lists everything you need to download and install **before** you build any VMs. Do this on the laptop that will host VMware Workstation, and **also** prepare a USB stick with the ISOs to take to the practice site.

> Allocate **one full day** for this file. Most of it is downloads + clicking *Next*.

---

## 1. Hypervisors

### 1.1 VMware Workstation Pro 17 (laptop)
- **Download:** `https://www.vmware.com/products/workstation-pro/workstation-pro-evaluation.html`
- Newer Broadcom path: log in to `https://support.broadcom.com` → "VMware Cloud Foundation" → "My Downloads" → search "Workstation Pro 17" (free for personal use as of 2024).
- **Install:** Default options. After install, open Workstation → *Help → Enter a License Key* (use personal-use key from the same page).
- **Why we need it:** runs the local VMs that mirror what the ESXi host will run on competition day.

### 1.2 VMware ESXi 8 (server)
- **Download:** Broadcom support portal → "VMware vSphere Hypervisor 8" (free licence after registration).
- **Install:** Boot the server from the ISO → *Install* → choose disk → set root password → reboot.
- **Post-install:**
  1. From a PC on the same wired network, browse to `https://<esxi-ip>/ui` → log in as `root`.
  2. *Manage → Licensing → Assign license* → paste the free licence key.
  3. *Networking → Virtual switches* — we'll create vSwitches in `02_Setup_Topology.md`.

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
| **Docker** | Easiest Juice Shop runner | `https://www.docker.com/products/docker-desktop/` (Win) or `apt install docker.io` (Linux) |
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
- **LinPEAS / WinPEAS:** `https://github.com/peass-ng/PEASS-ng/releases` — only needed if a non-Juice-Shop pivot challenge appears.

> **CTF rule reminder:** during the actual CTF you are **not allowed** to access internet write-ups, ChatGPT/Gemini/Copilot, or external help. Everything must be on the **competitor PC's local disk** before the CTF starts.

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
- [ ] Docker Desktop installed (or Node.js 20 if not using Docker)
- [ ] OWASP Juice Shop running locally on `http://localhost:3000`
- [ ] Burp Suite Community installed and CA cert imported into Firefox
- [ ] jwt_tool installed
- [ ] CyberChef offline build extracted
- [ ] Pwning OWASP Juice Shop PDF saved offline
- [ ] HackTricks PDF saved offline
- [ ] (Optional) Local CTFd running on `http://localhost:8000` with Juice Shop challenges imported

When all ticked, go to **`02_Setup_Topology.md`**.
