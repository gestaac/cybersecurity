# Network Discovery Cheat-Sheet

> 🧠 **MEMORIZE WITH TEAMMATE — the one-line `arp-scan -l` command + your team subnet `192.168.10.0/24`.** Quiz: "What's the fastest LAN discovery command?" "What's our team subnet?" Both teammates must answer instantly.

> Use this on **competition setup day** to verify your rig is wired correctly, find live hosts, and confirm you can reach ESXi + CTFD.

---

## Your team's IP scheme (per Infrastructure-List topology)

```
              192.168.10.5
                 [SWITCH]
                /    |    \
       .10    .11   .12   .100
      [ESXi] [PC1] [PC2] [CTFD/Scoreboard]
```

| Device | IP | Purpose |
|---|---|---|
| **eSXi Server** | `192.168.10.10/24` | Hosts MA1/MA2 VMs |
| **PC1** (competitor 1a) | `192.168.10.11/24` | Day 1 MA1 + Day 3 Red CTF |
| **PC2** (competitor 1b) | `192.168.10.12/24` | Day 2 MA2 + Day 3 Blue CTF |
| Unmanaged Switch | `192.168.10.5/24` | Team-local switch |
| **CTFD Server / Scoreboard** | `192.168.10.100/24` | Flag submission |

> 🚨 **Note from topology:** "All Competitor PCs need 2 NICs for the 2 networks." Each PC has TWO ethernet ports — one for team-internal subnet, one for the shared CTFD network. **Confirm both NICs got IPs before doing anything else.**

---

## Step 0 — Find YOUR OWN IPs first (always)

Before scanning anyone else, know your own interface state.

### Linux / Kali
```bash
ip a                       # see all NIC IPs + subnet masks
ip route                   # see default gateway
hostname -I                # quick: just the IPs
```

Look for output like:
```
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> ...
    inet 192.168.10.11/24 brd 192.168.10.255 ...
3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> ...
    inet 10.0.0.50/24 ...
```
That tells you: NIC1 is on `192.168.10.0/24`, NIC2 is on `10.0.0.0/24`.

### Windows (PowerShell — on competitor PC)
```powershell
Get-NetIPAddress -AddressFamily IPv4 | Format-Table InterfaceAlias, IPAddress, PrefixLength
ipconfig /all              # classic detailed view
```

### Windows (CMD — quick)
```cmd
ipconfig
route print
```

---

## Step 1 — Scan for live hosts (pick ONE method)

> 🧠 **MEMORIZE this one:** `sudo arp-scan -l` — fastest method, works on Kali by default.

### Method A — `arp-scan` (recommended, 2-3 sec)

```bash
# Auto-detect interface + subnet
sudo arp-scan -l

# OR scan a specific subnet
sudo arp-scan 192.168.10.0/24
```

Sample output:
```
Interface: eth0, type: EN10MB, MAC: d8:43:ae:11:22:33, IPv4: 192.168.10.11
192.168.10.10  00:0c:29:aa:bb:cc  VMware, Inc.       ← ESXi
192.168.10.12  d8:43:ae:44:55:66  HP Inc.            ← PC2
192.168.10.100 00:50:56:77:88:99  VMware, Inc.       ← CTFD
```

If `arp-scan` is missing on Kali:
```bash
sudo apt update && sudo apt install -y arp-scan
```

### Method B — `nmap` ping sweep (universal)

```bash
# Standard ICMP ping sweep
nmap -sn 192.168.10.0/24

# If ICMP is blocked, probe TCP ports instead:
nmap -PS22,80,443,3389 -sn 192.168.10.0/24
```

### Method C — `netdiscover` (passive + active)

```bash
sudo netdiscover -i eth0 -r 192.168.10.0/24
# Press Ctrl+C when you've seen enough hosts
```

### Method D — `fping` (lightweight)

```bash
fping -a -g 192.168.10.0/24 2>/dev/null
# -a = alive only, -g = generate IP range
```

### Method E — Windows PowerShell (no Linux tools)

```powershell
# Sweep ping the subnet
1..254 | ForEach-Object {
  $ip = "192.168.10.$_"
  if (Test-Connection -ComputerName $ip -Count 1 -Quiet -TimeoutSeconds 1) {
    Write-Host "$ip ALIVE"
  }
}

# Or check ARP cache after some traffic
Get-NetNeighbor -AddressFamily IPv4 | Where-Object {$_.State -in 'Reachable','Stale'} | Format-Table
```

### Method F — Windows CMD (no PowerShell)

```cmd
arp -a
for /L %i in (1,1,254) do @ping -n 1 -w 100 192.168.10.%i | find "Reply"
```

