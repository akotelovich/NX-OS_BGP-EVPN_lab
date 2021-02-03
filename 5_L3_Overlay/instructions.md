
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
vrf context 1000
  vni 1000
  rd 65000:1000
  address-family ipv4 unicast
    route-target both 65000:1000
    route-target both 65000:1000 evpn 
```
Step 2.  
Create Tenant SVI that does routing from VRF to BGP EVPN peers over VXLAN.
```
vlan 1000
  vn-segment 1000

interface Vlan1000
    no shutdown
    mtu 9216
    vrf member 1000
    no ip redirects
    ip forward
    no ipv6 redirects
```
Step 3.  
Attach VXLAN tunnel to the NVE (VTEP) interface.
```
interface nve1
 member vni 1000 associate-vrf
```
Step 4.  
Configure VRF routing under VRF.
```
route-map permit-all
router bgp 65000
  vrf 1000
    address-family ipv4 unicast
      redistribute direct route-map permit-all
```
Step 5.  
Create L3 interfaces in the VRF.
```
vlan 100X
interface Vlan100X
  no shutdown
  mtu 9216
  vrf member 1000
  no ip redirects
  ip address 10.X.100.254/24
  no ipv6 redirects
  fabric forwarding mode anycast-gateway
```

> Command: `show ip interface brief vrf 1000`  
  
The IP forward interface.  
  
> Command: `show ip route vrf 1000`, or `routing-context vrf 1000` and `show ip route`  

> Command: `show nve vni` , `show nve vni 1000 detail`  
  
Seek for VNI state Up  

> Command: `show bgp l2vpn evpn vni-id 1000`  
  
Type 5 route should be present  

> Command: `show bgp l2vpn evpn vni-id 1000 route-type 5`  
  
Find the RD, RT and VNI values in the output  

## Configuring endpoints

VRFs are used to separate routing domains between lab steps on endpoints.  
  
Apply configuration on podX-ep1:
```
vrf definition 1000
 description IP endpoint in VNI 1000
 !
 address-family ipv4
 exit-address-family

  ! X - pod ID
vlan 100X 
interface Vlan100X
 vrf forwarding 1000
  ! X|Y - enter your pod id - 1, 2, 3 or 4
 ip address 10.X.100.X 255.255.255.0
 no shutdown

ip route vrf 1000 0.0.0.0 0.0.0.0 10.1.100.254
```

> Command: `show ip interface brief`  

SVI should be up.
> Command: `sh ip route vrf 1000`  

You should the see the default route active.
> Command: `sh ip arp vrf 1000`  

You should see the gateway's MAC address that you previously set as anycast mac address.

## Verify and inspect L2 overlay

Let's initiate traffic from our endpoint.
> Command: `ping vrf 1000 10.Y.100.Y`, where `Y` is the oposing pod ID.  

You should get successfull ICMP responses  

If the all pods have also finished their steps, you should see all networks reachable via VXLAN.
> Command: `show bgp l2vpn evpn [vni-id 1000] [route-type 5]`  

From the endpoints, you should also reach all other network gateways.
> Command: `ping vrf 1000 10.X.100.254`, where `X` is 1, 2, 3 or 4.  

You should get successfull ICMP responses.

> Try pinging endpoints from the switch - what happens?  

**End of L3 overlay configuration guide instructions**