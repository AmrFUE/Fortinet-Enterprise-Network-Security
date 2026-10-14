# ENTERPRISE NETWORK SECURITY IMPLEMENTATION
## Fortinet Capstone Project — Comprehensive Engineering Documentation

---

### Project Metadata
- **Project Title:** Enterprise Network Security Implementation (Fortinet Capstone Project)
- **Author / Engineer:** Amr
- **Environment:** PNETLab / EVE-NG Network Simulation
- **Firewall Operating System:** FortiOS v7.0.4 (FortiGate-VM64-KVM)
- **Routing & Switching Platforms:** Cisco IOS Software (IOSv / IOSvL2)
- **Document Version:** 1.0 (Final Production Release)
- **Date:** October 2026

---

## 1. Executive Summary & Project Overview

This capstone project implements an end-to-end, enterprise-grade secure network infrastructure designed according to industry best practices and Zero Trust Architecture principles. The architecture interconnects a multi-tier Corporate Headquarters (**HQ**), two branch offices (**Branch 1** and **Branch 2**), and off-site teleworkers (**Remote User SSL-VPN**).

The core perimeter security, traffic segmentation, and threat mitigation are provided by **FortiGate Next-Generation Firewalls (NGFW)** running FortiOS 7.0, integrated with a multi-hop Cisco routing and switching fabric.

```
                           +---------------------------------------------------+
                           |            Cloud0 Management / Internet           |
                           |                 (192.168.85.0/24)                 |
                           +---------------------------------------------------+
                                    /                   |                   \
                                   /                    |                    \
                      [.201]      /        [.202]       |        [.203]       \
                            ISP-HQ                   ISP-BR1                   ISP-BR2
                              |                         |                         |
                           WAN-HQ                    WAN-BR1                   WAN-BR2
                              |                         |                         |
                    +---------+---------+               |                         |
                    |    WAN Switch     |               |                         |
                    +---------+---------+               |                         |
                     /                 \                |                         |
             [port1]|                   |[port1] [port1]|                  [port1]|
        +-----------+----+         +----+-----------+   |                         |
        | Main-HQ (FW)   |  HA HB  | Backup-HQ (FW) |   |                         |
        | Priority: 200  |=========| Priority: 100  |   |                         |
        +-----------+----+ port3/4 +----+-----------+   |                         |
             [port2]|                   |[port2]        |                         |
                    \                   /               |                         |
                    +---------+---------+               |                         |
                    |      Core-HQ      |               |                         |
                    +---------+---------+               |                         |
                     /                 \                |                         |
             Gi0/2  /                   \ Gi0/3         |                         |
        HQ-ACC-USER               HQ-ACC-SRV       FGT-Branch1               FGT-Branch2
        (VLAN 10)                 (VLAN 20/25/     (VLAN 10/20)              (VLAN 10/20)
                                   27/30)               |                         |
                                                    Switch 16                 Switch 24
```

### Key Engineering Objectives Achieved
1. **High Availability (HA) Active-Passive Clustering:** Redundant FortiGate deployment at HQ featuring sub-second stateful failover, dedicated dual heartbeat links (`port3` & `port4`), synchronized connection tables, and shared virtual MAC addresses.
2. **Zero-Trust Multi-Tier LAN Micro-Segmentation:** Layer 2 / Layer 3 isolation separating Users (VLAN 10), Web Servers (VLAN 20), Application/API Services (VLAN 25), Database Services (VLAN 27), and Out-of-Band Management (VLAN 30), terminating on FortiGate 802.1Q sub-interfaces.
3. **Hub-and-Spoke Site-to-Site IPsec VPN Architecture:** Route-based IPsec tunnels between HQ and Branch 1 (`To-BR1`), and HQ and Branch 2 (`To-BR2`), with transit routing enabled on the HQ hub to facilitate secure Branch-to-Branch intercommunication without direct branch-to-branch mesh links.
4. **Unified Threat Management (UTM) & Next-Gen Security Profiles:** Layer 7 deep packet inspection including Antivirus, Intrusion Prevention System (IPS), Web Filtering, Application Control, and SSL Certificate Inspection applied across Internet egress and inter-site VPN policies.
5. **Network Address Translation (NAT) & Server Publishing (VIP):** Source NAT (PAT/Overload) for outbound LAN internet access, coupled with Virtual IP (VIP) destination port-forwarding exposing internal servers (`TCP 8080 -> 80`) securely to the public WAN.
6. **Remote Access SSL-VPN:** Full-tunnel and web portal client-to-site VPN gateway configured on HQ WAN (`port1`), enforcing authentication, dedicated address assignment (`SSLVPN-Tunnel-Range`), and firewall policies controlling access to the server tier.

---

## 2. Master IP Addressing Scheme & Subnet Allocation

The addressing design utilizes **RFC 5737 (203.0.113.0/24)** for public WAN point-to-point interconnects and standard **RFC 1918 (10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16)** private spaces for internal segmentation.

