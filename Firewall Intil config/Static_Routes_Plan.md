# Static Routes — Complete Routing Plan

## The Golden Rule
> Every device must know how to reach subnets it is **NOT directly connected to**.

---

## 1. ISP Routers — Must Know Each Other's WAN Subnets

All ISPs share the same `192.168.85.0/24` cloud, so they talk to each other via that.
Each ISP must have routes to the **other sites' WAN links** through the other ISPs.

### ISP-HQ (`192.168.85.201`)
```cisco
! Reach Branch1 WAN links → via ISP-BR1
ip route 203.0.113.8  255.255.255.252 192.168.85.202
ip route 203.0.113.12 255.255.255.252 192.168.85.202

! Reach Branch2 WAN links → via ISP-BR2
ip route 203.0.113.16 255.255.255.252 192.168.85.203
ip route 203.0.113.20 255.255.255.252 192.168.85.203

! Default route to internet
ip route 0.0.0.0 0.0.0.0 192.168.85.2
```

### ISP-BR1 (`192.168.85.202`)
```cisco
! Reach HQ WAN links → via ISP-HQ
ip route 203.0.113.0 255.255.255.252 192.168.85.201
ip route 203.0.113.4 255.255.255.252 192.168.85.201

! Reach Branch2 WAN links → via ISP-BR2
ip route 203.0.113.16 255.255.255.252 192.168.85.203
ip route 203.0.113.20 255.255.255.252 192.168.85.203

! Default route to internet
ip route 0.0.0.0 0.0.0.0 192.168.85.2
```

### ISP-BR2 (`192.168.85.203`)
```cisco
! Reach HQ WAN links → via ISP-HQ
ip route 203.0.113.0 255.255.255.252 192.168.85.201
ip route 203.0.113.4 255.255.255.252 192.168.85.201

! Reach Branch1 WAN links → via ISP-BR1
ip route 203.0.113.8  255.255.255.252 192.168.85.202
ip route 203.0.113.12 255.255.255.252 192.168.85.202

! Default route to internet
ip route 0.0.0.0 0.0.0.0 192.168.85.2
```

---

## 2. WAN Routers — Just Need a Default Route

WAN routers are simple — they just forward everything upstream toward the ISP.

### WAN-HQ
```cisco
ip route 0.0.0.0 0.0.0.0 203.0.113.1
```
> Gateway = ISP-HQ Gi0/1 IP

### WAN-BR1
```cisco
ip route 0.0.0.0 0.0.0.0 203.0.113.9
```
> Gateway = ISP-BR1 Gi0/1 IP

### WAN-BR2
```cisco
ip route 0.0.0.0 0.0.0.0 203.0.113.17
```
> Gateway = ISP-BR2 Gi0/1 IP

---

## 3. FortiGate Firewalls — Most Important

### FortiGate-HQ
```bash
# Default route (internet)
config router static
    edit 1
        set dst 0.0.0.0 0.0.0.0
        set gateway 203.0.113.5
        set device "port1"
    next
end

# Route to Branch1 LAN subnets → via VPN tunnel
config router static
    edit 2
        set dst 10.10.10.0 255.255.255.0
        set device "VPN_BR1"
    next
    edit 3
        set dst 10.10.20.0 255.255.255.0
        set device "VPN_BR1"
    next
end

# Route to Branch2 LAN subnets → via VPN tunnel
config router static
    edit 4
        set dst 10.20.10.0 255.255.255.0
        set device "VPN_BR2"
    next
    edit 5
        set dst 10.20.20.0 255.255.255.0
        set device "VPN_BR2"
    next
end
```

### FortiGate-BR1
```bash
# Default route (internet) → via WAN-BR1
config router static
    edit 1
        set dst 0.0.0.0 0.0.0.0
        set gateway 203.0.113.13
        set device "port1"
    next
end

# Route to HQ LAN subnets → via VPN tunnel
config router static
    edit 2
        set dst 10.0.0.0 255.255.0.0
        set device "VPN_HQ"
    next
end

# Route to Branch2 LAN (goes THROUGH HQ) → via VPN tunnel to HQ
config router static
    edit 3
        set dst 10.20.0.0 255.255.0.0
        set device "VPN_HQ"
    next
end
```

### FortiGate-BR2
```bash
# Default route (internet) → via WAN-BR2
config router static
    edit 1
        set dst 0.0.0.0 0.0.0.0
        set gateway 203.0.113.21
        set device "port1"
    next
end

# Route to HQ LAN subnets → via VPN tunnel
config router static
    edit 2
        set dst 10.0.0.0 255.255.0.0
        set device "VPN_HQ"
    next
end

# Route to Branch1 LAN (goes THROUGH HQ) → via VPN tunnel to HQ
config router static
    edit 3
        set dst 10.10.0.0 255.255.0.0
        set device "VPN_HQ"
    next
end
```

---

## 4. VPCs / End Hosts — Just Default Gateway

No static routes needed on VPCs — just configure the **default gateway**:

| Device | Location | Default Gateway |
|---|---|---|
| VPC17, VPC18 | Branch1 User VLAN | `10.10.10.1` (FortiGate-BR1 port2.10) |
| Branch1 Server VPC | Branch1 Server VLAN | `10.10.20.1` (FortiGate-BR1 port2.20) |
| Branch2 VPCs | Branch2 User VLAN | `10.20.10.1` (FortiGate-BR2 port2.10) |
| Branch2 Server VPC | Branch2 Server VLAN | `10.20.20.1` (FortiGate-BR2 port2.20) |

> In EVE-NG VPC: `ip 10.10.10.10 10.10.10.1 24`

---

## 5. Your Real PC — Nothing Needed

Your **physical machine** running EVE-NG does not need any routes added. All routing is handled **inside** the lab topology.

---

## Summary Diagram

```
VPC → FortiGate (default GW)
      ↓ default route
   WAN Router
      ↓ default route
   ISP Router  ←→  [ISP-HQ / ISP-BR1 / ISP-BR2 know each other's subnets]
      ↓ Net cloud
   Other ISP → WAN Router → FortiGate
                                ↓ VPN tunnel routes
                             Destination LAN
```
