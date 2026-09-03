# VLAN & Network Segmentation Setup

This document outlines the VLAN configuration, subnets, and firewall rules used to segment the homelab network across the OPNsense router, the TP-Link TL-SG108PE core switch, and the TP-Link Omada EAP650 Wireless AP.

---

## VLAN & Subnet Overview

The network is divided into four primary VLANs to separate management interfaces, trusted devices, externally facing services, and guest access. All VLANs are trunked over the physical `igb0` LAN interface on the router.

| VLAN ID | Interface Name   | Subnet            | DHCP Pool                         | Parent Interface | Description                                            |
| :------ | :--------------- | :---------------- | :-------------------------------- | :--------------- | :----------------------------------------------------- |
| **10**  | `opt1` / MGMT    | `192.168.10.0/24` | `192.168.10.100 - 192.168.10.200` | `igb0` [LAN]     | Management network for hypervisors and infrastructure. |
| **20**  | `opt2` / TRUSTED | `192.168.20.0/24` | `192.168.20.100 - 192.168.20.200` | `igb0` [LAN]     | Primary network for trusted personal devices.          |
| **30**  | `opt3` / DMZ     | `192.168.30.0/24` | *None Configured*                 | `igb0` [LAN]     | Isolated network for externally exposed services.      |
| **40**  | `opt4` / GUEST   | `192.168.40.0/24` | `192.168.40.100 - 192.168.40.200` | `igb0` [LAN]     | Isolated network for guest Internet access.            |

---

## OPNsense Firewall Configuration

### Global Network Alias
To simplify firewall rule creation and ensure strict isolation, an alias is defined to encompass all private IP space.

* **Name:** `RFC1918`
* **Type:** Network(s)
* **Content:** `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`

### Floating Firewall Rules
Traffic routing and isolation are handled via OPNsense floating rules applied across the interfaces.

|  Action   | Protocol     | Source             | Destination   | Destination Port | Description                                           |
| :-------: | :----------- | :----------------- | :------------ | :--------------- | :---------------------------------------------------- |
| **Pass**  | IPv4 *       | `192.168.20.10/32` | MGMT network  | *                | Allow Workstation IP access to MGMT subnet            |
| **Pass**  | IPv4 TCP/UDP | TRUSTED network    | MGMT network  | 53 (DNS)         | Allow TRUSTED DNS access to MGMT                      |
| **Pass**  | IPv4 *       | TRUSTED network    | `! RFC1918`   | *                | Allow TRUSTED Internet access, block private networks |
| **Pass**  | IPv4 TCP/UDP | DMZ network        | DMZ address   | 53 (DNS)         | Allow DMZ DNS access to router                        |
| **Block** | IPv4 *       | DMZ network        | `RFC1918`     | *                | Block DMZ access to internal subnets                  |
| **Pass**  | IPv4 *       | DMZ network        | *             | *                | Allow DMZ outbound Internet                           |
| **Pass**  | IPv4 TCP/UDP | GUEST network      | GUEST address | 53 (DNS)         | Allow GUEST DNS access to router                      |
| **Block** | IPv4 *       | GUEST network      | `RFC1918`     | *                | Block GUEST access to internal subnets                |
| **Pass**  | IPv4 *       | GUEST network      | *             | *                | Allow GUEST outbound internet access                  |
| **Pass**  | IPv4 *       | MGMT network       | *             | *                | Allow MGMT access to all networks and internet        |
| **Pass**  | IPv4 TCP     | MGMT network       | This Firewall | 443 (HTTPS)      | Allow MGMT to access firewall web UI                  |

---

## Layer 2 Switching Configuration (TP-Link TL-SG108PE)

The core switch handles VLAN tagging (802.1Q) and port assignments. Port 1 serves as the primary trunk link connecting back to the OPNsense router's `igb0` interface.

### 802.1Q VLAN Mapping

| VLAN ID | Name    | Member Ports | Tagged Ports | Untagged Ports |
| :-----: | :------ | :----------- | :----------- | :------------- |
|  **1**  | Default | 7            | *None*       | 7              |
| **10**  | MGMT    | 1-6          | 1            | 2-6            |
| **20**  | TRUSTED | 1-2, 8       | 1-2          | 8              |
| **30**  | DMZ     | 1, 4-6       | 1, 4-6       | *None*         |
| **40**  | GUEST   | 1-2          | 1-2          | *None*         |

### PVID (Port VLAN ID) Assignments
Untagged traffic entering the switch is assigned the following PVIDs based on the physical port:

| PVID |  Port(s)  | Assigned Network |
| :--: | :-------: | :--------------- |
|  1   |     7     | Default          |
|  10  | **1 - 6** | MGMT             |
|  20  |   **8**   | TRUSTED          |

---

## Wireless Access Point Configuration (TP-Link Omada EAP650)

The wireless access point is connected to switch Port 2 (tagged for VLANs 20, 40). SSIDs are mapped to their respective VLAN IDs to extend the segmented networks over Wi-Fi.

---
## Migration & Troubleshooting Experience

Migrating from the flat network into a segmented 802.1Q VLAN came with a few lockout situations.

### Switch Configuration Lockouts & Order of Operations
The most significant problem during the transition was the sequence of configuration changes. Applying VLAN tag modifications or updating PVID assignments on the TP-Link switch out of order occasionally removed connectivity to active hosts and required troubleshooting.

* **The PVID Problem:** Changing a switch port's PVID before updating the corresponding host's network interface immediately caused problems connecting to the device. This required switching the port my workstation was connected to so it was on the same subnet, then manually configuring IPv4 to match, then connect and change the hosts network configuration from there.
* **Trunk Port Sensitivity:** Adjusting the tagged member ports on Port 1 (the OPNsense trunk) required can cause problems as well. Misconfiguring this uplink momentarily dropped the router's connection to the switch entirely, bringing down routing, DHCP, and the management web GUI for the entire lab simultaneously. 
* **The Solution:** To prevent lockouts, a strict "top-down" approach proved necessary. 
  1. Configure the OPNsense virtual interfaces, subnets, and floating rules first.
  2. Map the 802.1Q tagged ports on the switch so the infrastructure is aware of the VLANs.
  3. Finally, change the untagged PVIDs port-by-port while immediately releasing/renewing the IP on the connected host device so it goes onto the new subnet without losing its link.