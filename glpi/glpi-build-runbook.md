# GLPI Ticketing System Build Runbook

**System:** GLPI-01, Proxmox VMID 201
**Host:** Dell PowerEdge R320 running Proxmox VE
**Application:** GLPI 11.0.8 on Debian 13
**Built:** September 2026

Sensitive data within this document is obfuscated. Public IP addresses and Internet facing service listening port information, user account names and any keys or credentials are displayed using asterisks. Private addressing for internal networks are displayed as they appear, due to RFC 1918 space being non-routable from external locations, and this makes the documentation easier to follow.

## Purpose

This runbook documents how the GLPI IT service management system was built on the homelab so that issues and fixes across the lab can be tracked as tickets, with screenshots and documentation attached. It is written so another technician could rebuild the same system from scratch.

GLPI was chosen because it combines ticket management with an asset database and a knowledge base, and the self-hosted community edition is free and open source.

## Final State

VM 201 runs Debian 13 with Apache, MariaDB 11.8, and PHP 8.4. GLPI 11.0.8 is served over HTTPS only at https://10.10.50.201 on the SERVERS VLAN, and is reachable in a browser from both the desktop and the laptop. Program files, configuration, data, logs, and plugins are stored in separate directories following the layout recommended in the GLPI installation documentation. Automatic actions run every minute through cron in CLI mode. The default GLPI accounts are disabled, and a dedicated super-admin account is used instead.

## Environment

Hypervisor: Proxmox VE on a Dell PowerEdge R320, node name masked as ********

Proxmox host address: 10.10.50.10

VM storage: local-zfs for VM disks, local at /var/lib/vz for ISO images

Network: bridge vmbr0 with no VLAN tag, which places the VM on VLAN 50, 10.10.50.0/24, the same as the existing course VM 102

Firewall and DHCP: pfSense, VLAN 50 DHCP pool 10.10.50.100 to 10.10.50.200

## Phase 1: Create the VM from the Proxmox Host CLI

All steps in this phase run in the Proxmox host shell. No files were transferred from a workstation to the server; the installer ISO was downloaded directly on the host.

### 1.1 Confirm the VMID is free

VMID 201 was chosen to place the VM in the 200 range alongside the Jellyfin VM at VMID 200.

   qm status 201

The expected result is an error stating that the configuration file for 201 does not exist.

### 1.2 Confirm where ISO images are stored

   grep -A4 "^dir: local" /etc/pve/storage.cfg

   ls /var/lib/vz/template/iso

The local storage showed path /var/lib/vz with iso listed under content, so ISO images belong in /var/lib/vz/template/iso.

### 1.3 Download and verify the Debian netinst ISO

The netinst filename changes with each Debian point release, so the filename is read from Debian's official checksum file instead of being typed by hand.

   cd /var/lib/vz/template/iso

   wget https://cdimage.debian.org/debian-cd/current/amd64/iso-cd/SHA512SUMS

   ISO=$(grep -oE 'debian-[0-9.]+-amd64-netinst\\.iso' SHA512SUMS)

   echo $ISO

   wget https://cdimage.debian.org/debian-cd/current/amd64/iso-cd/$ISO

   sha512sum -c --ignore-missing SHA512SUMS

The checksum line must end in OK before the ISO is used. Keep the same shell open for the next step, because the qm create command uses the $ISO variable.

### 1.4 Create VM 201

The hardware settings match the existing course VM 102: same CPU type, VirtIO SCSI single controller, iothread on the disk, and a VirtIO NIC on vmbr0 with the Proxmox firewall enabled. The QEMU guest agent is enabled.

   qm create 201 --name GLPI-01 --ostype l26 --memory 4096 --cores 2 --cpu x86-64-v2-AES --scsihw virtio-scsi-single --scsi0 local-zfs:32,iothread=1 --net0 virtio,bridge=vmbr0,firewall=1 --ide2 local:iso/$ISO,media=cdrom --boot order='scsi0;ide2' --agent 1

   qm config 201

The boot order lists the disk first. While the disk is empty it is not bootable, so the VM falls through to the ISO. After installation the VM boots from the disk automatically.

### 1.5 Run the Debian installer

   qm start 201

