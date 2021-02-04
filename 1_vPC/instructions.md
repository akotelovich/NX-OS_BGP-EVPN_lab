
## Enabling required features

Apply configuration:
```
feature lacp
feature vpc
```

> Command: `show system internal sysmgr service name vpc`  
> Command: `show system internal sysmgr service dependency srvname vpc`  
> Command: `show system internal feature-mgr feature vpc current status`  

```
Service "vpc" ("vpc", 317):
        UUID = 0x251, PID = 12208, SAP = 450
        State: SRV_STATE_HANDSHAKED (entered at time Wed Feb  3 13:57:02 2021).
        Restart count: 4
        Time of last restart: Wed Feb  3 13:57:01 2021.
        The service never crashed since the last reboot.
        Tag = N/A
        Plugin ID: 1
```

> Command: `show system internal sysmgr service name lacp`  
> Command: `show system internal sysmgr service dependency srvname lacp`  


## vPC domain and keepalive configuration

Apply configuration:
```
vpc domain X00
  peer-switch
  ! role priority 4000 or 8000
  peer-keepalive destination {{ pod_peer_mgmt_address }}
  peer-gateway
  layer3 peer-router
```

> Command: `show vpc`  

```
Legend:
                (\*) - local vPC is down, forwarding via vPC peer-link

vPC domain id                     : X00 
Peer status                       : peer link not configured
vPC keep-alive status             : peer is alive
Configuration consistency status  : failed
Per-vlan consistency status       : failed
Configuration inconsistency reason: vPC peer-link does not exist
Type-2 consistency status         : failed  
Type-2 inconsistency reason       : vPC peer-link does not exist
vPC role                          : none established
Number of vPCs configured         : 0   
Peer Gateway                      : Enabled
Dual-active excluded VLANs        : -
Graceful Consistency Check        : Disabled (due to peer configuration)
Auto-recovery status              : Disabled
Delay-restore status              : Timer is off.(timeout = 30s)
Delay-restore SVI status          : Timer is off.(timeout = 10s)
Operational Layer3 Peer-router    : Disabled
Virtual-peerlink mode             : Disabled
```

## vPC Peer link configuration

Apply configuration:
```
interface port-channel1
  switchport mode trunk
  spanning-tree port type network
  vpc peer-link

interface Ethernet1/1
  switchport mode trunk
  channel-group 1 mode active
```

> Command: `show vpc`  

```
Legend:
                (\*) - local vPC is down, forwarding via vPC peer-link

vPC domain id                     : X00 
Peer status                       : peer adjacency formed ok
vPC keep-alive status             : peer is alive
Configuration consistency status  : success 
Per-vlan consistency status       : success
Type-2 consistency status         : success 
vPC role                          : primary                     
Number of vPCs configured         : 0
Peer Gateway                      : Enabled
Dual-active excluded VLANs        : -
Graceful Consistency Check        : Enabled
Auto-recovery status              : Disabled
Delay-restore status              : Timer is off.(timeout = 30s)
Delay-restore SVI status          : Timer is on.(timeout = 10s, 1s left)
Operational Layer3 Peer-router    : Enabled
Virtual-peerlink mode             : Disabled

vPC Peer-link status
-- ------------------------------------------------------------------
id    Port   Status Active vlans
--    ----   ------ -------------------------------------------------
1     Po1    up     1
```

> Command: `show vpc consistency-parameters global`  

## vPC configuration

Apply configuration to n-1:
```
interface port-channel1003
  switchport mode trunk
  switchport trunk allowed vlan 1
  vpc 1003

interface Ethernet1/3
  switchport mode trunk
  switchport trunk allowed vlan 1
  channel-group 1003 mode active
```

After adding to one member check:
> Command: `show vpc`  
```
vPC status
----------------------------------------------------------------------------
Id    Port          Status Consistency Reason                Active vlans
--    ------------  ------ ----------- ------                ---------------
1003  Po1003        up     failed      Peer does not have    -                           
                                       corresponding vPC        
```

