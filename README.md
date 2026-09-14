# Steelbyte Homelab

Documentation for a segmented home lab environment: a Dell PowerEdge R320 running Proxmox VE behind a pfSense firewall, with a Cisco Catalyst 3850 core switch and thirteen VLANs separating management, hypervisor, storage, server, user, camera, guest, SOC, and isolated lab traffic.

Everything here is work I did on my own equipment. Some of it started as coursework for CVNP 1601 and CVNP 1606 at Metro State, but the builds, the decisions, and the problems documented in them are mine.

Internal RFC 1918 addressing appears as written throughout, since that space is not routable from outside the network. Public addressing, listening ports, account names, and key material are masked in every document and screenshot.

## Contents

**network-topology.md** The VLAN layout, device inventory, switch port map, and management access paths. Start here. Every other document assumes this environment.

**vm-migration/** Moving two live virtual machines from VMware Workstation on a laptop to Proxmox VE on the R320, preserving the existing Windows 11 installation rather than rebuilding it. Covers the virtual TPM and UEFI requirements that do not survive an import, the storage driver problem that decides which bus the disk attaches to, and a total loss of network on the Linux guest traced to a stale interface name. Includes a runbook written so another technician could repeat the procedure.

**ubuntu-lab-build/** Building and hardening the Ubuntu Server lab host: replacing password authentication with an ed25519 key pair, removing direct root login, adding a brute force countermeasure, and verifying the result against the fully resolved SSH configuration rather than a single config file.

**remote-access/** A WireGuard road warrior VPN on pfSense providing remote access to the lab without exposing the hypervisor management interface, RDP, or SSH to the public internet. The tunnel is configured and the client access rule is scoped to a single VLAN. External validation is not yet complete, and the document says so rather than claiming otherwise.

**glpi/** Standing up GLPI as a ticketing, asset management, and knowledge base system for the lab. Built entirely from the Proxmox host CLI with no file transfer from a workstation, on Debian 13 with Apache, MariaDB, and PHP, served over HTTPS and separated into the directory layout GLPI recommends. Includes the six problems hit along the way, among them a single-character typo in a PHP limit that made the application silently reject form submissions, and a separate write-up of the category tree, templates, and workflow configured through the REST API.

## A Note on Status

Where something is finished, these documents say so and show the evidence. Where something is configured but not yet proven working, they say that instead. A control that has not been observed working is not a control that has been verified, and writing it up as complete would make the rest of the documentation less trustworthy rather than more impressive.
