# Proxmox VM Migration: VMware Workstation to Proxmox VE

**Reference:** HOMELAB-MIG-001
**Course:** CVNP 1606, Supporting Windows Operating Systems (extra credit)
**Systems:** CVNP1606-LAB-Win11 (VMID 101), CVNP1601-LAB-UBUNTU (VMID 102)
**Completed:** August 2026

Sensitive data within this document is obfuscated. Public IP addresses and Internet facing service listening port information, user account names and any keys or credentials are displayed using asterisks. Private addressing for internal networks are displayed as they appear, due to RFC 1918 space being non-routable from external locations, and this makes the documentation easier to follow.

---

## Objectives

- Move both course VMs off VMware Workstation on the laptop and onto Proxmox VE on the homelab server
- Free the RAM those VMs were reserving on the primary workstation
- Make both VMs reachable from either the desktop or the laptop over the LAN
- Preserve the existing Windows 11 installation rather than rebuilding it from scratch
- Establish a known good snapshot on each VM after migration
- Keep both VMs off the public internet

---

## Tools Used

- `df -h` (Proxmox host, free space check)
- `mkdir -p` (Proxmox host, staging directory)
- `ls -la` (Proxmox host, filename verification)
- `qm importdisk` (Proxmox host, VMDK import)
- `sed` (Ubuntu guest, netplan interface rename)
- `netplan apply` (Ubuntu guest)
- `ip a` (Ubuntu guest, interface state)
- `ping` (Ubuntu guest and laptop, connectivity)
- `apt update` (Ubuntu guest, repository reachability)
- WinSCP over SFTP (file transfer)
- VMware Workstation Snapshot Manager (snapshot consolidation)
- Proxmox VE web interface (VM shell creation, hardware, snapshots)
- Registry Editor (Windows profile path correction)
- virt-viewer and SPICE guest tools (clipboard)

---

## Configuration

### Ticket Information

- **Reference:** HOMELAB-MIG-001
- **Type:** Self-assigned infrastructure project, approved by instructor
- **Source host:** Laptop running VMware Workstation, VMs stored under D:\VM_Lab\VMs
- **Target host:** Dell PowerEdge R320, Proxmox VE, 96 GB RAM, 10.10.50.10
- **Network:** VLAN 50 (SERVERS), 10.10.50.0/24, behind pfSense with a Cisco 3850 core switch. See ../network-topology.md
- **Storage target:** local-zfs
- **Staging path:** /var/lib/vz/tmp/vmware-import
- **Data moved:** approximately 49 GB for the Windows 11 disk, approximately 54 GB for both VMs combined

### Scenario Summary

There were several reasons I wanted to migrate the VMs onto my homelab server. First, I had three VMs running in VMware Workstation on my laptop, and all three used RAM. After adding a fourth VM, an Ubuntu Server, I was concerned about exhausting my laptop's remaining RAM. Second, during the summer I transitioned from using my laptop as my primary workstation to using a desktop I had built at the end of the previous school year. Since the laptop and desktop use separate resource allocations, I faced the option of constantly switching between the two systems or migrating all existing VMs to the desktop, which would raise the same concern about RAM utilization on my working system. Third, although the chances of needing to work remotely during the semester may be rare, having the option to do so will be beneficial if needed, and now is the time to do it rather than when it is needed in the future. My server has 96 GB of RAM, so running several school related VMs on it would not consume excessive amounts of host RAM. Before making any decision, I contacted my instructor and requested permission to use my homelab server for this purpose. After he granted permission, he gave me a recommendation: run the VMs under Proxmox rather than installing VMware on the server. I followed his recommendation.

The R320 resides on a VLAN segmented network. Servers sit on their own VLAN separate from the user VLAN, so segmentation prevents communication between the two by default. A connectivity test was performed before the transfer began, because if the path did not exist the transfer would have failed at the start and would have required a pfSense rule change to resolve. All machines on the LAN can reach the lab VMs. None of the lab VMs are exposed to the public internet. External access requires the VPN documented in ../remote-access/remote-access-wireguard.md.

