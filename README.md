# Exploring MPLS Networking and Configuration
![[Pasted image 20250407152852.png]]
## Basic MPLS Core Configuration
### IPs and Loopbacks on Provider Routers
![[Pasted image 20250407125854.png]]

The first step in the configuration will be to setup the interfaces on each provider router with IPs and Loopbacks.

Example configuration for P1:
```IOS
hostname P1

interface Loopback0
 ip address 2.2.2.2 255.255.255.255

interface GigabitEthernet2
 ip address 10.1.1.2 255.255.255.252
 no shutdown

interface GigabitEthernet1
 ip address 10.1.2.1 255.255.255.252
 no shutdown

```

Checking the interfaces shows the IPs and respective interfaces configured
![[Pasted image 20250407131053.png]]
### Configure OSPF for Core Routers
OPFS is configured as the MPLS Core IGP to allow the service provider routers to reach each others loopbacks and communicate with each other. It is NOT used for customer data but rather "control plane" data. The `PE` and `P` router are configured with OSPF, `CE` customer routers are not.  

Example configuration for PE1:
```
router ospf 100
 network 1.1.1.1 0.0.0.0 area 0
 network 10.1.1.0 0.0.0.3 area 0
```

Verifying the configuration; OSPF routes to all other subnets and router loopbacks in the MPLS Core have been added, pinging `PE2`'s loopback from `PE1` also succeeds.
![[Pasted image 20250407133823.png]]

### Configure MPLS on `PE` and `P` routers
Next is the MPLS configuration which is what actually facilitates the customer data transfer. MPLS uses label switching rather than traditional IP routing which allows fast and efficient transfer of data. MPLS is configured only on P and PE routers, allowing label switching instead of traditional IP routing for fast and efficient data forwarding. CE routers do not run MPLS but exchange routing information with PE routers using BGP (or another protocol).

`PE1` example MPLS setup:
```
mpls ip

interface GigabitEthernet2
 mpls ip
```

We can verify the configuration by checking LDP neighbors (showing `P1` has become neighbors with `PE1` and `P2`) and the MPLS forwarding-table:
![[Pasted image 20250407135418.png]]
## BGP for MPLS Core
MPLS by itself only provides label switching. To actually **route customer traffic**, we need a way to exchange customer routes across the MPLS core. **BGP allows this by:**
1. **CE to PE:** The CE router advertises its network(s) to the PE router using eBGP.
2. **PE to PE (via MP-BGP):** The PE router **does NOT put these routes into the global BGP table**. Instead, it uses **MP-BGP (Multiprotocol BGP)** to advertise **VPN routes** between PE routers.
3. **PE to CE:** The PE router then sends the correct routes to the CE router on the other side
The **MPLS Core `P` routers do not run BGP at all**—only the `PE` routers do, to share customer prefixes.
### MP-BGP setup for `PE` Routers
![[Pasted image 20250407143736.png]]
**MP-BGP (BGP with VPNv4 address family)** allows `PE` routers to **exchange customer routes** while keeping them isolated in their VRFs. It is responsible for: 
- Exchanging **VPNv4 routes** (customer routes + VRF information).
- Carrying **route targets (RTs)** and **route distinguishers (RDs)** to keep customer routes **logically separated**.
- **Without MP-BGP, PE routers wouldn’t know which customer routes belong to which VPNs, breaking MPLS VPN functionality.**

`PE1` Configuration:
```
router bgp 65000
 bgp log-neighbor-changes
 neighbor 4.4.4.4 remote-as 65000
 neighbor 4.4.4.4 update-source Loopback0
 address-family vpnv4
  neighbor 4.4.4.4 activate
  neighbor 4.4.4.4 send-community both
 exit-address-family
```

## Customer VPN Configuration
![[Pasted image 20250407152852.png]]
We can now add customers to the topology. In this example I add two customers, Red and Blue, with two sites connected over the MPLS WAN. 
### Configuring VRFs on `PE` routers
**VRF (Virtual Routing and Forwarding)** allows multiple customers to use the same **IP address ranges** without conflicts.
- Each **customer gets its own isolated routing table** inside the PE router.
- **MP-BGP (VPNv4) relies on VRFs** to keep customer traffic separate.

First I create the VRFs on both `PE` routers:
```
ip vrf Red
 rd 65000:1
 route-target export 65000:1
 route-target import 65000:1

ip vrf Blue
 rd 65000:2
 route-target export 65000:2
 route-target import 65000:2
```

