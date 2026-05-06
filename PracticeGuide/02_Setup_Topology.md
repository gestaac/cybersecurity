# 02 — Topology & Network Setup

This file replicates the **per-team physical setup** from `Infrastructure-List (2).docx` (the topology used at the actual competition) and prepares the **virtual networks** on ESXi that MA1 + MA2 will use.

You only need Team 1's setup. The other two teams are not present.

---

## A. Physical equipment (per Infrastructure-List)

You should have brought to practice:

| Item | Quantity |
|---|---|
| ESXi server (your own, with ESXi 8 installed) | 1 |
| Workstation PC (laptop or desktop with VMware Workstation Pro) | 2 (one per teammate) |
| Unmanaged 5/8-port switch | 1 |
| RJ45 patch cords ≥ 3 m | 4 |
| Power extension | as needed |
| Optional: 2nd NIC USB-Ethernet adapter for each PC (so each PC has 2 NICs) | 2 |

> **Why 2 NICs per PC?** Per the topology diagram in `Infrastructure-List (2).docx`: one NIC reaches the team's ESXi (for VM consoles via VMware Workstation), the other NIC reaches the **competition LAN** (where the CTFD scoreboard lives). For solo practice, you can simulate the competition LAN with just the ESXi NIC and skip CTFD until Day 2 prep.

### Wiring diagram (Team 1, practice)

```
[ Laptop PC1 ]──┐
                 ├──[ Unmanaged Switch ]──[ ESXi Server  192.168.10.10 ]
[ Laptop PC2 ]──┘                                  │
                                                    └── (later, at venue) → Team competition uplink
```

### IP plan for the team underlay (between PCs and ESXi)

| Device | IP | Mask | Notes |
|---|---|---|---|
| ESXi mgmt vmk0 | 192.168.10.10 | /24 | Default ESXi mgmt |
| Laptop PC1 NIC | 192.168.10.11 | /24 | Static, no GW |
| Laptop PC2 NIC | 192.168.10.12 | /24 | Static, no GW |

> Set static IPs on each laptop NIC: *Settings → Network & Internet → Ethernet → Edit IP settings → Manual → IPv4*. No gateway needed if you don't have internet.

Verify by pinging from each laptop:
```cmd
ping 192.168.10.10
```
Expected: replies in <5 ms. If not — check cables, switch power, NIC enabled.

---

## B. ESXi virtual networking — the four MA2 VLANs

MA2 needs four isolated networks: **Internet**, **LAN**, **DMZ**, **Servers**. We model each as an **ESXi port group** on its own **vSwitch** (no physical uplinks needed for inter-VM traffic).

### Steps in the ESXi web UI (`https://192.168.10.10/ui`)

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

## E. Practice-Mode physical setup (Team 1 alone, with internet)

The competition diagram in `Infrastructure-List (2).docx` shows 3 teams + a shared CTFD scoring server + a TV scoreboard, all on a competition LAN. For **solo Team 1 practice** you do NOT need the multi-team setup. Build this instead:

### E.1 Wiring (practice)

```
                            ┌── (router / phone hotspot WAN)
                            │   for downloading installers
                            ▼
                  ┌─────────────────────┐
                  │   Home/team router  │
                  └──────────┬──────────┘
                             │
                ┌────────────┴────────────┐
                ▼                         ▼
      ┌─────────────────┐       ┌─────────────────┐
      │ ESXi Server     │       │  Unmanaged      │
      │ 192.168.10.10   │───────│  Switch         │
      └─────────────────┘       └─┬─────────┬─────┘
                                  │         │
                                  ▼         ▼
                         ┌──────────┐  ┌──────────┐
                         │ Laptop1  │  │ Laptop2  │
                         │ NIC1     │  │ NIC1     │
                         │ NIC2*    │  │ NIC2*    │
                         └──────────┘  └──────────┘
                          *NIC2 optional for practice
```

> **Internet uplink is for practice ONLY.** During the actual competition there is **no internet** — every package needed is pre-staged on the supplied VMs (per MA2 lines 178/181 *"Packages have been pre-downloaded"*). For Day 2 (Security Hardening) the competition organisers pre-download Security Onion + OpenVPN packages too, so you don't need internet there either. You only need internet during *practice* to download all the tools/VMs/ISOs.

### E.2 What plugs into what

| Cable | From | To |
|---|---|---|
| Cable 1 | Home router LAN port | Unmanaged switch port 1 |
| Cable 2 | Switch port 2 | ESXi server NIC1 (mgmt) |
| Cable 3 | Switch port 3 | Laptop1 NIC1 |
| Cable 4 | Switch port 4 | Laptop2 NIC1 |

Power up the switch first, then ESXi, then laptops.

