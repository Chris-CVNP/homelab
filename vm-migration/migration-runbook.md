# Runbook: Migrating a VM from VMware Workstation to Proxmox VE

## Executive Summary

This runbook covers moving a virtual machine off VMware Workstation on a desktop or laptop and onto a Proxmox VE server. Running virtual machines on dedicated server hardware frees memory on the workstation, makes the machines reachable from more than one computer, and puts them somewhere with the resources to run several at once. The procedure below was used to migrate two working course VMs, one Windows 11 and one Ubuntu Server, and is written so another technician could repeat it without prior migration experience.

The procedure is the same for both guest operating systems through step 8. Steps 9 and 10 differ by guest, and both are covered.

## Prerequisites

- Proxmox VE host with enough free storage for the virtual disk, plus headroom
- Network path from the workstation to the Proxmox host, tested before starting
- Root credentials for the Proxmox host
- WinSCP or another SFTP client on the workstation
- The source VM fully shut down, not suspended

Confirm the network path first. If the workstation and the Proxmox host sit on different VLANs, a firewall rule may be required before anything can transfer, and it is better to find that out now than partway through:

   ping 10.10.50.10

Confirm free space on the Proxmox host:

   df -h

## Procedure

### 1. Consolidate snapshots on the source VM

In VMware Workstation, open the snapshot manager and delete existing snapshots. VMware stores each snapshot as a delta file layered on the base disk. Deleting them merges those deltas back down so the VM is left with a single VMDK.

Migrating with an active snapshot chain risks an incomplete or inconsistent disk image. Verify afterward that the VM folder contains one VMDK and no files ending in -000001.vmdk, no .vmem files, and no .vmsn files.

### 2. Shut down and locate the source files

Shut the VM down completely. A suspended VM's disk is not in a consistent state to copy.

In VMware Workstation, go to VM, then Settings, then the Options tab, and note the Working directory path. The folder should contain a .vmx, a .vmdk, and a .nvram.

If a .vmx.lck folder is present, VMware still has the VM checked out. Close Workstation and confirm it clears before transferring.

### 3. Create a staging directory on the Proxmox host

From the Proxmox node shell:

   mkdir -p /var/lib/vz/tmp/vmware-import

Confirm it exists:

   ls -la /var/lib/vz/tmp/

### 4. Transfer the VM folder

Connect WinSCP to the Proxmox host over SFTP on port 22 as root. Accept the host key warning on first connection.

Navigate the local pane to the VM's working directory and the remote pane to /var/lib/vz/tmp/vmware-import, then drag the VM folder across.

Transfer speed is the limiting factor on total time. Over a gigabit link at roughly 80 MB per second, a 49 GB disk takes about ten minutes. Plan accordingly rather than assuming either extreme.

### 5. Verify the filename on the Proxmox side

Before using a filename in a command, read it off the destination rather than typing it from memory:

   ls -la /var/lib/vz/tmp/vmware-import/VMNAME/

This matters more than it sounds. VMware names the disk after the VM, so a VM named with spaces or an ampersand produces a filename containing both, and either one will break an unquoted command.

### 6. Build the VM shell in Proxmox

Create the VM in the Proxmox web interface before importing the disk. Match the source VM's CPU and memory allocation.

For a Windows 11 guest, two settings are required and must be in place before first boot:

- BIOS set to OVMF (UEFI), not SeaBIOS
- A TPM State device added, version 2.0

VMware's virtual TPM does not travel inside the disk file. If the VM boots without one, Windows 11 will not come up. For a Linux guest neither setting is required, though OVMF is still a reasonable default.

Note the VM ID assigned, since the import command needs it.

Do not attach a disk during creation. The imported disk is attached in a later step.

### 7. Import the disk

From the node shell, with the VM ID and the full path to the VMDK:

   qm importdisk 101 "/var/lib/vz/tmp/vmware-import/VMNAME/diskname.vmdk" local-zfs

Quote the path if the filename contains spaces or special characters.

qm importdisk converts the VMDK into the target storage format as it reads it, so no separate conversion step is required.

Progress is reported as transferred lines counting up to the provisioned disk size. The import can appear to sit at 100 percent for some time before completing. This is normal on large disks. Do not interrupt it. On success it reports the new disk as an unused volume attached to the VM.