The installer runs in the VM 201 Console tab in the Proxmox web interface. Settings used:

Install type: text mode Install

Hostname: GLPI

Domain name: the local domain configured in pfSense under System then General Setup, or blank

Accounts: a root password and one administrative user, masked as ********

Partitioning: Guided, use entire disk, all files in one partition. Encrypted LVM was not used, because it requires a passphrase at the console on every boot and GLPI would stay offline after any reboot until it was entered.

Software selection: SSH server and standard system utilities only. All desktop environments were unchecked.

Boot loader: GRUB installed to /dev/sda, the VM's only disk, scsi0

### 1.6 Eject the ISO

   qm set 201 --ide2 none,media=cdrom

Confirm with qm config 201 that the ide2 line shows none,media=cdrom.

## Phase 2: Base Operating System Setup

These commands were run in a root shell on the VM. Each apt command is separate; apt update takes no arguments and cannot be combined with full-upgrade on one line.

   apt update

   apt full-upgrade -y

   apt install -y sudo wget qemu-guest-agent

   usermod -aG sudo ********

Debian does not install sudo when a root password is set during installation, which is why it is installed here. Group membership takes effect at the next login, so the administrative user must log out and back in before sudo works. sudo prompts for the user's own password, not the root password.

   systemctl status qemu-guest-agent --no-pager

The guest agent should report active, running.

## Phase 3: Assign a Static Address in pfSense

The VM first received 10.10.50.105, a dynamic lease from the VLAN 50 pool. A static DHCP mapping was added so the address never changes.

1. Services then DHCP Server then the VLAN 50 interface: note the pool range, 10.10.50.100 to 10.10.50.200.
2. Status then DHCP Leases: find the lease for VM 201 by its MAC address and select Add static mapping, which fills in the MAC address and hostname.
3. Enter an IP address outside the pool and save.

pfSense only accepts static mappings outside the DHCP pool, because a mapping inside the pool does not remove the address from the pool, and another device could be given it while the VM is offline. The address 10.10.50.201 was chosen to match the VMID. Before saving, the address was checked for conflicts with ping from the Proxmox host and in pfSense under Diagnostics then ARP Table.

   ping -c 3 10.10.50.201

After a reboot, the VM came up on the new address.

   ip -br addr

The interface ens18 showed 10.10.50.201/24. From this point on, the VM was managed over SSH from the desktop.

## Phase 4: Install the Web Stack

   sudo apt install -y apache2 mariadb-server php libapache2-mod-php php-mysql php-curl php-gd php-intl php-xml php-mbstring php-bcmath php-zip php-bz2

   php -v

   mariadb --version

   ls /etc/php/

Installed versions: PHP 8.4.24 with Zend OPcache loaded, and MariaDB 11.8.6. The PHP configuration folder is /etc/php/8.4. GLPI 11 requires at least PHP 8.2 and MariaDB 10.6, so both versions meet the requirements.

## Phase 5: Database

### 5.1 Secure MariaDB

   sudo mariadb-secure-installation

Answers: press Enter for the current root password, no to switching to unix_socket authentication since the Debian root account already uses it, no to changing the root password, and yes to removing anonymous users, disallowing remote root login, removing the test database, and reloading privilege tables.

### 5.2 Load timezone data

GLPI's timezone support requires the timezone tables to be loaded, followed by a restart of the database server.

   sudo mariadb-tzinfo-to-sql /usr/share/zoneinfo | sudo mariadb -u root mysql

   sudo systemctl restart mariadb

### 5.3 Create the GLPI database and user

   sudo mariadb -u root

At the MariaDB prompt:

   CREATE DATABASE glpi CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

   CREATE USER 'glpi'@'localhost' IDENTIFIED BY '********';

   GRANT ALL PRIVILEGES ON glpi.\* TO 'glpi'@'localhost';

   FLUSH PRIVILEGES;

   EXIT;

### 5.4 Verify

   sudo mariadb -u root -e "SELECT COUNT(\*) FROM mysql.time_zone_name;"

   mariadb -u glpi -p -e "SHOW DATABASES;"

