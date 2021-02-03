
## Build a L3 overlay

In order to deploy a L3 overlay, NX-OS needs to:
1. Map a VRF to a VXLAN ID and Assign BGP address familiy distinguisher/targets to the VRF
2. Create Tenant SVI that does routing from VRF to BGP EVPN peers over VXLAN
3. Attach VXLAN tunnel to the NVE (VTEP) interface
4. Configure VRF routing under VRF
5. Create L3 interfaces in the VRF

Apply configuration:  
Step 1.  
Map a VRF to a VXLAN ID and Assign BGP address familiy distinguisher/targets to the VRF.
```
vrf context 2000
  vni 2000
  rd 65000:2000
  address-family ipv4 unicast
    route-target both 65000:2000
    route-target both 65000:2000 evpn 
```
Step 2.  
Create Tenant SVI that does routing from VRF to BGP EVPN peers over VXLAN.
```
vlan 2000
  vn-segment 2000

interface Vlan2000
    no shutdown
    mtu 9216
    vrf member 2000
    no ip redirects
    ip forward
    no ipv6 redirects
```
Step 3.  
Attach VXLAN tunnel to the NVE (VTEP) interface.
```
interface nve1
 member vni 2000 associate-vrf
```
Step 4.  
Configure VRF routing under VRF.
```
route-map permit-all
router bgp 65000
  vrf 2000
    address-family ipv4 unicast
      redistribute direct route-map permit-all
```
Step 5.  
Create L3 interfaces in the VRF.
```
vlan 20XY
interface Vlan20XY
  no shutdown
  mtu 9216
  vrf member 2000
  no ip redirects
  ip address 10.XY.200.254/24
  no ipv6 redirects
  fabric forwarding mode anycast-gateway
```

> Command: `show ip interface brief vrf 2000`  
  
The IP forward interface.  
  
> Command: `show ip route vrf 2000`, or `routing-context vrf 2000` and `show ip route`  

> Command: `show nve vni` , `show nve vni 2000 detail`  
  
Seek for VNI state Up  

> Command: `show bgp l2vpn evpn vni-id 2000`  
  
Type 5 route should be present  

> Command: `show bgp l2vpn evpn vni-id 2000 route-type 5`  
  
Find the RD, RT and VNI values in the output  

## Configuring endpoints

Herw we extend endpoint's connectivity in VRF 2000 by adding a default route towards the VXLAN network.
Additionally we create indirectly reachable networks on the endpoint.  
  
Apply configuration on podX-ep1:
```
ip route vrf 2000 0.0.0.0 0.0.0.0 10.XY.200.254

interface Loopback2000
 vrf forwarding 2000
   ! X - pod ID
 ip address 10.X.200.X 255.255.255.255

```

> Command: `show ip interface brief`  

SVI should be up.
> Command: `show ip route vrf 2000`  

You should the see the default route active.  

> Command: `sho2 ip arp vrf 2000`  

You should see the gateway's MAC address that you previously set as anycast mac address.

## Adding external routing

From the endpoint check connectivity to the gateway.

> Command: `ping vrf 2000 10.XY.200.254`  
  
Gateway should be reachable, but not when sourcing packets from the loopback.
> Command: `ping vrf 2000 10.XY.200.254 source loopback 2000`  
  
Now let's add the routes to the external networks and redistribute them to BGP.
Apply configuration:
```
vrf context 2000
  ip route 10.X.200.X/32 10.XY.200.X

router bgp 65000
  vrf 2000
    address-family ipv4 unicast
      redistribute static route-map permit-all
```

Check reachability from your endpoint.
> Command: `ping vrf 2000 10.XY.200.254 source loopback 2000`  
  
Additionally try reachability to other endpoints and their loopbacks.

## Verify and inspect L2 & L3 overlay

If the all pods have also finished their steps, you should see all networks reachable via VXLAN.
> Command: `show bgp l2vpn evpn [vni-id 2000] [route-type 5]`  
  
From the endpoints, you should also reach all other network loopbacks.  
> Command: `ping vrf 2000 10.X.200.X`, where `X` is 1, 2, 3 or 4.  
  
You should get successfull ICMP responses.
  
> Try pinging endpoints from the switch - what happens?  
  
**End of L2 & L3 overlay configuration guide instructions**