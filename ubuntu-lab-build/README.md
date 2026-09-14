# Ubuntu Server Lab VM: Build and Hardening on Proxmox

**Reference:** HOMELAB-UBU-001
**Scope:** Steelbyte homelab, Linux lab host
**System:** CVNP1601-LAB-UBUNTU, Proxmox VMID 102, hostname 1601
**Completed:** August 2026

Sensitive data within this document is obfuscated. Public IP addresses and Internet facing service listening port information, and any keys or credentials are displayed using asterisks. Private addressing for internal networks is displayed as it appears, due to RFC 1918 space being non-routable from external locations, and this makes the documentation easier to follow.

---

## Objectives

- Stand up the Ubuntu Server lab VM on Proxmox rather than on a laptop hypervisor
- Bring the machine to a current, fully patched state
- Enable SSH and reach the VM from a Windows workstation rather than through the Proxmox console
- Replace password authentication with key based authentication
- Remove direct root login over SSH
- Add a brute force countermeasure at the service level
- Establish a known good snapshot before coursework begins

The course setup guide required only an SSH session screenshot and a snapshot named clean-install. Everything in this document beyond those two items was done because the VM lives on a real server on a segmented network rather than inside a laptop hypervisor, and is not part of the graded submission.

---

## Tools Used

- `lsb_release -a` (release confirmation)
- `ip a` and `ip -4 addr show` (interface and address state)
- `systemctl status ssh` (service state)
- `apt update`, `apt list --upgradeable`, `apt upgrade` (patching)
- `ssh-keygen -t ed25519` (key generation, run on Windows)
- `ssh-agent` and `ssh-add` (passphrase caching, run on Windows)
- `sshd -T` (resolved configuration verification)
- `git config`, `git clone`, `git commit`, `git push` (portfolio repo)
- `ssh -T git@github.com` (GitHub authentication test)
- `df -h`, `lsblk` (storage state)
- Proxmox VE web interface (VM shell creation, snapshots)

---

## Configuration

### Ticket Information

- **Reference:** HOMELAB-UBU-001
- **Host:** Dell PowerEdge R320, node STEELBYTE-R320, Proxmox VE
- **VMID:** 102, name CVNP1601-LAB-UBUNTU
- **Shell:** 4 cores, 1 socket, 4096 MB memory, virtio-scsi-single, net0 virtio on vmbr0
- **Guest OS:** Ubuntu Server 24.04.4 LTS, codename noble
- **Hostname:** 1601
- **Address:** 10.10.50.103/24 on ens18, DHCP, VLAN 50 (SERVERS)
- **Primary account:** steelbyte, member of sudo
- **Repo:** github.com/Chris-CVNP/cvnp1601-portfolio
- **Related:** the disk this VM runs from was imported from VMware as documented in ../vm-migration/README.md (HOMELAB-MIG-001)

### Scenario Summary

The Ubuntu Server VM for this course runs on my homelab server under Proxmox rather than in a laptop hypervisor. That changes the security posture. A VM inside VMware Workstation on a laptop is reachable only from that laptop. This one sits on a VLAN alongside other servers and is reachable from every machine on my LAN, which means the same hardening I would apply to any other host on that segment applies here. The course setup guide does not ask for any of it, because most of the class is not running their lab on a real server. I did it because the machine is exposed to a real network rather than to one laptop, and because it is the correct habit to build.

The work below covers bringing the VM up on Proxmox, patching it, moving off the console onto SSH, replacing password authentication with an ed25519 key pair, removing root login, and putting a brute force countermeasure in front of the SSH service.

---

## Steps Taken

### Step 1. Build the VM shell on Proxmox

Action performed in the Proxmox web interface: Create VM, 4 cores on 1 socket, 4096 MB memory, virtio-scsi-single controller, net0 on vmbr0 with the firewall flag set, Start after created left unchecked.

**Expected output includes:** the Confirm tab listing vmid 102, name CVNP1601-LAB-UBUNTU, nodename STEELBYTE-R320, ostype l26, and scsihw virtio-scsi-single.