Then interfaces are assigned to each VRF and IP addresses add.
`PE1` Example:
```
interface GigabitEthernet1
 ip vrf forwarding Red
 ip address 192.168.1.1 255.255.255.252
 no shut

interface GigabitEthernet3
 ip vrf forwarding Blue
 ip address 192.168.1.1 255.255.255.252
 no shut
```

### Configure PE-to-CE BGP Sessions per VRF
This step creates **per-VRF BGP sessions** between the **PE** routers and the **CE** routers, allowing them to exchange **customer-specific routing information**. Since each customer is in a separate VRF, we define a BGP session **inside that VRF**. 

`PE1`:
```
router bgp 65000
 address-family ipv4 vrf Red
  neighbor 192.168.1.2 remote-as 65020
  neighbor 192.168.1.2 activate
  redistribute connected
 exit-address-family

 address-family ipv4 vrf Blue
  neighbor 192.168.1.2 remote-as 65010
  neighbor 192.168.1.2 activate
  redistribute connected
 exit-address-family
```

## `CE` Router Configurations
I configure the `CE` routers with hostnames, IP addresses and then BGP pairings with their respective `PE` routers. 

Configuration for `Red-CE1`
```
hostname Red-CE1
interface GigabitEthernet1
 ip address 172.16.10.1 255.255.255.0
 no shutdown
 
interface GigabitEthernet3
 ip address 192.168.2.2 255.255.255.252
 no shutdown

router bgp 65020
 bgp log-neighbor-changes
 neighbor 192.168.2.1 remote-as 65000
 network 172.16.10.0 mask 255.255.255.0
 address-family ipv4
  neighbor 192.168.2.1 allowas-in 2
```

>\***Note:** Since all CE routers in the VRF share the same AS ,the `allowas-in` command is used to override BGP's loop prevention.
## Verifications 
To make sure everything is working as it should I test to make sure both customer routers and the MPLS provider routers are all functioning as they should.
### CE Routers
#### Checking the Routing Table
Checking the routing table on `Blue-CE1` reveals that the BGP routes have been add. I also **don't** see any routes internal to the MPLS Core or from other customers. 
![[Pasted image 20250407165825.png]]
#### Check Connectivity
Both `traceroute` and `ping` show that `Red-CE2` has connectivity across the MPLS Core to the second site. 
![[Pasted image 20250407170345.png]]
### PE Routers
#### Checking VRFs
Running `sh ip vrf Red` on `PE1` verifies that all routes from the Red customer sites have successfully been added to the table
![[Pasted image 20250407170903.png]]

#### Checking MPLS Forwarding Tables
Output of `sh mpls forwarding-table` on `PE1`:
```IOS
Local      Outgoing   Prefix           Bytes Label   Outgoing   Next Hop
Label      Label      or Tunnel Id     Switched      interface
16         Pop Label  2.2.2.2/32       0             Gi2        10.1.1.2
17         17         3.3.3.3/32       0             Gi2        10.1.1.2
18         20         4.4.4.4/32       0             Gi2        10.1.1.2
19         Pop Label  10.1.2.0/30      0             Gi2        10.1.1.2
20         19         10.1.3.0/30      0             Gi2        10.1.1.2
21         No Label   192.168.1.0/30[V]   \
                                       2372          aggregate/Red
22         No Label   192.168.1.0/30[V]   \
                                       0             aggregate/Blue
23         No Label   172.16.20.0/24[V]   \
                                       0             Gi1        192.168.1.2
24         No Label   172.16.1.0/24[V] 0             Gi3        192.168.1.2
```

Output of `sh mpls forwarding-table` on `PE2`:
```IOS
Local      Outgoing   Prefix           Bytes Label   Outgoing   Next Hop
Label      Label      or Tunnel Id     Switched      interface
16         21         1.1.1.1/32       0             Gi2        10.1.3.1
17         17         2.2.2.2/32       0             Gi2        10.1.3.1
18         Pop Label  3.3.3.3/32       0             Gi2        10.1.3.1
19         19         10.1.1.0/30      0             Gi2        10.1.3.1
20         Pop Label  10.1.2.0/30      0             Gi2        10.1.3.1
21         No Label   192.168.2.0/30[V]   \
                                       0             aggregate/Red
22         No Label   192.168.2.0/30[V]   \
                                       0             aggregate/Blue
23         No Label   172.16.10.0/24[V]   \
                                       1812          Gi3        192.168.2.2
24         No Label   172.16.12.0/24[V]   \
                                       0             Gi1        192.168.2.2
```