
## Enabling required features

Apply configuration:
```
feature ospf
feature interface-vlan
```

> Command: `show system internal sysmgr service name ospf`  
> Command: `show system internal sysmgr service dependency srvname ospf`  
> Command: `show system internal feature-mgr feature ospf current status`  

## Building P2P connectivity

First of all we prepare P2P connectivity between vPC peers over the peer link.  

Apply configuration on n-1:
```
vlan 12
  name UNDERLAY_N1_N2

interface Vlan12
  description UNDERLAY_N1_N2_P2P
  no shutdown
  mtu 9216
  medium p2p
  no ip redirects
    ! X - pod #
  ip address 10.X.12.1/30
  no ipv6 redirects
```

Apply configuration on n-2:
```
vlan 12
  name UNDERLAY_N1_N2

interface Vlan12
  description UNDERLAY_N1_N2_P2P
  no shutdown
  mtu 9216
  medium p2p
  no ip redirects
    ! X - pod #
  ip address 10.X.12.2/30
  no ipv6 redirects
```

> Command: `show vpc`  
Should show VLAN 12 active on vPC peer link.

> Command: `ping 10.X.12.2`  ir `.1` depends from which peer we are testing  
Peer should be reachable over ICMP.  

> For HW platforms, `system nve infra-vlans 12` would be needed to use VLAN 12 as fabric uplink.

Secondly, we'll configure IP connectivity to the `r-underlay` fabric transport router.  

Apply configuration:
```
interface Ethernet1/2
  no switchport
  mtu 9216
  medium p2p
  no ip redirects
    ! X - pod #, N - Nexus #, pod1-n2 is 10.1.23.1/30
  ip address 10.X.N3.1/30
  no ipv6 redirects
  no shutdown
```

> Command: `ping 10.X.N3.2` to verify connectivity to fabric router.  
ICMP replies should be received.

## Looback interfaces for BGP and VTEPs

Here we create the interfaces for routing session and VTEP (VXLAN tunnel) source addressing

Apply configuration:
```
interface loopback0
  description UNDERLAY Router-ID
    ! X - pod #, N - Nexus #
  ip address 10.X.0.N/32

interface loopback1
  description UNDERLAY NVE ID
    ! X - pod #, N - Nexus #
  ip address 10.X.1.N/32
  ip address 10.X.1.99/32 secondary
```


## Enabling underlay routing

Underlay routing in this lab (and most deployments) is based on OSPF.  

Apply configuration:
```
router ospf 1
    ! X - pod #, N - Nexus #, pod2-n1 is 10.2.0.1
  router-id 10.X.0.N
  auto-cost reference-bandwidth 1000 Gbps
```

Include interfaces into OSPF 1 routing protocol process

Apply configuration:
```
interface Vlan12
  ip ospf network point-to-point
  ip router ospf 1 area 0.0.0.0

interface Ethernet1/2
  ip ospf network point-to-point
  ip router ospf 1 area 0.0.0.0

interface loopback0
  ip router ospf 1 area 0.0.0.0

interface loopback1
  ip router ospf 1 area 0.0.0.0

```

> Command: `show ip ospf neighbors` to verify dynamic routing sessions with vPC peer and fabric router.    
Each switch should have two peers in FULL state.

> Command: `show ip route ospf-1`  
> Command: `show ip ospf database [router-id] detail`   
Now you should sync to peers that are configuring other pods. At this point, if all lab members succeed, looback interface addresses of other pods should start to appear in your routing table. 
This is essential for next steps in this configuration guide.

**End of Underlay configuration guide instructions**