---

## Step 2 — Verify reachability of critical hosts

Once you have a list of live IPs:

```bash
# ESXi — should respond on 443 (web UI)
ping 192.168.10.10
curl -k https://192.168.10.10/                # accept self-signed cert

# CTFD — should respond on 80 or some web port
ping 192.168.10.100
curl http://192.168.10.100/

# Your teammate's PC — should ping
ping 192.168.10.12      # from PC1
ping 192.168.10.11      # from PC2
```

If any fails:
- Check cable seating
- Check switch power
- Check NIC link light
- Run `ip a` again — did the interface go DOWN?

---

## Step 3 — Port-scan a specific host

Once you know an IP is alive, see what's running on it:

```bash
# Quick top-1000 ports
nmap -sC -sV -T4 192.168.10.10

# Full 65535 ports (slower)
nmap -p- --min-rate 5000 192.168.10.10

# UDP scan (slow but sometimes needed)
sudo nmap -sU --top-ports 50 192.168.10.10
```

---

## Setup-day full sequence (run this tomorrow morning at the venue)

```bash
# 1. Confirm both NICs got addresses
ip a

# 2. Note both subnets you're connected to
ip route

# 3. Scan team-internal subnet
sudo arp-scan 192.168.10.0/24

# 4. Verify ESXi reachable
ping -c 4 192.168.10.10
curl -k https://192.168.10.10/

# 5. Verify CTFD reachable
ping -c 4 192.168.10.100
curl http://192.168.10.100/

# 6. Confirm teammate's PC reachable
ping -c 4 192.168.10.12     # from PC1 (or .11 from PC2)

# 7. If everything pings, you're set. Save outputs:
ip a > ~/setup_day_iface.txt
sudo arp-scan -l > ~/setup_day_arp.txt
```

---

## Common problems + fixes

| Symptom | Likely cause | Fix |
|---|---|---|
| `ip a` shows NIC but no IPv4 | DHCP didn't assign | `sudo dhclient eth0` (Linux) or `ipconfig /renew` (Windows) |
| Ping works to ESXi (.10) but not CTFD (.100) | 2nd NIC down or wrong cable | Check 2nd cable + `ip a` for second interface |
| `arp-scan` returns 0 hosts | Wrong interface or wrong subnet | Add `-I eth0` or use `-l` for auto-detect |
| `nmap -sn` says all hosts down | ICMP blocked by switch/firewall | Use `nmap -PS22,80,443 -sn` instead |
| Can ping IPs but ESXi web UI 403/timeout | ESXi service not started yet OR wrong port | Try `https://192.168.10.10:443` explicitly; wait 2 min for ESXi boot |
| You're getting `192.168.X.Y` instead of `192.168.10.X` | Venue uses different subnet than topology | Run `ip a`, note the actual subnet, scan THAT subnet |

---

## 🧠 Drill table — quiz your teammate tonight

| Question | Answer |
|---|---|
| Our team's internal subnet? | `192.168.10.0/24` |
| ESXi IP? | `192.168.10.10` |
| PC1 IP? | `192.168.10.11` |
| PC2 IP? | `192.168.10.12` |
| CTFD/Scoreboard IP? | `192.168.10.100` |
| Switch IP? | `192.168.10.5` |
| Fastest LAN discovery command? | `sudo arp-scan -l` |
| Universal ping sweep? | `nmap -sn 192.168.10.0/24` |
| First command on setup day? | `ip a` (verify your own NICs) |
| How many NICs per PC? | **2** — one for team subnet, one for CTFD |
| If ICMP is blocked, what nmap flag? | `-PS22,80,443` (probe TCP ports instead) |
| Windows equivalent of `ip a`? | `ipconfig` or `Get-NetIPAddress` |

---

## What to bring on USB

- [ ] `arp-scan` Debian package (in case Kali install needs offline)
- [ ] `nmap` installer for Windows (`nmap-setup.exe`) — sometimes useful on competitor PC
- [ ] This cheat-sheet printed
- [ ] Note paper for jotting actual venue IPs (in case different from topology)

---

## If venue IPs differ from topology

Don't panic. The same commands still work, just with the **actual** subnet:

1. Run `ip a` → see your real IP (e.g. `10.20.30.42/24`).
2. Adjust scan: `sudo arp-scan 10.20.30.0/24`.
3. Find ESXi by MAC vendor (`VMware, Inc.` in arp-scan output) — that's your hypervisor regardless of IP.
4. Find CTFD by visiting it: `curl http://<every-live-ip>/` and look for "CTFd" in the response.

The topology is the **expected** layout; the actual venue rig might differ slightly. The discovery flow stays the same.