---

## Steps Taken

### Step 1. Verify connectivity from the laptop to the Proxmox host

```
ping 10.10.50.10
```

**Expected output includes:** replies from 10.10.50.10 with no loss, confirming the user VLAN can reach the SERVERS VLAN before any transfer begins.

**Status:** Complete. Connectivity confirmed, so no pfSense rule change was required.

### Step 2. Verify free space on the Proxmox host

```
df -h
```

**Expected output includes:** the ZFS dataset backing the staging path reporting several terabytes available, well beyond the approximately 54 GB about to be transferred.

**Status:** Complete. Ample space confirmed before starting, eliminating the risk of a mid-transfer failure on insufficient disk space.

### Step 3. Consolidate VMware snapshots on both source VMs

Action performed in VMware Workstation: VM > Snapshot > Snapshot Manager, then delete each snapshot.

**Expected output includes:** each VM folder left with a single VMDK and no delta files ending in -000001.vmdk, no .vmem files, and no .vmsn files.

**Status:** Complete. Evidence: 02-snapshot-consolidation.png shows the 1606 snapshot chain before deletion. 02a and 02b show the 1606 folder before and after. 02c and 02d show the 1601 folder before and after. VMware stores snapshots as delta files layered on the base disk, so consolidating first meant only one disk file per VM had to move and no snapshot chain had to survive the migration.

### Step 4. Confirm the source working directory for each VM

Action performed in VMware Workstation: VM > Settings > Options tab > General.

**Expected output includes:** the Working directory field showing the folder holding the .vmx, .vmdk, .nvram, and .vmsd files for that VM.

**Status:** Complete. Evidence: 01-vmware-source-vm.png.

### Step 5. Create the staging directory on the Proxmox host

```
mkdir -p /var/lib/vz/tmp/vmware-import
```

**Expected output includes:** no output on success.

**Status:** Complete. The staging directory and the qm importdisk command both require root, so this work was done over the internal management path only. Root access to the host is not reachable externally.

### Step 6. Transfer both VM folders with WinSCP

Action performed in WinSCP: connect to 10.10.50.10 over SFTP, local pane at D:\VM_Lab\VMs, remote pane at /var/lib/vz/tmp/vmware-import, then drag each VM folder across.

**Expected output includes:** both VM folders present in the remote pane with sizes matching the source.

**Status:** Complete. Evidence: 03a-transfer-windows.png shows the Windows VMDK uploading. 03c-transfer-complete.png shows both folders staged on the host.

### Step 7. Verify the exact VMDK filename on the Proxmox side

```
ls -la "/var/lib/vz/tmp/vmware-import/CVNP1601-LAB/"
```

**Expected output includes:** the .vmdk, .vmx, .nvram, .vmsd, and .vmxf files with their byte sizes, confirming the exact filename before it is used in a command.

**Status:** Complete. Evidence: 05-importdisk-ubuntu.png. This step was run for each VM because the Windows VMDK filename contains spaces and an ampersand, which have to be quoted exactly rather than typed from memory.

### Step 8. Build the VM shell for the Windows 11 VM

Action performed in the Proxmox web interface: Create VM, matching the source CPU and memory allocation, with no disk attached during creation.

**Expected output includes:** BIOS set to OVMF (UEFI), a TPM State device added at version 2.0, and the assigned VMID recorded for the import command.

**Status:** Complete. Evidence: 04-vm-shell-config.png. The vTPM had to exist before first boot. VMware's virtual TPM does not travel inside the disk file, and Windows 11 will not boot without one, so this was configured as part of building the shell rather than added after a failed boot.

### Step 9. Import the Windows 11 disk

```
qm importdisk 101 "/var/lib/vz/tmp/vmware-import/CVNP1606-LAB/<disk-name>.vmdk" local-zfs
```

