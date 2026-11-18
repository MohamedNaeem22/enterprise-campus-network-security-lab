# Enterprise Campus Network Design & Security Lab

A multi-layer enterprise campus network designed, configured, and verified in **Cisco Packet Tracer**. The topology segments six corporate departments alongside centralized servers and shared network printers, implementing Layer 2 redundancy (LACP + RSTP), split-scope high-availability DHCP, inter-departmental Extended ACLs, NAT/PAT, and device-level security hardening.

---

## Topology Overview

<img width="1667" height="640" alt="topology png" src="https://github.com/user-attachments/assets/32a0a1f2-045b-481b-8607-cd03fa151e7a" />


### Network Architecture & Device Naming
- **Edge & WAN Layer:** `Edge-Router` (Cisco 2811) handling external connectivity, NAT/PAT overload, Router-on-a-Stick (`Fa0/0.70`) for the Servers VLAN, and backup DHCP pools.
- **Core & Distribution Layer:** `Core-SW`, `Dist-SW1`, and `Dist-SW2` (Catalyst 3560-24PS Multilayer Switches with `ip routing` enabled) providing Layer 3 Inter-VLAN routing via SVIs, LACP EtherChannel aggregation, and Rapid PVST+ Root Bridge election.
- **Access Layer:** Catalyst 2960-24TT switches (`ACC-HR-IT-SW`, `ACC-Creative-SW`, `ACC-PR-SW`, `ACC-Sales-SW`, `ACC-Marketing-SW`) connecting department workstations, laptops, and Cisco 7960 IP phones.
- **Servers & Shared Services:** `Server-Dist-SW`, `Servers-SW`, `Admin-SW`, and `Printers-SW` connecting `DHCP Server` (`192.168.100.2`), `HTTP Server`, `ICMP Server` (`192.168.70.10`), admin endpoints, and network printers (`Printer0`–`Printer2`).

---

## VLAN & Addressing Table

| VLAN ID | Segment / Name | Subnet | Default Gateway | Gateway Location |
| :---: | :--- | :--- | :--- | :--- |
| **10** | HR | `192.168.10.0/24` | `192.168.10.1` | `Core-SW` (`Vlan10` SVI) |
| **20** | IT (Data + Voice) | `192.168.20.0/24` | `192.168.20.1` | `Core-SW` (`Vlan20` SVI) |
| **30** | Creative | `192.168.30.0/24` | `192.168.30.1` | `Core-SW` (`Vlan30` SVI) |
| **40** | PR | `192.168.40.0/24` | `192.168.40.1` | `Core-SW` (`Vlan40` SVI) |
| **50** | Sales | `192.168.50.0/24` | `192.168.50.1` | `Core-SW` (`Vlan50` SVI) |
| **60** | Marketing | `192.168.60.0/24` | `192.168.60.1` | `Core-SW` (`Vlan60` SVI) |
| **70** | Servers | `192.168.70.0/24` | `192.168.70.1` | `Edge-Router` (`Fa0/0.70`) |
| **—** | Core-to-Edge P2P | `10.0.0.0/30` | `10.0.0.1` / `10.0.0.2` | `Edge-Router` (`Eth0/2/0`) ↔ `Core-SW` (`Fa0/7`) |
| **—** | Outside / DHCP Link | `192.168.100.0/24` | `192.168.100.1` | `Edge-Router` (`Fa0/1` Outside) ↔ `192.168.100.2` |

---

## Implemented Technologies & Key Configurations

### 1. Switching, EtherChannel & Spanning Tree
- **802.1Q Trunks & LACP EtherChannel:** Bundled inter-switch links into active LACP port channels (`Port-channel1`, `Port-channel2` on `Core-SW`, and `Port-channel3` on Access Switches) using `switchport trunk encapsulation dot1q` and `channel-group ... mode active`.
- **RSTP & Root Bridge:** Configured `spanning-tree mode rapid-pvst` across switches, set `Core-SW` as Root Bridge for VLANs `10,20,30,40,50,60,70` (`priority 24576`), and enabled `spanning-tree portfast` on edge access ports.

