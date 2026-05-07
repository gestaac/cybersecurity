# 02 — Topology & Network Setup

This file replicates the **per-team physical setup** from `Infrastructure-List (2).docx` (the topology used at the actual competition) and prepares the **virtual networks** on ESXi that MA1 + MA2 will use.

You only need Team 1's setup. The other two teams are not present.

---

## A. Physical equipment (Team 1's actual rig)

You will use these physical pieces:

| Item | Quantity | Role |
|---|---|---|
| Desktop PC (PC1, PC2) — both running VMware Workstation Pro 17 on Windows | 2 | Teammates' workstations |
| Desktop / system unit running ESXi 8 | 1 | Hosts all the practice VMs |
| TP-Link 8-port unmanaged gigabit switch | 1 | Connects everything together |
| TP-Link router (any model with WAN + LAN ports) | 1 | Provides internet during practice (download installers, updates) |
| Cat 5e/6 straight-through ethernet cables | 4 minimum | Wiring |
| Power extension / surge-protected power strip (≥ 8 outlets recommended) | 1 | Power for 3 PCs + 2 monitors + switch + router |

> **About 2 NICs per PC:** the Infrastructure-List requires it for the *competition* (one NIC to team ESXi, one to the venue's CTFD LAN). For **solo Team 1 practice you can skip the 2nd NIC** — a single onboard NIC per PC handles everything below. Add a USB-Ethernet adapter (~₱500) only when you join the multi-team venue setup.

### Wiring diagram (Team 1, practice — 4 cables total)

```
                      Internet
                         │
                         ▼
                ┌─────────────────┐
                │ TP-Link Router  │
                │  192.168.1.1    │
                └────────┬────────┘
                         │  Cable 1: router LAN → switch port 1
                         ▼
              ┌──────────────────────────┐
              │ TP-Link 8-port Switch    │
              └─┬──────┬──────┬──────────┘
                │      │      │
        Cable 2 │  C3  │  C4  │
                ▼      ▼      ▼
             ┌─────┐ ┌─────┐ ┌──────────────┐
             │ESXi │ │ PC1 │ │     PC2      │
             │svr  │ │     │ │              │
             └─────┘ └─────┘ └──────────────┘
              .10     .101    .102  ← typical DHCP-assigned addresses
```

### Cable plan

| # | From | To |
|---|---|---|
| 1 | TP-Link router LAN port (1, 2, 3 or 4) | TP-Link switch port 1 |
| 2 | Switch port 2 | ESXi server NIC |
| 3 | Switch port 3 | PC1 onboard NIC |
| 4 | Switch port 4 | PC2 onboard NIC |

Switch ports 5–8 are spare (for future TV/CTFD scoreboard or 2nd NICs).

> Modern devices have **Auto-MDIX** so straight-through cables work everywhere. No crossover cables needed.

### Power-on order

1. **TP-Link router** first → wait ~30 sec until WAN/Internet light is solid.
2. **TP-Link switch** next → wait ~5 sec until link lights blink.
3. **ESXi server** next → wait ~3 min until console shows the yellow ESXi splash with a `https://...` URL.
4. **PC1 + PC2** last.

This order ensures DHCP requests get answered cleanly.

### IP addressing plan

The TP-Link router runs DHCP automatically (default subnet usually `192.168.1.0/24` with router at `.1`). We let the PCs use DHCP (zero config) and **pin a static IP on ESXi only**, so the URL is always the same.

| Device | IP method | Final IP | Why |
|---|---|---|---|
| TP-Link router | (its own default) | `192.168.1.1` | Gateway, DHCP, internet uplink |
| ESXi server mgmt vmk0 | **Static** (set in ESXi console) | `192.168.1.10` | So `https://192.168.1.10/ui` always works |
| PC1 onboard NIC | DHCP automatic | e.g. `192.168.1.101` | Just plug in, it works |
| PC2 onboard NIC | DHCP automatic | e.g. `192.168.1.102` | Same |

> Adjust the `.10`/`.101`/`.102` addresses to match your TP-Link router's actual subnet. Some TP-Link routers default to `192.168.0.0/24` — check the sticker on the router or open `192.168.0.1` / `192.168.1.1` in a browser from PC1 once it's plugged in.

### Set ESXi static IP (1 minute, once)

At the ESXi server's physical console (the screen shows yellow/grey ESXi splash):
1. **F2** → log in as `root`.
2. *Configure Management Network → IPv4 Configuration*.
3. Pick *Set static IPv4 address and network configuration*:
   - IP: `192.168.1.10` (or `192.168.0.10` if your router uses that subnet)
   - Subnet mask: `255.255.255.0`
   - Default gateway: `192.168.1.1` (your TP-Link router IP)
4. *DNS Configuration*:
   - Primary DNS: `8.8.8.8`
   - Secondary DNS: `1.1.1.1`
   - Hostname: `esxi-team1`
5. **Esc** → **Y** to apply and restart management network.
6. Reserve `192.168.1.10` in the TP-Link router's DHCP table (web UI → DHCP → Address Reservation → enter the ESXi MAC + IP) so DHCP never tries to give that IP to anything else.

### Verify everything

From both PC1 and PC2, open Command Prompt:

```cmd
ping 192.168.1.1         :: TP-Link router
ping 192.168.1.10        :: ESXi server
ping 8.8.8.8             :: internet (Google DNS)
ping google.com          :: internet + DNS resolution
```

All four should reply in under 50 ms. If any fail:

| Symptom | Likely cause | Fix |
|---|---|---|
| `ping 192.168.1.1` fails | PC NIC down, cable not seated, switch port dead | Check link lights on switch + PC; try another switch port |
| `ping 192.168.1.10` fails | ESXi static IP not yet applied, or wrong subnet | Re-do "Set ESXi static IP" above |
| `ping 8.8.8.8` fails | TP-Link router not connected to internet | Check WAN cable on router; reboot router |
| `ping 8.8.8.8` works but `ping google.com` fails | DNS misconfigured on PC | Use DHCP on the PC NIC, or add `8.8.8.8` as primary DNS |

---

## B. ESXi virtual networking — the four MA2 VLANs

MA2 needs four isolated networks: **Internet**, **LAN**, **DMZ**, **Servers**. We model each as an **ESXi port group** on its own **vSwitch** (no physical uplinks needed for inter-VM traffic).

### Steps in the ESXi web UI (`https://192.168.1.10/ui`)

#### Step 1 — Create vSwitch1 (Internet) + port group
**Why:** isolates the "fake Internet" + Client3 from the rest.
**Where:** ESXi UI → *Networking → Virtual switches → Add standard virtual switch*
**Clicks:**
- Name: `vSwitch-Internet`
- MTU: 1500
- Uplink: *(none — leave blank for VM-only)*
- Save
- Then *Port groups → Add port group* → Name: `PG-Internet` → vSwitch: `vSwitch-Internet` → VLAN ID: 0 → Save
**Expected:** `PG-Internet` shows under *Port groups*.

#### Step 2 — Create vSwitch2 (LAN) + port group `PG-LAN`
Same as Step 1 but `vSwitch-LAN` and `PG-LAN`.

#### Step 3 — Create vSwitch3 (DMZ) + port group `PG-DMZ`
Same.

#### Step 4 — Create vSwitch4 (Servers) + port group `PG-Servers`
Same.

#### Step 5 — Create vSwitch5 (MA1) + port group `PG-MA1-LAN`
**Why:** MA1 uses 172.16.100.0/24 with grimshay.local — separate from MA2 to avoid IP collision.
**Clicks:** Name: `vSwitch-MA1`, port group: `PG-MA1-LAN`, VLAN 0.

### Final port-group inventory

| Port group | Used by |
|---|---|
| `PG-Internet` | ISP, pfSense WAN, Client3 |
| `PG-LAN` | pfSense LAN, Client1, Client2 |
| `PG-DMZ` | pfSense DMZ, LinSRV1 |
| `PG-Servers` | pfSense Servers, WinSRV1, WinSRV3, WinSRV4 |
| `PG-MA1-LAN` | DC.grimshay.local, www.grimshay.ca, AMClient1, AMClient2 |
| Default `VM Network` | only used to give VMs initial internet access during install (then disconnect) |

---

## C. The MA2 logical IP plan (must memorise)

From MA2 Table 1:

```
Internet (PG-Internet)              LAN (PG-LAN)
┌──────────────┐                    ┌──────────────────┐
│ ISP          │                    │ Client1 (DHCP)   │
│ Client3 DHCP │                    │ Client2 (DHCP)   │
└─────┬────────┘                    └────────┬─────────┘
      │                                      │
      │       ┌────────────────────┐         │
      └─WAN──►│  pfSense Firewall  │◄──LAN──┘
              │  WAN: DHCP         │
              │  Servers: 192.168.2.254/24
              │  DMZ:     192.168.1.254/24
              │  LAN:     172.16.100.254/24
              └────┬───────────┬───┘
                   │           │
                   │ Servers   │ DMZ
                   ▼           ▼
              ┌────────┐    ┌──────────┐
              │WINSRV1 │    │ LinSRV1  │
              │.10 DC  │    │ .10 web  │
              │WINSRV3 │    └──────────┘
              │.30 CA  │
              │WINSRV4 │
              │.50 RCA │
              └────────┘
```

| VM | Port group | IP | Role |
|---|---|---|---|
| ISP | PG-Internet | static (e.g. 10.0.0.1/24) | Fake Internet, DNS, DHCP for WAN, hosts test sites |
| pfSense WAN | PG-Internet | DHCP from ISP | gateway to "Internet" |
| pfSense Servers | PG-Servers | 192.168.2.254/24 | gateway for Servers |
| pfSense DMZ | PG-DMZ | 192.168.1.254/24 | gateway for DMZ |
| pfSense LAN | PG-LAN | 172.16.100.254/24 | gateway + DHCP for LAN |
| WINSRV1 | PG-Servers | 192.168.2.10/24 | DC manila.com / DNS / file svc |
| WINSRV3 | PG-Servers | 192.168.2.30/24 | Issuing CA |
| WINSRV4 | PG-Servers | 192.168.2.50/24 | Offline Root CA (powered off) |
| LINSRV1 | PG-DMZ | 192.168.1.10/24 | Apache + DNS public-facing |
| Client1 | PG-LAN | DHCP | LAN test |
| Client2 | PG-LAN | DHCP | LAN test |
| Client3 | PG-Internet | DHCP from ISP | external test + VPN |

---

## D. The MA1 IP plan (separate)

| VM | Port group | IP | Role |
|---|---|---|---|
| DC.grimshay.local | PG-MA1-LAN | 172.16.100.10/24 | AD + DNS for grimshay.local |
| www.grimshay.ca | PG-MA1-LAN | 172.16.100.13/24 | Apache (intentionally weak) |
| AMClient1 | PG-MA1-LAN | 172.16.100.1/24 | Win10 client |
| AMClient2 | PG-MA1-LAN | 172.16.100.2/24 | Win10 client |

> Same `172.16.100.0/24` is used by both MA1 and the MA2 LAN — this is fine because they live on **different vSwitches** that never bridge to each other. Just don't power both sets on at once if you ever route them.

---

## E. Practice-mode notes (in addition to Section A above)

The wiring + IP plan in Section A already matches Team 1's practice rig (TP-Link router + 8-port switch + 2 PCs + ESXi). Section A is the source of truth — these are just additional notes that apply to practice only.

### E.1 Internet uplink is for PRACTICE ONLY

The TP-Link router brings internet so you can:
- Download VMware Workstation Pro, ESXi ISO, OS ISOs.
- Pull Wazuh agents, Security Onion ISO, VulnHub VMs, Juice Shop ZIP.
- Update Kali, install packages with `dnf`/`apt` during VM build.

**During the actual competition there is NO internet** — every package needed is pre-staged on the supplied VMs (per MA2 lines 178/181 *"Packages have been pre-downloaded"*). For Day 2 (Security Hardening), the competition organisers pre-download Security Onion + OpenVPN too. You only need internet during **practice** to download all the tools/VMs/ISOs to your USB stick.

> 💡 **Dry-run tip:** for your last 2 dry-runs, **unplug Cable 1** (router → switch) so PC1, PC2 and ESXi lose internet. This forces you to check that everything you need is already on local disk — exactly like competition day.

### E.2 PC NIC settings (Windows 10/11)

DHCP from the TP-Link router is the simplest path:

1. *Settings → Network & Internet → Ethernet → IP assignment → Edit → Automatic (DHCP)* → Save.
2. Verify in cmd:
   ```cmd
   ipconfig
   ```
   You should see an IPv4 address from the router's range (e.g. `192.168.1.101`) and a Default Gateway pointing to the router (`192.168.1.1`).
3. Internet test:
   ```cmd
   ping 8.8.8.8
   ping google.com
   ```
   Both should reply.

### E.3 Optional: 2nd NIC per PC (matches competition exactly)

Competition uses 2 NICs per PC (one for team eSXi, one for competition LAN/CTFD). For solo practice you can:
- **Skip it.** Use only the onboard NIC for everything. Simpler, fully functional.
- **Buy USB-Ethernet adapters** (~₱500 each) — adds a 2nd NIC. Both NICs go into the same TP-Link switch during practice.
- **Buy a PCIe NIC card** (~₱700 each) — cleaner because no USB cable hanging out the back.

For Day 1 + Day 2 + CTF practice — single onboard NIC is enough. The 2-NIC setup only matters when you join the multi-team venue.

### E.4 Optional: TV / spare monitor for the CTFD scoreboard

The Infrastructure-List diagram shows a TV connected to the venue's CTFD server displaying the scoreboard for spectators. You don't need this to *play* — but if you want a realistic dress-rehearsal:

| Option | What you need | Effort |
|---|---|---|
| **A — Skip it** | Nothing. Use Juice Shop's native score-board (`/#/score-board`) on PC1 to track progress | None — recommended for normal practice |
| **B — TV mirrored from a PC** | Spare TV + HDMI cable + PC1's HDMI-out. Open the score-board in a browser, drag to the TV display, fullscreen (F11) | 5 min |
| **C — Dedicated scoreboard PC** | Spare PC or laptop showing the score-board fullscreen, plugged into the TP-Link switch with TV via HDMI | 30 min |

> ⚠️ At the competition the TV scoreboard is provided by the organisers — don't bring your own. Section E.4 is purely for practice realism.

### E.5 Summary — bare-minimum vs full-sim practice rig

| Item | Bare minimum (Team 1's plan, works for everything) | Full sim (matches competition feel) |
|---|---|---|
| PCs | 2 desktops, single NIC each, DHCP | 2 desktops, dual NIC each |
| ESXi server | 1 system unit, 1 NIC, static `192.168.1.10` | 1 system unit, 2 NICs (mgmt + VM separated) |
| Switch | 1× TP-Link 8-port unmanaged | Same |
| Router | 1× TP-Link (for internet during practice) | Same |
| Internet | Yes (during build); disconnect for last 2 dry-runs | Same |
| TV / monitor | Optional | HDMI from a scoreboard PC |
| CTFD scoreboard | None — use Juice Shop's built-in score-board on PC1 | Optional dedicated scoreboard PC |

Your current plan = **bare minimum**, which is fine for everything below.

---

## F. Verification before moving on

- [ ] TP-Link router has internet (Internet/WAN light is solid green)
- [ ] All 4 cables seated; switch link lights blinking on ports 1–4
- [ ] ESXi server boot screen shows `https://192.168.1.10/` (or whatever IP you set)
- [ ] PC1 + PC2 each have a DHCP IP from the router (`ipconfig` shows `192.168.1.10x`)
- [ ] From PC1: `ping 192.168.1.10` (ESXi) replies; `ping 8.8.8.8` (internet) replies
- [ ] From PC1: browser opens `https://192.168.1.10/ui` and you can log in as `root`
- [ ] All five ESXi port groups created (PG-Internet, PG-LAN, PG-DMZ, PG-Servers, PG-MA1-LAN) — see Section B above
- [ ] You understand which port group each VM should sit on (Section C)

If yes → next file: `03_Setup_VMs_MA1.md`.

---

## G. What's on each box at competition (reference)

Consolidated picture of what's pre-installed where on competition day. Use as reference; informs what you bring on USB and what to expect on each VM.

### G.1 Physical PC1 + PC2 (your team workstations)

Mostly **VM viewers + documentation machines**. Minimal security tooling on the host OS itself.

| Layer | What |
|---|---|
| OS | Windows 10/11 |
| Hypervisor client | VMware Workstation Pro 17 (to view VMs on the ESXi server) |
| Browser | Chrome + Firefox (to reach ESXi web UI, pfSense web UI, Juice Shop) |
| Remote access | PuTTY, WinSCP, OpenSSH client (built in) |
| Documentation | LibreOffice / MS Office, Greenshot, Notepad++, Print-to-PDF |
| Auth credentials | `Competitor0 / CharterDressing` per MA1 line 20 |

> The login `Competitor0` is on the physical workstation. Inside the VMs, separate credentials apply (e.g. `MANILA\Anorbert / P@ssw0rd`).

### G.2 Physical ESXi server (the 3rd box)

Just one thing on it: **VMware ESXi 8.x**. ESXi is the bare-metal hypervisor — all real work happens inside the VMs it hosts.

| Layer | What |
|---|---|
| OS | VMware ESXi 8 (no Linux/Windows underneath) |
| Auth | `pmuser / Paul Bocuse` afternoon (MA2 line 38) and `amuser / Mt Blanc` morning (MA1 line 60) |

### G.3 VMs hosted on the ESXi server (the actual practice surface)

Pre-built and pre-configured by the organisers — already on the ESXi datastore when you arrive.

#### G.3.1 Morning — MA1 set (assess Apache vulnerabilities)
| VM | Role | Pre-installed inside |
|---|---|---|
| **DC.grimshay.local** | Domain controller, DNS for grimshay.local | AD, DNS |
| **www.grimshay.ca** | The deliberately weak Apache web server (your assessment target) | Apache + LDAP auth misconfigured (the flaws you're hunting) |
| **AMClient1** | Win10 LAN client | Chrome, PuTTY, Wireshark *(MA1 line 50)* |
| **AMClient2** | Win10 LAN client | Same |

#### G.3.2 Afternoon — MA2 set (build/harden manila.com)
| VM | Role | Pre-installed inside |
|---|---|---|
| **ISP** | Fake Internet, DNS, DHCP, hosts test sites | dnsmasq + Apache with self-signed certs |
| **pfSense** | Firewall, base install only — you configure it | pfSense 2.7.2 base + **OpenVPN package pre-downloaded** + **Snort package pre-downloaded** *(MA2 lines 178, 181)* |
| **WINSRV1** | DC for manila.com, file server | Win Server 2022 + AD + DNS + DHCP, **`googleChromeEnterpriseBundle64.zip` pre-staged in Documents** *(MA2 line 204)*, plus `manila.jpg` placeholder |
| **WINSRV3** | Issuing CA (half-built — you finish it) | Win Server 2022 + AD CS subordinate role partially configured |
| **WINSRV4** | Offline Root CA (already configured) | Win Server 2022 + AD CS standalone root |
| **LINSRV1** | Apache web server in DMZ, you harden it | CentOS Stream 9 + httpd + base packages, deliberately un-hardened |
| **Client1** | LAN client, DHCP | Win 10 + Chrome, PuTTY, Wireshark |
| **Client2** | LAN client, DHCP | Same |
| **Client3** | External client, DHCP from ISP | Win 10 + Chrome, PuTTY, Wireshark, **Nmap** *(MA2 line 235)*, OpenVPN Connect |

#### G.3.3 Day 2 — Security Hardening (per chief Marlon, speculative)
Official Day 2 doc not released yet. Likely:

| VM | Role | Pre-installed inside |
|---|---|---|
| **Security Onion** | SOC platform | Either pre-installed Security Onion 2.4 OR a fresh CentOS/Ubuntu with the SO ISO pre-staged on the datastore for you to install |
| **OpenVPN service** | Standalone OpenVPN server | Either pre-installed CentOS with OpenVPN pre-staged, OR fresh build using locally cached RPMs |
| **Possibly: monitored hosts** | Receivers of Wazuh agents | Day 1's LinSRV1 + WINSRV1 may carry over from morning |

#### G.3.4 Days 3–4 — CTF (random-pick from VulnHub + Juice Shop)
| VM | Role |
|---|---|
| **Kali Linux** | Your attacker box — pre-loaded with Burp Community, sqlmap, nmap, gobuster, ffuf, jwt_tool, hashcat, john, Volatility 3, Ghidra, etc. |
| **OWASP Juice Shop** | Web CTF target (confirmed by Marlon) — likely as a Node.js install on a server VM |
| **1–N VulnHub VMs** | Boot-to-root targets, randomly picked by chief from VulnHub catalogue |
| **CTFD server** | Scoreboard (separate from team ESXi — runs on competition LAN per Infrastructure-List, managed by organisers) |

### G.4 What you BRING on USB

Per CTF rules: **no internet at the venue**. Bring everything pre-downloaded:

| Category | Items |
|---|---|
| Reference docs | Pwning OWASP Juice Shop PDF, HackTricks PDF, GTFOBins offline mirror, OWASP Top 10 PDF |
| Wordlists | rockyou.txt, SecLists, PayloadsAllTheThings |
| Privesc tools | LinPEAS, WinPEAS, LinEnum.sh |
| Web tools | Burp Suite Community installer (in case Kali doesn't have it), CyberChef offline build |
| Day-2 specific | Wazuh agent RPM (Linux) + MSI (Windows), Security Onion 2.4 ISO (backup), NetworkMiner |
| Networking tools | Nmap installer (Windows), Wireshark installer |
| Documentation | LibreOffice installer, Greenshot installer |
| Backup | Full copy of your `PracticeGuide/` folder so you have offline reference |

### G.5 Network connections at competition (from Infrastructure-List)

Each PC has 2 NICs:
- **NIC #1** → team switch → reaches the **team's ESXi server** (where all team VMs live)
- **NIC #2** → competition LAN switch → reaches the **shared CTFD server + TV scoreboard** (managed by organisers)

Each team has:
- Its own **eSXi server** (the 3rd box)
- Its own **unmanaged switch** for team-internal traffic
- Connection to the **competition LAN** for scoring

The ESXi server is **not on the competition LAN** — it's behind the team switch. CTFD only sees scoring traffic from PC1/PC2's NIC #2.

### G.6 What is NOT pre-installed (you must do during competition)

These are the actual deliverables that earn you marks:

| Day | Action |
|---|---|
| MA1 | Write the executive summary + 2 vulnerability reports (no install — just analysis + writing) |
| MA2 | Configure pfSense rules, OpenVPN, Snort; harden LinSRV1; create AD GPOs/share/audit; finish PKI on WINSRV3 |
| Day 2 | Build/configure Security Onion, deploy Wazuh agents, configure OpenVPN service, investigate alerts |
| Days 3–4 | Solve the challenges (Juice Shop + VulnHub VMs) |

### G.7 Open unknowns (will be resolved when chief releases more docs)

| Unknown | Impact |
|---|---|
| Whether Day 2 VMs ship pre-installed with SO, or fresh OS for you to install SO from local ISO | Both cases covered in `24_Day2_SecurityHardening.md` |
| Which specific VulnHub VMs the chief picks | We practise on the top-8 most likely (per `06_…` Part F) |
| Whether CTFD is on a single shared server or per-team | Doesn't affect prep — you submit flags to whatever URL is given |
| Whether OpenVPN Day 2 is a separate VM or just a service on an existing one | `24_…` covers building from scratch, which works either way |

If the chief releases the Day 2 official doc, several of these unknowns disappear.
