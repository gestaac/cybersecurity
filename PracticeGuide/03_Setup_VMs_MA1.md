# 03 — Build the MA1 Practice Environment (CMS pentest target)

The actual MA1 (per the official PDF in `testpacakge_pdf/`) is a **pentest** of a Linux server running a CMS. You have 2 VMs:

| VM | IP | Role |
|---|---|---|
| **Linux Server with CMS** | 192.168.2.1 | Target machine (intentionally vulnerable CMS) |
| **Kali Linux** | 192.168.2.2 | Attacker box (Kali + standard pentest tools) |

For practice we build the same 2 VMs ourselves. The chief hasn't disclosed which CMS the competition will use — we go with **Drupal 7** because:
- It's the most-used CMS in WSC pentest scenarios.
- Maps cleanly to the 4 MA1 tasks (Info Gathering / CMS Vuln Assess / System Weaknesses / Report).
- Ships with the famous **Drupalgeddon 2 (CVE-2018-7600)** — exploitable for RCE.
- Same family as VulnHub's **DC-1** which has documented walkthroughs.

> Time to build: ~90 min first time, 5 min from snapshot afterward.

> **At competition** the chief pre-installs the target VM and Kali. You don't build them — you just run the pentest in `10_Day1_MA1_Solution.md`. This file is for **practice prep** so you've seen the same shape of target before competition day.

---

## Part 1 — Network setup (on the ESXi server)

Both VMs sit on the **PG-MA1-CMS** port group on the ESXi server with subnet `192.168.2.0/24`. They don't need internet after the build is done.

### Create PG-MA1-CMS on ESXi
1. ESXi UI → **Networking** → **Port groups** → **Add port group**.
2. Fill in:
   - **Name:** `PG-MA1-CMS`
   - **VLAN ID:** `0`
   - **Virtual switch:** the same vSwitch you set up for PG-Servers (or a separate one if you want full isolation).
3. Click **Add**.

> 💡 Both Kali and the CMS target will connect to this same port group. They talk to each other via this virtual cable.

---

## Part 2 — Build the CMS target VM (Drupal 7)

### Step 2.1 — Create the VM (on ESXi)
- ESXi UI → Virtual Machines → **Create / Register VM** → **Create a new virtual machine** → Next.
- **Name:** `CMS-Target` (this is just the ESXi label — the OS hostname is set later)
- **Compatibility:** ESXi 8.0 virtual machine
- **Guest OS family:** Linux
- **Guest OS version:** **Ubuntu Linux (64-bit)** OR **CentOS 9 (64-bit)** (Ubuntu is simpler for beginners; pick one and stick with it)
- Click Next → select your datastore → Next.
- **Customize hardware:**
  - **CPU:** 1 vCPU
  - **Memory:** `2048` MB (2 GB)
  - **Hard disk 1:** `20` GB, **Thin Provisioned** (under Disk Provisioning)
  - **Network adapter 1:** `PG-MA1-CMS` ← important
  - **CD/DVD Drive 1:** **Datastore ISO file** → browse to the Ubuntu Server 22.04 OR CentOS Stream 9 ISO you uploaded to the datastore → tick **Connect at power on**
- Review → **Finish**.
- Power on the VM → click **Console** to open the install screen.

### Step 2.1.1 — Walk through the Ubuntu Server 22.04 installer (~15 min)

> Beginner notes: navigation is **arrow keys** + **Tab** + **Enter**. The mouse won't work in this text installer. **Space** toggles checkboxes. **Tab** moves between [Done] / [Back] / [Cancel] buttons at the bottom.

**Screen 1 — GRUB boot menu** (5-second countdown)
- Just wait, or press **Enter** on `Try or Install Ubuntu Server`.

**Screen 2 — Language**
- Highlight `English` → **Enter**.

**Screen 3 — Installer update available?** (may or may not appear)
- Pick **Continue without updating** → **Enter**. *(We don't have internet access from this VLAN, and we don't need updates.)*

**Screen 4 — Keyboard configuration**
- Layout: `English (US)` — leave default.
- Variant: `English (US)` — leave default.
- Tab to **[Done]** → **Enter**.

**Screen 5 — Choose type of install**
- Highlight **Ubuntu Server** (NOT "Ubuntu Server (minimized)" — we want the full one).
- Tab to **[Done]** → **Enter**.