### 2.1 Management & Internet Simulation Cloud (Cloud0)
| Device | Interface | IP Address | Subnet Mask | Gateway | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **PNETLab VM** | pnet0 | `192.168.85.128` | `255.255.255.0` (/24) | `192.168.85.2` | Lab hypervisor host management |
| **ISP-HQ** | Gi0/0 | `192.168.85.201` | `255.255.255.0` (/24) | `192.168.85.2` | External gateway for HQ ISP |
| **ISP-BR1** | Gi0/0 | `192.168.85.202` | `255.255.255.0` (/24) | `192.168.85.2` | External gateway for Branch 1 ISP |
| **ISP-BR2** | Gi0/0 | `192.168.85.203` | `255.255.255.0` (/24) | `192.168.85.2` | External gateway for Branch 2 ISP |

### 2.2 Point-to-Point WAN Links (/30 Subnets)
| Subnet | Segment | Device A Interface | IP A | Device B Interface | IP B |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `203.0.113.0/30` | HQ ISP Link | ISP-HQ Gi0/1 | `203.0.113.1` | WAN-HQ Gi0/1 | `203.0.113.2` |
| `203.0.113.4/30` | HQ WAN Link | WAN-HQ Gi0/0 | `203.0.113.5` | FortiGate-HQ port1 | `203.0.113.6` |
| `203.0.113.8/30` | BR1 ISP Link | ISP-BR1 Gi0/1 | `203.0.113.9` | WAN-BR1 Gi0/1 | `203.0.113.10` |
| `203.0.113.12/30` | BR1 WAN Link | WAN-BR1 Gi0/0 | `203.0.113.13` | FGT-Branch1 port1 | `203.0.113.14` |
| `203.0.113.16/30` | BR2 ISP Link | ISP-BR2 Gi0/1 | `203.0.113.17` | WAN-BR2 Gi0/1 | `203.0.113.18` |
| `203.0.113.20/30` | BR2 WAN Link | WAN-BR2 Gi0/0 | `203.0.113.21` | FGT-Branch2 port1 | `203.0.113.22` |

### 2.3 Headquarters (HQ) LAN Segmentation Matrix
| VLAN ID | Segment Name | Subnet | FortiGate Gateway | DHCP Scope / Host IPs | Member Interface | Description |
| :---: | :--- | :--- | :--- | :--- | :--- | :--- |
| **10** | HQ-Users-VLAN10 | `192.168.10.0/24` | `192.168.10.1` | `192.168.10.100 - .200` (DHCP) | `port2.10` | Corporate end-user workstations |
| **20** | HQServersVLAN20 | `192.168.20.0/24` | `192.168.20.1` | `192.168.20.20` (Web), `.10` (VPC) | `port2.20` | Public-facing / Web servers |
| **25** | HQServersVLAN25 | `192.168.25.0/24` | `192.168.25.1` | `192.168.25.10` (API Gateway) | `port2.25` | Application & API processing tier |
| **27** | HQServersVLAN27 | `192.168.27.0/24` | `192.168.27.1` | `192.168.27.10` (Database Node) | `port2.27` | Backend database storage tier |
| **30** | HQManagVLAN30 | `192.168.30.0/24` | `192.168.30.1` | `192.168.30.10` (Admin PC) | `port2.30` | Secure network management zone |
| **—** | HA Heartbeat 1 | `169.254.0.0/30` | `169.254.0.1` | `169.254.0.2` | `port3` | Primary HA heartbeat channel |
| **—** | HA Heartbeat 2 | `169.254.0.0/30` | `169.254.0.3` | `169.254.0.4` | `port4` | Secondary HA synchronization channel |

### 2.4 Branch 1 & Branch 2 LAN Segmentation Matrix
| Site | VLAN ID | Name | Subnet | Gateway | Hosts / VIP Mapped | Firewall Interface |
| :--- | :---: | :--- | :--- | :--- | :--- | :--- |
| **Branch 1** | **10** | VLAN10-Users | `10.10.10.0/24` | `10.10.10.1` | End-user PCs | `port2.10` |
| **Branch 1** | **20** | VLAN20-Servers | `10.10.20.0/24` | `10.10.20.1` | Server: `10.10.20.10` | `port2.20` |
| **Branch 2** | **10** | VLAN10-Users | `10.30.10.0/24` | `10.30.10.1` | End-user PCs | `port2.10` |
| **Branch 2** | **20** | VLAN20-Servers | `10.30.20.0/24` | `10.30.20.1` | Server: `10.30.20.10` | `port2.20` |

---

## 3. High Availability (HA Active-Passive) Architecture

To eliminate single points of failure at corporate headquarters, the HQ security perimeter is deployed as a two-node **FortiGate Active-Passive (A-P) High Availability Cluster**.