**Expected output includes:** incremental transferred lines counting up to the provisioned disk size, then confirmation of the new volume attached to VM 101 as an unused disk.

**Status:** Complete. qm importdisk converts the VMDK into Proxmox storage as it reads it, so no separate conversion step was required.

### Step 10. Attach the imported Windows disk on the SATA bus

Action performed in the Proxmox web interface: VM 101 > Hardware > select the unused disk > Edit > bus SATA.

**Expected output includes:** Hard Disk (sata0) listed in the Hardware tab pointing at local-zfs.

**Status:** Complete. Evidence: 04-vm-shell-config.png. VirtIO SCSI performs better than SATA, but Windows had been running under VMware's storage drivers with no VirtIO drivers installed. Attaching to VirtIO at this point would have produced an INACCESSIBLE_BOOT_DEVICE bugcheck on first boot. SATA is natively supported by Windows and was the safe choice for the first boot after migration.

### Step 11. First boot of the Windows 11 VM

Action performed in the Proxmox web interface: VM 101 > Start, then Console.

**Expected output includes:** Windows 11 reaching the login screen with no recovery prompt and no safe mode.

**Status:** Complete. The VM booted successfully on the first attempt.

### Step 12. Replace the guest tooling in the Windows 11 VM

Action performed inside the guest: uninstall VMware Tools through Programs and Features, then install the VirtIO guest tools and the QEMU guest agent from the virtio-win ISO.

**Expected output includes:** the VirtIO installer reporting Installation Successfully, and Qemu-ga, Red Hat, and Virtio-Win directories present under C:\Program Files.

**Status:** Complete. Evidence: 07a-virtio-installer-welcome.png, 07b-virtio-installing.png, 07c-virtio-installed.png, and 08-guest-tools-installed.png.

### Step 13. Import the Ubuntu Server disk

```
qm importdisk 102 "/var/lib/vz/tmp/vmware-import/CVNP1601-LAB/CVNP1601-LAB.vmdk" local-zfs
```

**Expected output includes:** transferred lines counting up to 20.0 GiB, then confirmation of the new volume attached to VM 102.

**Status:** Complete. Evidence: 05-importdisk-ubuntu.png. The Ubuntu disk was attached to VirtIO SCSI directly, since Linux does not have the Windows boot driver limitation described in Step 10.

### Step 14. First boot of the Ubuntu Server VM

Action performed in the Proxmox web interface: VM 102 > Start, then Console.

**Expected output includes:** a normal login prompt.

**Status:** Complete on boot, failed on network. The VM reached a login prompt but had no network connectivity. See Troubleshooting Notes, Issue 3.

### Step 15. Take a baseline snapshot on each VM

Action performed in the Proxmox web interface: VM > Snapshots > Take Snapshot.

**Expected output includes:** the snapshot listed in the Snapshots tab with a name, timestamp, and description.

**Status:** Complete. Evidence: 09-snapshot-task-101.png shows the VM 101 snapshot task saving 10.65 GiB of VM state. 10-snapshots-102.png shows the VM 102 snapshot tree.

Snapshots in Proxmox are managed under each VM's own Snapshots tab rather than through a standalone snapshot manager as in VMware. Any VMware based instruction referring to a snapshot manager translates to this tab.

Current snapshots on VMID 101:

- Win11ProBaseline, created 2026-08-26. The known good state immediately following post-migration baseline configuration.
- W01_CleanBaseline, created 2026-08-27. Taken after accounts and Windows Updates were complete.

Current snapshots on VMID 102:

- UbuntuServerBaseline, created 2026-08-26. A known good baseline of the Ubuntu Server.
- Clean-Install, created 2026-08-27. A clean install and a known good starting point.

Having both on each VM allows rollback to a point before coursework begins and to a point after the baseline is established.

---

## Troubleshooting Notes

### Issue 1: Virtual TPM does not migrate with the disk image

**Incorrect approach:** Building the VM shell with default firmware and no TPM device, then importing the disk and booting.