The first command returned 486 timezone names. The second, after entering the glpi password, listed glpi and information_schema, confirming the user can log in and reach its database.

## Phase 6: Download GLPI and Set Up the Directory Layout

### 6.1 Download and extract

   cd /tmp

   wget https://github.com/glpi-project/glpi/releases/download/11.0.8/glpi-11.0.8.tgz

   sudo tar -xzf glpi-11.0.8.tgz -C /var/www/

   ls /var/www/glpi

### 6.2 Separate configuration, data, logs, and plugins

The layout follows the GLPI installation documentation: program files in /var/www/glpi, configuration in /etc/glpi, data in /var/lib/glpi/files, logs in /var/log/glpi, and plugins in /var/lib/glpi/plugins. Only the public folder is served to browsers, and /var/www/glpi stays owned by root.

   sudo mkdir -p /etc/glpi /var/lib/glpi /var/log/glpi /var/lib/glpi/plugins

   sudo cp -a /var/www/glpi/config/. /etc/glpi/

   sudo cp -a /var/www/glpi/files /var/lib/glpi/

   sudo cp -a /var/www/glpi/marketplace/. /var/lib/glpi/plugins/

   sudo cp -a /var/www/glpi/plugins/. /var/lib/glpi/plugins/

### 6.3 Point GLPI to the new locations

Create /var/www/glpi/inc/downstream.php with this content. The release archive does not include this file, so nothing is overwritten.

   \<?php

   define('GLPI_CONFIG_DIR', '/etc/glpi/');

   if (file_exists(GLPI_CONFIG_DIR . '/local_define.php')) {

      require_once GLPI_CONFIG_DIR . '/local_define.php';

   }

Create /etc/glpi/local_define.php with this content:

   \<?php

   define('GLPI_VAR_DIR', '/var/lib/glpi/files');

   define('GLPI_LOG_DIR', '/var/log/glpi');

   define('GLPI_MARKETPLACE_DIR', '/var/lib/glpi/plugins');

### 6.4 Permissions

Apache runs as www-data, and GLPI needs read and write access to its configuration, data, log, and plugin directories.

   sudo chown -R www-data:www-data /etc/glpi /var/lib/glpi /var/log/glpi

   ls -ld /etc/glpi /var/lib/glpi /var/lib/glpi/files /var/log/glpi /var/lib/glpi/plugins

All directories should show www-data www-data as owner.

## Phase 7: HTTPS and Apache

### 7.1 Self-signed certificate

The certificate is issued to the IP address itself through the subject alternative name, which browsers require before they will trust it.

   sudo openssl req -x509 -nodes -days 825 -newkey rsa:2048 -keyout /etc/ssl/private/glpi.key -out /etc/ssl/certs/glpi.crt -subj "/CN=10.10.50.201" -addext "subjectAltName=IP:10.10.50.201"

### 7.2 Apache site

Create /etc/apache2/sites-available/glpi.conf with this content. The port 80 virtual host redirects to HTTPS. The port 443 virtual host is based on the Apache configuration in the GLPI documentation: only the public folder is served, and every request goes through the GLPI router.

   \<VirtualHost \*:80>

      Redirect permanent / https://10.10.50.201/

   \</VirtualHost>

   \<VirtualHost \*:443>

      DocumentRoot /var/www/glpi/public

      SSLEngine on

      SSLCertificateFile /etc/ssl/certs/glpi.crt

      SSLCertificateKeyFile /etc/ssl/private/glpi.key

      \<Directory /var/www/glpi/public>

         Require all granted

         RewriteEngine On

         RewriteCond %{HTTP:Authorization} ^(.+)$

         RewriteRule .\* - [E=HTTP_AUTHORIZATION:%{HTTP:Authorization}]

         RewriteCond %{REQUEST_FILENAME} !-f

         RewriteRule ^(.\*)$ index.php [QSA,L]

      \</Directory>

   \</VirtualHost>

### 7.3 PHP settings

Create /etc/php/8.4/apache2/conf.d/99-glpi.ini with the lines below. The session cookie settings are the ones recommended by the GLPI documentation. session.cookie_secure is safe to enable because the site is HTTPS only.

   session.cookie_secure = on

   session.cookie_httponly = on

   session.cookie_samesite = Lax

   date.timezone = America/Chicago

   upload_max_filesize = 200M

   post_max_size = 250M