### 2. Device Hardening & Port Security
- **Port Security:** Enabled `switchport port-security` (`maximum 2`, `mac-address sticky`, `violation restrict`) on active access ports to support PC + IP Phone connections.
- **Unused Port Hardening:** Administratively shut down unused FastEthernet and GigabitEthernet ports across Core, Distribution, and Access switches (`shutdown`).
- **Basic Device Security:** Configured standardized hostnames, `enable secret`, `service password-encryption`, and `banner motd` across network devices, and removed unused virtual subinterfaces.

### 3. High-Availability DHCP (Split-Scope Design)
- **Primary DHCP Server (`192.168.100.2`):** Reached across VLANs via `ip helper-address 192.168.70.10` and `ip helper-address 192.168.100.2` on each SVI.
- **Backup DHCP on `Edge-Router`:** Configured backup pools (`BACKUP_HR`, `BACKUP_IT`, `BACKUP_CREATIVE`, `BACKUP_PR`, `BACKUP_SALES`, `BACKUP_MARKETING`) with `ip dhcp excluded-address 192.168.X.1 192.168.X.150` on every VLAN so backup leases (`.151–.254`) never conflict with the primary server's pool.

### 4. Inter-Department ACL Security
Extended ACLs are applied **inbound** on the `Core-SW` SVIs (closest to the source) to block restricted inter-department traffic without disrupting Server or Internet access:

| ACL | Applied On | Rule Summary | Purpose |
| :---: | :--- | :--- | :--- |
| **101** | `Vlan60` (`in`) | `deny ip 192.168.60.0/24 192.168.10.0/24`<br>`deny ip 192.168.60.0/24 192.168.20.0/24`<br>`permit ip any any` | Blocks **Marketing → HR** and **Marketing → IT** while permitting Servers & Internet |
| **102** | `Vlan10` (`in`) | `deny ip 192.168.10.0/24 192.168.60.0/24`<br>`permit ip any any` | Blocks **HR → Marketing** while permitting Servers & Internet |

### 5. NAT / PAT (`Edge-Router`)
- **NAT Overload:** Configured `ip nat inside source list 1 interface FastEthernet0/1 overload` matching `access-list 1 permit 192.168.0.0 0.0.255.255`.
- **Interface Roles:** `Fa0/0.70` and `Eth0/2/0` configured as `ip nat inside`; `Fa0/1` (`192.168.100.1`) configured as `ip nat outside`.
- **Routing:** Static summary route `ip route 192.168.0.0 255.255.0.0 10.0.0.2` on `Edge-Router` and default route `ip route 0.0.0.0 0.0.0.0 10.0.0.1` on `Core-SW`.

---

## Verification & Testing

- **DHCP Allocation:** Confirmed workstations across all six department VLANs receive valid IP leases, subnet masks, and gateways automatically.
- **Server Reachability:** Verified successful ICMP connectivity from all departments to `HTTP Server` / `ICMP Server` (`192.168.70.10`).
- **ACL Enforcement:** Confirmed blocked traffic (`Destination host unreachable`) for Marketing → HR, Marketing → IT, and HR → Marketing while keeping Server and external access functional.
- **Trunks & NAT/PAT:** Verified active `802.1Q` LACP trunking via `show interfaces trunk` on `Core-SW` and live PAT translations via `show ip nat translations` on `Edge-Router`.

---

## Device Access Credentials

Use the following password to enter Privileged EXEC mode (`enable`) on `Edge-Router` and all switches when testing the `.pkt` file:

- **Privileged EXEC (`enable secret`):** `Cisco@123`

---

## Repository Contents

- `Enterprise Campus Network Security Lab.pkt` — Cisco Packet Tracer project file
- `topology.png` — Network topology diagram screenshot
- `README.md` — Project documentation

---

## Author

**Mohamed Ahmed Naeem**  
Junior Network Engineer | IT Support Specialist  
- GitHub: [MohamedNaeem22](https://github.com/MohamedNaeem22)  
- Email: `naeemmohamed500@gmail.com`
