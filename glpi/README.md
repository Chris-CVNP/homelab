# GLPI: IT Service Management for the Homelab

**Reference:** HOMELAB-GLPI-001
**Scope:** Steelbyte homelab, ticketing and asset management
**System:** GLPI-01, Proxmox VMID 201, 10.10.50.201
**Built:** September 2026

Sensitive data within this document is obfuscated. Public IP addresses and Internet facing service listening port information, user account names and any keys or credentials are displayed using asterisks. Private addressing for internal networks is displayed as it appears, due to RFC 1918 space being non-routable from external locations, and this makes the documentation easier to follow.

---

## Objectives

- Track issues and fixes across the lab as tickets rather than in scattered notes
- Attach screenshots and documentation to the work they belong to
- Maintain an asset inventory of the machines, VMs, and network devices in the environment
- Build a knowledge base so a fix found once does not have to be rediscovered
- Run the whole thing on the lab itself rather than on a hosted service
- Build it entirely from the server side, with no files moved from a workstation

---

## Tools Used

- `qm status`, `qm create`, `qm set`, `qm config`, `qm snapshot`, `qm listsnapshot` (Proxmox host, VM lifecycle)
- `wget` and `sha512sum` (ISO download and integrity verification)
- `apt` (package installation and patching)
- `mariadb-secure-installation`, `mariadb-tzinfo-to-sql`, `mariadb` (database)
- `openssl req` (self-signed certificate with a subject alternative name)
- `a2enmod`, `a2ensite`, `a2dissite`, `apache2ctl configtest` (Apache)
- `php -i` and `php -v` (verifying the configuration PHP actually loaded)
- `glpi:system:check_requirements` (GLPI console)
- `cron` in CLI mode (automatic actions)
- `curl` and `jq` (GLPI REST API scripting)
- `certutil` (trusting the certificate on Windows clients)

---

## Configuration

### Ticket Information

- **Reference:** HOMELAB-GLPI-001
- **Type:** Self-assigned infrastructure project
- **Host:** Dell PowerEdge R320, Proxmox VE, node STEELBYTE-R320
- **VMID:** 201, name GLPI-01
- **Shell:** 2 cores, 4096 MB memory, 32 GB on local-zfs, virtio-scsi-single, net0 virtio on vmbr0
- **Guest OS:** Debian 13
- **Stack:** Apache, MariaDB 11.8.6, PHP 8.4.24
- **Application:** GLPI 11.0.8
- **Address:** 10.10.50.201 on VLAN 50, static DHCP reservation in pfSense
- **Access:** HTTPS only, LAN only
- **Related:** the network this sits on is documented in ../network-topology.md

### Why GLPI

I wanted a ticketing system for the lab so that when something breaks and I fix it, the work is recorded somewhere with the screenshots and the commands attached, instead of living in notes I will not find again. Four options were considered.

osTicket does ticketing well and little else. FreeScout is closer to a shared inbox than a service desk. Zammad has a strong interface but is heavier to run and is aimed at customer support rather than internal IT. GLPI was the one that covered all three things I wanted in a single application: ticket management, an asset database, and a knowledge base. It is free, open source, and self-hosted, which matters because the entire point was to run it on my own hardware.

There is a second reason. GLPI is real ITSM software used in production by actual organizations, and it follows ITIL conventions. Learning it on my own equipment is more useful than learning a toy tracker, and the category structure, templates, and workflows transfer to a job.

### Scenario Summary

The build was done entirely from the server side. The Debian installer ISO was downloaded onto the Proxmox host and verified against Debian's published checksums, the VM was created with qm from the host shell rather than the web interface, and everything after first boot was done over SSH. Nothing was transferred from a workstation to the server at any point, which was a deliberate constraint rather than an accident.

The VM hardware matches VMID 102, the existing Linux lab host: same CPU type, VirtIO SCSI single with iothread, and a VirtIO NIC on vmbr0 with the Proxmox firewall enabled. VMID 201 was chosen to group it with the 200 range alongside the Jellyfin VM, and the address 10.10.50.201 was chosen to match the VMID, which is a convention used across this environment.

The application layout follows GLPI's own installation documentation rather than dropping everything under one directory. Program files stay at /var/www/glpi owned by root, with configuration in /etc/glpi, data in /var/lib/glpi/files, logs in /var/log/glpi, and plugins in /var/lib/glpi/plugins. Only the public subdirectory is served to browsers. The reason is that if Apache serves the GLPI root instead, the configuration and data directories become reachable over HTTP, and GLPI's own health check flags that as a finding.

Evidence: 01-requirements-check.png shows the health check with every required item passing, including the two permission checks and the marketplace directory. SELinux is skipped because Debian does not use it, the session security check is skipped because the command line cannot test browser cookies, and the ldap extension is reported as not present because nothing here authenticates against an LDAP server.

---

## Steps Taken

The full procedure is in glpi-build-runbook.md, written so another technician could rebuild the system from scratch. The phases were:

1. Create the VM from the Proxmox host CLI, including reading the Debian ISO filename out of the official checksum file rather than typing it, then verifying the download against that file
2. Base operating system setup and guest agent
3. Static DHCP reservation in pfSense, outside the pool
4. Apache, MariaDB, and PHP with the extensions GLPI requires
5. Database creation, timezone table load, and securing MariaDB
6. GLPI download and the separated directory layout
7. HTTPS with a certificate issued to the IP through a subject alternative name, and the Apache site from GLPI's documentation
8. Snapshot, requirements check, and the installation wizard
9. Disabling the four default accounts after creating a real super-admin
10. Automatic actions moved to CLI mode under cron
11. Upload limits raised so full screen screenshots can be attached
12. Trusting the certificate on client machines
13. Test data cleanup and counter reset so the first real ticket is number 1
14. Twenty-two ITIL categories created through the REST API