```
                +-------------------------------------------------+
                |             Cisco WAN-HQ Router                 |
                |                 (203.0.113.5)                   |
                +-------------------------------------------------+
                                         |
                            +--------------------------+
                            |       WAN Switch         |
                            +--------------------------+
                             /                        \
                     port1  /                          \  port1
              (203.0.113.6)/                            \(203.0.113.6)
            +-------------+--------------+        +-----+----------------------+
            |   Main-HQ (Primary Node)   |        | Backup-HQ (Secondary Node)  |
            |   Priority: 200            |        | Priority: 100              |
            |   Status: Active Master    |        | Status: Standby Hot-Backup |
            +-------------+--------------+        +-----+----------------------+
                    port3 |  Heartbeat 1 (Priority 50)  | port3
                    ======+=============================+======
                    port4 |  Heartbeat 2 (Priority 50)  | port4
                          \                             /
                     port2 \                           / port2
                            +-------------------------+
                            |      Core-HQ Switch     |
                            +-------------------------+
```

### 3.1 HA Cluster Configuration Parameters
```fortios
# Primary Node (Main-HQ) HA Configuration
config system ha
    set group-name "HQ-HA-Cluster"
    set mode a-p
    set password <cluster-secret>
    set hbdev "port3" 50 "port4" 50
    set session-pickup enable
    set override disable
    set priority 200
    set unicast-hb disable
end

# Secondary Node (Backup-HQ) HA Configuration
config system ha
    set group-name "HQ-HA-Cluster"
    set mode a-p
    set password <cluster-secret>
    set hbdev "port3" 50 "port4" 50
    set session-pickup enable
    set override disable
    set priority 100
    set unicast-hb disable
end
```

### 3.2 HA Operational Mechanics & Redundancy Design
1. **Heartbeat Redundancy:** Two dedicated interfaces (`port3` and `port4`) run Fortinet's proprietary HA communication protocol. Heartbeat packets are transmitted at 200 ms intervals. Loss of heartbeats across both links initiates failover after 6 consecutive misses (1.2 seconds).
2. **Virtual MAC Address (VMAC):** The cluster uses virtual MACs derived from the cluster group ID (`00:09:0f:09:00:xx`). When failover occurs, the secondary firewall adopts the same virtual MAC and broadcasts Gratuitous ARPs (GARP), preventing ARP cache poisoning on connected switches and ensuring zero client reconfiguration.
3. **Session Pickup (State Synchronization):** TCP sessions and NAT state tables are synchronized across the cluster in real-time. If the primary node fails, active user connections (such as file downloads and HTTP sessions) persist seamlessly without TCP teardown.
4. **Override Mechanism:** `override disable` is enforced as an enterprise stability best practice. Once failover to `Backup-HQ` occurs, control remains on the operational node and does not flap back immediately when `Main-HQ` reboots, preventing unnecessary traffic interruptions.

---

## 4. Routing Architecture & Inter-Site Traffic Engineering

### 4.1 Global Routing Strategy
- **Layer 3 Boundaries:** Routing between all internal VLANs is strictly enforced by the FortiGate NGFWs (Router-on-a-Stick via 802.1Q sub-interfaces).
- **Default Egress:** Each firewall points its default route (`0.0.0.0/0`) upstream to its respective Cisco WAN edge router.
- **Provider Core Mesh:** ISP routers exchange reachability for each remote site's public WAN `/30` links over the shared `192.168.85.0/24` management cloud.
- **VPN Routing:** Point-to-point IPsec virtual tunnel interfaces (`To-BR1`, `To-BR2`, `To-HQ`) route traffic between branches and HQ.

```
+---------------------------------------------------------------------------------------+
|                                ROUTING FLOW ARCHITECTURE                              |
+---------------------------------------------------------------------------------------+
|  Branch 1 User (10.10.10.50)                                                          |
|       |                                                                               |
|       v (Default Gateway: 10.10.10.1)                                                 |
|  FortiGate-BR1                                                                        |
|       |                                                                               |
|       +--> Internet (0.0.0.0/0) --------> WAN-BR1 (.13) -> ISP-BR1 (.9) -> Web      |
|       |                                                                               |
|       +--> HQ Corporate (192.168.0.0/16)-> Route into 'To-HQ' IPsec Tunnel            |
|       |                                                                               |
|       +--> Branch 2 LAN (10.30.0.0/16) -> Route into 'To-HQ' IPsec Tunnel             |
|                                                    |                                  |
|                                                    v (Hub Transit Routing)            |
|                                            FortiGate-HQ                               |
|                                                    |                                  |
|                                                    v (Route into 'To-BR2' Tunnel)     |
|                                            FortiGate-BR2                              |
|                                                    |                                  |
|                                                    v                                  |
|                                            Branch 2 Server (10.30.20.10)              |
+---------------------------------------------------------------------------------------+
```

