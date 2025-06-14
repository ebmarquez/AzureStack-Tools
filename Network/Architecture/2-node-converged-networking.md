# Azure Local - Two machines switchless dual link network architecture

- [Azure Local - Two machines switchless dual link network architecture](#azure-local---two-machines-switchless-dual-link-network-architecture)
  - [Terminology](#terminology)
  - [Example Device](#example-device)
  - [Scope](#scope)
  - [Two Node Switchless Environment](#two-node-switchless-environment)
  - [Attributes](#attributes)
    - [Nodes](#nodes)
  - [Cable Map](#cable-map)
    - [Node 1](#node-1)
    - [Node 2](#node-2)
    - [TOR 1](#tor-1)
    - [TOR 2](#tor-2)
  - [Switches](#switches)
  - [Switch Configuration](#switch-configuration)
    - [QOS Policy](#qos-policy)
    - [Compute, Management Intent Networks](#compute-management-intent-networks)
  - [ToR Configuration](#tor-configuration)
    - [Required Switch Features](#required-switch-features)
    - [VLAN configuration](#vlan-configuration)
    - [HSRP Configuration](#hsrp-configuration)
    - [Host Side Configuration](#host-side-configuration)
    - [MLAG Connection](#mlag-connection)
    - [Example Routing](#example-routing)
  - [Example SDN configuration](#example-sdn-configuration)
  - [Layer 3 Forwarding Gateway](#layer-3-forwarding-gateway)
    - [BGP Mode](#bgp-mode)
    - [Static Mode](#static-mode)
  - [Reference Documents](#reference-documents)

![2 node Switchless with 2 ToR](../images/2node-switchless-two-switch.png)

## Terminology

- **ToR**: Top of Rack network switch
- **p-NIC**: Physical Network Card attached to a node.
- **v-Switch**: Virtual Switch configured on the Azure Local Cluster.
- **VLAN**: Virtual Local Area Network.
- **SET**: Switch Embedded Teaming, supporting switch-independent teaming.
- **MLAG**: Multi-Chassis Link Aggregation, a technique that lets two or more network switches work together as if they were one logical switch.
- **Border Router**: Uplink device with the ToR switches, providing routing to endpoints external to the Azure Local environment.
- **AS**: Autonomous System number used to define a BGP neighbor.

## Example Device

- **Make**: Cisco
- **Model**: Nexus 98180YC-FX
- **Firmware**: 10.3(4a)

## Scope

This document assists administrators in designing a network architecture that aligns with Azure Local cluster requirements. It provides reference architectures and sample configurations for network devices supporting cluster deployments. Equipment such as switches, firewalls, or routers located on customer premises is considered out of scope, as it is assumed to be part of the existing infrastructure. The focus is on Node-to-ToR, ToR-to-ToR, and ToR uplink configurations.

## Two Node Switchless Environment

This solution deploys a two-node environment using an Azure Local switchless storage architecture. Storage traffic is directly connected between the two nodes, bypassing network switches to reduce latency and complexity for storage workloads. The ToR switches are dedicated to compute and management traffic, providing redundancy and high availability for cluster operations. Compute and management p-NICs are configured with Switch Embedded Teaming (SET), enabling switch-independent teaming and seamless integration with the physical network.

This approach ensures storage traffic is isolated and optimized through direct node-to-node connections, while compute and management traffic benefit from the resiliency and scalability of dual ToR switches and MLAG/vPC configurations.

## Attributes

The two-node converged networking configuration has the following attributes:

### Nodes

1. Each node is equipped with two physical network interface cards, each with two physical interfaces (p-NICs).
2. p-NICs A and B handle both compute and management traffic.
3. p-NICs A and B are configured as part of a Switch Embedded Teaming (SET) team, transmitting compute and management traffic using VLAN tags. These NICs are assigned to a virtual switch (v-Switch) to support multiple network intents.
4. p-NICs C and D are dedicated to storage traffic and support RDMA protocol workloads.
5. p-NICs C and D are directly connected between node 1 and node 2, with traffic transmitted using VLAN tags over this direct connection.

## Cable Map

### Node 1

| Device    | Interface |      | Device | Interface   |
| --------- | --------- | ---- | ------ | ----------- |
| **Node1** | p-NIC A   | <==> | TOR1   | Ethernet1/1 |
| **Node1** | p-NIC B   | <==> | TOR2   | Ethernet1/1 |
| **Node1** | p-NIC C   | <==> | Node2  | p-NIC C     |
| **Node1** | p-NIC D   | <==> | Node2  | p-NIC D     |

### Node 2

| Device    | Interface |      | Device | Interface   |
| --------- | --------- | ---- | ------ | ----------- |
| **Node2** | p-NIC A   | <==> | TOR1   | Ethernet1/2 |
| **Node2** | p-NIC B   | <==> | TOR2   | Ethernet1/2 |
| **Node2** | p-NIC C   | <==> | Node1  | p-NIC C     |
| **Node2** | p-NIC D   | <==> | Node1  | p-NIC D     |

### TOR 1

| Device   | Interface    |      | Device  | Interface    |
| -------- | ------------ | ---- | ------- | ------------ |
| **TOR1** | Ethernet1/1  | <==> | Node1   | p-NIC A      |
| **TOR1** | Ethernet1/2  | <==> | Node2   | p-NIC B      |
| **TOR1** | Ethernet1/41 | <==> | TOR2    | Ethernet1/41 |
| **TOR1** | Ethernet1/42 | <==> | TOR2    | Ethernet1/42 |
| **TOR1** | Ethernet1/47 | <==> | Border1 | Ethernet1/x  |
| **TOR1** | Ethernet1/48 | <==> | Border2 | Ethernet1/x  |
| **TOR1** | Ethernet1/49 | <==> | TOR2    | Ethernet1/49 |
| **TOR1** | Ethernet1/50 | <==> | TOR2    | Ethernet1/50 |
| **TOR1** | Ethernet1/51 | <==> | TOR2    | Ethernet1/51 |

### TOR 2

| Device   | Interface    |      | Device  | Interface    |
| -------- | ------------ | ---- | ------- | ------------ |
| **TOR2** | Ethernet1/1  | <==> | Node1   | p-NIC A      |
| **TOR2** | Ethernet1/2  | <==> | Node2   | p-NIC B      |
| **TOR2** | Ethernet1/41 | <==> | TOR1    | Ethernet1/41 |
| **TOR2** | Ethernet1/42 | <==> | TOR1    | Ethernet1/42 |
| **TOR2** | Ethernet1/47 | <==> | Border1 | Ethernet1/x  |
| **TOR2** | Ethernet1/48 | <==> | Border2 | Ethernet1/x  |
| **TOR2** | Ethernet1/49 | <==> | TOR1    | Ethernet1/49 |
| **TOR2** | Ethernet1/50 | <==> | TOR1    | Ethernet1/50 |
| **TOR2** | Ethernet1/51 | <==> | TOR1    | Ethernet1/51 |

## Switches

- Two physical switches are used as Top of Rack (ToR) devices (Cisco Nexus 93180YC-FX).
- Management and compute p-NICs connect to the ToR devices.
- Storage p-NICs do not connect to the ToR devices.
- The ToRs are configured in MLAG for redundancy.
- The core network (switch/router/firewall) is considered out of scope.

## Switch Configuration

### QOS Policy

Switch QoS is not required in this configuration, as storage traffic is direct node-to-node. QoS is typically only needed if storage interfaces connect to a switch.

### Compute, Management Intent Networks

Compute and management intents are separated into VLANs (e.g., 7 and 8). These VLANs are configured as Layer 3 SVIs on both ToR switches. MLAG is used to support east-west traffic for compute and management.

## ToR Configuration

### Required Switch Features

Enable the following features on Cisco Nexus 93180YC-FX to support this example Azure Local environment:

**Sample Cisco Nexus 93180YC-FX Configuration:**

```console
feature bgp
feature interface-vlan
feature hsrp
feature lacp
feature vpc
feature lldp
```

### VLAN configuration

In this configuration, VLANs are used to logically separate management and compute traffic within the Azure Local environment:

- **VLAN 7** is designated for management traffic. It is configured as the native VLAN on trunk ports and supports Azure Local backend services, including cluster management and infrastructure communication.
- **VLAN 8** is assigned for compute traffic, which carries customer or tenant workloads.

Both VLANs are configured as Layer 2 and Layer 3 interfaces, with each assigned to a Switch Virtual Interface (SVI). The gateway IP address for each VLAN SVI ends in ".1" to provide a consistent and predictable default gateway for devices within each subnet.

### HSRP Configuration

Hot Standby Router Protocol (HSRP) is implemented on both VLAN interfaces to provide gateway redundancy and high availability. HSRP version 2 is used, and each VLAN is assigned an HSRP group number that matches its VLAN ID for maintainablity. The HSRP configuration ensures that if one ToR switch becomes unavailable, the other can immediately take over gateway responsibilities, maintaining uninterrupted network connectivity for both management and compute networks.

The VLAN Maximum Transmission Unit (MTU) is set to 9216 bytes to support jumbo frames, which is required for SDN workloads. If SDN is not used, the default MTU (typically 1500 bytes) is sufficient, and customers should select the MTU size based on their workload requirements.

**Sample Cisco Nexus 93180YC-FX Configuration:**

```console
vlan 7
  name Management_7
vlan 8
  name Compute_8
!
interface Vlan7
  description Management
  no shutdown
  mtu 9216
  no ip redirects
  ip address 10.101.176.2/24
  no ipv6 redirects
  hsrp version 2
  hsrp 7
    priority 150 forwarding-threshold lower 1 upper 150
    ip 100.101.176.1
!
interface Vlan8
  description Compute
  no shutdown
  mtu 9216
  no ip redirects
  ip address 10.101.177.2/24
  no ipv6 redirects
  hsrp version 2
  hsrp 8
    priority 150 forwarding-threshold lower 1 upper 150
    ip 100.101.177.1
```

### Host Side Configuration

The following configuration example demonstrates how to configure the ToR switch interfaces that connect directly to the Azure Local cluster nodes. These interfaces are responsible for carrying both management and compute traffic between the nodes and the network infrastructure.

**Key Configuration Elements:**

- **Interface Assignment:**  
  `Ethernet1/1` and `Ethernet1/2` are connected to Node 1 and Node 2. Each interface is clearly labeled with a descriptive name.

- **Trunk Mode with VLAN Tagging:**  
  Both interfaces are configured as trunk ports, allowing them to carry multiple VLANs (VLAN 7 for management and VLAN 8 for compute). The native VLAN is set to 7, which is typically used for management traffic. This setup enables the use of Switch Embedded Teaming (SET) on the host side, supporting multiple network intents over the same physical connection.  Additional compute VLANs tags can be included to support tenant workloads.

- **Spanning Tree and Edge Port Configuration:**  
  The `spanning-tree port type edge trunk` command is applied to optimize convergence times and protect against accidental loops, as these ports connect directly to servers rather than other switches.

- **Performance Optimization:**  
  The MTU is set to 9216 bytes to support jumbo frames, which is recommended for Azure Local and SDN workloads to maximize throughput and reduce CPU overhead. The interface speed is explicitly set to 10 Gbps to match the physical NIC capabilities.

- **Operational Enhancements:**  
  CDP (Cisco Discovery Protocol) is disabled to minimize unnecessary protocol traffic, and link status logging is enabled to monitor and record changes in the status of server-facing ports.

**Sample Cisco Nexus 93180YC-FX Interface Configuration:**

```console
interface Ethernet1/1
  description AzLocalNode1
  no cdp enable
  switchport
  switchport mode trunk
  switchport trunk native vlan 7
  switchport trunk allowed vlan 7-8
  spanning-tree port type edge trunk
  mtu 9216
  speed 10000
  logging event port link-status
  no shutdown
!
interface Ethernet1/2
  description AzLocalNode2
  no cdp enable
  switchport
  switchport mode trunk
  switchport trunk native vlan 7
  switchport trunk allowed vlan 7-8
  spanning-tree port type edge trunk
  mtu 9216
  speed 10000
  logging event port link-status
  no shutdown
```

### MLAG Connection

**Spanning Tree Configuration:**  
The configuration begins by enabling BPDU Guard on all edge ports to protect against accidental network loops. Multiple Spanning Tree (MST) is used to support multiple VLANs, with specific priorities set for each MST instance to influence root bridge selection. The MST region is defined with a unique name and revision, and VLANs are mapped to specific MST instances for optimal traffic management.

**vPC Domain and Peer Link:**
The vPC domain is established to allow the two ToR switches to operate as a single logical switch for downstream devices. Key parameters such as role priority, peer-keepalive (for monitoring the health of the vPC peer), and auto-recovery are configured to enhance resiliency. The peer-gateway feature is enabled to allow seamless gateway operations across both switches.

**Port-Channel and Interface Configuration:**

- Port-Channel 50 is configured as a dedicated Layer 3 link for vPC heartbeat and iBGP communication between the ToR switches, with jumbo frames enabled for high-performance workloads.
- Port-Channel 101 serves as the vPC peer link, configured as a trunk to carry multiple VLANs, with priority flow control and spanning tree network port settings applied for reliability and performance.

**Physical Interface Bundling:**
Interfaces Ethernet1/49, Ethernet1/50, and Ethernet1/51 are bundled into Port-Channel 101 using LACP in active mode. These interfaces are configured as trunk ports, with the native VLAN set to 99 and priority flow control enabled. Logging is enabled for link status changes, and CDP is disabled to reduce unnecessary protocol traffic.

**Cisco Nexus 93180YC-FX Configuration:**

```console
spanning-tree port type edge bpduguard default
spanning-tree mst 0-1 priority 8192
spanning-tree mst 2 priority 16384
spanning-tree mst configuration
  name AzureStack
  revision 1
  instance 1 vlan 1-1999
  instance 2 vlan 2000-4094

vpc domain 1
  role priority 1
  peer-keepalive destination 10.100.19.18 source 10.100.19.17 vrf default
  delay restore 150
  peer-gateway
  auto-recovery

interface port-channel50
  description VPC:Heartbeat_P2P_IBGP_Link
  logging event port link-status
  mtu 9216
  ip address 10.10.19.17/30

interface port-channel101
  description VPC:MLAG_PEER
  switchport
  switchport mode trunk
  switchport trunk native vlan 99
  priority-flow-control mode on
  spanning-tree port type network
  logging event port link-status
  vpc peer-link

interface Ethernet1/49
  description MLAG_Peer
  no cdp enable
  switchport
  switchport mode trunk
  switchport trunk native vlan 99
  priority-flow-control mode on
  logging event port link-status
  channel-group 101 mode active
  no shutdown

interface Ethernet1/50
  description MLAG_Peer
  no cdp enable
  switchport
  switchport mode trunk
  switchport trunk native vlan 99
  priority-flow-control mode on
  logging event port link-status
  channel-group 101 mode active
  no shutdown

interface Ethernet1/51
  description MLAG_Peer
  no cdp enable
  switchport
  switchport mode trunk
  switchport trunk native vlan 99
  priority-flow-control mode on
  logging event port link-status
  channel-group 101 mode active
  no shutdown
```

### Example Routing

The configuration begins by defining the router's BGP process under autonomous system 64511, using a static router-id assigned to a loopback interface. This ensures a stable identifier, crucial for consistent BGP operation even if physical interfaces change state. The command bestpath as-path multipath-relax allows the router to consider multiple paths to the same destination—even when AS paths are not identical—thus providing greater flexibility in equal-cost multipath routing. The log-neighbor-changes command ensures that any changes in neighbor status are logged, aiding in operational troubleshooting.

Under the IPv4 unicast address family, several networks are explicitly advertised, including the loopback (used for the router ID), point-to-point links (for border and port channel connectivity), and VLAN-specific subnets (supporting internal segments such as VLAN7 and VLAN8). The use of maximum-paths 8 (for both eBGP and iBGP with the maximum-paths ibgp command) enables the router to handle up to eight equal-cost paths, enhancing load balancing and redundancy within the network.

Prefix lists are utilized within this configuration to enforce clear routing policies:
**DefaultRoute Prefix List**:
This list is designed to only advertise the default route. It explicitly permits the default route (0.0.0.0/0) and then denies any other routes within the same address space, ensuring that no extraneous subnets are erroneously advertised as default routes.
**FROM-BORDER Prefix List**:
Applied to incoming BGP updates from border neighbors, this list allows only the default route to be received. It permits 0.0.0.0/0 while denying any other prefixes that fall under the 0.0.0.0/0 umbrella, thereby filtering out unwanted routes from entering the local routing table.
**TO-BORDER Prefix List:**
Used when advertising routes to border neighbors, this list prevents the default route from being advertised. The early deny rule (seq 5) filters out the 0.0.0.0/0 prefix, while the subsequent permit rule allows all other routes (any IPv4 prefix with a mask length less than or equal to 32) to be advertised.

Neighbor relationships are then established for three interfaces. Two of these neighbors (with IPs \<Border1-IP\> and \<Border2-IP\>) are external peers in AS 64404—each configured with descriptive labels and both employing the FROM-BORDER and TO-BORDER prefix lists within the IPv4 unicast address family. These peers also have a safeguard with maximum-prefix 12000 warning-only to notify if the count of received prefixes nears a risky threshold. The third neighbor, associated with \<Port-Channel50-IP\>, is an internal (iBGP) peer in AS 64511 and is configured similarly, though without prefix filtering in the outbound direction

**Cisco Nexus 93180YC-FX Configuration:**

```console
!!! Only advertise the default route
ip prefix-list DefaultRoute seq 10 permit 0.0.0.0/0
ip prefix-list DefaultRoute seq 50 deny 0.0.0.0/0 le 32

!!! Receive BGP Advertisements for 0.0.0.0/0, deny all others.
ip prefix-list FROM-BORDER seq 10 permit 0.0.0.0/0
ip prefix-list FROM-BORDER seq 30 deny 0.0.0.0/0 le 32

!!! Advertise any network except for 0.0.0.0/0
ip prefix-list TO-BORDER seq 5 deny 0.0.0.0/0
ip prefix-list TO-BORDER seq 10 permit 0.0.0.0/0 le 32

router bgp 64511
  router-id <Loopback-IP>
  bestpath as-path multipath-relax
  log-neighbor-changes
  address-family ipv4 unicast
    network <Loopback-IP>/32
    network <Border1-IP>/30
    network <Border2-IP>/30
    network <Port-Channel50-IP>/30
    ! VLAN7
    network 10.101.176.0/24
    ! VLAN8
    network 10.101.177.0/24
    maximum-paths 8
    maximum-paths ibgp 8
  neighbor <Border1-IP>
    remote-as 64404
    description TO_Border1
    address-family ipv4 unicast
      prefix-list FROM-BORDER in
      prefix-list TO-BORDER out
      maximum-prefix 12000 warning-only
  neighbor <Border2-IP>
    remote-as 64404
    description TO_Border2
    address-family ipv4 unicast
      prefix-list FROM-BORDER in
      prefix-list TO-BORDER out
      maximum-prefix 12000 warning-only
  neighbor <Port-Channel50-IP>
    remote-as 64511
    description TO_TOR2
    address-family ipv4 unicast
      maximum-prefix 12000 warning-only
```

## Example SDN configuration

This section of the BGP configuration is tailored to support an [Azure Local SDN](https://learn.microsoft.com/en-us/azure/azure-local/manage/load-balancers) scenario using VLAN8.

**Dynamic BGP Neighbor Definition**:
A BGP neighbor is defined using the 10.101.177.0/24 subnet, which corresponds to VLAN8 and is reserved for the SLBMUX. The SLBMUX can use any IP address within this subnet, so the configuration specifies the entire subnet as the neighbor. The remote AS is set to 65158, and the neighbor is labeled TO_SDN_SLBMUX for clarity. When a subnet is used as the BGP neighbor, the switch operates in passive mode and waits for the SLBMUX to initiate the BGP connection.

**Peering and Connectivity**:

- `neighbor 10.101.177.0/24`
  Defines the BGP neighbor as the entire 10.101.177.0/24 subnet, which is associated with VLAN8. This allows any device within that subnet—such as a SLBMUX to establish a BGP session with the switch, provided it matches the remote AS number.
- `remote-as 65158`  
  Specifies the remote Autonomous System (AS) number that the switch will allow to form a BGP session. In this case, the remote AS is set to 65158, which should match the AS number configured on the Gateway VM. Only eBGP sessions are supported in this configuration.
- `update-source loopback0`
  This ensures that BGP traffic originates from the stable loopback interface on the TOR, which helps maintain consistent peering even if physical interfaces change.
- `ebgp-multihop 3`
  Allows the BGP session to traverse up to three hops, accommodating scenarios where the SLBMUX is not directly connected.
- `prefix-list DefaultRoute out`
  Within the IPv4 unicast address family for this neighbor, the outbound route policy is governed by the DefaultRoute prefix list. This list is designed to advertise only the default route (0.0.0.0/0) to the SLBMUX. This aligns with the design goal of having the SLBMUX receive only the default route from the switch.
- `maximum-prefix 12000 warning-only`
  This command serves as a safeguard, issuing warnings if the number of received prefixes approaches a set limit, thereby helping maintain stability in the peer session.

**Cisco Nexus 93180YC-FX Configuration:**

```console
  neighbor 10.101.177.0/24
    remote-as 65158
    description TO_SDN_SLBMUX
    update-source loopback0
    ebgp-multihop 3
    address-family ipv4 unicast
      prefix-list DefaultRoute out
      maximum-prefix 12000 warning-only
```

## Layer 3 Forwarding Gateway

There are two primary methods for supporting [Layer 3 Forwarding Gateways](https://learn.microsoft.com/en-us/azure/azure-local/manage/gateway-connections?view=azloc-2505#create-an-l3-connection) in Azure Local configurations: BGP and static routing.

With BGP, the Layer 3 Gateway establishes a BGP session with the ToR switch and advertises its V-NET routes directly into the ToR routing table. This dynamic approach allows the routing table to be automatically updated as new networks are added or removed, reducing manual intervention and supporting scalable, automated network operations.

In contrast, static routing requires the Forwarding Gateway to be a member of the VLAN, with the ToR switch manually configured with static routes for each required network. This method is more manual and requires the network team to update the ToR configuration whenever new internal networks are introduced. While BGP is recommended for environments that require flexibility and automation, static routing may be preferred in scenarios where a controlled, predictable routing configuration is needed.

For both methods, the subnet used for the gateway can be much smaller than the examples shown—typically as small as a /30 or /28, depending on the number of required IP addresses.

### BGP Mode

As mentioned above, BGP provides a dynamic way to add internal networks to the ToR switch routing table. When an administrator adds new networks in the portal, the Layer 3 Gateway advertises these networks to the switch using BGP, allowing the routing table to update automatically.

In the sample configuration below

- `neighbor 10.101.177.0/24`  
  Defines the BGP neighbor as the entire 10.101.177.0/24 subnet, which is associated with VLAN8. This allows any device within that subnet—such as a Layer 3 Gateway VM to establish a BGP session with the switch, provided it matches the remote AS number.

- `remote-as 65158`  
  Specifies the remote Autonomous System (AS) number that the switch will allow to form a BGP session. In this case, the remote AS is set to 65158, which should match the AS number configured on the Gateway VM. Only eBGP sessions are supported in this configuration.

- `update-source Vlan8`  
  Ensures that BGP peering uses the VLAN8 interface as the source IP address. The source IP can be any address assigned to the VLAN8 configuration. Refer to the VLAN section above for more details.

- `ebgp-multihop 5`  
  Allows the BGP session to be established even if the Gateway VM is up to five hops away from the switch. This is useful in scenarios where the VM is not directly connected.

- `prefix-list DefaultRoute out`  
  Within the IPv4 unicast address family, this command restricts the switch to only advertise the default route (0.0.0.0/0) to the Layer 3 Gateway. The Gateway VM must receive at least the default route from the ToR switch.

- `maximum-prefix 12000 warning-only`  
  Sets a safeguard by issuing a warning if the number of received prefixes approaches 12,000. This helps prevent routing table overload and maintains network stability.

This configuration enables the ToR switch to dynamically learn and advertise routes as the network evolves, reducing manual intervention and supporting scalable, automated network operations.

**Cisco Nexus 93180YC-FX Configuration:**

```console
    neighbor 10.101.177.0/24
      remote-as 65158
      description TO_L3Forwarder
      update-source Vlan8
      ebgp-multihop 5
      address-family ipv4 unicast
        prefix-list DefaultRoute out
        maximum-prefix 12000 warning-only
```

### Static Mode

In static routing mode, the network team must plan in advance which subnet will be used by the V-NET and which IP address will serve as the gateway for internal routing. For example, in the configuration below, 10.101.177.226 is designated as the gateway VM. This IP address acts as the Layer 3 peering point with the ToR switch and serves as the gateway to the internal subnet 10.68.239.0/24.

It is recommended to configure the required static routes on the ToR switch before deploying the gateway VM. If additional internal networks are needed in the future, the ToR configuration must be updated to include static routes for those networks prior to their deployment. This ensures that traffic destined for the internal subnet is correctly forwarded to the gateway VM, supporting seamless connectivity within the larger Azure Local environment.

This approach is particularly useful in environments where dynamic routing protocols like BGP are not used, or where a more controlled, manual routing configuration is preferred.

**Cisco Nexus 93180YC-FX Configuration:**

```console
  ip route 10.101.177.226/32 10.68.239.0/24
```

## Reference Documents

- [Network considerations for cloud deployments of Azure Local](https://learn.microsoft.com/en-us/azure/azure-local/plan/cloud-deployment-network-considerations)
- [Physical network requirements for Azure Local](https://learn.microsoft.com/en-us/azure/azure-local/concepts/physical-network-requirements)
- [Teaming in Azure Stack HCI](https://techcommunity.microsoft.com/blog/networkingblog/teaming-in-azure-stack-hci/1070642)
- [Manage Azure Local gateway connections](https://learn.microsoft.com/en-us/azure/azure-local/manage/gateway-connections?view=azloc-2505#create-an-l3-connection)