> Command: `show vpc consistency-parameters interface port-channel 1003`  
> Command: `show vpc consistency-parameters vpc 1003`  

Apply configuration to n-2:

> Command: `show vpc`  

```
vPC status
----------------------------------------------------------------------------
Id    Port          Status Consistency Reason                Active vlans
--    ------------  ------ ----------- ------                ---------------
1003  Po1003        up     success     success               1                           
                                                                                         
```

## Adding vlans to vPC

> Command: `show vpc`  
Check field `Active vlans` for vpc 1003

Apply configuration to n-1:
```
! X - pod # - 1, 2, 3 or 4
vlan 100X
! XY - pod pair # - 12 or 34
vlan 20XY
interface port-channel1003
  switchport trunk allowed vlan add 100X,20XY
```

> `show vpc`  
 Only VLAN 1 in `Active vlans` section.

> Command: `show vpc consistency-parameters interface port-channel 1003`  

```
[ OMITTED OUTPUT ... ]            
Allowed VLANs               -     1,100X,20XY            1                     
Local suspended VLANs       -     100X,20XY              -
```

> Command: `sh logging | grep suspended`  

Apply configuration to n-2:
```
! X - pod # - 1, 2, 3 or 4
vlan 100X
! XY - pod pair # - 12 or 34
vlan 20XY
interface port-channel1003
  switchport trunk allowed vlan add 100X,20XY
```

> Command: `show vpc`  
 All vlans active in `Active vlans` section.  

> Command: `show vpc consistency-parameters interface port-channel 1003 | diff`  
No susspended VLANs.  

> Command: `sh logging | grep suspended | diff`  

```
> 2021 Feb  3 14:54:17 podX-n1 %ETHPORT-3-IF_ERROR_VLANS_REMOVED: VLANs 100X on Interface port-channel1003 are removed from suspended state.
> 2021 Feb  3 14:54:17 podX-n1 %ETHPORT-3-IF_ERROR_VLANS_REMOVED: VLANs 100X on Interface port-channel1 are removed from suspended state.
> 2021 Feb  3 14:54:17 podX-n1 %ETHPORT-3-IF_ERROR_VLANS_SUSPENDED: VLANs 100X on Interface port-channel1003 are being suspended. (Reason: Vlan is not allowed on Peer-link)
> 2021 Feb  3 14:54:17 podX-n1 %ETHPORT-3-IF_ERROR_VLANS_REMOVED: VLANs 100X on Interface port-channel1 are removed from suspended state.
> 2021 Feb  3 14:54:17 podX-n1 %ETHPORT-3-IF_ERROR_VLANS_REMOVED: VLANs 100X on Interface port-channel1003 are removed from suspended state.
> 2021 Feb  3 14:54:24 podX-n1 %ETHPORT-3-IF_ERROR_VLANS_REMOVED: VLANs 20XY on Interface port-channel1003 are removed from suspended state.
> 2021 Feb  3 14:54:24 podX-n1 %ETHPORT-3-IF_ERROR_VLANS_REMOVED: VLANs 20XY on Interface port-channel1 are removed from suspended state.
> 2021 Feb  3 14:54:24 podX-n1 %ETHPORT-3-IF_ERROR_VLANS_SUSPENDED: VLANs 20XY on Interface port-channel1003 are being suspended. (Reason: Vlan is not allowed on Peer-link)
> 2021 Feb  3 14:54:24 podX-n1 %ETHPORT-3-IF_ERROR_VLANS_REMOVED: VLANs 20XY on Interface port-channel1 are removed from suspended state.
> 2021 Feb  3 14:54:24 podX-n1 %ETHPORT-3-IF_ERROR_VLANS_REMOVED: VLANs 20XY on Interface port-channel1003 are removed from suspended state.
```

**End of vPC configuration guide instructions**