**Screen 6 — Network connections**
- You'll see one entry like `ens33 eth -` or `ens34` or `ens160` showing it's trying to get DHCP and **failing** (since PG-MA1-CMS has no DHCP server). Status will say `DISABLED` / `auto-config failed` — that's expected.
- 📝 **Write down the interface name** (e.g. `ens33`, `ens34`, `ens160`) — you'll need it in Step 2.2 to replace `ens33` in the netplan config.
- **Don't try to fix DHCP** — leave it as is. We set a static IP after install in Step 2.2.
- Tab to **[Done]** → **Enter**.
- If a "Continue without network?" prompt appears → **[Continue without network]** → Enter.

> 💡 The interface name varies by VMware hardware version + BIOS — `ens33` / `ens34` / `ens160` are all normal. Whatever name appears, use that exact name everywhere `ens33` is mentioned in Step 2.2.

> 💡 If install hangs here for 2+ minutes waiting for DHCP, just Tab to [Done] anyway — install will continue offline.

**Screen 7 — Configure proxy**
- Leave blank.
- Tab to **[Done]** → **Enter**.

**Screen 8 — Configure Ubuntu archive mirror**
- Leave default (`http://archive.ubuntu.com/ubuntu`).
- Tab to **[Done]** → **Enter**.

**Screen 9 — Guided storage configuration**
- Leave **Use an entire disk** ticked (default).
- The disk shown is your 20 GB virtual disk — leave selected.
- Leave **Set up this disk as an LVM group** ticked.
- Tab to **[Done]** → **Enter**.

**Screen 10 — Storage configuration (review)**
- Don't change anything.
- Tab to **[Done]** → **Enter**.

**Screen 11 — Confirm destructive action**
- A red box pops up: *"The installer will format the disk."*
- Tab to **[Continue]** → **Enter**.

**Screen 12 — Profile setup** ← THE IMPORTANT ONE
- **Your name:** `Competitor`
- **Your server's name:** `cms-target` ← lowercase, no spaces
- **Pick a username:** `competitor`
- **Choose a password:** `P@ssw0rd`
- **Confirm your password:** `P@ssw0rd`
- Tab to **[Done]** → **Enter**.

**Screen 13 — Upgrade to Ubuntu Pro**
- Highlight **Skip for now**.
- Tab to **[Continue]** → **Enter**.

**Screen 14 — SSH Setup**
- ✅ **Press Space to tick `Install OpenSSH server`** (so we can SSH in later from Kali).
- Leave **Import SSH identity** as `No`.
- Tab to **[Done]** → **Enter**.

**Screen 15 — Featured Server Snaps**
- **Don't select anything** — we install LAMP manually.
- Tab to **[Done]** → **Enter**.

**Screen 16 — Installing the system** (5–10 min wait)
- A green log scrolls. The installer downloads/installs base packages.
- When done, the bottom button changes from `Cancel update and reboot` to **[Reboot Now]**.
- Tab to **[Reboot Now]** → **Enter**.

**Screen 17 — "Please remove the installation medium"**
- The VM tries to reboot but stalls because the ISO is still mounted.

> ⚠️ **EJECT THE ISO HERE — important step:**
> 1. Don't close the console.
> 2. Switch to the ESXi UI tab in your browser.
> 3. CMS-Target VM → **Edit** (top button).
> 4. Find **CD/DVD Drive 1** → **uncheck "Connect at power on"** AND change dropdown from "Datastore ISO file" to **"Host device"** (or just disconnect).
> 5. Click **Save**.
> 6. Switch back to the VM console → press **Enter** → reboot continues.

**Screen 18 — GRUB after reboot**
- 5-second countdown → boots Ubuntu.

**Screen 19 — Login prompt**
```
cms-target login: _
```
- Type `competitor` → Enter
- Password: `P@ssw0rd` → Enter (won't show as you type — that's normal)
- You see a shell prompt: `competitor@cms-target:~$`

**You're done with the installer.** Continue to Step 2.2 to set the static IP.

### Common Ubuntu installer issues

| Problem | Fix |
|---|---|
| Installer keyboard doesn't respond | Click inside the VM console window first (gives it focus). Press **Ctrl+Alt** to release mouse afterwards. |
| Stuck at "waiting for DHCP" for 5+ minutes | Tab to **[Done]** anyway — install continues without internet. |
| Install screen freezes mid-way | ESXi → VM → Power → **Reset**. Re-mount ISO if needed. |
| After reboot it boots installer again instead of OS | The ISO is still mounted. Edit VM → CD/DVD → uncheck Connect at power on → Save → reset VM. |
| Login fails — "Login incorrect" | The `c` you typed might have been Caps Lock'd. Confirm Caps Lock is OFF. Username + password are case-sensitive. |

