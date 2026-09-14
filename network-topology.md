# Steelbyte Homelab Network Topology

Segmented home lab and network architecture. Reflects the network state following the March 2026 cutover, when the TP-Link was converted to access point mode and all addressing moved onto the 10.10.0.0/16 VLAN scheme.

Internal RFC 1918 addressing is shown as written, since that space is not routable from outside the network and masking it would make the document unusable. Public addressing, listening ports, external hostnames, account names, and key material do not appear here.

## Overview

pfSense serves as the perimeter firewall and inter-VLAN router. A Cisco Catalyst 3850 is the core switch. A TP-Link Archer AX4400 runs in access point mode, providing wireless onto an existing VLAN rather than routing its own network. Thirteen VLANs separate management, switching infrastructure, hypervisors, storage, servers, users, office client access, cameras, guests, administration, SOC tooling, and isolated lab traffic.

Every other document in this repository assumes this layout. When a write-up says a machine sits on VLAN 50, or that a firewall rule is scoped to 10.10.50.0/24, this is what it refers to.

| Field | Value |
| --- | --- |
| Environment | Steelbyte homelab |
| Core firewall | pfSense |
| Core switch | Cisco Catalyst 3850, STEELBYTE-SW-01 |
| Wireless | TP-Link Archer AX4400, access point mode |
| Hypervisor | Dell PowerEdge R320, Proxmox VE, node STEELBYTE-R320 |
| Primary addressing model | 10.10.x.0/24 VLAN networks |

## Core Physical Topology

    Internet
      |
    Spectrum EU2251 Modem (bridged)
      |
    pfSense Firewall / Router
      |
    Cisco Catalyst 3850 Core Switch, STEELBYTE-SW-01
      |
      +-- VLAN 10,  MGMT
      +-- VLAN 20,  SWITCH_INFRA
      +-- VLAN 30,  HYPERVISOR
      +-- VLAN 40,  STORAGE
      +-- VLAN 50,  SERVERS
      +-- VLAN 60,  USERS
      +-- VLAN 65,  TPLINK_TRANSIT
      +-- VLAN 70,  CAMERAS
      +-- VLAN 80,  GUESTS
      +-- VLAN 90,  ADMIN
      +-- VLAN 100, SOC
      +-- VLAN 200, DIRTY_LAB
      +-- VLAN 999, BLACKHOLE

## VLAN and Gateway Layout

| VLAN | Name | Gateway | Purpose | In Use |
| --- | --- | --- | --- | --- |
| 10 | MGMT | 10.10.10.1 | General management network | Defined |
| 20 | SWITCH_INFRA | 10.10.20.1 | Switch and infrastructure management | Yes |
| 30 | HYPERVISOR | 10.10.30.1 | Hypervisor management | Defined |
| 40 | STORAGE | 10.10.40.1 | Storage network | Defined |
| 50 | SERVERS | 10.10.50.1 | Servers and application hosts | Yes |
| 60 | USERS | 10.10.60.1 | User and client access network | Yes |
| 65 | TPLINK_TRANSIT | 10.10.65.1 | Office clients, wireless, cameras, printer | Yes |
| 70 | CAMERAS | 10.10.70.1 | Camera segment | Defined |
| 80 | GUESTS | 10.10.80.1 | Guest network | Defined |
| 90 | ADMIN | 10.10.90.1 | Administrative workstation segment | Defined |
| 100 | SOC | 10.10.100.1 | SOC, monitoring, and security tooling | Defined |
| 200 | DIRTY_LAB | 10.10.200.1 | Isolated lab and dirty side testing segment | Defined |
| 999 | BLACKHOLE | No gateway | Disabled and unused switch ports | Yes |

Several VLANs are provisioned but not yet populated. They exist so the addressing plan is settled before the workloads arrive rather than being retrofitted later, but a reader should not assume every segment above carries traffic today.

Two naming notes. VLAN 65 is still named TPLINK_TRANSIT in pfSense, from when the TP-Link ran as a downstream router and VLAN 65 carried its WAN side. Since the conversion to access point mode it functions as the office client and wireless segment. The name is stale, the function is current. Cameras sit on VLAN 65 rather than VLAN 70 because they are wireless and the access point already serves that segment.

