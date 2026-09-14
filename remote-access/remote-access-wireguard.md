# Remote Access to the Lab Environment via WireGuard

**Reference:** HOMELAB-VPN-001
**Scope:** Steelbyte homelab, remote access
**Firewall:** pfSense Community Edition
**Status:** Configured. Tunnel established on LAN. External validation still pending.

There is sensitive data contained within this document. All public IP addresses and Internet-facing service listening ports have been represented using asterisks (*) to represent such information. User account names along with keys and/or credentials have also been represented by asterisks. Internal private addresses of subnets have been listed as they appear because RFC 1918 space represents non routable addresses and therefore make the documentation simpler to follow.

---

## Objectives

- Provide remote access to the migrated course VMs from outside the home network
- Avoid exposing the Proxmox management interface, RDP, or SSH to the public internet
- Scope connected clients to the SERVERS VLAN only, with no reach into other segments
- Verify the tunnel against defined criteria before declaring it working

---

## Tools Used

- pfSense Package Manager (WireGuard package installation)
- pfSense VPN > WireGuard (tunnel and peer configuration)
- pfSense Interfaces (interface assignment)
- pfSense Firewall > Rules (WAN and tunnel interface rules)
- pfSense Status > System Logs > Firewall (block log review)
- WireGuard for Windows (client tunnel and log)

No command line tooling was used. This configuration was performed entirely through the pfSense web interface and the WireGuard Windows client.

---

## Configuration

### Ticket Information

- **Reference:** HOMELAB-VPN-001
- **Type:** Self-assigned infrastructure project
- **Firewall:** pfSense, WAN on igb1
- **Tunnel:** tun_wg0, description Home Lab Remote Access
- **Tunnel subnet:** 10.10.55.0/24
- **Firewall tunnel address:** 10.10.55.1/24
- **Assigned interface:** WIREGUARDVPN (opt12)
- **Listening port:** masked
- **Permitted destination:** 10.10.50.0/24 (SERVERS VLAN)
- **Client peer:** laptop, 10.10.55.2/32
- **Dependency:** the VMs documented in ../vm-migration/README.md (HOMELAB-MIG-001)

### Scenario Summary

The course VMs exist on a Proxmox server residing on a segmented home network. They are accessible internally on any of the systems I utilize. They are accessible externally from no location, which is the proper default. I want to access the course VMs remotely from outside the network without changing this status. Thus, establishing a reverse proxy or creating a static port forward to the Proxmox Web Interface or to RDP on the Windows VM are not viable options.

Reverse proxies handle only HTTP and HTTPS traffic. RDP and SSH into the VMs are raw TCP-based communication and cannot be carried by a reverse proxy. More importantly, exposing a hypervisor's management interface or RDP to the internet creates significant exposure. Both are high-value targets and continuously scanned for attacks. Therefore, publishing either of them will allow any entity on the internet to attempt to authenticate against it.

A VPN provides an opposite effect. No services are exposed. The only item that listens on the WAN is the VPN tunnel itself. WireGuard endpoints do not reply to any traffic that cannot be validated as part of a valid cryptographic session. As such, it does not reply to scans like an open RDP or web login would. After the tunnel is brought online, all services on the inside are accessible as if the client were physically present on the home network.

---

## Steps Taken

### Step 1. Install the WireGuard package and enable the service

Action performed in pfSense: System > Package Manager > Available Packages, install WireGuard, then VPN > WireGuard > Settings and enable the service.

**Expected output includes:** WireGuard listed under Installed Packages and a VPN > WireGuard menu entry present.

**Status:** Complete.

### Step 2. Create the tunnel

Action performed in pfSense: VPN > WireGuard > Tunnels > Add Tunnel. Description Home Lab Remote Access, listen port set, tunnel address 10.10.55.1/24.

**Expected output includes:** tun_wg0 listed in the Tunnels tab with its description, public key, listen port, and peer count.