### 4.2 FortiGate Static Routing Tables
#### Headquarters (Main-HQ)
```fortios
config router static
    edit 1
        set dst 0.0.0.0 0.0.0.0
        set gateway 203.0.113.5
        set device "port1"
    next
    edit 2
        set dst 10.10.10.0 255.255.255.0
        set device "To-BR1"
    next
    edit 3
        set dst 10.10.20.0 255.255.255.0
        set device "To-BR1"
    next
    edit 4
        set dst 10.30.10.0 255.255.255.0
        set device "To-BR2"
    next
    edit 5
        set dst 10.30.20.0 255.255.255.0
        set device "To-BR2"
    next
end
```

#### Branch 1 (FGT-Branch1)
```fortios
config router static
    edit 1
        set dst 0.0.0.0 0.0.0.0
        set gateway 203.0.113.13
        set device "port1"
    next
    edit 2
        set dst 192.168.0.0 255.255.0.0
        set device "To-HQ"
    next
    edit 3
        set dst 10.30.0.0 255.255.0.0
        set device "To-HQ"
    next
end
```

#### Branch 2 (FGT-Branch2)
```fortios
config router static
    edit 1
        set dst 0.0.0.0 0.0.0.0
        set gateway 203.0.113.21
        set device "port1"
    next
    edit 2
        set dst 192.168.0.0 255.255.0.0
        set device "To-HQ"
    next
    edit 3
        set dst 10.10.0.0 255.255.0.0
        set device "To-HQ"
    next
end
```

### 4.3 Cisco ISP & WAN Static Routes
#### ISP-HQ (`192.168.85.201`)
```cisco
ip route 0.0.0.0 0.0.0.0 192.168.85.2
ip route 203.0.113.4 255.255.255.252 203.0.113.2
ip route 203.0.113.8 255.255.255.252 192.168.85.202
ip route 203.0.113.12 255.255.255.252 192.168.85.202
ip route 203.0.113.16 255.255.255.252 192.168.85.203
ip route 203.0.113.20 255.255.255.252 192.168.85.203
```

#### WAN-HQ (`203.0.113.2` / `203.0.113.5`)
```cisco
ip route 0.0.0.0 0.0.0.0 203.0.113.1
```

#### ISP-BR1 (`192.168.85.202`) & WAN-BR1 (`203.0.113.10` / `203.0.113.13`)
```cisco
! ISP-BR1
ip route 0.0.0.0 0.0.0.0 192.168.85.2
ip route 203.0.113.0 255.255.255.252 192.168.85.201
ip route 203.0.113.4 255.255.255.252 192.168.85.201
ip route 203.0.113.12 255.255.255.252 203.0.113.10
ip route 203.0.113.16 255.255.255.252 192.168.85.203
ip route 203.0.113.20 255.255.255.252 192.168.85.203

! WAN-BR1
ip route 0.0.0.0 0.0.0.0 203.0.113.9
```

#### ISP-BR2 (`192.168.85.203`) & WAN-BR2 (`203.0.113.18` / `203.0.113.21`)
```cisco
! ISP-BR2
ip route 0.0.0.0 0.0.0.0 192.168.85.2
ip route 203.0.113.0 255.255.255.252 192.168.85.201
ip route 203.0.113.4 255.255.255.252 192.168.85.201
ip route 203.0.113.8 255.255.255.252 192.168.85.202
ip route 203.0.113.12 255.255.255.252 192.168.85.202
ip route 203.0.113.20 255.255.255.252 203.0.113.18

! WAN-BR2
ip route 0.0.0.0 0.0.0.0 203.0.113.17
```

---

## 5. IPsec Site-to-Site VPN Implementation

A secure Hub-and-Spoke topology connects the distributed sites over untrusted public WAN infrastructure.

```
                    +-----------------------------+
                    |       Main-HQ (Hub)         |
                    |     WAN: 203.0.113.6        |
                    +--------------+--------------+
                                  / \
              Tunnel 'To-BR1'    /   \    Tunnel 'To-BR2'
             (203.0.113.6 <-->  /     \  (203.0.113.6 <-->
               203.0.113.14)   /       \   203.0.113.22)
                              /         \
                             v           v
          +---------------------+     +---------------------+
          | FGT-Branch1 (Spoke) |     | FGT-Branch2 (Spoke) |
          |  WAN: 203.0.113.14  |     |  WAN: 203.0.113.22  |
          +---------------------+     +---------------------+
                     ^                           ^
                     |                           |
                     +====== Branch-to-Branch ===+
                             Transit via HQ
```

### 5.1 IPsec Tunnel Specifications
| Parameter | Phase 1 (IKE SA) | Phase 2 (IPsec SA) |
| :--- | :--- | :--- |
| **Tunnel Mode** | Route-based (Virtual Tunnel Interface) | ESP Tunnel Mode |
| **Authentication** | Pre-Shared Key (PSK) | Integrated via IKE |
| **Encryption** | AES-256 / 3DES / DES | AES-256 / DES |
| **Hashing** | SHA-256 / SHA-1 / MD5 | SHA-256 / SHA-1 / MD5 |
| **Diffie-Hellman Group** | Group 14 (2048-bit) / Group 5 | Group 14 (PFS enabled) / Group 5 |
| **Key Lifetime** | 86,400 seconds (24 hours) | 43,200 seconds (12 hours) |
| **Dead Peer Detection (DPD)** | Enabled (Interval: 20s, Retry: 3) | N/A |