**Status:** Complete. Evidence: 01-vm-shell-102.png. No disk was attached at creation because the imported disk is attached afterward, as covered in the migration write-up.

### Step 2. Confirm the release and the starting state

```
lsb_release -a
```

**Expected output includes:** Distributor ID Ubuntu, Description Ubuntu 24.04.4 LTS, Release 24.04, Codename noble.

**Status:** Complete. Evidence: 02-baseline-state.png.

### Step 3. Confirm the network address

```
ip -4 addr show | grep inet
```

**Expected output includes:** ens18 holding 10.10.50.103/24 as a dynamic global address.

**Status:** Complete. Evidence: 12-ip-verification.png. The interface rename that had to happen before this worked is documented as Issue 3 in the migration write-up.

### Step 4. Check the SSH service state

```
sudo systemctl status ssh
```

**Expected output includes:** ssh.service loaded, and on a fresh Ubuntu Server install, disabled and inactive with ssh.socket listed as the trigger.

**Status:** Complete. Evidence: 02-baseline-state.png shows the service disabled and inactive before any change. This is the before state the rest of the document works against.

### Step 5. Enable and start SSH

```
sudo systemctl enable --now ssh
```

**Expected output includes:** the journal reporting sshd listening on 0.0.0.0 port 22 and on :: port 22, and ssh.service started.

**Status:** Complete. Evidence: 03-ssh-enabled.png.

### Step 6. Patch the system

```
sudo apt update
```

```
apt list --upgradeable
```

```
sudo apt upgrade
```

**Expected output includes:** the upgradeable list enumerating each package with its current and candidate version, then the upgrade completing.

**Status:** Complete. Evidence: 04-system-updates.png and 05-updates-complete.png. Five python3.12 packages were reported as deferred due to phasing rather than failing. Phasing is Ubuntu's staged rollout of an update to a fraction of machines at a time, so a deferred package is expected behavior and not an error.

### Step 7. Generate an ed25519 key pair on the Windows workstation

Action performed on Windows in PowerShell:

```
ssh-keygen -t ed25519
```

**Expected output includes:** a private key and a matching .pub file written under the user profile .ssh directory, with a passphrase set.

**Status:** Complete. ed25519 was chosen over RSA because it is the current default, uses a fixed and adequate key size, and produces a shorter key to move by hand.

### Step 8. Install the public key on the VM

Action performed manually: append the contents of the .pub file to ~/.ssh/authorized_keys on the VM.

**Expected output includes:** a key login from Windows succeeding without a password prompt for the account.

**Status:** Complete. This was done by hand because ssh-copy-id does not exist in the Windows OpenSSH client, so the usual one command method is not available.

### Step 9. Cache the passphrase with ssh-agent

Action performed on Windows in PowerShell:

```
ssh-add $env:USERPROFILE\.ssh\id_ed25519
```

**Expected output includes:** Identity added, followed by key logins no longer prompting for the passphrase.

**Status:** Complete.

### Step 10. Disable password authentication and root login

Action performed on the VM: write PasswordAuthentication no and PubkeyAuthentication yes into a drop-in under /etc/ssh/sshd_config.d/, set PermitRootLogin no, validate, and restart.

```
sudo sshd -t && echo "config OK"
```

```
sudo systemctl restart ssh
```

```
sudo sshd -T | grep -iE "passwordauthentication|pubkeyauthentication|permitrootlogin"
```

**Expected output includes:** permitrootlogin no, pubkeyauthentication yes, passwordauthentication no.

**Status:** Complete. sshd -T was used rather than reading the config file, because it prints the fully resolved configuration after all includes and drop-ins are merged. Reading a single file does not tell you what the service is actually running. A load order problem that made this distinction matter is recorded in the Week 2 portfolio entry and is not repeated here, since that was graded work.

Before disabling password authentication, a second session was left open and a fresh key login was confirmed from a new terminal. Locking password authentication without a working key first means losing access to a machine whose console does not reliably accept the keystrokes needed for recovery.

### Step 11. Install and configure fail2ban