### Step 2.2 — Set static IP `192.168.2.1`
After OS install, log in as root:

**Ubuntu:**

First, confirm your interface name:
```bash
ip link show
```
Look for the line that's NOT `lo` — example output:
```
2: ens34: <BROADCAST,MULTICAST,UP,LOWER_UP> ...
```
Note your actual name (here `ens34`). It might be `ens33`, `ens34`, `ens160`, or other. **Use whatever YOUR machine shows** in the YAML below.

Open the netplan file in nano:
```bash
sudo nano /etc/netplan/00-installer-config.yaml
```

> 💡 **If you see a blank screen after running this** — that's normal. The installer didn't create the file (because DHCP failed), so nano opens an empty new file. Just type the content below.

Type these contents (substitute `ens34` with YOUR actual interface name):
```yaml
network:
  version: 2
  ethernets:
    ens34:                                # ← change to YOUR interface name
      addresses: [192.168.2.1/24]
      nameservers:
        addresses: [8.8.8.8]
      routes:
        - to: default
          via: 192.168.2.254
```

#### ⚠️ Typing tips (YAML is strict)

If you're typing this manually in nano (not pasting), follow these 3 rules to avoid errors:

1. **NEVER press Tab. Always press the Space bar.** YAML treats tabs as invalid.
2. **Indent in steps of 2 spaces.** Each level of nesting is exactly 2 spaces deeper than its parent.
3. **Indent count cheat sheet:**

   | Line | Spaces before first character |
   |---|---|
   | `network:` | 0 (column 1, far left) |
   | `version: 2` | 2 spaces |
   | `ethernets:` | 2 spaces |
   | `ens34:` (your interface) | 4 spaces |
   | `addresses: [192.168.2.1/24]` | 6 spaces |
   | `nameservers:` | 6 spaces |
   | `addresses: [8.8.8.8]` (the inner one) | 8 spaces |
   | `routes:` | 6 spaces |
   | `- to: default` | 8 spaces, then `-`, then space, then `to:` |
   | `via: 192.168.2.254` | 10 spaces |

After typing → **Ctrl+O** → **Enter** → **Ctrl+X**.

Verify your file looks right:
```bash
cat /etc/netplan/00-installer-config.yaml
```
Compare line-by-line with the YAML above. If anything is wrong, re-run `sudo nano /etc/netplan/00-installer-config.yaml` and fix it.

#### Step 2.2.1 — Fix file permissions FIRST (before applying)

By default, the file you just created is world-readable, which makes netplan complain on every apply with warnings like:

```
WARNING: Permissions for /etc/netplan/00-installer-config.yaml are too open.
         Configuration should NOT be accessible by others.
```

Fix it once now so you don't see the warnings later:

```bash
sudo chmod 600 /etc/netplan/00-installer-config.yaml
```

This makes the file readable + writable **only by root**. No more permission warnings.

#### Step 2.2.2 — Apply the network config

```bash
sudo netplan apply
```

**Expected output: no warnings, no output, just back to the prompt.**