**Status:** Complete. Evidence: 02-wireguard-tunnel.png, showing tun_wg0 assigned to WIREGUARDVPN (opt12) with a single peer. The tunnel public key and listening port are masked in the capture.

### Step 3. Assign the tunnel as its own interface

Action performed in pfSense: Interfaces > Assignments, assign tun_wg0, then Interfaces > WIREGUARDVPN, enable, IPv4 Configuration Type Static IPv4, address 10.10.55.1/24, upstream gateway None.

**Expected output includes:** the interface page showing Enable interface checked, Static IPv4 selected, and 10.10.55.1 with a /24 mask.

**Status:** Complete. Evidence: 04-wireguard-interface.png. Assigning the tunnel rather than leaving it unassigned creates its own firewall rule tab, which is what makes it possible to scope what VPN clients can reach instead of granting full access.

### Step 4. Add the WAN rule permitting the tunnel handshake

Action performed in pfSense: Firewall > Rules > WAN > Add. Protocol UDP, destination WAN address, destination port set to the tunnel listening port.

**Expected output includes:** the rule listed on the WAN tab as a pass rule for UDP to the firewall itself.

**Status:** Complete. Without this rule the first handshake packet would never arrive at the firewall.

### Step 5. Add the tunnel interface rule scoping client access

Action performed in pfSense: Firewall > Rules > WIREGUARDVPN > Add. Source WIREGUARDVPN net, destination 10.10.50.0/24.

**Expected output includes:** the rule listed on the WIREGUARDVPN tab permitting traffic only to the SERVERS VLAN.

**Status:** Complete. Scoping this rule to that single VLAN, rather than any destination, means a connected client will have full connectivity to the lab environment but absolutely no connectivity to any other network segment.

### Step 6. Disable bogon blocking on the tunnel interface

Action performed in pfSense: Interfaces > WIREGUARDVPN, uncheck Block bogon networks.

**Expected output includes:** the bogon block rule no longer present on the WIREGUARDVPN rules tab.

**Status:** Complete. Evidence: 03-bogon-rule.png. Bogons are suspicious because they originate from address space that has not been allocated for use on the public internet. Bogon filtering belongs on a WAN facing interface where unallocated source addresses might legitimately appear. On an internal tunnel interface it only filters legitimate traffic.

### Step 7. Create the client peer

Action performed in pfSense: VPN > WireGuard > Peers > Add Peer. Tunnel tun_wg0, description Laptop, dynamic endpoint, peer public key, pre-shared key, Allowed IP 10.10.55.2/32.

**Expected output includes:** the peer listed under tun_wg0 with its allowed IP.

**Status:** Complete. Evidence: 01-wireguard-peer.png. The /32 is intentional. Each client receives exactly one address inside the tunnel with no ability to claim additional ones. Each peer has its own key pair and a pre-shared key, which is an optional additional layer on top of the normal key exchange. None of that material appears in this document, and none of it belongs in a repository.

### Step 8. Configure the client tunnel in the WireGuard Windows app

Action performed on the laptop: WireGuard for Windows, add a tunnel named Homelab-Laptop with the firewall's public endpoint and an AllowedIPs directive listing only the internal networks to be reached through the tunnel.

**Expected output includes:** the tunnel present in the client with the correct endpoint and a scoped AllowedIPs value rather than 0.0.0.0/0.

**Status:** Complete. Scoping AllowedIPs rather than routing all traffic through the tunnel means only lab traffic crosses the VPN.

### Step 9. Test the tunnel from an external network

Action performed on the laptop: connect to a cellular hotspot, activate the Homelab-Laptop tunnel, review the client log and the pfSense firewall log.

**Expected output includes:** a completed handshake with a recent timestamp.

**Status:** Not yet verified. The handshake did not complete on this attempt and the external test has not been repeated, so this step remains incomplete rather than passed or failed. See Troubleshooting Notes, Issue 3.

---

## Troubleshooting Notes

### Issue 1: Interface description collision