Step 14 is worth calling out. The categories were created by a bash script against the GLPI REST API rather than entered by hand in the web interface. That was not just to save clicking. GLPI stores calculated fields for every node in a category tree, including the full path name, the depth level, and parent and child caches. Inserting rows directly with SQL would have produced categories that looked right and behaved wrong. Going through the API makes GLPI populate those fields itself.

---

## Troubleshooting Notes

Six issues came up during the build. All six are written out in full in the runbook with symptom, cause, and fix. The one worth reading is the fourth.

Saving a setting in GLPI returned "The action you have requested is not allowed," which reads like a permissions problem and is not. The cause was a typo setting post_max_size to 25M0 instead of 250M. When a request exceeds post_max_size, PHP discards the entire submitted form, including GLPI's CSRF token, so GLPI correctly rejected a request that arrived with no token. The error message was accurate and pointed nowhere near the cause.

That one stuck with me because the symptom and the cause were in completely different parts of the stack, and no amount of looking at GLPI permissions would have found it.

---

## Current Status

GLPI is running and in use. It is reachable over HTTPS on the LAN only. It is not exposed to the internet, and remote access would go through the WireGuard tunnel documented in ../remote-access/remote-access-wireguard.md.

Evidence: 03-dashboard.png shows the application running and signed in as the dedicated super-admin account. The absence of GLPI's default password warning banner on that page is what confirms the four default accounts were disabled, since GLPI displays it on the home page for any default account still using its original password. 04-itil-categories.png shows the category tree the API script created. 02-snapshots-201.png shows both restore points on the VM.

Open items:

- The certificate is trusted on the desktop but not yet installed on the laptop
- External access has not been validated, because the external WireGuard handshake is still unverified
- Inline image insertion in ticket followups has not been tested; file attachments through the Files area were confirmed working

---

## Portfolio Card

Deployed GLPI 11.0.8 as a self-hosted IT service management platform on Debian 13, built end to end from the hypervisor command line with no files transferred from a workstation. Verified the installer image against the vendor's published checksums before use, separated program files, configuration, data, logs, and plugins per the vendor's hardening guidance so the web server cannot write to its own program directory, served the application over HTTPS only, disabled all four default accounts after provisioning a real administrator, and moved scheduled tasks to CLI cron rather than page-load triggered. Configured the ITIL category tree programmatically through the REST API rather than by hand, because the application maintains calculated tree metadata that direct database inserts would have corrupted. Diagnosed six build issues including an application-level authorization error whose actual cause was a single mistyped character in a PHP request size limit silently discarding the CSRF token.

---

## AI Use Statement

AI was used throughout this build, and this section records what it actually did rather than a general acknowledgment.

Where it helped: laying out the build in phases before anything was created, which mattered because I had not stood up a LAMP application from scratch before and did not know what order things needed to happen in. Checking claims against the GLPI and Debian documentation rather than answering from memory, including the recommended directory separation, the Apache site configuration, the session cookie settings, and the requirements the installer checks. Reading the GLPI source to confirm table and field names before the category script was written, rather than guessing at the API schema.

The decisions are mine. Building entirely from the server side with no file transfer from a workstation was my constraint, not a suggestion. So were the VMID and addressing conventions, the choice of GLPI over the alternatives, the category tree and its naming, and the decision to disable the API again afterward.

Where it was wrong, because that is the more useful half of this:

It gave me apt update and apt full-upgrade joined on one line. That does not work, and the resulting error is the first issue in the runbook. A command that fails on the first phase of a build is not a small thing to hand someone.

The category script failed twice on the first run for reasons it had not anticipated. Debian 13 does not ship curl or jq, which it should have known and checked for, and the Legacy REST API had not actually been saved as enabled, which took a manual request to the API endpoint to diagnose rather than being caught in the instructions.

It initially proposed a different VMID than the one I wanted and had to be corrected, and it made an assumption about VMID 102's network configuration based on a different VM's settings rather than asking for the actual line.

The API token was pasted into the chat during the scripting work. That is on me, but it is also the kind of thing the tooling makes easy to do without thinking. The token was regenerated afterward, the script file was deleted, shell history was cleared, and the Legacy REST API was turned back off.

Across this and the other projects in this repository, the pattern is consistent: AI is good at structure and at telling me what to go read, and unreliable enough on specifics that every command was verified against the actual system state before it was run on anything live. That is not a complaint about the tool. It is the working method, and the errors above are the reason for it.

---

## Files In This Folder

- README.md, this document
- glpi-build-runbook.md, the full fourteen phase build procedure with the issues encountered
- glpi-configuration.md, the category tree, templates, groups, and workflow configured through the REST API
- screenshots/01-requirements-check.png, the GLPI health check with all required items passing
- screenshots/02-snapshots-201.png, both snapshots on VMID 201
- screenshots/03-dashboard.png, GLPI running and signed in as the super-admin account
- screenshots/04-itil-categories.png, the ITIL category tree created through the REST API
- ../network-topology.md, the VLAN layout and VM inventory this system sits inside
- ../remote-access/remote-access-wireguard.md, the VPN that would provide external access