```
sudo apt install fail2ban
```

Then create /etc/fail2ban/jail.local with an sshd stanza and enable the service. The policy set was four failed attempts within a findtime window of 1000 seconds, roughly seventeen minutes, producing a bantime of 8000 seconds, roughly two hours and thirteen minutes. Those values are stricter than the fail2ban defaults of five attempts and a ten minute ban, which was deliberate on a machine reachable from every host on the LAN.

**Expected output includes:** fail2ban.service active and running, and fail2ban-client status sshd reporting the sshd jail with a filter and an action attached.

**Status:** Complete and verified. Evidence: 13-fail2ban-service.png and 14-fail2ban-jail.png. See Troubleshooting Notes, Issue 2, for why this step took two attempts separated by several weeks.

### Step 12. Configure Git and authenticate to GitHub

```
git config --global user.name Chris-CVNP
```

```
git config --global user.email <school-address>
```

```
git config --global init.defaultBranch main
```

```
ssh -T git@github.com
```

**Expected output includes:** git config --list reflecting the set values, and the GitHub test returning a successful authentication greeting for the account while noting that GitHub does not provide shell access.

**Status:** Complete. Evidence: 06-git-config.png and 07-github-auth.png. The placeholder values from the setup instructions were written first and then corrected, which is visible in the capture.

### Step 13. Clone the portfolio repo and push an initial commit

```
git clone git@github.com:Chris-CVNP/cvnp1601-portfolio.git
```

```
git add README.md && git commit -m "Initial portfolio setup" && git push
```

**Expected output includes:** the clone completing and the push reporting a commit hash advancing main.

**Status:** Complete. Evidence: 08-repo-clone.png and 09-initial-commit.png, commit 67c7883.

### Step 14. Take a baseline snapshot

Action performed in the Proxmox web interface: VM 102 > Snapshots > Take Snapshot.

**Expected output includes:** the task reporting the VM state saved and the disk snapshotted, ending in TASK OK.

**Status:** Complete. Evidence: 10-snapshot-102.png shows the task saving 881.76 MiB of VM state and snapshotting drive-scsi0 on local-zfs. Two snapshots exist on this VM: UbuntuServerBaseline from 2026-08-26 and Clean-Install from 2026-08-27.

---

## Troubleshooting Notes

### Issue 1: fail2ban install rejected by apt

**Incorrect command:**

```
sudo apt install fail2ban/jail.local
```

**Root cause:** the slash in an apt package argument is release pinning syntax, not a path. apt read jail.local as the name of a release to install the fail2ban package from, could not find a release by that name, and refused. jail.local is a configuration file created after installation, not part of the package name.

**Resolution:** install the package by name alone, then create the jail configuration as a separate step.

**Result:** apt accepted the corrected command. Evidence: 03-ssh-enabled.png shows the rejected attempt and the error text.

### Issue 2: fail2ban enabled, running, and doing nothing

**Incorrect configuration,** in /etc/fail2ban/jail.local:

```
[sshd]
enable = true
port = ssh
filter = sshd
logpath = /var/log/auth.log
maxretry 4
bantime = 8000
findtime = 1000
```

**Root cause:** two separate faults on two lines. Line 6 was missing an equals sign, and fail2ban configuration is INI format, so the parser threw on it and the daemon exited with status 255 on every start attempt. Line 2 read `enable` rather than `enabled`, which is not a recognized key. Either one alone breaks the jail, and they fail differently: the missing equals sign kills the service outright, while the wrong key name would have left the service running with no sshd jail loaded at all.

**How it was found:** not by testing. `systemctl is-enabled fail2ban` returns enabled whether or not the service can actually start, so nothing in the ordinary checks caught it. It surfaced weeks later during a power outage recovery, when this document's own unverified item was finally checked and `systemctl status` reported failed rather than active.

**Resolution:**

```
sudo sed -i -e 's/^enable = true/enabled = true/' -e 's/^maxretry 4/maxretry = 4/' /etc/fail2ban/jail.local
```

```
sudo systemctl restart fail2ban
```

