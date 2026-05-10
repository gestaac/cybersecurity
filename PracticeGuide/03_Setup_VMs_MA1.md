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

### Step 2.2.5 — Give the VM temporary internet (REQUIRED before Step 2.3)

🚨 **Do NOT skip this step.** Your VM is currently on `PG-MA1-CMS` which has no internet — `apt update` and `wget` will both fail. You'll see:

```
W: Some index files failed to download. They have been ignored, or old ones used instead.
W: Failed to fetch http://archive.ubuntu.com/...
```

**Fix:** before running Step 2.3, follow **`09_Temporary_Internet_For_VMs.md`** to:
1. Create the `PG-TempInternet` port group (one-time, ~2 min).
2. Switch CMS-Target's NIC from `PG-MA1-CMS` to `PG-TempInternet`.
3. Switch the VM's netplan from static `192.168.2.1` to DHCP (`dhcp4: true`).
4. Verify internet works (`ping 8.8.8.8`).

When `ping 8.8.8.8` returns 3 replies, come back here and run Step 2.3.

After Step 2.3 + Step 2.4 + Step 2.5 + Step 2.6 are done (= LAMP + Drupal + weak user + privesc path), follow **`09_…` Stage F + G** to switch back:
1. Restore the static netplan (`192.168.2.1/24`, gateway `192.168.2.254`).
2. Power off the VM.
3. Switch the NIC back to `PG-MA1-CMS` in ESXi UI.
4. Power on.
5. Snapshot as `cms-target-vulnerable`.

This way the VM's "competition starting state" is sealed on the isolated practice subnet — exactly like at the actual competition.

### Step 2.3 — Install LAMP stack + Drupal 7

> ⚠️ **CRITICAL:** Drupal 7.57 (the version with Drupalgeddon 2) was released in 2018 and **does NOT work on PHP 8.x**. Ubuntu 22.04 ships PHP 8.1 by default — installing it will give you a 500 error in Step 2.4 when you try the Drupal installer. We avoid this by installing **PHP 7.4** explicitly via the Ondrej PPA. The 5 sub-steps below handle this correctly.

> 🌐 **Make sure you completed Step 2.2.5** (CMS-Target on `PG-TempInternet` with DHCP). All commands below need internet.

#### Step 2.3.1 — Install Apache + MariaDB + base packages

```bash
sudo apt update
sudo apt install -y apache2 mariadb-server wget unzip software-properties-common
sudo systemctl enable --now apache2 mariadb
```

✅ Verify: `sudo systemctl status apache2 | head -3` shows `Active: active (running)`.

#### Step 2.3.2 — Add the Ondrej PPA + install PHP 7.4 (NOT default php 8.1)

The Ondrej Sury PPA is the official source for older PHP versions on Ubuntu.

```bash
sudo add-apt-repository -y ppa:ondrej/php
sudo apt update
```

Now install PHP 7.4 + the modules Drupal needs:

```bash
sudo apt install -y \
  php7.4 \
  php7.4-gd \
  php7.4-mysql \
  php7.4-curl \
  php7.4-mbstring \
  php7.4-xml \
  php7.4-cli \
  php7.4-zip \
  libapache2-mod-php7.4
```

> ⚠️ **Watch the last package name:** `libapache2-mod-**php**7.4` (with `php` before `7.4`). NOT `libapache2-mod-7.4`. The `php` is required.

#### Step 2.3.3 — Switch Apache from PHP 8.1 to PHP 7.4

By default Apache uses PHP 8.1. We need to disable it and enable 7.4.

```bash
sudo a2dismod php8.1
sudo a2enmod php7.4
sudo systemctl restart apache2
```

✅ **Expected output:**
```
Module php8.1 disabled.
Enabling module php7.4.
```

#### Step 2.3.4 — Verify Apache is using PHP 7.4 (not 8.1)

`php -v` shows the **CLI** version, which is independent of Apache. To check what **Apache** uses, create a tiny test file:

```bash
echo '<?php phpinfo(); ?>' | sudo tee /var/www/html/phpinfo.php
sudo chown www-data:www-data /var/www/html/phpinfo.php
```

Then from **Kali's Firefox** (Kali at `192.168.2.2` should be on the same `PG-MA1-CMS` port group — temporarily on `PG-TempInternet` is also fine):

```
http://<CMS-Target-current-IP>/phpinfo.php
```

(Use whatever IP CMS-Target has now — likely `192.168.1.X` from PG-TempInternet's DHCP, or `192.168.2.1` if back on PG-MA1-CMS.)

At the very top of the page in big text:

| What you see | Meaning | Action |
|---|---|---|
| `PHP Version 7.4.X` | ✅ Apache uses PHP 7.4 — Drupal will work | Continue |
| `PHP Version 8.1.X` | ❌ Switch didn't work | Re-run `sudo a2dismod php8.1 && sudo a2enmod php7.4 && sudo systemctl restart apache2` |

When you see `7.4.X`, **delete the test file** (it leaks server details):

```bash
sudo rm /var/www/html/phpinfo.php
```

#### Step 2.3.5 — Create the MariaDB database for Drupal

```bash
sudo mysql -e "CREATE DATABASE drupal; CREATE USER 'drupal'@'localhost' IDENTIFIED BY 'drupalpass'; GRANT ALL ON drupal.* TO 'drupal'@'localhost'; FLUSH PRIVILEGES;"
```

✅ Verify: `sudo mysql -e "SHOW DATABASES;"` lists `drupal` in the output.

#### Step 2.3.6 — Download + extract Drupal 7.57

```bash
cd /var/www
sudo wget https://ftp.drupal.org/files/projects/drupal-7.57.tar.gz
sudo tar xzf drupal-7.57.tar.gz
sudo cp -r drupal-7.57/. /var/www/html/
sudo rm -f /var/www/html/index.html  # remove default Apache page
```

> 💡 The `cp -r drupal-7.57/. /var/www/html/` syntax (with the trailing `/.`) copies all files **including hidden ones** like `.htaccess` — important for Drupal's URL rewriting.

#### Step 2.3.7 — Set up Drupal's settings.php and upload folder

```bash
sudo cp /var/www/html/sites/default/default.settings.php /var/www/html/sites/default/settings.php
sudo chown -R www-data:www-data /var/www/html
sudo chmod 666 /var/www/html/sites/default/settings.php
sudo mkdir -p /var/www/html/sites/default/files
sudo chown -R www-data:www-data /var/www/html/sites/default/files
sudo chmod -R 777 /var/www/html/sites/default/files
sudo systemctl restart apache2
```

✅ Verify: `ls -la /var/www/html/sites/default/settings.php` shows the file with `-rw-rw-rw-` (666) permissions and `www-data:www-data` ownership.

#### Common Step 2.3 issues

| Error | Cause | Fix |
|---|---|---|
| `httplib2.error.ServerNotFoundError: Unable to find the server at api.launchpad.net` (during `add-apt-repository`) | No internet on the VM | Verify `ping -c 2 8.8.8.8` works. If not, switch NIC to `PG-TempInternet` + netplan to DHCP per `09_…` |
| `E: Unable to locate package libapache2-mod-7.4` | Typo — missing `php` in the package name | Use `libapache2-mod-php7.4` (with `php` before `7.4`) |
| `E: Unable to locate package php7.4` (after PPA add) | Forgot `sudo apt update` after PPA | Run `sudo apt update`, then retry the install |
| 500 Internal Server Error in Step 2.4 | Apache still using PHP 8.1 | Confirm Step 2.3.4 phpinfo shows 7.4. If not, redo Step 2.3.3 |
| `cp: cannot stat '...settings.php'` in Step 2.3.7 | Drupal didn't extract properly | Re-run Step 2.3.6 (download + tar + cp) |
| `chmod: cannot access '...settings.php'` | Settings.php not created (cp was skipped) | Run `sudo cp /var/www/html/sites/default/default.settings.php /var/www/html/sites/default/settings.php` first, then chmod |

### Step 2.4 — Run the Drupal web installer

The Drupal install wizard runs in a **browser**. Since CMS-Target is a server (no GUI), you have 3 options for the browser:

| Option | When to use |
|---|---|
| **A — Kali's Firefox** (recommended) | Once Kali is built (`192.168.2.2` on PG-MA1-CMS). Browse to `http://192.168.2.1/install.php`. |
| **B — PC1 browser** | If CMS-Target is on `PG-TempInternet` with a DHCP IP (e.g., `192.168.1.50`), browse to `http://192.168.1.50/install.php` from PC1. |
| **C — w3m text-mode browser inside CMS-Target** | If neither of the above is available. Run `sudo apt install -y w3m && w3m http://localhost/install.php` |

> 💡 **Most common path for beginners:** Option B (browse from PC1 while CMS-Target is on PG-TempInternet) — install Drupal first, THEN switch back to PG-MA1-CMS for final state.

#### Walking through the installer

Once the wizard loads, follow these screens:

1. **Choose profile:** Standard → **Save and continue**.
2. **Choose language:** English (default) → **Save and continue**.
3. **Verify requirements:** if you see green checkmarks → continue. If you see red errors about PHP missing extensions, go back to Step 2.3.2 and confirm all `php7.4-*` packages installed.
4. **Database configuration:**
   - Database type: **MySQL, MariaDB, or equivalent**
   - Database name: `drupal`
   - Database username: `drupal`
   - Database password: `drupalpass`
   - Click **Save and continue**.
5. **Install profile** runs (~30 sec progress bar).
6. **Configure site:**
   - Site name: `Manila CMS`
   - Site email address: `admin@manila.local`
   - **Site maintenance account:**
     - Username: `admin`
     - Email: `admin@manila.local`
     - Password: `admin` (deliberately weak — the "vulnerable user account" task)
     - Confirm password: `admin`
   - Default country: Philippines
   - Default time zone: Asia/Manila
   - Click **Save and continue**.
7. ✅ **"Welcome to your new Drupal site!"** — install complete.

#### After install completes:

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

### Step 2.7 — Inject Task 1 Q2 secret message into Drupal's homepage HTML

The MA1 walkthrough (`10_Day1_MA1_Solution.md` Task 1 Q2) expects a hidden HTML comment in the Drupal homepage source. We inject it into the active theme so the comment appears in every rendered page.

```bash
SECRET='<!-- secret: flag{worldskills_manila_2025} -->'
THEME=/var/www/html/themes/bartik/templates/page.tpl.php

# Backup first
sudo cp $THEME ${THEME}.bak

# Inject at line 1 (raw HTML — appears before any other output)
sudo sed -i "1i$SECRET" $THEME

# Clear Drupal's page cache so the change is immediately visible
mysql -u drupal -pdrupalpass drupal -e "DELETE FROM cache_page; DELETE FROM cache;"

# Optional: clear filesystem CSS/JS caches
sudo rm -rf /var/www/html/sites/default/files/css/*
sudo rm -rf /var/www/html/sites/default/files/js/*
```

✅ Verify from Kali (or the same VM):
```bash
curl -s http://192.168.2.1/ | grep -i "secret"
# Expected: <!-- secret: flag{worldskills_manila_2025} -->
```

> 🧠 **Why the Bartik theme path?** Bartik is Drupal 7's default active theme. Its `page.tpl.php` is the top-level template rendered on every page. Adding raw HTML at line 1 outputs it before any PHP `header()` calls, so it appears in every page's source without breaking Drupal.

> If the verify command returns nothing, try `sudo systemctl restart apache2` then re-test. If `mysql` says command not found, try `mariadb` instead.

> **Rollback (if you need to undo):**
> ```bash
> sudo cp /var/www/html/themes/bartik/templates/page.tpl.php.bak \
>         /var/www/html/themes/bartik/templates/page.tpl.php
> mysql -u drupal -pdrupalpass drupal -e "DELETE FROM cache_page; DELETE FROM cache;"
> ```

### Step 2.8 — Sanity test
From the same VM browser: `http://192.168.2.1` → see Drupal home page. Login as `admin/admin` → confirm admin panel loads. Also `curl -s http://192.168.2.1/ | head -3` should show the injected HTML secret comment as the first line.

### Step 2.9 — Snapshot
Snapshot the VM as `cms-target-vulnerable`. This is the **starting state** for every MA1 dry-run.

---

## Part 3 — Build the Kali attacker VM (from ISO)

> 💡 **Why ISO instead of OVA?** Same install pattern as the CMS-Target (you already know it — boot ISO → walk through installer). No OVF Tool, no `.vmx` conversion, no extra steps on PC1. Trade-off: Kali ISO install takes ~30 min vs OVA deploy at ~5 min — but it's more beginner-friendly and predictable.

> 🌐 **Internet during Kali install — strongly recommended.**
> The Kali Installer ISO (~4 GB) bundles all default tools, so install **works offline**. But Kali also tries to:
> - Configure a package manager mirror (faster with internet).
> - Download security updates during install.
> - Pull the latest tool definitions.
>
> **Recommendation:** before powering on the Kali VM for first install, switch its NIC to **`PG-TempInternet`** (per `09_Temporary_Internet_For_VMs.md` Stage A). After install completes + you've set the static IP in Step 3.2, switch the NIC back to `PG-MA1-CMS`.
>
> Doing it offline is ~30% slower on the mirror config step, otherwise fine.

### Step 3.0 — Download the Kali ISO (do this on PC1)

#### Download
- **URL:** `https://www.kali.org/get-kali/#kali-installer-images`
- **Click:** the **Installer** tile (NOT "Live Boot" or "Net Installer" or "Virtual Machines")
- **File:** `kali-linux-2025.x-installer-amd64.iso` (~4 GB)
- **Save to:** PC1 → `D:\ISO\` folder

> 💡 **Why "Installer" not "Net Installer"?** The Net Installer is small (~500 MB) but downloads all packages during install (needs fast internet). The full Installer (~4 GB) bundles everything and works reliably even if internet is slow.

#### Upload to ESXi datastore
1. ESXi UI → **Storage** → click your datastore → **Datastore browser**.
2. Click into the `ISO/` folder you created earlier.
3. Click **Upload** → select `kali-linux-2025.x-installer-amd64.iso` from PC1.
4. Wait ~10 min for the upload.

### Step 3.1 — Create the VM on ESXi

ESXi UI → Virtual Machines → **Create / Register VM** → **Create a new virtual machine** → Next.

#### Screen 2 — Name and Guest OS
| Field | Value |
|---|---|
| **Name** | `Kali` |
| **Compatibility** | ESXi 8.0 virtual machine |
| **Guest OS family** | Linux |
| **Guest OS version** | Debian GNU/Linux 12 (64-bit) ← Kali is based on Debian 12 |

Click Next.

#### Screen 3 — Storage
Pick your default datastore → Next.

#### Screen 4 — Customize hardware
| Field | Value |
|---|---|
| **CPU** | 2 |
| **Memory** | `4096` MB (4 GB) |
| **Hard disk 1** | `80` GB, **Thin Provisioned** |
| **Network Adapter 1** | `PG-MA1-CMS` |
| **CD/DVD Drive 1** | Datastore ISO file → `kali-linux-2025.x-installer-amd64.iso` → ✅ Connect at power on |

Click Next.

#### Screen 5 — Ready to complete
Review → **Finish**.

Power on the VM → click **Console**.

### Step 3.1.1 — Walk through the Kali graphical installer (~30 min)

> Beginner notes: the Kali installer has both **Graphical Install** (mouse + keyboard) and **Install** (text-mode, faster). Pick **Graphical Install** — easier for beginners.

**Screen 1 — Boot menu**

You see a Kali boot menu with several options:
```
Kali GNU/Linux Installer Boot Menu
─────────────────────────────────
  Live system (amd64)
  Live system (amd64 forensic mode)
  Install
  ► Graphical install
  Advanced options
  Help
  Boot from first hard disk
```

- Highlight **`Graphical install`** with arrow keys.
- Press **Enter**.

**Screen 2 — Select a language**

- Pick **English** → **Continue**.

**Screen 3 — Select your location**

- Click **other** → **Asia** → **Philippines** → **Continue**.

**Screen 4 — Configure locales**

- Pick **United States — en_US.UTF-8** (default). Continue.

**Screen 5 — Configure the keyboard**

- Pick **American English** (default). Continue.

**Screen 6 — Loading installer components** (auto, ~30 sec wait)

The installer pulls additional components from the ISO. No interaction.

**Screen 7 — Configure the network — Hostname**

- **Hostname:** `kali`
- Click **Continue**.

**Screen 8 — Domain name**

- Leave blank.
- Click **Continue**.

**Screen 9 — Set up users and passwords — Full name**

- **Full name for the new user:** `Kali User`
- Click **Continue**.

**Screen 10 — Username for your account**

- **Username:** `kali`
- Click **Continue**.

**Screen 11 — Password for the new user**

- **Choose a password:** `kali`
- **Re-enter password to verify:** `kali`
- Click **Continue**.

> 💡 The MA1 PDF Table 1 specifies Kali credentials as `kali / kali`. Use exactly that.

**Screen 12 — Configure the clock**

- **Timezone:** Asia/Manila (or whichever city is closest)
- Click **Continue**.

**Screen 13 — Partition disks (method)**

- Pick **Guided - use entire disk** (default).
- Click **Continue**.

**Screen 14 — Select disk to partition**

- You see one disk: `SCSI3 (0,0,0) (sda) - 80 GB VMware Virtual disk`.
- Highlight it → **Continue**.

**Screen 15 — Partitioning scheme**

- Pick **All files in one partition (recommended for new users)**.
- Click **Continue**.

**Screen 16 — Review partitioning**

- You see the proposed partition layout (1× ext4 root, 1× swap).
- Highlight **Finish partitioning and write changes to disk** → **Continue**.

**Screen 17 — Confirm write changes**

- "Write the changes to disks?" → ✅ **Yes** → **Continue**.

**Screen 18 — Installing the base system** (5 min wait)

Progress bar fills. No interaction.

**Screen 19 — Software selection**

You see checkboxes for desktop + tools:
```
Choose software to install:
  ☑ Xfce (Kali's default desktop environment)
  ☐ GNOME
  ☐ KDE Plasma
  ─────────
  ☑ Collection of tools — top10 — the 10 most popular tools
  ☐ Collection of tools — default — recommended tools (default)
  ☐ Collection of tools — large — default selection PLUS additional tools
  ☐ Collection of tools — everything
  ─────────
  ☐ Standard system utilities
```

For our use, recommended:
- ✅ **Xfce** — keep default. Lightweight, fast.
- ✅ **Collection of tools — default** — has nmap, sqlmap, hydra, john, hashcat, Burp, Wireshark, Metasploit, gobuster, ffuf, etc. (**Untick "top10" if you tick "default"** — they overlap.)
- ✅ **Standard system utilities** — keep default.

> 💡 **Don't pick `everything`** — that's ~30 GB of tools, takes 1+ hour to install. **default** has everything we need (~15 GB).

Click **Continue**.

**Screen 20 — Installing software** (15–20 min wait)

Big wait. Get coffee.

**Screen 21 — Install the GRUB boot loader**

- "Install the GRUB boot loader to your primary drive?" → ✅ **Yes** → **Continue**.

**Screen 22 — Device for boot loader installation**

- Highlight `/dev/sda` → **Continue**.

**Screen 23 — Finish the installation**

- "Installation complete." → **Continue**.

> ⚠️ **EJECT THE ISO before reboot completes:**
> 1. Don't close the console.
> 2. Switch to ESXi UI → Kali VM → **Edit**.
> 3. **CD/DVD Drive 1** → uncheck "Connect at power on" → dropdown to **`Host device`**.
> 4. Click **Save**.
> 5. Switch back to console → wait for reboot.

**Screen 24 — GRUB after reboot**

- 5-second countdown → boots Kali GNU/Linux.

**Screen 25 — Login screen (graphical)**

After boot you see a graphical login screen with a Kali dragon background.

- Username: `kali`
- Password: `kali`
- Click **Log In**.

You land in the Xfce desktop. Top bar has a Kali menu, terminal icon, file manager, browser.

✅ **Kali is installed.** Continue with Step 3.2.

#### Common Kali installer issues

| Problem | Fix |
|---|---|
| Boot menu doesn't appear / black screen | Wait 60 sec — early boot is silent. If still black, reset VM. |
| Mouse pointer stuck or jumpy | Click inside the VM console window first. Press Ctrl+Alt to release. |
| Software install hangs at >30 min on one package | Some packages have post-install scripts that take a while. Wait 5 more min before resetting. |
| GRUB install fails | Pick `/dev/sda` explicitly (not "Use entire disk" radio button if it appeared). |
| After reboot, boots installer again | ISO not ejected. Edit VM → CD/DVD → uncheck Connect at power on → reset VM. |
| First desktop login takes 1+ minute | Xfce first-run is slow. Subsequent logins are fast (~10 sec). |

### Step 3.2 — Set static IP `192.168.2.2`

Kali uses **NetworkManager** by default with a GUI applet — easiest for beginners.

#### Step 3.2.1 — Find your interface name

Open a terminal (top bar → Terminal Emulator icon, or right-click desktop → Open Terminal):

```bash
ip link show
```

Look for the line that's NOT `lo` — your interface might be `eth0`, `ens34`, `ens160`, etc. **Note your name** (the example below uses `eth0`).

#### Step 3.2.2 — Set the static IP via NetworkManager GUI

1. Top-right of the screen → click the **network icon** (looks like 2 stacked arrows or a wired plug).
2. Click **Edit Connections...**
3. You see a list of saved connections. Highlight **Wired connection 1** (or the one matching your interface name).
4. Click the **gear icon** (Edit) at the bottom.
5. In the dialog, click the **IPv4 Settings** tab.
6. Method drop-down → change from `Automatic (DHCP)` to **`Manual`**.
7. Click **Add** next to the (empty) addresses table.
8. Fill in:
   - **Address:** `192.168.2.2`
   - **Netmask:** `24` (or `255.255.255.0`)
   - **Gateway:** `192.168.2.254` (leave empty if you don't need internet through this NIC)
9. **DNS servers:** `8.8.8.8`
10. Click **Save**.
11. Close the Connections window.
12. Top-right network icon → click your connection → click **Disconnect**, then click it again to **Connect**. (This re-applies the new IP.)

#### Step 3.2.3 — Set static IP via terminal (alternative if GUI fails)

```bash
sudo nmcli con mod "Wired connection 1" \
    ipv4.addresses 192.168.2.2/24 \
    ipv4.gateway 192.168.2.254 \
    ipv4.dns 8.8.8.8 \
    ipv4.method manual

sudo nmcli con up "Wired connection 1"
```

> 💡 If your connection name isn't "Wired connection 1", run `nmcli con show` to find the actual name and substitute it.

#### Step 3.2.4 — Verify

```bash
ip a
```

Look for the interface line showing `inet 192.168.2.2/24`. ✅

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