Create /etc/php/8.4/cli/conf.d/99-glpi.ini with the first four lines only. The timezone is set for both the web server and the command line, because a mismatch between the two can cause automatic actions to run at the wrong times. The upload limits were added later; see Phase 11.

### 7.4 Enable the site

   sudo a2enmod rewrite ssl

   sudo a2dissite 000-default

   sudo a2ensite glpi

   sudo apache2ctl configtest

   sudo systemctl restart apache2

### 7.5 Verify

   sudo systemctl status apache2 --no-pager | head -5

   php -i | grep -E "date.timezone|session.cookie_(secure|httponly|samesite)"

Apache reported active, running. PHP reported America/Chicago, cookie_secure On, cookie_httponly On, and cookie_samesite Lax.

## Phase 8: Snapshot, Requirements Check, and Installation

### 8.1 Snapshot before installing

On the Proxmox host:

   qm snapshot 201 GLPI_PreInstall --description "Debian 13, Apache, MariaDB, PHP 8.4 configured. GLPI files in place, before install."

   qm listsnapshot 201

### 8.2 Requirements check

The check runs as www-data so it tests the same permissions GLPI will use.

   sudo -u www-data php /var/www/glpi/bin/console glpi:system:check_requirements

All required items passed. Non-passing items and their disposition:

SELinux configuration: skipped, because Debian does not use SELinux.

Security configuration for sessions: skipped on the command line because it cannot test browser cookies. The browser installer's check later confirmed it.

ldap extension: not installed. It is only needed for authentication against an LDAP server such as Active Directory, which this single-user system does not use.

Permissions for marketplace directory: GLPI could not write to /var/www/glpi/marketplace. This was resolved by moving plugin storage to /var/lib/glpi/plugins with the GLPI_MARKETPLACE_DIR setting, included in Phase 6, which keeps the program folder read-only for the web server. The check then reported OK.

### 8.3 Installation wizard

In a browser, go to https://10.10.50.201 and accept the certificate warning. The warning is removed in Phase 12.

1. Language: English.
2. License: accept.
3. Select Install.
4. Environment checks: all passed except the optional ldap extension. The session security check passed in the browser.
5. Database connection: SQL server localhost, SQL user glpi, and the glpi password.
6. Database: select the existing glpi database. Do not create a new one.
7. Allow initialization to finish, choose a telemetry preference, and continue to the login page.

## Phase 9: Secure the Default Accounts

GLPI creates four default accounts, each with a password equal to its username: glpi, tech, normal, and post-only. The GLPI home page shows a warning listing any defaults that still use their original passwords.

1. Log in once as glpi.
2. Administration then Users then Add: create a new user, masked as ********, with a strong password, profile Super-Admin, entity Root entity, recursive Yes.
3. Log out and log back in as the new user. Confirm the Administration and Setup menus are present before continuing.
4. Open each of the four default accounts and set Active to No.
5. Confirm the default password warning no longer appears on the home page.

## Phase 10: Automatic Actions in CLI Mode

By default, GLPI automatic actions only run when someone loads a page. CLI mode runs them from the server's scheduler every minute instead, which is the mode the GLPI documentation recommends.

   systemctl status cron --no-pager

   sudo -u www-data /usr/bin/php /var/www/glpi/front/cron.php

The manual run should finish without output. Then create the system cron file:

   echo "\* \* \* \* \* www-data /usr/bin/php /var/www/glpi/front/cron.php" | sudo tee /etc/cron.d/glpi

In GLPI: Setup then Automatic actions, show all rows on one page, select all, Actions then Update, choose the Run Mode field, set the value to CLI, and select Post.

Verification: the cron log showed cron.php running as www-data once per minute, and the Last run column in Setup then Automatic actions updated with current times.

## Phase 11: Upload Size Limits

The installer sets GLPI's document size limit from PHP's upload_max_filesize, which defaults to 2 MB. A full screen screenshot on a high resolution monitor can exceed that, so the limits were raised.