## DHCP Pools

| Interface | Pool |
| --- | --- |
| SERVERS, VLAN 50 | 10.10.50.100 to 10.10.50.200 |
| USERS, VLAN 60 | 10.10.60.100 to 10.10.60.200 |
| TPLINK_TRANSIT, VLAN 65 | 10.10.65.100 to 10.10.65.200 |

Static assignments are made as DHCP reservations outside the pool range. pfSense enforces this, and the reason is sound: a reservation inside the pool does not actually remove the address from circulation, so another device could be handed it while the reserved host is offline.

## Core Devices

| Device | Role | Network / Addressing |
| --- | --- | --- |
| Spectrum EU2251 modem | ISP handoff, bridged | Feeds pfSense WAN |
| pfSense firewall | Perimeter router, inter-VLAN firewall, DHCP, DNS | VLAN gateways on 10.10.x.1 networks |
| Cisco Catalyst 3850, STEELBYTE-SW-01 | Core switch | Management SVI on VLAN 20, 10.10.20.2 |
| TP-Link Archer AX4400 | Wireless access point | VLAN 65 |
| Dell PowerEdge R320 | Proxmox VE host | VLAN 50 |
| Laptop | Office and administration client | 10.10.65.10 |
| Desktop | Office and administration client | 10.10.65.11 |
| Printer | Network printer | 10.10.65.106 |
| Cameras | Wireless camera devices | VLAN 65 via the access point |

Switch management was moved off VLAN 1. The 3850 originally answered on the default VLAN at 192.168.1.2 and now sits on VLAN 20 at 10.10.20.2. Leaving management on VLAN 1 means it is reachable from any access port that was never explicitly assigned, since VLAN 1 is what an unconfigured port falls into.

## Virtual Machines

| VMID | Name | Guest | Address | Purpose |
| --- | --- | --- | --- | --- |
| 101 | CVNP1606-LAB-Win11 | Windows 11 Enterprise Evaluation | VLAN 50 | Coursework lab endpoint |
| 102 | CVNP1601-LAB-UBUNTU | Ubuntu Server 24.04 LTS | 10.10.50.103 | Linux lab host, see ubuntu-lab-build/ |
| 200 | Ubuntu-RIP-01 | Ubuntu | 10.10.50.100 | Jellyfin media host |
| 201 | GLPI-01 | Debian 13 | 10.10.50.201 | GLPI ITSM, asset and ticket management |

VM addressing follows a convention of matching the last octet to the VMID where the address is statically reserved. VMID 201 at 10.10.50.201 is the clearest example.

## Cisco Catalyst 3850 Port Map

| Port | Mode | VLANs / Assignment | Connected Device / Purpose |
| --- | --- | --- | --- |
| Gi1/0/1 | 802.1Q trunk | 1,10,20,30,40,50,60,65,70,80,90,100,200 | pfSense trunk |
| Gi1/0/11 | Access | VLAN 60, USERS | User and client access port |
| Gi1/0/17 | Access | VLAN 65, TPLINK_TRANSIT | TP-Link access point uplink |
| Gi1/0/45 | Access | VLAN 50, SERVERS | R320 server link |
| Gi1/0/47 | Access | VLAN 50, SERVERS | R320 second server link |
| Gi1/0/12 | Access, shutdown | VLAN 999, BLACKHOLE | Unused |
| Gi1/0/2-10, Gi1/0/13-16, Gi1/0/18-44, Gi1/0/46, Gi1/0/48, Gi1/1/1-2, Te1/1/3-4 | Access, shutdown | VLAN 999, BLACKHOLE | Unused ports |

Unused ports are assigned to an unrouted VLAN and administratively shut rather than simply left alone. A port in the default VLAN with no configuration is a live network drop for anyone who plugs into it. Assigning it to a blackhole VLAN with no gateway and shutting it means a cable in an unused port reaches nothing.

## Remote Access

A WireGuard tunnel on pfSense provides remote access to the lab from outside the network. It is not a VLAN and does not appear in the table above, but it is a routable network on this firewall.