**Root cause:** Windows 11 requires a TPM. VMware's virtual TPM is a property of the VMware VM, not of the VMDK, so it does not exist on the Proxmox side after an import. A VM shell built with SeaBIOS and no TPM State device cannot boot a Windows 11 image.

**Resolution:** Set BIOS to OVMF (UEFI) and add a TPM State device at version 2.0 while building the VM shell, before importing the disk and before first boot. Proxmox provides vTPM natively through swtpm.

**Result:** Windows 11 booted on the first attempt. This was identified during research rather than after a failed boot, so no failed boot occurred.

### Issue 2: No shared clipboard inside the Windows 11 VM

**Incorrect approach:** Attempting to copy and paste between the host machine and the guest through the Proxmox noVNC browser console.

**Root cause:** The default noVNC browser console does not provide a shared clipboard. This is a known limitation of that console rather than a fault introduced by the migration.

**Resolution:** Change the VM's Display hardware from the default to SPICE, install the SPICE guest tools inside Windows, and install virt-viewer on the connecting machine so the SPICE console opens in a real client rather than the browser panel.

**Result:** Bidirectional clipboard works. This was the most time consuming part of the migration. While installing virt-viewer I also looked at display memory allocation and determined 32 MB is suitable for single monitor operation at normal desktop resolutions. Larger values only benefit multi monitor SPICE configurations that require a separate video device per display, which adds overhead with no benefit in this setup. Evidence: 06-spice-guest-tools.png and 08-guest-tools-installed.png.

### Issue 3: Ubuntu Server VM had no network after migration

**Incorrect command:**

```
sudo dhclient ens18
```

**Root cause:** The netplan configuration at /etc/netplan/50-cloud-init.yaml still referenced ens33, the interface name assigned by VMware. Proxmox presents the VirtIO NIC as ens18, so netplan was configuring an interface that did not exist and ens18 was left with no address. Bringing the link up manually and calling dhclient did not resolve it, and dhclient is not present on Ubuntu 24.04 in any case.

**Resolution:**

```
sudo sed -i 's/ens33/ens18/' /etc/netplan/50-cloud-init.yaml
sudo netplan apply
```

**Result:** ens18 received a DHCP lease, ping to the gateway succeeded, and apt update completed against the Ubuntu repositories. Evidence: 11-apt-dns-failure.png, 12-interface-down.png, 13-manual-link-up-failed.png, and 14-netplan-ens33.png show the failure chain and the root cause in the netplan file.

### Issue 4: Duplicate user profile folder on the Windows 11 VM

**Incorrect approach:** Treating the second profile folder as the live profile.

**Root cause:** Not confirmed. The account ended up with two profile folders. I am not claiming the migration caused this, because Windows creates a suffixed profile folder any time it determines the expected profile path is already occupied.

**Resolution:** Rename the active profile folder to the intended name, then update the ProfileImagePath value under HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\ProfileList\(SID) so Windows points at the correct folder.

**Result:** The account loads the correct profile. The orphaned folder was removed afterward.

---

## Portfolio Card

Migrated two live course virtual machines from VMware Workstation on a laptop to Proxmox VE on a Dell PowerEdge R320, preserving the existing Windows 11 installation rather than rebuilding it. Handled the platform differences that do not carry across an import: provisioned a virtual TPM 2.0 and UEFI firmware on the target before first boot, attached the imported disk on SATA to avoid an INACCESSIBLE_BOOT_DEVICE bugcheck under drivers the guest did not yet have, and replaced VMware Tools with the VirtIO guest tools and QEMU guest agent afterward. Diagnosed a total loss of network on the Ubuntu guest down to a stale VMware interface name in the netplan configuration and corrected it. Established baseline snapshots on both machines and kept both off the public internet. Requested and received instructor approval before starting.

---

## AI Use Statement

I used AI throughout this migration, and the way I used it shifted as the work progressed. Initially, it provided an outline of what needed completing and in what order, which mattered because this represented my first migration and I did not know what the phases were. Once issues arose, it became more hands-on, supplying commands to run and helping me work through what was failing.