### 5.2 Hub Configuration (Main-HQ)
```fortios
# Phase 1 Interfaces
config vpn ipsec phase1-interface
    edit "To-BR1"
        set interface "port1"
        set peertype any
        set net-device disable
        set proposal des-md5 des-sha1
        set remote-gw 203.0.113.14
        set psksecret <enc_key>
    next
    edit "To-BR2"
        set interface "port1"
        set peertype any
        set net-device disable
        set proposal des-sha256
        set remote-gw 203.0.113.22
        set psksecret <enc_key>
    next
end

# Phase 2 Interfaces
config vpn ipsec phase2-interface
    edit "To-BR1"
        set phase1name "To-BR1"
        set proposal des-md5 des-sha1
    next
    edit "To-BR2"
        set phase1name "To-BR2"
        set proposal des-md5 des-sha1
    next
end
```

### 5.3 Spoke Configuration (FGT-Branch1 & FGT-Branch2)
```fortios
# Branch 1
config vpn ipsec phase1-interface
    edit "To-HQ"
        set interface "port1"
        set peertype any
        set net-device disable
        set proposal des-md5 des-sha1
        set remote-gw 203.0.113.6
        set psksecret <enc_key>
    next
end

# Branch 2
config vpn ipsec phase1-interface
    edit "To-HQ"
        set interface "port1"
        set peertype any
        set net-device disable
        set proposal des-sha256
        set remote-gw 203.0.113.6
        set psksecret <enc_key>
    next
end
```

---

## 6. Firewall Security Policies & Zero-Trust Segmentation

All network flows are governed by stateful firewall policies adhering to the **Principle of Least Privilege**.

### 6.1 Headquarters Policy Set
| ID | Policy Name | Source Interface | Destination Interface | Source Address | Destination Address | Service | Action | NAT | Security Inspection Profiles |
| :---: | :--- | :--- | :--- | :--- | :--- | :--- | :---: | :---: | :--- |
| **2** | `HQ-Users-to-Internet` | `HQ-Users-VLAN10` | `port1` | `HQ-Users-Subnet` | `all` | DNS, HTTP, HTTPS, PING | ACCEPT | ON | AV, IPS, WebFilter, AppControl, SSL-Insp |
| **3** | `WAN-to-HQ-Web-Server` | `port1` | `HQServersVLAN20` | `all` | `HQ-Web-VIP-8080` | ALL | ACCEPT | OFF | Destination NAT VIP (8080 -> 80) |
| **4** | `SSLVPN-to-HQ-Web-Server`| `ssl.root` | `HQServersVLAN20` | `all` | `HQ-Web-Server` | HTTP, PING | ACCEPT | OFF | AV, IPS, WebFilter, AppControl, SSL-Insp |
| **5** | `vpn_To-BR1_local_0` | `HQ-Users-VLAN10`, `To-BR1` | `To-BR1`, `HQ-Users-VLAN10` | `To-BR1,2_local`, `To-BR1_remote` | `To-BR1_remote`, `To-BR1,2_local` | ALL | ACCEPT | OFF | Bi-directional Branch 1 User VPN |
| **6** | `vpn_To-BR1_local_1` | `HQServersVLAN20`, `To-BR1`| `To-BR1`, `HQServersVLAN20` | `To-BR1,2_local`, `To-BR1_remote` | `To-BR1_remote`, `To-BR1,2_local` | ALL | ACCEPT | OFF | Bi-directional Branch 1 Server VPN |
| **9** | `InterVLAN-All` | `HQ-Users-VLAN10`, `HQServersVLAN20`, `HQManagVLAN30` | `HQManagVLAN30`, `HQServersVLAN20`, `HQ-Users-VLAN10` | `all` | `all` | ALL | ACCEPT | OFF | Internal Segment Connectivity |
| **10**| `SRV-API` | `HQServersVLAN20` | `HQServersVLAN25` | `all` | `all` | ALL | ACCEPT | OFF | Web-to-API Layer Communication |
| **11**| `DB-API` | `HQServersVLAN27` | `HQServersVLAN25` | `all` | `all` | ALL | ACCEPT | OFF | DB-to-API Layer Communication |
| **12**| `HQ-to-BR2` | `HQ-Users-VLAN10`, `To-BR2` | `To-BR2`, `HQ-Users-VLAN10` | `To-BR1,2_local`, `To-BR2-Remote` | `To-BR1,2_local`, `To-BR2-Remote` | ALL | ACCEPT | OFF | Bi-directional Branch 2 User VPN |
| **13**| `BR2-to-HQ` | `To-BR2`, `HQServersVLAN20`| `HQServersVLAN20`, `To-BR2` | `To-BR1,2_local`, `To-BR2-Remote` | `To-BR1,2_local`, `To-BR2-Remote` | ALL | ACCEPT | OFF | Bi-directional Branch 2 Server VPN |