### 8. Attach the disk

In the VM's Hardware tab, select the unused disk and attach it.

For a Windows guest, attach it as SATA, not VirtIO SCSI. VirtIO offers better performance, but the guest was running under VMware's storage drivers and has no VirtIO drivers installed. Attaching to VirtIO before those drivers exist produces an INACCESSIBLE_BOOT_DEVICE bugcheck on first boot. SATA is the reliable choice for the initial boot after migration, and the guest can be moved to VirtIO later once the drivers are in place.

For a Linux guest, VirtIO SCSI can be used directly. Linux carries the VirtIO drivers in the kernel and does not have the Windows boot driver problem.

Set the boot order so the imported disk is first.

### 9. First boot

Start the VM and open the console. The guest should boot normally.

If a Windows guest does not, the most likely causes in order are: firmware set to SeaBIOS instead of OVMF, no TPM device present, the disk attached to VirtIO, or the boot order not pointing at the imported disk.

If a Linux guest boots but has no network, see the interface rename note under step 10.

### 10. Post-migration guest configuration

For a Windows guest, inside the running VM:

- Uninstall VMware Tools through Programs and Features
- Install the VirtIO guest tools for the correct storage, network, and balloon drivers
- Install the QEMU guest agent so Proxmox can read the guest's state and shut it down cleanly

Reboot after installing.

For a Linux guest, expect the network interface name to change. VMware presents its virtual NIC as ens33 while Proxmox presents the VirtIO NIC as ens18. The guest's network configuration still references the old name after migration, so it configures an interface that does not exist and the real one comes up with no address. Check what the kernel actually sees:

   ip a

Then correct the netplan configuration to match:

   sudo sed -i 's/ens33/ens18/' /etc/netplan/50-cloud-init.yaml

   sudo netplan apply

Confirm the interface picked up an address and can reach the gateway before moving on. Note that dhclient is not present on Ubuntu 24.04, so bringing the link up by hand and calling dhclient will not work as a shortcut. Fix the configuration instead.

Install the QEMU guest agent on the Linux guest as well.

### 11. Enable clipboard sharing

The Proxmox browser console (noVNC) does not support a shared clipboard. If copy and paste between host and guest is needed:

- Change the VM's Display hardware to SPICE
- Install the SPICE guest tools inside the guest and reboot
- Install virt-viewer on the machine you connect from
- Use the SPICE console option, which downloads a connection file and opens a real client window rather than the browser panel

This is a configuration limitation, not a fault. Budget time for it. Display memory of 32 MB is adequate for single monitor operation at normal desktop resolutions; larger values only benefit multi monitor SPICE configurations that require a separate video device per display.

### 12. Take a baseline snapshot

Once the VM boots cleanly and guest tools are installed, take a snapshot from the VM's Snapshots tab. Give it a name that says what state it captures and a description.

Snapshots in Proxmox live under the VM's own Snapshots tab. VMware-based instructions referring to a separate snapshot manager translate to this tab.

## Verification

Confirm the migration from more than one angle before considering it complete:

- The VM boots to a login prompt or desktop without safe mode or recovery prompts
- On a Windows guest, Get-ComputerInfo reports the expected edition, build, and a firmware type of UEFI, and msinfo32 confirms Secure Boot state and BIOS mode
- On a Linux guest, lsb_release -a reports the expected release
- The Proxmox Hardware tab shows the expected memory, disk size, processor count, firmware, and, for Windows 11, TPM 2.0
- Network connectivity works from inside the guest, tested against the gateway rather than assumed from an interface being up
- The snapshot appears in the Snapshots tab with a timestamp

## Rollback

If a later change breaks the VM, revert from the Snapshots tab: select the snapshot, click Rollback, and confirm. The VM returns to the state captured at that snapshot.

Keep the source VM files on the workstation until the migrated VM has been in use long enough to trust. They are the fallback if the migration has to be redone.

## Cleanup

Once the migration is verified and the source files are no longer needed, remove the staging directory to reclaim space:

   rm -rf /var/lib/vz/tmp/vmware-import

## AI Disclosure

Drafted with AI assistance from my own migration work and notes. The procedures, technical decisions, and verification steps reflect the actual build. See the AI disclosure in the migration README for details on how AI was used during this project.