Decisions remained mine. Shell placement, allocation sizing, file naming, and the overall layout of the machines were determined by me, along with the decision to request approval from my instructor prior to beginning.

Because these commands were being executed against a live machine rather than a document, I did not accept suggestions at face value. I required the AI to verify against multiple reputable sources before acting upon its answers, and I requested the sources it drew from, including URLs, when I intended to read the documentation myself.

That caution proved warranted. The AI was wrong at several points. It directed me to stage the transferred files in an incorrect location based upon an assumption regarding filesystem layout that the actual disk usage output contradicted. When the clipboard issue was ultimately resolved, the fix turned out to be a step the AI had never instructed me to take, meaning every component had been configured correctly and its instructions contained the gap. In a later session, it claimed a technical detail about display memory had caused the clipboard failure, and I knew from the original work that wasn't what happened. There were multiple points across the project where we disagreed regarding how to proceed.

I anticipate using AI tools as regular collaborators within IT rather than as occasional use. This project showed me that the value (or lack thereof) of the AI ultimately lies with the individual who operates it. While the AI provided an excellent resource for structuring tasks and quickly identifying potential problem areas, I found it wrong often enough that I had to verify all AI outputs against actual system states before deciding how to proceed. In short, running an unverified command on a live machine is a great way to create the outage you are attempting to avoid.

---

## Conclusion

Would I do it again? Yes. While I expected challenges that might take considerable time to resolve, I never felt like the task would fail or that I was defeated at any point.

Now that both VMs have been moved to new hardware with sufficient resources, my workstation no longer reserves any RAM for them, and I can access the lab environment from either workstation, whereas previously I had limited mobility because I could use only one workstation.

---

## Files In This Folder

- README.md, this document
- migration-runbook.md, the repeatable procedure for moving a VM from VMware Workstation to Proxmox
- ../remote-access/remote-access-wireguard.md, the WireGuard VPN configuration and its current status
- ../network-topology.md, the VLAN layout and device inventory this migration sits inside
- screenshots/01-vmware-source-vm.png, VMware VM Settings showing the source working directory
- screenshots/02-snapshot-consolidation.png, VMware Snapshot Manager before deletion
- screenshots/02a-snapshots-before-1606.png, 1606 VM folder with the snapshot chain present
- screenshots/02b-snapshots-after-1606.png, 1606 VM folder after consolidation
- screenshots/02c-snapshots-before-1601.png, 1601 VM folder with the snapshot chain present
- screenshots/02d-snapshots-after-1601.png, 1601 VM folder after consolidation
- screenshots/03a-transfer-windows.png, WinSCP transferring the Windows VMDK
- screenshots/03c-transfer-complete.png, both VM folders staged on the Proxmox host
- screenshots/04-vm-shell-config.png, Proxmox Hardware tab for VMID 101
- screenshots/05-importdisk-ubuntu.png, ls -la verification and qm importdisk for VMID 102
- screenshots/06-spice-guest-tools.png, SPICE guest tools download
- screenshots/07a-virtio-installer-welcome.png, VirtIO installer
- screenshots/07b-virtio-installing.png, VirtIO installer progress
- screenshots/07c-virtio-installed.png, VirtIO installer completion
- screenshots/08-guest-tools-installed.png, Program Files showing Qemu-ga, Red Hat, Spice Agent, Virtio-Win
- screenshots/09-snapshot-task-101.png, Proxmox snapshot task for VMID 101
- screenshots/10-snapshots-102.png, Snapshots tab for VMID 102
- screenshots/11-apt-dns-failure.png, apt update failing name resolution
- screenshots/12-interface-down.png, ip a showing ens18 down and ping unreachable
- screenshots/13-manual-link-up-failed.png, manual link up and dhclient not found
- screenshots/14-netplan-ens33.png, netplan still referencing ens33