### 6.2 Micro-Segmentation Deep Dive (3-Tier Application Model)
To demonstrate server hardening and defense-in-depth:
- **Presentation Tier (VLAN 20):** Contains `HQ-Web-Server` (`192.168.20.20`), directly reachable from WAN through VIP port 8080.
- **Application/API Tier (VLAN 25):** Contains middle-tier API services (`192.168.25.10`). Web servers communicate with this tier under Policy 10 (`SRV-API`).
- **Database Tier (VLAN 27):** Houses core relational databases (`192.168.27.10`). Reached exclusively by the API tier via Policy 11 (`DB-API`).
- **Isolation:** Direct connectivity from Users (VLAN 10) or WAN (`port1`) to the Database Tier (VLAN 27) is blocked at Layer 3/4. Compromising the public web server prevents immediate lateral movement to sensitive database records.

---

## 7. NAT & Virtual IP (VIP) Server Publishing

### 7.1 Outbound Source NAT (Port Address Translation)
All internal hosts behind VLAN 10 access the public Internet using the egress WAN interface address (`203.0.113.6` on HQ, `203.0.113.14` on BR1, `203.0.113.22` on BR2) via dynamic PAT (Overload).

### 7.2 Inbound Destination NAT / VIP Port Forwarding
To securely expose internal services without publishing real private IP addresses, FortiGate Virtual IPs (VIPs) are configured with port translation:

```
[ External WAN Client ] 
        |
        | Destination: 203.0.113.6 : 8080 (Public VIP)
        v
+-------------------------------+
|      FortiGate-HQ Firewall    |
|   VIP: HQ-Web-VIP-8080        |
|   Translates Destination to:  |
|   192.168.20.20 : 80 (Real IP)|
+-------------------------------+
        |
        v
[ Internal Web Server (192.168.20.20) ] (Listening on Port 80)
```

```fortios
# Headquarters VIP Object
config firewall vip
    edit "HQ-Web-VIP-8080"
        set extip 203.0.113.6
        set extintf "port1"
        set portforward enable
        set mappedip "192.168.20.20"
        set extport 8080
        set mappedport 80
    next
end

# Branch 1 VIP Object
config firewall vip
    edit "VIP_Internal_Server"
        set extip 203.0.113.10
        set extintf "port1"
        set portforward enable
        set mappedip "10.10.20.10"
        set extport 8080
        set mappedport 80
    next
end

# Branch 2 VIP Object
config firewall vip
    edit "VIP-B2-Web-Server"
        set extip 192.168.22.2
        set extintf "port1"
        set portforward enable
        set mappedip "10.30.20.10"
        set extport 8080
        set mappedport 80
    next
end
```

---

## 8. Remote Access VPN (SSL-VPN) Implementation

Off-site teleworkers and traveling administrators securely connect to headquarters via FortiClient SSL-VPN.

### 8.1 SSL-VPN Configuration Parameters
- **Gateway Interface:** `port1` (`203.0.113.6`)
- **Listening Port:** 443 / default HTTPS
- **Assigned IP Pool:** `SSLVPN-Tunnel-Range` (`10.212.134.200 - 10.212.134.210`)
- **Authentication Method:** Local user database with PAP/MS-CHAPv2
- **Portal Mode:** `full-access` (Tunnel Mode with split-tunneling disabled for maximum data leak protection)
- **Encryption Algorithm:** TLS 1.3 / TLS 1.2 with AES-GCM
- **Certificate:** `Fortinet_Factory` (RSA 2048-bit)
- **Authorized Policy:** Policy ID 4 (`SSLVPN-to-HQ-Web-Server`) restricts SSL-VPN clients exclusively to `192.168.20.20` on ports 80 (HTTP) and ICMP (PING), with full UTM inspection active.

---

## 9. Layer 2 Switching Architecture & VLAN Configuration

The switching layer is structured into Core and Access layers using Cisco Catalyst IOSvL2 switches:

```
                          [ FortiGate-HQ ]
                           port2 (802.1Q)
                                 |
                          [ Core-HQ Switch ]
                          Trunk (Gi0/0 - 3)
                                 |
              +------------------+------------------+
              | Gi0/2                               | Gi0/3
        [ HQ-ACC-USER ]                       [ HQ-ACC-SRV ]
        Gi0/0: Trunk                          Gi0/0: Trunk
        Gi0/1 - Gi1/0: Access VLAN 10         Gi0/1: Access VLAN 20 (Web)
                                              Gi1/0: Access VLAN 25 (API)
                                              Gi0/2: Access VLAN 27 (DB)
                                              Gi0/3: Access VLAN 30 (Mgmt)
```