1. Add upload_max_filesize = 200M and post_max_size = 250M to /etc/php/8.4/apache2/conf.d/99-glpi.ini. Final file contents are in Phase 7.3. post_max_size is set higher than upload_max_filesize because a form submission contains the file plus the other form fields.
2. Restart Apache.

   sudo systemctl restart apache2

3. Verify the values the web server's PHP actually uses:

   PHP_INI_SCAN_DIR=/etc/php/8.4/apache2/conf.d php -c /etc/php/8.4/apache2/php.ini -i | grep -E "upload_max_filesize|post_max_size"

4. In GLPI: Setup then General then the Management tab, set Document files maximum size in MB to 100 and save. GLPI's setting can only lower the PHP limit, never raise it, so 100 MB is the effective limit.

## Phase 12: Trust the Certificate on Client PCs

The certificate is public; the private key never leaves the VM. To avoid transferring files to or from the server, the certificate was displayed in the terminal and copied.

   cat /etc/ssl/certs/glpi.crt

1. Copy everything from the BEGIN CERTIFICATE line through the END CERTIFICATE line, including both of those lines, into Notepad.
2. Save as glpi.crt with Save as type set to All Files, so the file is not saved as glpi.crt.txt.
3. Double-click glpi.crt, select Install Certificate, choose Local Machine, approve the administrator prompt, choose Place all certificates in the following store, and select Trusted Root Certification Authorities.
4. Finish, then fully close and reopen the browser.

Alternative from an elevated PowerShell prompt in the folder containing the file:

   certutil -addstore -f Root glpi.crt

Result: https://10.10.50.201 loads with a padlock and no warning on the desktop.

## Phase 13: Test Data Cleanup

Test tickets with screenshot attachments were created to confirm that attachments save correctly, then removed so the first real ticket starts at number 1.

### 13.1 Delete the test tickets and documents

Assistance then Tickets: select the tickets, Actions then Put in trashbin, then open the trash bin view, select them again, and choose Delete permanently. Repeat the same two steps in Management then Documents for the attached screenshots.

### 13.2 Reset the counters

GLPI never reuses ticket or document numbers. The counters are MariaDB AUTO_INCREMENT values and can only be reset when the table is empty; MariaDB ignores a lower value while higher IDs still exist, so the reset cannot overwrite existing records.

   sudo mariadb -u root glpi -e "SELECT COUNT(\*) FROM glpi_tickets;"

   sudo mariadb -u root glpi -e "ALTER TABLE glpi_tickets AUTO_INCREMENT = 1;"

   sudo mariadb -u root glpi -e "SELECT AUTO_INCREMENT FROM information_schema.TABLES WHERE TABLE_SCHEMA='glpi' AND TABLE_NAME='glpi_tickets';"

The same three commands were run with glpi_documents in place of glpi_tickets. Both counts were 0 before the reset, and both counters returned 1 afterward.

### 13.3 Staging files

Screenshot copies remained in /var/lib/glpi/files/_tmp. This is GLPI's upload staging folder, not document storage. The Clean temporary files automatic action deletes anything there older than one hour on an hourly schedule. They were cleared immediately instead:

   sudo find /var/lib/glpi/files/_tmp -type f ! -name ".gitkeep" -delete

## Phase 14: ITIL Categories

Twenty-two categories were created in one pass through the GLPI Legacy REST API with a bash script, instead of entering them one at a time in the web interface. The API was used instead of inserting rows directly with SQL because GLPI stores calculated fields for each category in the tree, including full path name, depth level, and parent and child caches, and the API has GLPI fill those in correctly.

Category tree:

Network: Firewall and Routing, Switching, Wireless, VPN

Virtualization: Proxmox Host, Virtual Machines

Media Services: Jellyfin

Telephony (VoIP): IP PBX, IP Phones

End Devices: Desktop, Laptop

Coursework: CVNP 1601 Linux Administration, CVNP 1606 Supporting Windows Operating Systems, CVNP 1612 Cisco 2, CVNP 2601 Virtual Computing, CVNP 2615 Security Fundamentals

Naming notes: the VoIP phone system is an IP PBX, a private telephone system that carries calls over IP, with IP phones as its endpoints. End Devices follows Cisco's CCNA terminology for hosts where communication starts or ends, as opposed to intermediary devices such as switches and routers.

