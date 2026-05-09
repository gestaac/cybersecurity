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
- During the OS install:
  - **Hostname:** `cms-target`
  - **User:** `competitor` / **Password:** `P@ssw0rd`
  - **Root password:** `P@ssw0rd`

### Step 2.2 — Set static IP `192.168.2.1`
After OS install, log in as root:

**Ubuntu:**
```bash
sudo nano /etc/netplan/00-installer-config.yaml
```
Replace contents:
```yaml
network:
  version: 2
  ethernets:
    ens33:
      addresses: [192.168.2.1/24]
      nameservers:
        addresses: [8.8.8.8]
      routes:
        - to: default
          via: 192.168.2.254
```
```bash
sudo netplan apply
```

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