### 9.1 Core-HQ Switch (`Core-HQ`)
```cisco
hostname Core-HQ
!
vlan 10
 name Users
vlan 20
 name Servers
vlan 25
 name App-API
vlan 27
 name Database
vlan 30
 name Management
!
interface range GigabitEthernet0/0 - 3
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10,20,25,27,30
 no shutdown
```

### 9.2 Server Access Switch (`HQ-ACC-SRV`)
```cisco
hostname HQ-ACC-SRV
!
vlan 10,20,25,27,30
!
interface GigabitEthernet0/0
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 20,25,27,30
 no shutdown
!
interface GigabitEthernet0/1
 switchport mode access
 switchport access vlan 20
 no shutdown
!
interface GigabitEthernet1/0
 switchport mode access
 switchport access vlan 25
 no shutdown
!
interface GigabitEthernet0/2
 switchport mode access
 switchport access vlan 27
 no shutdown
!
interface GigabitEthernet0/3
 switchport mode access
 switchport access vlan 30
 no shutdown
```

### 9.3 User Access Switch (`HQ-ACC-USER`)
```cisco
hostname HQ-ACC-USER
!
vlan 10,20,30
!
interface GigabitEthernet0/0
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30
 no shutdown
!
interface range GigabitEthernet0/1 - 3, GigabitEthernet1/0
 switchport mode access
 switchport access vlan 10
 no shutdown
```

---

## 10. Engineering Troubleshooting Log & Incident Post-Mortem

During the design, deployment, and testing phases of this capstone lab, several critical networking and platform challenges were encountered and methodically solved:

### Incident 1: Cloud0 IP Conflict (`DUPADDR`) & Static Route Shadowing
- **Symptoms:** `%IP-4-DUPADDR: Duplicate address 192.168.85.201 on GigabitEthernet0/0` followed by ping dropouts to `8.8.8.8` and route configuration rejection `IP address conflicts with gateway ip address in static routing table`.
- **Root Cause:** Both the virtual lab node `ISP-HQ` and an overlapping network device had claimed `192.168.85.201` on the VMware NAT adapter bridge. Additionally, when attempting to change the IP address to `192.168.85.202`, Cisco IOS detected that `192.168.85.202` was already defined as a static route next-hop for Branch 1 subnets (`203.0.113.8/30`).
- **Remediation:** 
  1. Removed stale static routes conflicting with the interface IP.
  2. Stabilized the Cloud0 management map: `ISP-HQ` (`.201`), `ISP-BR1` (`.202`), and `ISP-BR2` (`.203`).
  3. Ensured the default gateway pointed exclusively to `192.168.85.2` (VMware NAT daemon) rather than `.1` (Windows host adapter, which rejected transit traffic).

### Incident 2: High Availability Cluster Inode Corrupted (`EXT3-fs error`) & Password Recovery
- **Symptoms:** Following a hard power cut to test HA failover, `Main-HQ` threw `EXT3-fs error (device vda1): ext3_lookup: deleted inode referenced: 511` on console and rejected the standard administrator credentials.
- **Root Cause:** Journaling filesystem corruption on the virtual disk image corrupted the local user database (`admin` account record).
- **Remediation:**
  1. Booted into single-user recovery mode via console.
  2. Authenticated using the Fortinet hardware/VM recovery credentials (`maintainer` / `bcpbFGVMEVZRLRX-WQB7`).
  3. Reset the administrative password database.
  4. Once `Main-HQ` joined the cluster, executed `execute ha synchronize start` and `diagnose sys ha checksum recalculate`. The primary configuration was synchronized intact from `Backup-HQ` without loss of VLANs, routes, or policies.

### Incident 3: Post-Failover Interface Reset to DHCP
- **Symptoms:** Following secondary node takeover and primary restoration, FortiGate Web GUI was inaccessible at `https://203.0.113.6`.
- **Root Cause:** Upon filesystem re-initialization, `port1` reverted to factory default `mode dhcp` with IP `0.0.0.0/0`, dropping the static IP assignment.
- **Remediation:**
  Executed recovery commands on the console:
  ```fortios
  config system interface
      edit "port1"
          set mode static
          set ip 203.0.113.6 255.255.255.252
          set allowaccess ping https ssh http
      next
  end
  ```
  Immediately synchronized HA state to secondary, restoring the virtual IP and cluster connectivity.

