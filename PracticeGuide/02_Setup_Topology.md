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

## E. Verification before moving on

- [ ] All five port groups visible in ESXi UI under *Networking → Port groups*
- [ ] Each laptop can reach ESXi web UI at `https://192.168.10.10/ui`
- [ ] You understand which port group each VM should sit on

If yes → next file: `03_Setup_VMs_MA1.md`.