| Item | Value |
| --- | --- |
| Tunnel | tun_wg0, Home Lab Remote Access |
| Assigned interface | WIREGUARDVPN, opt12 |
| Tunnel network | 10.10.55.0/24, firewall at 10.10.55.1 |
| Listening port | Masked |
| Permitted destination | 10.10.50.0/24 only |

Connected clients reach the SERVERS VLAN and nothing else. The interface is assigned rather than left unassigned specifically so it has its own firewall rule tab, which is what makes that scoping possible. Full configuration and current status are in remote-access/remote-access-wireguard.md.

## Internet Facing Services

Two inbound paths exist. Everything else is denied at the perimeter.

The WireGuard tunnel above, listening on UDP. A WireGuard endpoint does not respond to traffic it cannot cryptographically validate as belonging to an established session, so it does not answer scans the way an exposed login page does.

A single HTTPS port forward to a Caddy reverse proxy, which fronts the Jellyfin media host on VLAN 50. Caddy handles TLS certificate issuance and renewal automatically. The external hostname is not published here.

The split is deliberate. Services genuinely intended for outside consumption go through the reverse proxy, where exactly one port is open and one process terminates TLS. Anything administrative goes through the VPN instead, which places the client inside the network before it can reach a management interface at all. Neither the Proxmox web interface, nor the pfSense web interface, nor SSH on any host is reachable from the internet.

## Management Access Paths

| Source Network | Destination | Protocol / Port | Purpose |
| --- | --- | --- | --- |
| VLAN 65 | pfSense, 10.10.60.1 | HTTPS 8443 | pfSense web interface |
| VLAN 65 | pfSense | SSH 22 | Firewall administration, key based only |
| VLAN 65 | Cisco 3850, 10.10.20.2 | SSH 22 | Core switch administration |
| VLAN 65 | Server network, 10.10.50.0/24 | SSH 22 | Server administration |
| VLAN 65 | Jellyfin host, 10.10.50.100 | HTTP 8096 | Local application access |

Management access originates from the office client segment rather than from every VLAN, and pfSense SSH accepts key based authentication only, with passwords disabled. Nothing in this table is reachable from outside the network. Remote administration goes through the VPN, which places a connected client inside the network first rather than exposing any of these services to the internet.

## Segmentation Rationale

| Segment | Reason |
| --- | --- |
| MGMT | Dedicated management plane |
| SWITCH_INFRA | Switch and infrastructure administration, off VLAN 1 |
| HYPERVISOR | Hypervisor management separation |
| STORAGE | Storage traffic separation |
| SERVERS | Server and application workload isolation |
| USERS | User and client access segmentation |
| TPLINK_TRANSIT | Office clients, wireless, cameras, and printer |
| CAMERAS | Camera isolation, provisioned |
| GUESTS | Guest device isolation |
| ADMIN | Administrative workstation separation |
| SOC | Security monitoring and SOC tooling |
| DIRTY_LAB | Isolated lab and dirty side testing |
| BLACKHOLE | Disabled unused switch ports |

## Summary Diagram

                          Internet
                             |
                    Spectrum EU2251 Modem
                             |
                      pfSense Firewall
              Perimeter Router / Inter-VLAN Firewall
                             |
                     802.1Q Trunk to Core
                             |
              Cisco Catalyst 3850, STEELBYTE-SW-01
                             |
      +---------+---------+---------+---------+---------+---------+
      |         |         |         |         |         |         |
    VLAN 10  VLAN 20   VLAN 30   VLAN 40   VLAN 50   VLAN 60   VLAN 65
     MGMT    SWITCH    HYPERV.   STORAGE   SERVERS    USERS    TPLINK
             INFRA                                             TRANSIT
                |                             |                    |
           3850 Mgmt                   R320 / Proxmox          TP-Link AP
           10.10.20.2                  VM 101 Win11            Laptop
                                       VM 102 Ubuntu           10.10.65.10
                                       VM 200 Jellyfin         Desktop
                                       10.10.50.100            10.10.65.11
                                       VM 201 GLPI             Printer
                                       10.10.50.201            10.10.65.106
                                                               Cameras

      +---------+---------+---------+---------+
      |         |         |         |         |
    VLAN 70  VLAN 80   VLAN 90  VLAN 100  VLAN 200
    CAMERAS  GUESTS     ADMIN     SOC     DIRTY_LAB