> 💡 If you forgot Step 2.2.1 and ran `netplan apply` first, you may have seen a wall of yellow `WARNING` text plus a separate `WARNING:root:Cannot call Open vSwitch: ovsdb-server.service is not running.` — both are **non-fatal**. The "Open vSwitch" one is harmless (Ubuntu probing for advanced networking we don't use; ignore forever). The permissions ones disappear once you run `chmod 600`. The IP gets applied either way.

#### Step 2.2.3 — Verify the IP is set

```bash
ip a show ens34                            # ← use YOUR interface name
```

Look for the line:
```
inet 192.168.2.1/24 brd 192.168.2.255 scope global ens34
```

If you see `inet 192.168.2.1/24` → ✅ **static IP is set.**

Also check the route:

```bash
ip route
```

Should show:
```
default via 192.168.2.254 dev ens34 proto static
192.168.2.0/24 dev ens34 proto kernel scope link src 192.168.2.1
```

#### Common warnings during `netplan apply`

| Warning | Meaning | Action |
|---|---|---|
| `Permissions for /etc/netplan/...yaml are too open` | File is readable by all users | Run `sudo chmod 600 /etc/netplan/00-installer-config.yaml` once. Re-apply. Goes away. |
| `WARNING:root:Cannot call Open vSwitch: ovsdb-server.service is not running` | Netplan checked for OpenVSwitch (advanced virtual switching). We don't use it. | **Ignore — completely harmless.** Will appear forever, doesn't affect anything. |
| `Cannot find unique matching interface for ens34` | The interface name in your YAML doesn't match `ip link show` | Re-run `ip link show`, fix the name in the YAML, save, re-apply. |
| YAML parse error like `mapping values are not allowed here` | Indentation is wrong in your YAML | Re-open with nano, count spaces carefully (use the cheat sheet in Step 2.2). |

**CentOS:**
```bash
sudo nmcli con mod ens160 ipv4.addresses 192.168.2.1/24 ipv4.method manual ipv4.dns 8.8.8.8
sudo nmcli con up ens160
```

> If you have NO gateway in this practice net, omit the route line. Connectivity to internet is needed only during the install of Drupal — afterwards the VM works isolated.

### Step 2.3 — Install LAMP stack + Drupal 7

```bash
# Ubuntu
sudo apt update
sudo apt install -y apache2 mariadb-server php php-gd php-mysql php-curl php-mbstring php-xml php-cli wget unzip

# Start + enable
sudo systemctl enable --now apache2 mariadb

# Secure MariaDB minimal
sudo mysql -e "CREATE DATABASE drupal; CREATE USER 'drupal'@'localhost' IDENTIFIED BY 'drupalpass'; GRANT ALL ON drupal.* TO 'drupal'@'localhost'; FLUSH PRIVILEGES;"

# Get Drupal 7 (the version with Drupalgeddon 2)
cd /var/www
sudo wget https://ftp.drupal.org/files/projects/drupal-7.57.tar.gz
sudo tar xzf drupal-7.57.tar.gz
sudo mv drupal-7.57 html-drupal
sudo cp -r html-drupal/. /var/www/html/
sudo rm /var/www/html/index.html  # remove default Apache page
sudo cp /var/www/html/sites/default/default.settings.php /var/www/html/sites/default/settings.php
sudo chown -R www-data:www-data /var/www/html
sudo chmod 666 /var/www/html/sites/default/settings.php
sudo mkdir -p /var/www/html/sites/default/files
sudo chown -R www-data:www-data /var/www/html/sites/default/files
sudo chmod -R 777 /var/www/html/sites/default/files
sudo systemctl restart apache2
```

### Step 2.4 — Run the Drupal web installer
From any browser (or `curl` from the same VM):
- Visit `http://192.168.2.1/install.php`
- *Standard* profile → Save.
- Database type: MySQL/MariaDB.
- DB name: `drupal`, user: `drupal`, password: `drupalpass`. → Save.
- Site information:
  - Site name: `Manila CMS`
  - Site email: `admin@manila.local`
  - **Site maintenance account → username `admin`, password `admin`** (deliberately weak — the "vulnerable user account" task)
  - Default country: Philippines.
  - Save.

After install completes:
```bash
# Tighten settings.php so Drupal stops complaining
sudo chmod 444 /var/www/html/sites/default/settings.php
```

### Step 2.5 — Create a deliberately weak user (the "system weakness" target)
The MA1 PDF Task 3 says *"a user account on the CMS service that exposes the system weakness... violate security policies"*. We'll create a user with a known-easy password:

In the Drupal admin UI (`/?q=admin/people/create`):
- Username: `john`
- Email: `john@manila.local`
- **Password: `password123`** (weak, in `rockyou.txt`)
- Status: Active.
- Roles: authenticated user.
- Save.

Also create the OS-level user `john` with a matching weak password (the MA1 Task 3 says "locate sensitive information stored within the vulnerable user account's home directory"):

```bash
sudo useradd -m -s /bin/bash john
echo "john:password123" | sudo chpasswd
echo "Hidden flag in john's home: flag{john_was_here_2025}" | sudo tee /home/john/secret.txt
sudo chown john:john /home/john/secret.txt
sudo chmod 600 /home/john/secret.txt
```

### Step 2.6 — Set up a privesc path (for Task 4 root access)
Plant a SUID misconfiguration that mirrors what's commonly seen in WSC-style targets:

```bash
# Make 'find' SUID — classic GTFOBins escape
sudo chmod u+s /usr/bin/find

# Or alternatively (pick ONE for practice realism — both are valid pentest paths):
# Sudo NOPASSWD vim:
echo "john ALL=(ALL) NOPASSWD: /usr/bin/vim" | sudo tee /etc/sudoers.d/john
sudo chmod 440 /etc/sudoers.d/john
```

Add a flag in `/root/`:
```bash
echo "Root flag: flag{root_compromise_complete_2025}" | sudo tee /root/proof.txt
```

### Step 2.7 — Sanity test
From the same VM browser: `http://192.168.2.1` → see Drupal home page. Login as `admin/admin` → confirm admin panel loads.

### Step 2.8 — Snapshot
Snapshot the VM as `cms-target-vulnerable`. This is the **starting state** for every MA1 dry-run.

---

## Part 3 — Build the Kali attacker VM

### Step 3.1 — Import Kali into ESXi
- Download the **Kali Linux 2025.x VMware image** from `https://www.kali.org/get-kali/#kali-virtual-machines` (need internet on PC1).
- Extract the 7z / zip — you get a folder with `.vmx` and several `.vmdk` files.
- **Convert VMware Workstation format → ESXi-compatible OVA** (one extra step since you're going to ESXi):
  1. On PC1 install **VMware OVF Tool** (free, from VMware): `https://developer.vmware.com/web/tool/4.6.0/ovf`.
  2. Open Command Prompt → `cd` to the extracted Kali folder.
  3. Convert:
     ```cmd
     "C:\Program Files\VMware\VMware OVF Tool\ovftool.exe" kali-linux-*.vmx kali-linux.ova
     ```
- Upload `kali-linux.ova` to your ESXi datastore (Storage → Datastore browser → Upload).
- ESXi UI → Virtual Machines → **Create / Register VM** → **Deploy a virtual machine from an OVF or OVA file** → select the uploaded OVA.
- Configure: **Network adapter → PG-MA1-CMS**. Memory **4 GB**. Disk thin.
- Finish.

### Step 3.2 — Set static IP `192.168.2.2`
After Kali boots:
```bash
sudo nmcli con mod "Wired connection 1" ipv4.addresses 192.168.2.2/24 ipv4.method manual
sudo nmcli con up "Wired connection 1"
```

Default credentials: `kali / kali` (matches the MA1 PDF Table 1).

### Step 3.3 — Confirm tooling is present
```bash
which nmap sqlmap hydra john hashcat searchsploit drupalgeddon2 wpscan
# All should print a path. Drupalgeddon may need install:
sudo apt update
sudo apt install -y exploitdb metasploit-framework hashcat hydra john sqlmap nikto whatweb
sudo searchsploit -u   # update local exploit-db
```

### Step 3.4 — Verify Kali can reach the target
```bash
ping -c 3 192.168.2.1
nmap -sV 192.168.2.1
```
Expected: ping replies, nmap sees ports 22 (ssh) + 80 (http).

### Step 3.5 — Stage offline references on Kali
Per CTF rules — no internet at competition. Pre-stage on Kali:
```bash
mkdir -p ~/refs
cd ~/refs
# Copy from your USB:
#   - rockyou.txt (extracted from /usr/share/wordlists/rockyou.txt.gz)
#   - linpeas.sh
#   - GTFOBins offline mirror

# Make rockyou available
sudo gunzip -k /usr/share/wordlists/rockyou.txt.gz
ls -la /usr/share/wordlists/rockyou.txt
```

### Step 3.6 — Snapshot
Snapshot Kali as `kali-clean`.

---

## Part 4 — Final verification before MA1 practice

From Kali:
- [ ] `ping 192.168.2.1` replies
- [ ] `nmap -sC -sV 192.168.2.1` shows ports 22 + 80, with HTTP banner identifying Drupal 7
- [ ] Browser → `http://192.168.2.1` → Drupal home page loads
- [ ] `john` user exists at OS level (`getent passwd john` shows it from a quick SSH attempt)
- [ ] `/root/proof.txt` exists on target (verify after privesc later)

When all green → snapshot both VMs and proceed to **`10_Day1_MA1_Solution.md`**.

---

## Snapshot inventory

| VM | Snapshot name | When |
|---|---|---|
| CMS target | `cms-target-vulnerable` | After Steps 2.6 + verification |
| Kali | `kali-clean` | After Step 3.6 |

Restore both before each dry-run so the lab state is identical.

---

## Notes for competition day

The actual competition target may be a different CMS (Joomla, WordPress, MediaWiki). The **methodology** in `10_…` is what transfers — the techniques (nmap recon, CMS version detection, exploit search, password brute, privesc) work on any CMS.

Build this practice rig once. Run the pentest playbook 3+ times until it's muscle memory. Then on competition day, regardless of which CMS shows up, you're applying a familiar workflow.

Next file: **`10_Day1_MA1_Solution.md`** — the 4-task walkthrough.