### E.3 Network settings on each laptop (Windows 10/11)

For practice, set **NIC1 to DHCP** (so the home router gives it both an IP and an internet route). The static-IP approach in section A is for the **isolated competition rig** where there's no DHCP — for practice with internet, DHCP is simpler.

1. *Settings → Network & Internet → Ethernet → NIC1 → IP assignment → Edit → Automatic (DHCP)* → Save.
2. Verify: `ipconfig` shows an IP from your home router's range (e.g. 192.168.1.x) plus a Default Gateway.
3. Verify internet: `ping 8.8.8.8` and `ping google.com` both reply.

### E.4 ESXi management IP for practice

Either:
- Let DHCP give ESXi an IP automatically (note it from the ESXi console at boot), then browse to `https://<dhcp-ip>/ui`.
- Or set a static IP on ESXi (`F2` at console → Configure Management Network → IPv4 Configuration → Static → 192.168.1.10, mask 255.255.255.0, gateway 192.168.1.1). Adjust to match your home router's subnet.

> Take note of the ESXi IP — you'll use it from both laptops to reach the web UI.

### E.5 Optional: 2nd NIC per laptop (matches competition exactly)

Competition uses 2 NICs per PC (one for team eSXi, one for competition LAN/CTFD). For practice you can:
- **Skip it.** Use only NIC1 for everything. Simpler, fully functional for solo practice.
- **Buy USB-Ethernet adapters** (~$10 each) — gives each laptop a 2nd NIC. Plug into the same switch. Then:
  - NIC1 = team subnet (192.168.1.x with internet)
  - NIC2 = "competition" subnet (could be a 2nd small switch, or another VLAN)
- **Use a virtual NIC inside Workstation** — VMware Workstation can present a 2nd "host-only" virtual NIC. Cheapest option, but only works for VMs, not the host laptop's apps.

> For Day 1 + Day 2 + CTF practice — single NIC1 is enough. The 2-NIC setup only matters for the spectator-scoreboard simulation (next section).

### E.6 Optional: TV / spare monitor for the CTFD scoreboard

The Infrastructure-List diagram shows a TV connected to a CTFD server displaying the scoreboard for spectators. You don't need this to *play* — but if you want a realistic dress-rehearsal, here are 3 options ranked by effort:

| Option | What you need | Effort |
|---|---|---|
| **A — Skip it** | Nothing. Just open the CTFD UI in a browser tab on Laptop1 when you want to check score | None — recommended for normal practice |
| **B — TV mirrored from a laptop** | Spare TV + HDMI cable + Laptop1's HDMI-out. Open CTFD scoreboard in a browser, drag to the TV display, fullscreen (F11) | 5 min |
| **C — Dedicated CTFD box** | Spare PC or laptop running CTFD locally, plugged into the switch, with TV via HDMI showing the scoreboard. *(Setting up CTFd is non-trivial without Docker; for solo practice, use the Juice Shop native score-board instead — see `05_…` Step 4.)* | 30 min |

**For local CTFD setup, see `05_Setup_JuiceShop.md` Step 4** — you already have the install steps. Just leave that PC running CTFD in the corner of the room with the TV plugged into it.

> ⚠️ TV scoreboard at competition is provided by the organisers — don't bring your own. Section E.6 is purely for practice realism.

### E.7 Summary — the bare-minimum vs maximum practice rig

| Item | Bare minimum (works for everything) | Full sim (matches competition feel) |
|---|---|---|
| Laptops | 2, single NIC each, DHCP | 2, dual NIC each, NIC1 team / NIC2 competition |
| ESXi server | 1 with 1 NIC | 1 with 2 NICs (mgmt + VM traffic separated) |
| Switch | 1 unmanaged 5-port | 1 per network (2 switches total) |
| Internet | Yes (during build) | Yes (build) / disconnect for CTF dry-runs |
| TV / monitor | None | HDMI from CTFD-running PC |
| CTFD scoreboard | None — use Juice Shop's built-in score-board on Laptop1 | Optional dedicated PC (only for spectator dress-rehearsal) |

Recommendation: start with **bare minimum** for Days 1 + 2 prep. Add CTFD/TV during Week 2 only if you want the dress-rehearsal experience.

---

## F. Verification before moving on

- [ ] All five port groups visible in ESXi UI under *Networking → Port groups*
- [ ] Each laptop can reach ESXi web UI at `https://<esxi-ip>/ui`
- [ ] Each laptop has internet (`ping google.com` works)
- [ ] You understand which port group each VM should sit on
- [ ] You've decided: bare-minimum or full-sim practice rig (both are fine)

If yes → next file: `03_Setup_VMs_MA1.md`.
