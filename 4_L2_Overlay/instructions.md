
## Build a L2 overlay

In order to deploy a L2 overlay, NX-OS needs to:
1. Map a VLAN ID to a VXLAN ID
2. Assign BGP address familiy distinguisher/targets
3. Attach VXLAN tunnel to the NVE (VTEP) interface

Apply configuration:
```
Step 1:
! XY - pod pairs (12 or 34)
vlan 20XY
  vn-segment 20XY

Step 2:
evpn
  ! XY - pod numbers in pairs (12 or 34)
  vni 20XY l2
    rd 65000:20XY
    route-target import 65000:20XY
    route-target export 65000:20XY

Step 3:
interface nve1
  ! XY - pod pairs (12 or 34)
  member vni 20XY
    ingress-replication protocol bgp
    suppress-arp
```

> Command: `show nve vni` , `sh nve vni 20XY detail`  

Seek for VNI state Up  
  
> Command: `show bgp l2vpn evpn vni-id 20XY`  

> Command: `show bgp l2vpn evpn vni-id 2012 route-type 3`  

Find the RD, RT and VNI values in the output  

## Configuring endpoints

VRFs are used to separate routing domains between lab steps on endpoints. 

Apply configuration on podX-ep1:
```
vrf definition 2000
 description IP endpoint in VNI 2000
 !
 address-family ipv4
 exit-address-family

  ! XY - pod pairs (12 or 34)
vlan 20XY 
interface Vlan20XY
 vrf forwarding 2000
  ! X|Y - enter your pod id - 1, 2, 3 or 4
 ip address 10.XY.200.X|Y 255.255.255.0
 no shutdown
```

> Command: `show ip interface brief`  

SVI should be up.  

> Command: `sh ip vrf 2000`  

## Verify and inspect L2 overlay

Let's initiate traffic from our endpoint.
> Command: `ping vrf 2000 10.XY.200.254`  

The address does not exist in the network, so only ARP requests are sent out. But that is enought for the network to learn the MAC address.  

If the peer pod that shares the same L2 domain 20XY has also finished their part, you will see not only your, but the peer endpoint's podY-ep1 MAC address.
> Command: `show bgp l2vpn evpn [vni-id 20XY] [route-type 2]`  
```
BGP routing table information for VRF default, address family L2VPN EVPN
BGP table version is 9, Local Router ID is 10.X.0.N
Status: s-suppressed, x-deleted, S-stale, d-dampened, h-history, *-valid, >-best
Path type: i-internal, e-external, c-confed, l-local, a-aggregate, r-redist, I-injected
Origin codes: i - IGP, e - EGP, ? - incomplete, | - multipath, & - backup, 2 - best2

   Network            Next Hop            Metric     LocPrf     Weight Path
Route Distinguisher: 65000:20XY    (L2VNI 20XY)
*>l[2]:[0]:[0]:[48]:[5254.001c.87dc]:[0]:[0.0.0.0]/216
                      10.1.1.99                         100      32768 i
*>l[3]:[0]:[32]:[10.1.1.99]/88
                      10.1.1.99                         100      32768 i
```

> Command: `show l2route mac topology 20XY`  
```
Flags -(Rmac):Router MAC (Stt):Static (L):Local (R):Remote (V):vPC link 
(Dup):Duplicate (Spl):Split (Rcv):Recv (AD):Auto-Delete (D):Del Pending
(S):Stale (C):Clear, (Ps):Peer Sync (O):Re-Originated (Nho):NH-Override
(Pf):Permanently-Frozen, (Orp): Orphan

Topology    Mac Address    Prod   Flags         Seq No     Next-Hops      
----------- -------------- ------ ------------- ---------- ----------------
20XY        5254.001c.87dc Local  L,            0          Po1003         
```

> Command: `show l2route topology 20XY detail`  


Verify connectivity between peer pods:
> Command: `ping vrf 2000 10.XY.200.X|Y`, where last X|Y - is the oposing pod id  
  
The end result should be mutual connectivity.

**End of L2 overlay configuration guide instructions**