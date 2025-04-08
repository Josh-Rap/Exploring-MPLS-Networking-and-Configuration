# Exploring MPLS Networking and Configuration
![image](images/Pasted%20image%2020250407152852.png)
## Basic MPLS Core Configuration
### IPs and Loopbacks on Provider Routers
![image](images/Pasted%20image%2020250407125854.png)

The first step in the configuration is to set up the interfaces on each provider router with IP addresses and loopbacks.

Example configuration for `P1`:
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

Verifying the interfaces shows the IP addresses assigned to each interface:
![image](images/Pasted%20image%2020250407131053.png)
### Configure OSPF for Core Routers
OSPF is used as the IGP (Interior Gateway Protocol) for the MPLS core. It enables the provider (`P` and `PE`) routers to **learn each other’s loopbacks and establish connectivity**. This is **strictly for control plane communication**, not for customer traffic. Only `PE` and `P` routers run OSPF; `CE` routers do not.

Example configuration for `PE1`:
```
router ospf 100
 network 1.1.1.1 0.0.0.0 area 0
 network 10.1.1.0 0.0.0.3 area 0
```

To verify, we can confirm that OSPF routes to all loopbacks and subnets within the MPLS core are present. Pinging `PE2`'s loopback from `PE1` is successful:
![image](images/Pasted%20image%2020250407133823.png)

### Configure MPLS on `PE` and `P` routers
The next step is configuring MPLS, which **enables label switching for customer data**. MPLS **replaces traditional IP routing** in the core for more efficient forwarding. MPLS is **only enabled on `P` and `PE` routers**; `CE` routers do not participate in MPLS. They exchange routes with `PE` routers using BGP or another protocol.

`PE1` example MPLS setup:
```
mpls ip

interface GigabitEthernet2
 mpls ip
```

We verify the configuration by checking LDP neighbors (e.g., `P1` is now an LDP neighbor of `PE1` and `P2`) and inspecting the MPLS forwarding table:
![image](images/Pasted%20image%2020250407135418.png)
## BGP for MPLS Core
MPLS provides the label-switched forwarding plane, but we still need a routing protocol to exchange customer routes. This is where BGP comes in:
1. **CE to PE:** Customer (`CE`) routers advertise their routes to the connected `PE` router using eBGP.
2. **PE to PE (MP-BGP):** PE routers use **MP-BGP** (Multiprotocol BGP) to advertise customer VPN routes to other `PE` routers. These are **not** inserted into the global BGP table.
3. **PE to CE:** The remote `PE` router then advertises the appropriate customer routes to the connected CE.
> Note: Core P routers do not run BGP. Only PE routers participate in BGP to carry customer routes.
### MP-BGP setup for `PE` Routers
![image](images/Pasted%20image%2020250407143736.png)

**MP-BGP (BGP with VPNv4 address family)** allows `PE` routers to **exchange customer routes** while keeping them isolated in their VRFs. It is responsible for: 
- Exchanging **VPNv4 routes** (customer routes + VRF information).
- Carrying **route targets (RTs)** and **route distinguishers (RDs)** to keep customer routes **logically separated**.
- **Without MP-BGP, PE routers wouldn’t know which customer routes belong to which VPNs, breaking MPLS VPN functionality.**

**MP-BGP using the VPNv4 address famil**y allows `PE` routers to **exchange customer routes** while keeping them isolated by VPN:
- It carries **VPNv4 routes** (IPv4 + RD).
- It includes **route distinguishers (RDs)** and **route targets (RTs)** to maintain **logical separation** of customer networks.
- **Without MP-BGP**, `PE` routers wouldn’t know which routes belong to which customer VPNs.

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
![image](images/Pasted%20image%2020250407152852.png)
Now we add customers to the MPLS topology. In this example, there are two customers: **Red** and **Blue**, each with two sites connected over the MPLS WAN.
### Configuring VRFs on `PE` routers
**VRFs (Virtual Routing and Forwarding)** create logically separate routing instances for each customer. This allows customers to use overlapping IP ranges without conflicts.
- Each customer has **its own routing table** on the `PE` router.
- **MP-BGP uses VRFs** to maintain route separation.

First, create VRFs on both PE routers:
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

Then assign interfaces to the appropriate VRF and configure IP addresses. `PE1` Example:
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
This step configures **per-VRF BGP** sessions between the **`PE`** and **`CE`** routers. Each VRF gets its own BGP session to exchange customer-specific routes.

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
`CE` routers are configured with hostnames, IP addresses, and eBGP peering toward their respective `PE` routers.

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

>\***Note:** Since all `CE` routers in the VRF share the same AS ,the `allowas-in` command is used to override BGP's default AS path loop prevention.
## Verifications 
We verify the configuration to ensure that both customer and provider routers are operating correctly.
### CE Routers
#### Checking the Routing Table
On `Blue-CE1`, the routing table shows BGP-learned routes. No core MPLS routes or routes from other customers appear, which confirms isolation.
![image](images/Pasted%20image%2020250407165825.png)
#### Check Connectivity
`Traceroute` and `ping` confirm that `Red-CE2` can reach its remote site across the MPLS core.
![image](images/Pasted%20image%2020250407170345.png)
### PE Routers
#### Checking VRFs
Running `show ip route vrf Red` on `PE1` confirms that routes from the Red customer are correctly installed in the VRF:
![image](images/Pasted%20image%2020250407170903.png)

#### Checking MPLS Forwarding Tables
`PE1` MPLS forwarding table::
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

`PE2` MPLS forwarding table:
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
## Full Configurations for Each Router
to be added.