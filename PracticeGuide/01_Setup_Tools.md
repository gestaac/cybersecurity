# 01 — Tools & Downloads (Setup from Zero)

This file lists everything you need to download and install **before** you build any VMs. Do this on the laptop that will host VMware Workstation, and **also** prepare a USB stick with the ISOs to take to the practice site.

> Allocate **one full day** for this file. Most of it is downloads + clicking *Next*.

---

## 0. Host hardware budget (read this first)

Team 1's actual hardware:
| Box | RAM | Disk | Role |
|---|---|---|---|
| **PC1** | 16 GB | 500 GB | Person A's workstation (Windows + VMware Workstation Pro) |
| **PC2** | 16 GB | 500 GB | Person B's workstation (Windows + VMware Workstation Pro) |
| **ESXi server** | TBD — see decision tree below | TBD | Hosts all the practice VMs |

### Per-VM resource table (full inventory)

| VM | RAM | Disk |
|---|---|---|
| **Day 1 — MA1 (CMS pentest):** | | |
| CMS target VM (Drupal 7 / Linux) | 2 GB | 20 GB |
| Kali Linux (attacker) | 4 GB | 80 GB |
| **Day 2 — MA2 (Security Hardening):** | | |
| ISP (CentOS) | 1 GB | 10 GB |
| pfSense | 2 GB | 20 GB |
| WINSRV1 (DC) | 4 GB | 80 GB |
| WINSRV3 (CA) | 3 GB | 60 GB |
| WINSRV4 (Root CA, off most of time) | 2 GB | 60 GB |
| LINSRV1 (CentOS) | 2 GB | 20 GB |
| Client1 (Win10) | 2 GB | 40 GB |
| Client2 (Win10) | 2 GB | 40 GB |
| Client3 (Win10) | 2 GB | 40 GB |
| **Day 3 — CTF (run one VM at a time):** | | |
| 1× VulnHub VM (varies) | 1–2 GB | 4 GB |
| Juice Shop (runs on host or Kali, no separate VM) | — | 1 GB |
| **Total nominal** *(MA2 active + Kali, WINSRV4 off)* | **~22 GB** | **~410 GB** |

### Disk reality check (with thin-provisioning)
ESXi defaults all disks to **thin-provisioned**, meaning a 60 GB VM disk only consumes **what's actually written** (typically 15–25 GB). So 930 GB nominal ≈ **300 GB real disk usage** in practice.

---

### Decision tree — what ESXi server do you need?

#### Scenario A — ESXi has ≥ 32 GB RAM and ≥ 1 TB SSD ✅ Ideal
- Power on every VM as designed.
- PC1 + PC2 are pure viewers. 16 GB / 500 GB each is plenty.
- No special workarounds.

#### Scenario B — ESXi has 16 GB RAM and 500 GB SSD ⚠️ Tight but workable
You can still do everything; you just **never run two phases simultaneously**. Power discipline becomes a habit:

| Session | Power on | RAM used |
|---|---|---|
| Day 1 — MA1 pentest | CMS target + Kali | ~6 GB |
| Day 2 — MA2 hardening | pfSense + WINSRV1 + WINSRV3 + LINSRV1 + Client1 + Client2 (skip ISP/WINSRV4 unless testing) | ~14 GB |
| Day 3 — CTF | Kali + 1 VulnHub VM | ~6 GB |

Disk: 500 GB on ESXi → with thin-prov, real usage ~150–250 GB. Manageable. Delete old snapshots aggressively.

#### Scenario C — ESXi is older / weaker than your PCs ⚡ Split workload across 3 boxes
If your ESXi server is the weakest box (say 8 GB RAM), flip the model:

- **ESXi server**: runs MA1 stack only (light VMs, ~10 GB total).
- **PC1**: runs MA2 stack via VMware Workstation Pro (16 GB available locally).
- **PC2**: runs Kali + Juice Shop + occasional VulnHub VMs locally.

Each PC has 16 GB RAM and 500 GB → comfortable to host one slice of the lab. Trade-off: you lose the "all VMs in one place" simplicity and the Infrastructure-List authenticity (competition uses one ESXi per team).

---

### What to do RIGHT NOW

1. **Find out your ESXi server's RAM and disk.** Boot it, look at the splash, or check `dmidecode` on Linux / Task Manager.
2. Match it to A / B / C above.
3. If **A**: proceed normally.
4. If **B**: read the "Power discipline" rules below before practising.
5. If **C**: tell me — I'll add a Scenario-C split-workload supplement to the guide.