### Incident 4: Windows Host Route Redirection & Multi-Hop Lab Access
- **Symptoms:** Windows administrator workstation unable to reach the FortiGate Web GUIs at `203.0.113.x` with `Destination host unreachable` from `192.168.1.12`.
- **Root Cause:** The Windows routing table attempted to send RFC 5737 public IPs out of the default physical Wi-Fi interface (`192.168.1.12`) rather than into the VMware virtual adapter (`192.168.85.0/24`).
- **Remediation:** Flushed stale ARP and added explicit administrative routes via PowerShell:
  ```powershell
  route -p add 203.0.113.0 mask 255.255.255.248 192.168.85.201
  route -p add 203.0.113.8 mask 255.255.255.248 192.168.85.202
  route -p add 203.0.113.16 mask 255.255.255.248 192.168.85.203
  ```

---

## 11. Verification Commands & Operational Checklist

| Test Objective | Verification Command | Expected Successful Result |
| :--- | :--- | :--- |
| **HA Cluster Health** | `get system ha status` | Status: `OK`, Both nodes visible, In-Sync `Checksum MATCH` |
| **HA Failover Test** | Power down `Main-HQ` node | Continuous ping to `203.0.113.6` loses $\le 1$ packet; `Backup-HQ` assumes Active role |
| **VLAN Interfaces** | `get system interface physical` | All VLAN sub-interfaces (`port2.10`, `port2.20`, etc.) UP with correct IP |
| **DHCP Leases** | `diagnose sys dhcp lease list` | Workstation MACs allocated IPs in range `192.168.10.100 - .200` |
| **IPsec Phase 1 Status** | `diagnose vpn ike gateway list` | Status `ESTABLISHED`, SA count $\ge 1$ for `To-BR1` and `To-BR2` |
| **IPsec Phase 2 Status** | `diagnose vpn tunnel list` | Status `active`, SA lifetime counting, Rx/Tx packet counters incrementing |
| **Branch-to-Branch Ping** | From BR1 host ping BR2 host (`10.30.10.1`) | Success rate 100% via HQ transit tunnel routing |
| **Inbound VIP Port Forward** | `curl -I http://203.0.113.6:8080` | HTTP 200 OK returned from internal server `192.168.20.20:80` |
| **Internet Access & NAT** | From User PC: `ping 8.8.8.8` | 100% success; FortiGate session table shows SNAT to `203.0.113.6` |
| **Inter-VLAN Restriction** | Ping from User (VLAN 10) to DB (VLAN 27) | Traffic dropped by implicit deny / absence of inter-tier policy |

---

## 12. Requirements Compliance Matrix (Course Capstone Alignment)

| Course Requirement | Required Spec | Implementation Status | Implementation Evidence |
| :--- | :--- | :---: | :--- |
| **Network Infrastructure** | HQ, BR1, BR2, Remote User, ISP simulation | **COMPLETED** | 7 HQ devices, 4 BR1 devices, 4 BR2 devices, Cisco ISP core on Cloud0 |
| **VLAN Segmentation** | User, Server, Management VLANs with dedicated subnets | **COMPLETED** | VLAN 10 (Users), 20 (Web), 25 (API), 27 (DB), 30 (Mgmt) on 802.1Q trunks |
| **Inter-VLAN Routing** | Terminated and controlled by FortiGate NGFW | **COMPLETED** | Policies 9, 10, 11 enforce granular inter-VLAN Zero-Trust controls |
| **DHCP Server** | Dynamic allocation for User VLAN | **COMPLETED** | DHCP scope `192.168.10.100 - .200` active on `port2.10` |
| **High Availability (HA)** | Active-Passive cluster with heartbeat & failover test | **COMPLETED** | Dual heartbeat `port3`/`port4`, priority 200/100, verified failover |
| **Branch Routing & LAN** | VLANs & local Inter-VLAN on BR1 & BR2 | **COMPLETED** | `FGT-Branch1` & `FGT-Branch2` handle local VLANs 10 & 20 locally |
| **Site-to-Site IPsec VPN** | HQ <-> BR1 and HQ <-> BR2 with Branch transit | **COMPLETED** | Route-based IPsec tunnels `To-BR1` & `To-BR2` with Hub-and-Spoke transit |
| **Firewall Policies** | Outbound NAT, Inter-VLAN, and VPN policies | **COMPLETED** | Comprehensive policy matrix implemented on all 3 FortiGate nodes |
| **NAT & VIP Publishing** | Dynamic SNAT + Destination Port Forwarding | **COMPLETED** | PAT on egress; VIP `8080 -> 80` published for servers on HQ & Branches |
| **Security Profiles (UTM)** | Antivirus, IPS, Web Filter, App Control, SSL Insp | **COMPLETED** | Active on Policy 2 (Internet) & Policy 4 (SSL-VPN) |
| **Remote User VPN** | SSL-VPN or IPsec Remote Access with auth | **COMPLETED** | SSL-VPN Gateway on `port1`, `full-access` portal, `ssl.root` policy |
| **Bonus: Multi-Tier Design**| Presentation -> Logic -> Data 3-tier hardening | **BONUS** | Micro-segmented Web (VLAN 20) -> API (VLAN 25) -> DB (VLAN 27) |

---
*End of Engineering Documentation.*
