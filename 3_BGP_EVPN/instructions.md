
## Enabling required features

Apply configuration:
```
nv overlay evpn
feature bgp
feature vn-segment-vlan-based
feature nv overlay
```

> Command: `show system internal sysmgr service name ospf`  
> Command: `show system internal sysmgr service dependency srvname ospf`  
> Command: `show system internal feature-mgr feature ospf current status`  

## Setting Anycast MAC address

Anyucast MAC address, that matches in all BGP EVPN fabric members allows endpoint migration between VTEPs without a requirement to renew it's gateway MAC address and preventingblackholing traffic after migration to a different network location.  

Apply configuration:
```
fabric forwarding anycast-gateway-mac 2021.2021.2021
```

## Configuring BGP

Here we enable BGP routing and configure a full mesh to all lab memebers

Apply configuration:
```
router bgp 65000
    ! X - pod #, N - Nexus #
  router-id 10.X.0.N
  log-neighbor-changes
  address-family ipv4 unicast
  address-family l2vpn evpn
    maximum-paths ibgp 2
  template peer BGP_EVPN_Peers
    remote-as 65000
    update-source loopback0
    timers 3 9
    address-family ipv4 unicast
    address-family l2vpn evpn
      send-community
      send-community extended
    !
    ! Peerings to all neighbors in the lab
    ! X - 1,2,3,4 and N - 1,2
    ! Except the router id of this switch.
  neighbor 10.X.0.N
    inherit peer BGP_EVPN_Peers
```

> Command: `show bgp l2vpn evpn summary`  
Eventually all neighbors should have 0 in State column.

## Enabling VTEP functionality

Apply configuration:
```
interface nve1
  no shutdown
  description Overlay VTEP interface
  source-interface loopback1
  host-reachability protocol bgp
```

> Command: `show nve interface`  
See the VIP address (secondary loopback1 address)
> Command: `nve vxlan-params`  

**End of BGP EVPN and VXLAN configuration guide instructions**