---

### Power discipline rules (apply to Scenario B and tight setups)

- **Always shut down VMs you're not actively using.** RAM is the constraint, not disk.
- **WINSRV4 is OFF by default.** Only power on when you need to sign a CSR (~5 min once).
- **CMS target + Kali OFF** once you've moved past Day 1 practice.
- **Skip ISP** during MA2 unless specifically testing Internet-bound rules.
- **Take VM snapshots before shutting down** — so you can resume the exact state next session.

---

### Thin-provisioning vs thick (disk choice when creating VMs)

When ESXi prompts during VM creation:
| Option | When to use |
|---|---|
| **Thin Provision** ✅ default | Use this for everything. Disk grows as needed. |
| Thick Provision Lazy Zeroed | Only if you have abundant disk and want max performance. |
| Thick Provision Eager Zeroed | Skip — for production clusters only. |

> 🔑 **Memory tactics summary** (apply always):
> - Power off WINSRV4 except when signing certs.
> - Power off MA1 VMs once you've moved to MA2 (Day 2) practice.
> - Power off MA2 VMs before doing CTF / Day 3 practice — only Kali + 1 VulnHub VM needed.
> - Thin-provision all VMware disks (default in ESXi) — disk usage is much lower than the table totals suggest.

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
| **Google Chrome** | General browser for client-side verification | `https://www.google.com/chrome/` |
| **Mozilla Firefox** | Backup browser for cert inspection | `https://www.mozilla.org/firefox/` |
| **PuTTY 0.79+** | SSH to LinSRV1 over port 2022 | `https://www.putty.org/` |
| **WinSCP** | File copy to LinSRV1 | `https://winscp.net/` |
| **OpenVPN Connect** | VPN client on Client3 | `https://openvpn.net/client/` |
| **VMware Workstation Player** (if not Pro) | View VMs locally | same page as Pro |

### 3.2 Network + analysis
| Tool | Why | Source |
|---|---|---|
| **Wireshark 4.x** | Packet capture for troubleshooting LDAP/HTTPS | `https://www.wireshark.org/` |
| **Nmap + Zenmap** | FIN scan from Client3 to test Snort (MA2 PDF page 10) | `https://nmap.org/download.html` |

### 3.3 Microsoft management bundles
| Tool | Why | Source |
|---|---|---|
| **RSAT (Remote Server Administration Tools)** | If you ever manage AD from a Win10 client | `Settings → Apps → Optional Features → Add → RSAT` |


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

**Day 2 (MA2 — Security Hardening) tooling is already covered above:**
- pfSense 2.7.2 ISO (Section 2)
- CentOS Stream 9 ISO (Section 2) — for LINSRV1
- Win Server 2022 Eval (Section 2) — for WINSRV1/3/4
- Win 10 Eval (Section 2) — for Client1/2/3
- OpenVPN Connect (Section 3.1) — for Client3 dial-in
- Snort + OpenVPN packages on pfSense — pre-staged in MA2 PDF (no separate download)
- Wireshark + PuTTY (Section 3.2) — for client-side troubleshooting

> No additional Day-2-only tools required. Marlon's earlier "Security Onion + OpenVPN" mention referred to the OpenVPN that's already in MA2 — there's no separate SOC module.

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

### Day 1 + Day 2 prep (MA1 pentest + MA2 hardening)
- [ ] VMware OVF Tool installed on PC1 + PC2 (for OVA conversions before ESXi upload)
- [ ] ESXi 8 installed on the 3rd PC (the team server) per `02c_Setup_ESXi_Server.md`
- [ ] All OS ISOs downloaded to `D:\ISO\` on PC1, then uploaded to ESXi datastore (pfSense, CentOS Stream 9, Win Server 2022 Eval, Win 10 Eval, Security Onion 2.4)
- [ ] Chrome, Firefox, PuTTY, WinSCP, Wireshark, Nmap installed on PC1 + PC2
- [ ] OpenVPN Connect installed on Client3 VM (for VPN dial-in test)

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
- [ ] ESXi `PG-CTF` port group created (or reuse `PG-MA1-CMS`)
- [ ] Kali running on PG-CTF, can ping VulnHub target VMs on the same subnet
- [ ] At least 8 VulnHub VMs downloaded as `.ova` to `D:\OVA\` on PC1, then uploaded to ESXi datastore
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

When all ticked, go to **`02_Setup_Topology.md`**.