**Result:** the service reports active and running with Server ready, and `fail2ban-client status sshd` returns the jail with its filter and action attached. Evidence: 13-fail2ban-service.png and 14-fail2ban-jail.png.

One detail worth recording. The jail reports its source as `_SYSTEMD_UNIT=sshd.service` rather than the logpath in the configuration. Debian and Ubuntu builds of fail2ban default to the systemd journal backend, which overrides the logpath setting. The line is not wrong, it is simply ignored, and the jail reads the journal instead.

---

## Verification

The hardening was confirmed from more than one angle rather than by trusting that a change had been made:

- `sshd -T` reporting the resolved values, not a single config file
- A fresh key login from a new terminal before password authentication was disabled
- `whoami`, `pwd`, and `ls /` from the SSH session confirming the account and a normal filesystem. Evidence: 11-lab-verification.png
- `ip -4 addr show` confirming the expected address. Evidence: 12-ip-verification.png

- `systemctl status fail2ban` and `fail2ban-client status sshd` confirming the service is running and the jail is loaded. Evidence: 13-fail2ban-service.png and 14-fail2ban-jail.png

The fail2ban item is worth singling out, because for several weeks this document recorded it as configured rather than verified, on the principle that a control which has not been observed working is not a control that has been proven. That turned out to be exactly right. When it was finally checked, the service had never started successfully, for the reasons in Issue 2. Had it been written up as complete on the strength of the configuration file existing, the gap would have gone unnoticed indefinitely.

The distinction matters more than it sounds. `systemctl is-enabled` returns enabled for a service that fails on every start, and a configuration file with a typo in it looks correct at a glance. Neither is evidence. Only the running service and the loaded jail are.

---

## Portfolio Card

Built and hardened an Ubuntu Server 24.04 LTS lab machine on Proxmox VE, going well beyond the course requirement because the VM lives on a segmented server VLAN rather than inside a laptop hypervisor. Moved administration off the hypervisor console onto SSH, replaced password authentication with an ed25519 key pair installed by hand, removed direct root login, and put a brute force countermeasure in front of the service. Verified the result with sshd -T against the fully resolved configuration rather than a single config file, and confirmed a working key login before locking password authentication so the change could not strand the machine. Diagnosed and corrected an apt release pinning syntax error that blocked the fail2ban install, and later found that the brute force countermeasure had never actually run: two typos in its configuration, one killing the daemon on startup and one silently preventing the jail from loading, neither of which showed up in the service's enabled state.

---

## AI Use Statement

AI assisted with the structure of this document and with checking technical claims against documentation. The build, the hardening decisions, and the order the work was done in are mine, and the commands recorded here are the ones I ran. Where a command failed, the failure is written down as it happened rather than replaced with the corrected version. See the AI disclosure in the Proxmox migration README for details on how AI was used across this project.

---

## Files In This Folder

- README.md, this document
- ../network-topology.md, the VLAN layout this VM sits inside
- screenshots/01-vm-shell-102.png, Proxmox Create VM confirm page for VMID 102
- screenshots/02-baseline-state.png, lsb_release, ip a, and ssh disabled and inactive
- screenshots/03-ssh-enabled.png, sshd listening on port 22 and the rejected fail2ban install
- screenshots/04-system-updates.png, apt list --upgradeable
- screenshots/05-updates-complete.png, upgrade complete with five packages deferred due to phasing
- screenshots/06-git-config.png, git config corrected from the guide placeholders
- screenshots/07-github-auth.png, ssh -T git@github.com authenticating
- screenshots/08-repo-clone.png, cvnp1601-portfolio cloned
- screenshots/09-initial-commit.png, initial commit pushed
- screenshots/10-snapshot-102.png, Proxmox snapshot task for VMID 102 ending TASK OK
- screenshots/11-lab-verification.png, whoami, pwd, and ls / over SSH
- screenshots/12-ip-verification.png, ip -4 addr show
- screenshots/13-fail2ban-service.png, fail2ban service active and running
- screenshots/14-fail2ban-jail.png, the sshd jail loaded with its filter and action