Procedure:

1. Setup then General then the API tab: turn on Enable Legacy REST API, confirm, and Save. Confirm Enable login with external token is on. The default API client, full access from localhost, already permits requests from the VM itself.
2. My settings then Passwords and access keys: regenerate the API token and save.
3. Install the script's dependencies:

   sudo apt install -y curl jq

4. Save the category script to the VM, replace the token placeholder on the UTOKEN line with the API token, and run it:

   bash ~/glpi-categories.sh

Result: all 22 categories were created with IDs 1 through 22, and the script closed its API session.

Cleanup, because the token was stored in the script file and the shell history:

   rm ~/glpi-categories.sh

   history -c

Then regenerate the API token again in My settings, and turn off Enable Legacy REST API in Setup then General then API. The categories are stored in the database and are not affected by disabling the API.

## Snapshots

GLPI_PreInstall: Debian 13, Apache, MariaDB, and PHP 8.4 configured, GLPI files in place, before the installation wizard. Taken before the marketplace directory fix; after a rollback to this snapshot, repeat the marketplace steps from Phase 6.

GLPI_CleanInstall: GLPI 11.0.8 installed, default accounts disabled, cron in CLI mode, HTTPS trusted, 100 MB upload limit. This is the restore point for the working system.

## Issues Encountered

### apt commands combined on one line

Symptom: apt update full-upgrade returned "The update command takes no arguments," and variations with a space inside full-upgrade returned option errors.

Cause: update and full-upgrade are separate apt commands, and each must start with apt on its own line.

Fix: run apt update, then apt full-upgrade -y as separate commands.

### sudo denied after adding the user to the sudo group

Symptom: the administrative user could not use sudo immediately after usermod -aG sudo.

Cause: group membership only applies to new login sessions, and sudo requires the user's own password rather than the root password.

Fix: confirmed sudo was installed with dpkg -l sudo, the user was in the sudo group with id, and the %sudo rule was present in /etc/sudoers, then logged out and back in. sudo whoami returned root.

### Marketplace directory not writable

Symptom: the requirements check reported that the directory could not be created in /var/www/glpi/marketplace.

Cause: /var/www/glpi is owned by root, which is intended, so GLPI cannot write plugins there.

Fix: set GLPI_MARKETPLACE_DIR to /var/lib/glpi/plugins, copied the marketplace and plugins contents there, and gave www-data ownership.

### "The action you have requested is not allowed" when saving settings

Symptom: saving the document size setting in GLPI returned "The action you have requested is not allowed."

Cause: a typo set post_max_size to 25M0 instead of 250M. When a request exceeds post_max_size, PHP discards all submitted form data, including GLPI's security token, so GLPI rejected the request.

Fix: rewrote /etc/php/8.4/apache2/conf.d/99-glpi.ini with the correct values, restarted Apache, and verified the values with php -i against the Apache configuration. The setting then saved.

### Category script: curl and jq missing

Symptom: the script reported "curl: command not found" and "jq: command not found," followed by Login failed.

Cause: the Debian standard system utilities do not include curl or jq.

Fix: apt install -y curl jq.

### Category script: Login failed with a valid token

Symptom: after installing curl and jq, the script still reported Login failed.

Diagnosis: a direct request to the initSession endpoint returned an API Disabled error with HTTP 400.

Cause: the Legacy REST API setting had not been saved as enabled.

Fix: enabled the Legacy REST API in Setup then General then API and saved. The same request then returned HTTP 200, and the script completed.

## Open Items

The certificate still needs to be installed on the laptop using the Phase 12 procedure.

From outside the home network, the laptop reaches GLPI only through the WireGuard tunnel, and the external WireGuard handshake has not yet been verified. See ../remote-access/remote-access-wireguard.md.

Inline image insertion in ticket followups through the editor's image button has not yet been tested. Attachments through the Files area were confirmed to save with tickets.

Drafted with AI assistance from my own build work and notes. The procedures, technical decisions, and verification steps reflect the actual build. See the AI disclosure in the GLPI README for details on how AI was used during this project.