**Incorrect value:** naming the assigned interface WIREGUARD.

**Root cause:** that name was already in use, so pfSense rejected it on save.

**Resolution:** named the interface WIREGUARDVPN instead.

**Result:** the interface saved and appears as WIREGUARDVPN (opt12).

### Issue 2: Duplicate address conflict on a second peer

**Incorrect value:** a second peer configured on the same tunnel while 10.10.55.3/32 was already in use.

**Root cause:** allowed IP entries must be unique between peers on the same tunnel. When they conflict, traffic to the overlapping network routes only to the last peer in the list.

**Resolution:** corrected the address, then later removed that peer entirely. It had been created for my desktop computer, and I determined the desktop would never leave the LAN, so a VPN profile for it served little to no purpose. Removing it was the correct decision, because every credential that exists has potential for leakage.

**Result:** one peer remains on the tunnel, the laptop at 10.10.55.2/32.

### Issue 3: Handshake does not complete from an external network

**Incorrect assumption:** that a configuration verified from inside the LAN is proof the tunnel works from outside it.

**Root cause:** not confirmed. The WireGuard client log shows repeated handshake initiations to the endpoint with no response, retrying and incrementing the try counter, which is the signature of initiation packets not reaching the endpoint rather than being rejected by it. Based on what I have learned about carrier-based UDP filtering on that port, which is common on some mobile networks, I believe this is why the handshake failed to complete.

**Resolution:** proposed, not yet applied. Configure the tunnel to listen on a port less likely to be filtered by the carrier, then test again from an external network.

**Result:** Open. This issue stays open because the remaining step has not been performed, not because the build is known to be broken. Evidence: 05-handshake-failure.png and 06-firewall-block-log.png. Until I successfully complete a handshake from outside my network, I consider remote-access functionality unproven; thus, documenting otherwise would be inaccurate.

---

## Verification Methodology

When testing resumes, confirmation that the tunnel is functioning correctly is obtained when all of the following conditions are met:

- Handshake is completed, with a recent timestamp reflecting a successful handshake
- Laptop can connect to the Proxmox server
- Laptop can connect to the course VMs
- Laptop cannot connect to VLANs outside of the specified scope, indicating the pass rule is effective

Note that the last point is equally important as the previous three points. A VPN that affords greater levels of access than intended is considered a finding, not a success.

---

## Portfolio Card

Designed and configured a WireGuard road warrior VPN on pfSense to provide remote access to a lab environment without exposing a hypervisor management interface, RDP, or SSH to the public internet. Assigned the tunnel as its own firewall interface so client access could be scoped by rule to a single VLAN rather than granted network wide, and disabled bogon filtering where it would have blocked legitimate internal traffic. Issued each peer its own key pair and pre-shared key with a /32 allowed IP, and removed a peer whose credential served no purpose once the machine was determined never to leave the LAN. Diagnosed a failed external handshake down to a probable carrier UDP filter, defined the verification criteria the build must meet before it is considered working, and documented the feature as unproven rather than complete.

---

## AI Use Statement

AI assisted with the structure and initial draft of this document. The configuration described, the technical decisions behind it, and the verification criteria reflect the actual build. See the AI disclosure in the Proxmox migration README for details on how AI was used during this project.

---

## Files In This Folder

- remote-access-wireguard.md, this document
- ../vm-migration/README.md, the migration write-up this VPN provides remote access to
- ../network-topology.md, the VLAN layout the tunnel rule is scoped against
- screenshots/01-wireguard-peer.png, pfSense peer configuration for the laptop
- screenshots/02-wireguard-tunnel.png, the Tunnels tab showing tun_wg0
- screenshots/03-bogon-rule.png, the bogon block rule
- screenshots/04-wireguard-interface.png, the WIREGUARDVPN interface at 10.10.55.1/24
- screenshots/05-handshake-failure.png, WireGuard client log showing repeated failed handshakes
- screenshots/06-firewall-block-log.png, pfSense firewall block log during the failed test
