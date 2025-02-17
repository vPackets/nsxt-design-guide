
**Table Of Contents**

<!--ts-->
   * [4 - NSX-T Routing](#4-nsx-t-logical-routing)
     * [4.1 - NSX-T Single Tier Routing](#41-single-tier-routing)
       * [4.1.1 - Distributed Router (DR)](#411-distributed-router-dr)
       * [4.1.2 - Services Router (SR)](#412-services-router-sr)
     * [4.2 - NSX-T Multi Tier Routing](#42-single-tier-routing)
       * [4.2.1 - Interfaces Types on Tier-1 and Tier-0 Gateway](#421-interfaces-types-on-tier-1-and-tier-0-gateway)
       * [4.2.2 - Route Types on Tier-0 and Tier-1 Gateways](#422-route-types-on-tier-0-and-tier-1-gateways)
       * [4.2.3 - Fully Distributed Two Tier Routing](#423-fully-distributed-two-tier-routing)
     * [4.3 - Routing Capabilities](#43-routing-capabilities)
       * [4.3.1 - Static Routing](#431-static-routing)
       * [4.3.2 - Dynamic Routing](#432-dynamic-routing)
     * [4.4 - VRF Lite](#44-vrf-lite)
       * [4.4.1 - VRF Lite Generalities](#441-vrf-lite-generalities)
     * [4.5 - IPv6 Routing Capabilities](#45-ipv6-routing-capabilities)
     * [4.6 - Services High Availability](#46-services-high-availability)
       * [4.6.1 - Active/Active](#461-activeactive)
       * [4.6.2 - Active/Standby](#462-activestandby)
         * [4.6.2.1 - Graceful Restart and BFD Interaction with Active/Standby](#4621graceful-restart-and-bfd-interaction-with-active)
       * [4.6.3 - High Availibility Failover triggers](#463-high-availbility-failover-triggers)
     * [4.7 - Edge Node](#47-edge-node)
     * [4.8 - Multi-TEP support on Edge Node](#48-multi-tep-support-on-edge-node)
       * [4.8.1 - Bare Metal Edge Node](#481-bare-metal-edge-node)
         * [4.8.1 - Management Plane Configuration Choice with Bare metal node](#4811-management-plane-configuration-choices-with-bare-metal-node)
         * [4.8.1 - Single NVDS Bare Metal configuration with 2 Pnics](#4812-single-n-vds-bare-metal-configuration-with-2-pnics)
         * [4.8.2 - Single NVDS Bare Metal configuration with 6 Pnics](#4813-single-n-vds-bare-metal-configuration-with-six-pnics)
       * [4.8.2 - VM Edge Node](#482-vm-edge-node)
         * [4.8.2.1	- Multiple N-VDS per Edge VM Configuration – NSX-T 2.4 or Older ](#4821-multiple-n-vds-per-edge-vm-configuration--nsx-t-24-or-older)
         * [4.8.2.2	- Single N-VDS Based Configuration - Starting with NSX-T 2.5 release](#4822-single-n-vds-based-configuration---starting-with-nsx-t-25-release)
         *  [4.8.2.3 - VLAN Backed Service Interface on Tier-0 or Tier-1 Gateway](#4823-vlan-backed-service-interface-on-tier-0-or-tier-1-gateway)
        * [4.8.3 - Edge Cluster](#483-edge-cluster)
        * [4.8.4 - Failure Domain](#484-failure-domain)
     * [4.9 - Other Network Services](#49-other-network-services)
       * [4.9.1 - Unicast Reverse Path Forwarding](#491-unicast-reverse-path-forwarding-urpf)
       * [4.9.2 - Network Address Translation](#492-network-address-translation)
       * [4.9.3 - DHCP Services](#493-dhcp-services)
       * [4.9.4 - Metadata Proxy Services](#494-metadata-proxy-service)
       * [4.9.5 - Gateway Firewall Service](#495-gateway-firewall-services)
       * [4.9.6 - Proxy ARP](#496-proxy-arp)
     * [4.10 - Topology Consideration](#410-topology-consideration)
       * [4.10.1 - Supported Topology](#4101-supported-topologies)
       * [4.10.2 - Unsupported Topology](#4102-unsupported-topologies)


# 4 NSX-T Logical Routing

The logical routing capability in the NSX-T platform provides the ability to interconnect both virtual and physical workloads deployed in different logical L2 networks. NSX-T enables the creation of network elements like segments (Layer 2 broadcast domains) and gateways (routers) in software as logical constructs and embeds them in the hypervisor layer, abstracted from the underlying physical hardware. Since these network elements are logical entities, multiple gateways can be created in an automated and agile fashion.
 
The previous chapter showed how to create segments; this chapter focuses on how gateways provide connectivity between different logical L2 networks. Figure 4-1 shows both logical and physical view of a routed topology connecting segments/logical switches on multiple hypervisors. Virtual machines “Web1” and “Web2” are connected to “overlay Segment 1” while “App1” and “App2” are connected to “overlay Segment 2”. 


<p align="center">
    <img src="images/Figure4-1.png">
</p>
<p align="center">
Figure 4‑1: Logical and Physical View of Routing Services
</p>


In a data center, traffic is categorized as East-West (E-W) or North-South (N-S) based on the origin and destination of the flow. When virtual or physical workloads in a data center communicate with the devices external to the data center (e.g., WAN, Internet), the traffic is referred to as North-South traffic. The traffic between workloads confined within the data center is referred to as East-West traffic. In modern data centers, more than 70% of the traffic is East-West.

For a multi-tiered application where the web tier needs to talk to the app tier and the app tier needs to talk to the database tier and, these different tiers sit in different subnets. Every time a routing decision is made, the packet is sent to the router. Traditionally, a centralized router would provide routing for these different tiers. With VMs that are hosted on same the ESXi or KVM hypervisor, traffic will leave the hypervisor multiple times to go to the centralized router for a routing decision, then return to the same hypervisor; this is not optimal.

NSX-T is uniquely positioned to solve these challenges as it can bring networking closest to the workload. Configuring a Gateway via NSX-T Manager instantiates a local distributed gateway on each hypervisor. For the VMs hosted (e.g., “Web 1”, “App 1”) on the same hypervisor, the E-W traffic does not need to leave the hypervisor for routing.


## 4.1 Single Tier Routing

NSX-T Gateway provides optimized distributed routing as well as centralized routing and services like NAT, AVI Load balancer, DHCP server etc. A single tier routing topology implies that a Gateway is connected to segments southbound providing E-W routing and is also connected to physical infrastructure to provide N-S connectivity. This gateway is referred to as Tier-0 Gateway. 

Tier-0 Gateway consists of two components: distributed routing component (DR) and centralized services routing component (SR).

### 4.1.1 Distributed Router (DR)

A DR is essentially a router with logical interfaces (LIFs) connected to multiple subnets. It runs as a kernel module and is distributed in hypervisors across all transport nodes, including Edge nodes. The traditional data plane functionality of routing and ARP lookups is performed by the logical interfaces connecting to the different segments. Each LIF has a vMAC address and an IP address being the default IP gateway for its connected segment. The IP address is unique per LIF and remains the same everywhere the segment exists. The vMAC associated with each LIF remains constant in each hypervisor, allowing the default gateway IP and MAC addresses to remain the same during vMotion. 

The left side of Figure 4-2 shows a logical topology with two segments, “Web Segment” with a default gateway of 172.16.10.1/24 and “App Segment” with a default gateway of 172.16.20.1/24 are attached to Tier-0 Gateway. In the physical topology view on the right, VMs are shown on two hypervisors, “HV1” and “HV2”.  A distributed routing (DR) component for this Tier-0 Gateway is instantiated as a kernel module and will act as a local gateway or first hop router for the workloads connected to the segments. Please note that the DR is not a VM and the DR on both hypervisors has the same IP addresses.

<p align="center">
    <img src="images/Figure4-2.png">
</p>
<p align="center">
Figure 4‑2: E-W Routing with Workloads on the same Hypervisor
</p>

**East-West Routing - Distributed Routing with Workloads on the Same Hypervisor**

In this example, “Web1” VM is connected to “Web Segment” and “App1” is connected to “App-Segment” and both VMs are hosted on the same hypervisor. Since “Web1” and “App1” are both hosted on hypervisor “HV1”, routing between them happens on the DR located on that same hypervisor.

Figure 4-3 presents the logical packet flow between two VMs on the same hypervisor

<p align="center">
    <img src="images/Figure4-3.png">
</p>
<p align="center">
Figure 4‑3: Packet Flow between two VMs on same Hypervisor
</p>

1.	“Web1” (172.16.10.11) sends a packet to “App1” (172.16.20.11). The packet is sent to the default gateway interface (172.16.10.1) for “Web1” located on the local DR. 
2.	The DR on “HV1” performs a routing lookup which determines that the destination subnet 172.16.20.0/24 is a directly connected subnet on “LIF2”. A lookup is performed in the “LIF2” ARP table to determine the MAC address associated with the IP address for “App1”. If the ARP entry does not exist, the controller is queried. If there is no response from controller, an ARP request is flooded to learn the MAC address of “App1”.
3.	Once the MAC address of “App1” is learned, the L2 lookup is performed in the local MAC table to determine how to reach “App1” and the packet is delivered to the App1 VM. 
4.	The return packet from “App1” follows the same process and routing would happen again on the local DR.

In this example, neither the initial packet from “Web1” to “App1” nor the return packet from “App1” to “Web1” left the hypervisor.

**East-West Routing - Distributed Routing with Workloads on Different Hypervisor**

In this example, the target workload “App2” differs as it rests on a hypervisor named “HV2”. If “Web1” needs to communicate with “App2”, the traffic would have to leave the hypervisor “HV1” as these VMs are hosted on two different hypervisors. Figure 4-4 shows a logical view of topology, highlighting the routing decisions taken by the DR on “HV1” and the DR on “HV2”.
When “Web1” sends traffic to “App2”, routing is done by the DR on “HV1”. The reverse traffic from “App2” to “Web1” is routed by DR on “HV2”. Routing is performed on the hypervisor attached to the source VM.

<p align="center">
    <img src="images/Figure4-4.png">
</p>
<p align="center">
Figure 4‑4: E-W Packet Flow between two Hypervisors
</p>


Figure 4-5 shows the corresponding physical topology and packet walk from “Web1” to “App2”.

<p align="center">
    <img src="images/Figure4-5.png">
</p>
<p align="center">
Figure 4‑5: End-to-end E-W Packet Flow
</p>

1.	“Web1” (172.16.10.11) sends a packet to “App2” (172.16.20.12). The packet is sent to the default gateway interface (172.16.10.1) for “Web1” located on the local DR. Its L2 header has the source MAC as “MAC1” and destination MAC as the vMAC of the DR. This vMAC will be the same for all LIFs.
2.	The routing lookup happens on the HV1 DR, which determines that the destination subnet 172.16.20.0/24 is a directly connected subnet on “LIF2”. A lookup is performed in the “LIF2” ARP table to determine the MAC address associated with the IP address for “App2”. This destination MAC, “MAC2”, is learned via the remote HV2 TEP 20.20.20.20.
3.	HV1 TEP encapsulates the original packet and sends it to the HV2 TEP with a source IP address of 10.10.10.10 and destinations IP address of 20.20.20.20 for the encapsulating packet. The destination virtual network identifier (VNI) in the Geneve encapsulated packet belongs to “App Segment”.
4.	HV2 TEP 20.20.20.20 decapsulates the packet, removing the outer header upon reception. It performs an L2 lookup in the local MAC table associated with “LIF2”. 
5.	Packet is delivered to “App2” VM.

The return packet from “App2” destined for “Web1” goes through the same process. For the return traffic, the routing lookup happens on the HV2 DR. This represents the normal behavior of the DR, which is to always perform routing on the DR instance running in the kernel of the hypervisor hosting the workload that initiates the communication. After the routing lookup, the packet is encapsulated by the HV2 TEP and sent to the remote HV1 TEP. The HV1 decapsulates the Geneve packet and delivers the encapsulated frame from “App2” to “Web1”.



### 4.1.2 Services Router (SR)

East-West routing is completely distributed in the hypervisor, with each hypervisor in the transport zone running a DR in its kernel. However, some services of NSX-T are not distributed, due to its locality or stateful nature such as:

* Physical infrastructure connectivity (BGP Routing with Address Families – VRF lite)
* NAT
* DHCP server
* VPN
* Gateway Firewall
* Bridging
* Service Interface
* Metadata Proxy for OpenStack

A services router (SR) is instantiated on an edge cluster when a service is enabled that cannot be distributed on a gateway. 

A centralized pool of capacity is required to run these services in a highly available and scaled-out fashion. The appliances where the centralized services or SR instances are hosted are called Edge nodes. An Edge node is the appliance that provides connectivity to the physical infrastructure.

Left side of Figure 4-6 shows the logical view of a Tier-0 Gateway showing both DR and SR components when connected to a physical router. Right side of Figure 4-6 shows how the components of Tier-0 Gateway are realized on Compute hypervisor and Edge node. Note that the compute host (i.e. HV1) has just the DR component and the Edge node shown on the right has both the SR and DR components. SR/DR forwarding table merge has been done to address future use-cases. SR and DR functionality remains the same after SR/DR merge in NSX-T 2.4 release, but with this change SR has direct visibility into the overlay segments. Notice that all the overlay segments are attached to the SR as well.

<p align="center">
    <img src="images/Figure4-6.png">
</p>
<p align="center">
Figure 4‑6: Logical Router Components and Interconnection
</p>

A Tier-0 Gateway can have following interfaces:
* External Interface – Interface connecting to the physical infrastructure/router. Static routing and BGP are supported on this interface. This interface was referred to as uplink interface in previous releases. This interface can also be used to extend a VRF (Virtual routing and forwarding instance) from the physical networking fabric into the NSX domain.
* Service Interface: Interface connecting VLAN segments to provide connectivity and services to VLAN backed physical or virtual workloads. Service interface can also be connected to overlay segments for Tier-1 standalone load balancer use-cases explained in Load balancer Chapter 6 . This interface was referred to as centralized service port (CSP) in previous releases. Note that a gateway must have a SR component to realize service interface.  NSX-T 3.0 supports static and dynamic routing over this interface.
* Linked Segments – Interface connecting to an overlay segment. This interface was referred to as downlink interface in previous releases. Static routing is supported over that interface
* Intra-Tier Transit Link (Internal link between the DR and SR). A transit overlay segment is auto plumbed between DR and SR and each end gets an IP address assigned in 169.254.0.0/24 subnet by default. This address range is configurable only when creating the Tier-0 gateway.

As mentioned previously, connectivity between DR on the compute host and SR on the Edge node is auto plumbed by the system. Both the DR and SR get an IP address assigned in 169.254.0.0/24 subnet by default. The management plane also configures a default route on the DR with the next hop IP address of the SR’s intra-tier transit link IP. This allows the DR to take care of E-W routing while the SR provides N-S connectivity.


**North-South Routing by SR Hosted on Edge Node**

From a physical topology perspective, workloads are hosted on hypervisors and N-S connectivity is provided by Edge nodes. If a device external to the data center needs to communicate with a virtual workload hosted on one of the hypervisors, the traffic would have to come to the Edge nodes first. This traffic will then be sent on an overlay network to the hypervisor hosting the workload. Figure 4-7 shows the traffic flow from a VM in the data center to an external physical infrastructure.


<p align="center">
    <img src="images/Figure4-7.png">
</p>
<p align="center">
Figure 4‑7: N-S Routing Packet Flow


Figure 4-8 shows a detailed packet walk from data center VM “Web1” to a device on the L3 physical infrastructure. As discussed in the E-W routing section, routing always happens closest to the source. In this example, eBGP peering has been established between the physical router interface with the IP address 192.168.240.1 and the Tier-0 Gateway SR component hosted on the Edge node with an external interface IP address of 192.168.240.3. Tier-0 Gateway SR has a BGP route for 192.168.100.0/24 prefix with a next hop of 192.168.240.1 and the physical router has a BGP route for 172.16.10.0/24 with a next hop of 192.168.240.3. 


<p align="center">
    <img src="images/Figure4-8.png">
</p>
<p align="center">
Figure 4‑8: End-to-end Packet Flow – Application “Web1” to External

1.	“Web1” (172.16.10.11) sends a packet to 192.168.100.10. The packet is sent to the “Web1” default gateway interface (172.16.10.1) located on the local DR. 
2.	The packet is received on the local DR. DR doesn’t have a specific connected route for 192.168.100.0/24 prefix. The DR has a default route with the next hop as its corresponding SR, which is hosted on the Edge node. 
3.	The HV1 TEP encapsulates the original packet and sends it to the Edge node TEP with a source IP address of 10.10.10.10 and destination IP address of 30.30.30.30.
4.	The Edge node is also a transport node. It will encapsulate/decapsulate the traffic sent to or received from compute hypervisors. The Edge node TEP decapsulates the packet, removing the outer header prior to sending it to the SR.
5.	The SR performs a routing lookup and determines that the route 192.168.100.0/24 is learned via external interface with a next hop IP address 192.168.240.1. 
6.	The packet is sent on the VLAN segment to the physical router and is finally delivered to 192.168.100.10.

Observe that routing and ARP lookup happens on the DR hosted on the HV1 hypervisor to determine that the packet must be sent to the SR. On the edge node, the packet is directly sent to the SR after the tunnel encapsulation has been removed.

Figure 4-9 follows the packet walk for the reverse traffic from an external device to “Web1”.


<p align="center">
    <img src="images/Figure4-9.png">
</p>
<p align="center">
Figure 4‑9: End-to-end Packet Flow – Application “Web1” to External

1.	An external device (192.168.100.10) sends a packet to “Web1” (172.16.10.11). The packet is routed by the physical router and sent to the external interface of Tier-0 Gateway hosted on Edge node.
2.	A single routing lookup happens on the Tier-0 Gateway SR which determines that 172.16.10.0/24 is a directly connected subnet on “LIF1”. A lookup is performed in the “LIF1” ARP table to determine the MAC address associated with the IP address for “Web1”. This destination MAC “MAC1” is learned via the remote TEP (10.10.10.10), which is the “HV1” host where “Web1” is located.
3.	The Edge TEP encapsulates the original packet and sends it to the remote TEP with an outer packet source IP address of 30.30.30.30 and destination IP address of 10.10.10.10. The destination VNI in this Geneve encapsulated packet is of “Web Segment”.
4.	The HV1 host decapsulates the packet and removes the outer header upon receiving the packet. An L2 lookup is performed in the local MAC table associated with “LIF1”.
5.	The packet is delivered to Web1.


This time routing and ARP lookup happened on the merged SR/DR hosted on the Edge node. No such lookup was required on the DR hosted on the HV1 hypervisor, and packet was sent directly to the VM after removing the tunnel encapsulation header.

Figure 4-9 showed a Tier-0 gateway with one external interface that leverages Edge node to connect to physical infrastructure. If this Edge node goes down, N-S connectivity along with other centralized services running on Edge node will go down as well. 

To provide redundancy for centralized services and N-S connectivity, it is recommended to deploy a minimum of two edge nodes. High availability modes are discussed in section 4.6 .


## 4.2 Multi Tier Routing

In addition to providing optimized distributed and centralized routing functions, NSX-T supports a multi-tiered routing model with logical separation between different gateways within the NSX-T infrastructure. The top-tier gateway is referred to as a Tier-0 gateway while the bottom-tier gateway is a Tier-1 gateway. This structure gives complete control and flexibility over services and policies. Various stateful services can be hosted on the Tier-1 while the Tier-0 can operate in an active-active manner.

Configuring two tier routing is not mandatory. It can be single tiered as shown in the previous section. Figure 4 10 presents an NSX-T two-tier routing architecture. 


<p align="center">
    <img src="images/Figure4-10.png">
</p>
<p align="center">
Figure 4‑10: Two Tier Routing and Scope of Provisioning

Northbound, the Tier-0 gateway connects to one or more physical routers/L3 switches and serves as an on/off ramp to the physical infrastructure. Southbound, the Tier-0 gateway connects to one or more Tier-1 gateways or directly to one or more segments as shown in North-South routing section. Northbound, the Tier-1 gateway connects to a Tier-0 gateway using a RouterLink port. Southbound, it connects to one or more segments using downlink interfaces.

Concepts of DR/SR discussed in the [section 4.1](#41-single-tier-routing) remain the same for multi-tiered routing. Like Tier-0 gateway, when a Tier-1 gateway is created, a distributed component (DR) of the Tier-1 gateway is intelligently instantiated on the hypervisors and Edge nodes. Before enabling a centralized service on a Tier-0 or Tier-1 gateway, an edge cluster must be configured on this gateway. Configuring an edge cluster on a Tier-1 gateway, instantiates a corresponding Tier-1 services component (SR) on two Edge nodes part of this edge cluster. Configuring an Edge cluster on a Tier-0 gateway does not automatically instantiate a Tier-0 service component (SR), the service component (SR) will only be created on a specific edge node along with the external interface creation. 

Unlike the Tier-0 gateway, the Tier-1 gateway does not support northbound connectivity to the physical infrastructure. A Tier-1 gateway can only connect northbound to:
•	a Tier-0 gateway,
•	a service port, this is used to connect a one-arm load-balancer to a segment. More details are available in Chapter 6.

Note that connecting Tier-1 to Tier-0 is a one click configuration or one API call configuration regardless of components instantiated (DR and SR) for that gateway. 

### 4.2.1 Interfaces Types on Tier-1 and Tier-0 Gateway

External and Service interfaces were previously introduced in the services router section. Figure 4 11 shows these interfaces types along with a new “RouterLink” interface in a two-tiered topology.

<p align="center">
    <img src="images/Figure4-11.png">
</p>
<p align="center">
Figure 4‑11: Anatomy of Components with Logical Routing

-   **External Interface:** Interface connecting to the physical infrastructure/router. Static routing and BGP are supported on this interface. This interface only exists on Tier-0 gateway. This interface was referred to as Uplink interface in previous releases. This interface type will also be used to extend a VRF (Virtual Routing and Forwarding) from the physical networking fabric into the NSX domain.
-   **Router Link Interface/Linked Port:** Interface connecting Tier-0 and Tier-1 gateways. Each Tier-0-to-Tier-1 peer connection is provided a /31 subnet within the 100.64.0.0/16 reserved address space (RFC6598). This link is created automatically when the Tier-0 and Tier-1 gateways are connected. This subnet can be changed when the Tier-0 gateway is being created. It is not possible to change it afterward. 
-   **Service Interface:**  Interface connecting VLAN segments to provide connectivity to VLAN backed physical or virtual workloads. Service interface can also be connected to overlay/VLAN segments for standalone load balancer use cases explained in load balancer Chapter 6. Service Interface supports static and dynamic routing starting with NSX-T 3.0. It is supported on both Tier-0 and Tier-1 gateways configured in Active/Standby high-availability configuration mode explained in section 4.6.2. Note that a Tier-0 or Tier-1 gateway must have an SR component to realize service interfaces. This interface was referred to as centralized service interface in previous releases.
-   **Loopback Interface:** Tier-0 gateway supports the loopback interfaces. A Loopback interface is a virtual interface, and it can be redistributed into a routing protocol.


### 4.2.2 Route Types on Tier-0 and Tier-1 Gateways

There is no dynamic routing between Tier-0 and Tier-1 gateways. The NSX-T platform takes care of the auto-plumbing between Tier-0 and Tier-1 gateways. The following list details route types on Tier-0 and Tier-1 gateways.


-   **Tier-0 Gateway**

    -   Connected – Connected routes on Tier-0 include external
        interface subnets, service interface subnets, loopback and
        segment subnets connected to Tier-0. In Figure 4‑12, 
        172.16.20.0/24 (Connected segment), 192.168.20.0/24 (Service
        Interface) and 192.168.240.0/24 (External interface) are
        connected routes for the Tier-0 gateway.

    -   Static – User configured static routes on Tier-0.

    -   NAT IP – NAT IP addresses owned by the Tier-0 gateway discovered
        from NAT rules configured on Tier-0 Gateway.

    -   BGP Routes learned via BGP neighbors.

    -   IPsec Local IP – Local IPsec endpoint IP address for
        establishing VPN sessions.

    -   DNS Forwarder IP – Listener IP for DNS queries from clients.
        Also used as the source IP to forward DNS queries to the
        upstream DNS server.

    -   Inter SR. SRs of a same Tier-0 gateway in the same edge cluster
        will create an automatic iBGP peering adjacency between them to
        exchange routing information. This topology is only supported
        with Active/Active topologies and with NSX-T Federation.

-   **Tier-1 Gateway**

    -   Connected – Connected routes on Tier-1 include segment subnets
        connected to Tier-1 and service interface subnets configured on
        Tier-1 gateway. In Figure 4‑12, 172.16.10.0/24 (Connected segment) and 192.168.10.0/24 (Service Interface) are connected routes for Tier-1 gateway.

-   Static– User configured static routes on Tier-1 gateway.

-   NAT IP – NAT IP addresses owned by the Tier-1 gateway discovered
    from NAT rules configured on the Tier-1 gateway.

-   LB VIP – IP address of load balancing virtual server.

-   LB SNAT – IP address or a range of IP addresses used for Source NAT
    by load balancer.

-   IPsec Local IP – Local IPsec endpoint IP address for establishing
    VPN sessions.

-   DNS Forwarder IP – Listener IP for DNS queries from clients. Also
    used as the source IP to forward DNS queries to the upstream DNS
    server.

**Route Advertisement on the Tier-1 and Tier-0 Logical Router**

The Tier-0 gateway could use static routing or BGP to connect to the
physical routers. The Tier-1 gateway cannot connect to physical routers
directly; it must connect to a Tier-0 gateway to provide N-S
connectivity to the subnets attached to it. When a Tier-1 gateway is
connected to a Tier-0 gateway, a default route is automatically created
on the Tier-1. That default route is pointing to the RouterLink IP
address that is owned by the Tier-0.

Figure 4‑12 explains the route advertisement on both the Tier-1 and
Tier-0 gateway.

<p align="center">
    <img src="images/Figure4-12.png">
</p>
<p align="center">
Figure 4‑12: Routing Advertisement 

“Tier-1 Gateway” advertises connected routes to Tier-0 Gateway. Figure
4‑12 shows an example of connected routes (172.16.10.0/24 and
192.168.10.0/24). If there are other route types, like NAT IP etc. as
discussed in section 4.2.2, a user can advertise those route types as
well. As soon as “Tier-1 Gateway” is connected to “Tier-0 Gateway”, the
management plane configures a default route on “Tier-1 Gateway” with
next hop IP address as RouterLink interface IP of “Tier-0 Gateway” i.e.
100.64.224.0/31 in the example above.

Tier-0 Gateway sees 172.16.10.0/24 and 192.168.10.1/24 as Tier-1
Connected routes (t1c) with a next hop of 100.64.224.1/31. Tier-0
Gateway also has Tier-0 “Connected” routes (172.16.20.0/24) in Figure
4‑12.

Northbound, “Tier-0 Gateway” redistributes the Tier-0 connected and
Tier-1 connected routes in BGP and advertises these routes to its BGP
neighbor, the physical router.

### 4.2.3 Fully Distributed Two Tier Routing

NSX-T provides a fully distributed routing architecture. The motivation
is to provide routing functionality closest to the source. NSX-T
leverages the same distributed routing architecture discussed in
distributed router section and extends that to multiple tiers.

Figure 4‑13 shows both logical and per transport node views of two
Tier-1 gateways serving two different tenants and a Tier-0 gateway. Per
transport node view shows that the distributed component (DR) for Tier-0
and the Tier-1 gateways have been instantiated on two hypervisors.

<p align="center">
    <img src="images/Figure4-13.png">
</p>
<p align="center">
Figure 4‑12: Logical Routing Instances 

If “VM1” in tenant 1 needs to communicate with “VM3” in tenant 2,
routing happens locally on hypervisor “HV1”. This eliminates the need to
route of traffic to a centralized location to route between different
tenants or environments.

**Multi-Tier Distributed Routing with Workloads on the same Hypervisor**

The following list provides a detailed packet walk between workloads
residing in different tenants but hosted on the same hypervisor.

1.  “VM1” (172.16.10.11) in tenant 1 sends a packet to “VM3”
    (172.16.201.11) in tenant 2. The packet is sent to its default
    gateway interface located on tenant 1, the local Tier-1 DR.

2.  Routing lookup happens on the tenant 1 Tier-1 DR and the packet is
    routed to the Tier-0 DR following the default route. This default
    route has the RouterLink interface IP address (100.64.224.0/31) as a
    next hop.

3.  Routing lookup happens on the Tier-0 DR. It determines that the
    172.16.201.0/24 subnet is learned from the tenant 2 Tier-1 DR
    (100.64.224.3/31) and the packet is routed there.

4.  Routing lookup happens on the tenant 2 Tier-1 DR. This determines
    that the 172.16.201.0/24 subnet is directly connected. L2 lookup is
    performed in the local MAC table to determine how to reach “VM3” and
    the packet is sent.

The reverse traffic from “VM3” follows the similar process. A packet
from “VM3” to destination 172.16.10.11 is sent to the tenant-2 Tier-1
DR, then follows the default route to the Tier-0 DR. The Tier-0 DR
routes this packet to the tenant 1 Tier-1 DR and the packet is delivered
to “VM1”. During this process, the packet never left the hypervisor to
be routed between tenants.

**Multi-Tier Distributed Routing with Workloads on different
Hypervisors**

Figure 4‑14 shows the packet flow between workloads in different tenants
which are also located on different hypervisors.

<p align="center">
    <img src="images/Figure4-14.png">
</p>
<p align="center">
Figure 4‑14: Logical routing end-to-end packet Flow between hypervisor

The following list provides a detailed packet walk between workloads residing in different tenants and hosted on the different hypervisors.
1.	“VM1” (172.16.10.11) in tenant 1 sends a packet to “VM2” (172.16.200.11) in tenant 2. VM1 sends the packet to its default gateway interface located on the local Tier-1 DR in HV1. 
2.	Routing lookup happens on the tenant 1 Tier-1 DR and the packet follows the default route to the Tier-0 DR with a next hop IP of 100.64.224.0/31.
3.	Routing lookup happens on the Tier-0 DR which determines that the 172.16.200.0/24 subnet is learned via the tenant 2 Tier-1 DR (100.64.224.3/31) and the packet is routed accordingly.
4.	Routing lookup happens on the tenant 2 Tier-1 DR which determines that the 172.16.200.0/24 subnet is a directly connected subnet. A lookup is performed in ARP table to determine the MAC address associated with the “VM2” IP address. This destination MAC is learned via the remote TEP on hypervisor “HV2”.
5.	The “HV1” TEP encapsulates the packet and sends it to the “HV2” TEP, finally leaving the host.
6.	The “HV2” TEP decapsulates the packet and recognize the VNI in the Geneve header. A L2 lookup is performed in the local MAC table associated to the LIF where “VM2” is connected.
7.	The packet is delivered to “VM2”.

The return packet follows the same process. A packet from “VM2” gets routed to the local hypervisor Tier-1 DR and is sent to the Tier-0 DR. The Tier-0 DR routes this packet to tenant 1 Tier-1 DR which performs the L2 lookup to find out that the MAC associated with “VM1” is on remote hypervisor “HV1”. The packet is encapsulated by “HV2” and sent to “HV1”, where this packet is decapsulated and delivered to “VM1". It is important to notice that in this use case, routing is performed locally on the hypervisor hosting the VM sourcing the traffic.

## 4.3 Routing Capabilities

NSX-T supports static routing and the dynamic routing protocol BGP on Tier-0 Gateways for IPv4 and IPv6 workloads. In addition to static routing and BGP, Tier-0 gateway also supports a dynamically created iBGP session between its Services router component. This feature is referred as Inter-SR routing and is available for active-active Tier-0 topologies only.

 Tier-1 Gateways support static routes but do not support any dynamic routing protocols.


### 4.3.1 Static Routing

Northbound, static routes can be configured on Tier-1 gateways with the next hop IP as the Routerlink IP of the Tier-0 gateway (100.64.0.0/16 range or a range defined by user for Routerlink interface). Southbound, static routes can also be configured on Tier-1 gateway with a next hop as a layer 3 device reachable via Service interface.

Tier-0 gateways can be configured with a static route toward external subnets with a next hop IP of the physical router. Southbound, static routes can be configured on Tier-0 gateways with a next hop of a layer 3 device reachable via Service interface.

ECMP is supported with static routes to provide load balancing, increased bandwidth, and fault tolerance for failed paths or Edge nodes. Figure 4 15 shows a Tier-0 gateway with two external interfaces leveraging Edge node, EN1 and EN2 connected to two physical routers. Two equal cost static default routes configured for ECMP on Tier-0 Gateway. Up to eight paths are supported in ECMP. The current hash algorithm for ECMP is two-tuple, based on source and destination IP of the traffic.

<p align="center">
    <img src="images/Figure4-15.png">
</p>
<p align="center">
Figure 4‑15: Static Routing Configuration

BFD can also be enabled for faster failure detection of next hop and is configured in the static route. In NSX-T 3.0, BFD keep alive TX/RX timer can range from a minimum of 50ms (for Bare Metal Edge Node) to maximum of 10,000ms. Default BFD keep alive TX/RX timers are set to 500ms with three retries.

### 4.3.2 Dynamic Routing

BGP is the de facto protocol on the WAN and in most modern data centers. A typical leaf-spine topology has eBGP running between leaf switches and spine switches. 

Tier-0 gateways support eBGP and iBGP on the external interfaces with physical routers. BFD can also be enabled per BGP neighbor for faster failover. BFD timers depend on the Edge node type. Bare metal Edge supports a minimum of 50ms TX/RX BFD keep alive timer while the VM form factor Edge supports a minimum of 500ms TX/RX BFD keep alive timer.

With NSX-T 3.0 release, the following BGP features are supported:

BGP is the de facto protocol on the WAN and in most modern data centers.
A typical leaf-spine topology has eBGP running between leaf switches and
spine switches.

Tier-0 gateways support eBGP and iBGP on the external interfaces with
physical routers. BFD can also be enabled per BGP neighbor for faster
failover. BFD timers depend on the Edge node type. Bare metal Edge
supports a minimum of 50ms TX/RX BFD keep alive timer while the VM form
factor Edge supports a minimum of 500ms TX/RX BFD keep alive timer.

With NSX-T 3.0 release, the following BGP features are supported:

-   Two and four bytes AS numbers in *asplain, asdot* and *asdot+*
    format.

-   eBGP multi-hop support, allowing eBGP peering to be established on
    loopback interfaces.

-   iBGP

-   eBGP multi-hop BFD

-   ECMP support with BGP neighbors in same or different AS numbers.
    (Multi-path relax)

-   BGP Allow AS in

-   BGP route aggregation support with the flexibility of advertising a
    summary route only to the BGP peer or advertise the summary route
    along with specific routes. A more specific route must be present in
    the routing table to advertise a summary route.

-   Route redistribution in BGP to advertise Tier-0 and Tier-1 Gateway
    internal routes as mentioned in section 4.2.2.

-   Inbound/outbound route filtering with BGP peer using prefix-lists or
    route-maps.

-   Influencing BGP path selection by setting Weight, Local preference,
    AS Path Prepend, or MED.

-   Standard, Extended and Large BGP community support.

-   BGP well-known community names (e.g., no-advertise, no-export,
    no-export-subconfed) can also be included in the BGP route updates
    to the BGP peer.

-   BGP communities can be set in a route-map to facilitate matching of
    communities at the upstream router.

-   Graceful restart (Full and Helper mode) in BGP.

-   BGP peering authentication using plaintext or MD5.

-   MP-BGP as the control plane protocol for VXLAN overlays or IPv6
    address families.

Active/active ECMP services supports up to eight paths. The ECMP hash algorithm is 5-tuple northbound of Tier-0 SR.  ECMP hash is based on the source IP address, destination IP address, source port, destination port and IP protocol. The hashing algorithm determines how incoming traffic is forwarded to the next-hop device when there are multiple paths. ECMP hashing algorithm from DR to multiple SRs is 2-tuple and is based on the source IP address and destination IP address.

**Graceful Restart**

Graceful restart in BGP allows a BGP speaker to preserve its forwarding table while the control plane restarts. It is recommended to enable BGP Graceful restart when the BGP peer has multiple supervisors. A BGP control plane restart could happen due to a supervisor switchover in a dual supervisor hardware, planned maintenance, or active routing engine crash. As soon as a GR-enabled router restarts (control plane failure), it preserves its forwarding table , marks the routes as stale, and sets a grace period restart timer for the BGP session to reestablish. If the BGP session reestablishes during this grace period, route revalidation is done, and the forwarding table is updated. If the BGP session does not reestablish within this grace period, the router flushes the stale routes. 
 
The BGP session will not be GR capable if only one of the peers advertises it in the BGP OPEN message; GR needs to be configured on both ends. GR can be enabled/disabled per Tier-0 gateway. The GR restart timer is 180 seconds by default and cannot be change after a BGP peering adjacency is in the established state, otherwise the peering needs to be negotiated again. 

# 4.4 VRF Lite

# 4.4.1 VRF Lite Generalities

Virtual Routing Forwarding (VRF) is a virtualization method that consists of creating multiple logical routing instances within a physical routing appliance. It provides a complete control plane isolation between routing instances. VRF instances are commonly used in enterprise and service providers networks to provide control and data plane isolation, allowing several use cases such as overlapping IP addressing between tenants, isolation of regulated workload, isolation of external and internal workload as well as hardware resources consolidation. Starting with NSX-T 3.0 it is possible to extent the VRF present on the physical network onto the NSX-T domain. Creating a development environment that replicates the production environment is a typical use case for VRF. 

Another representative use case for VRF is when multiple environments needs to be isolated from each other. As stated previously, VRF instances are isolated between each other by default; allowing communications between these environments using the Route Leaking VRF feature is possible. While this feature allows inter-VRF communications, it is important to emphasize that scalability can become an issue if a design permits all VRF to communicate between each other. In this case, VRF might not be the option. VRF should not be replaced in lieu of the DFW construct.
 

Figure 4-16 pictures a traditional VRF architecture. Logical (Switch Virtual Interface – Vlan interface) or Physical interface should be dedicated to a single VRF instance. In the following diagram, interface e1/1 and e2/2 belong to VRF-A while interface e1/2 and e2/2 belong to VRF-B. Each VRF will run their own dynamic routing protocol (or use static routes)

<p align="center">
    <img src="images/Figure4-16.png">
</p>
<p align="center">
Figure 4‑16: Networking VRF Architecture

With NSX-T 2.x, several multi-tenant designs are possible.
The first option was to deploy a Tier-1 gateway for each tenant while a shared Tier-0 provides connectivity to the physical networking fabric for all tenants. Figure 4-17 diagrams that option.


<p align="center">
    <img src="images/Figure4-17.png">
</p>
<p align="center">
Figure 4‑17: NSX-T 2.x multi-tenant architecture. – Shared Tier-0 Gateway


Another supported design is to deploy a separate Tier-0 gateway for each tenant on a dedicated tenant edge node. Figure 4-18 shows a traditional multi-tenant architecture using dedicated Tier-0 per tenant in NSX-T 2.X .


<p align="center">
    <img src="images/Figure4-18.png">
</p>
<p align="center">
Figure 4‑18: NSX-T 2.x multi-tenant architecture. Dedicated Tier0 for each tenant

In traditional networking, VRF instances are hosted on a physical appliance and share the resources with the global routing table. Starting with NSX-T 3.0, Virtual Routing and Forwarding (VRF) instances configured on the physical fabric can be extended to the NSX-T domain. A VRF Tier-0 gateway must be hosted on a traditional Tier-0 gateway identified as the “Parent Tier-0”. Figure 4-19 diagrams an edge node hosting a traditional Tier-0 gateway with two VRF gateways. Control plane is completely isolated between all the Tier-0 gateways instances. 

<p align="center">
    <img src="images/Figure4-19.png">
</p>
<p align="center">
Figure 4‑19: Tier-0 VRF Gateways hosted on a Parent Tier-0 Gateway 


The parent Tier-0 gateway can be considered as the global routing table and must have connectivity to the physical fabric. A unique Tier-0 gateway instance (DR and SR) will be created and dedicated to a VRF. Figure 4-20 shows a detailed representation of the Tier-0 VRF gateway with their respective Service Router and Distributed Router components.


<p align="center">
    <img src="images/Figure4-20.png">
</p>
<p align="center">
Figure 4‑20: Detailed representation of the SR/DR component for Tier-0 VRF hosted on an edge node 

Figure 4-21 shows a typical single tier routing architecture with two Tier-0 VRF gateways connected to their parent Tier-0 gateway. Traditional segments are connected to a Tier-0 VRF gateway.


<p align="center">
    <img src="images/Figure4-21.png">
</p>
<p align="center">
Figure 4‑21: NSX-T 3.0 multi-tenant architecture. Dedicated Tier-0 VRF Instance for each VRF

Since control plane is isolated between Tier-0 VRF instances and the parent Tier-0 gateway, each Tier-0 VRF needs their own routing configuration using either static routes or BGP. It implies that each Tier-0 VRF will have their own dedicated BGP process and needs to have their dedicated BGP peers. From a data plane standpoint, 802.1q VLAN tags are used to differentiate traffic between the VRFs instances as demonstrated in the previous figure.

NSX-T 3.0 supports BGP and static routes for the Tier-0 VRF gateway. It offers the flexibility to use static routes on a particular Tier-0 VRF while another Tier-0 VRF would use BGP. 

Figure 4-22 shows a topology with two Tier-0 VRF instances and their respective BGP peers on the physical networking fabric. It is important to emphasize that the Parent Tier-0 gateway has a BGP peering adjacency with the physical routers using their respective global routing table and BGP process. 


<p align="center">
    <img src="images/Figure4-22.png">
</p>
<p align="center">
Figure 4‑22: BGP peering Tier-0 VRF gateways and VRF on the networking fabric.

When a Tier-0 VRF is attached to parent Tier-0, multiple parameters will
be inherited by design and cannot be changed:

-   Edge Cluster

-   High Availability mode (Active/Active – Active/Standby)

-   BGP Local AS Number

-   Internal Transit Subnet

-   Tier-0, Tier-1 Transit Subnet.



All other configuration parameters can be independently managed:

-   External Interface IP addresses   

-   BGP neighbor

-   Prefix list, route-map, Redistribution

-   Firewall rules

-   NAT rules

As mentioned previously, The Tier-0 VRF is hosted on the Parent Tier-0 and will follow the high availability mode and state of its Parent Tier-0.
Both Active/Active or Active/Standby high availability mode are supported on the Tier-0 VRF gateways. It is not possible to have an Active/Active Tier-0 VRF hosted on an Active/Standby Parent Tier-0.

In a traditional Active/Standby design, a Tier-0 gateway failover can be triggered if all northbound BGP peers are unreachable. Similar to the high availability construct between the Tier-0 VRF and the Parent Tier-0, the BGP peering design must match between the VRF Tier-0 and the Parent Tier-0.

Inter-SR routing is not supported in Active/Active VRF topologies. 

Figure 4-23 represents a BGP instance from both parent Tier-0 and Tier-0 VRF point of view. This topology is supported as each Tier-0 SR (on the parent and on the VRF itself) have a redundant path towards the network infrastructure. Both the Parent Tier-0 gateway and the Tier-0 VRF gateway are peering with the same physical networking device but on a different BGP process. 
The Parent Tier-0 gateways peer with both top of rack switches on their respective global BGP process while the Tier-0 VRF gateways peer with both top of rack switch on another BGP process dedicated to the VRF. 
In this particular case the Tier-0 VRF leverages physical redundancy towards the networking fabric if one of its northbound link fails. 


<p align="center">
    <img src="images/Figure4-23.png">
</p>
<p align="center">
Figure 4‑23: Supported BGP peering Design

Figure 4-24 represents an unsupported VRF Active/Active design where different routes are learned from different physical routers. Both the Parent Tier-0 and VRF Tier-0 gateways are learning their default route from a single physical router. The active VRF Tier-0 must specific routes from a single BGP peer. This kind of scenario would be supported for traditional Tier-0 architecture as Inter-SR would provide a redundant path to the networking fabric. This VRF architecture is not supported in NSX-T 3.0.


<p align="center">
    <img src="images/Figure4-24.png">
</p>
<p align="center">
Figure 4‑24: Unsupported Active/Active Topology with VRF 

Figure 4-25 demonstrates the traffic being as one internet router fails and Tier-0 VRF gateways can’t leverage another redundant path to reach the destination. Since the Parent Tier-0 gateway has an established BGP peering adjacency, failover will not be triggered, and traffic will be blackholed on the Tier-0 VRF.


<p align="center">
    <img src="images/Figure4-25.png">
</p>
<p align="center">
Figure 4‑25: Unsupported Active-Active Topology with VRF – Failure

On the Parent Tier-0:

1.  VM “172.16.10.0” sends its IP traffic towards the internet through
    the Tier-0 DR.

2.  Since the Tier-0 topology is Active/Active, the Tier-0 DR sends the
    traffic to both Tier-0 SR1 and Tier-0 SR2 using a 2 tuple.

3.  From a Tier-0 SR1 point of view, the traffic that needs to be routed
    towards the internet will be sent towards Tier-0 SR2 as there is an
    inter-SR BGP adjacency and that Tier-0 SR2 learns the route from
    another internet switch.

4.  Traffic is received by the Tier-0 SR2 and routed towards the
    physical fabric.

On the Tier-0 VRF:

1.  VM “172.16.10.0” sends its IP traffic towards the internet through
    the Tier-0 DR.

2.  Since the Tier-0 topology is Active/Active, the Tier-0 DR sends the
    traffic to both Tier-0 SR1 and Tier-0 SR2 using a 2 tuple.

3.  From a Tier-0 SR1 point of view, the traffic is blackholed as there
    is no inter-SR BGP adjacency between the Tier-0 SRs VRF.


Following the same BGP peering design and principle for Active/Standby
topologies is also mandatory for VRF architectures as the Tier-0 VRF
will inherit the behavior of the parent Tier-0 gateway.

Figure 4-26 represents another unsupported design with VRF architecture.


<p align="center">
    <img src="images/Figure4-26.png">
</p>
<p align="center">
Figure 4‑26: Unsupported design BGP architecture Different peering with the networking fabric.


In this design, traffic will be blackholed on the Tier-0 VRF SR1 as the internet router fails. Since the Tier-0 VRF share its high availability running mode with the Parent Tier-0, it is important to note that the Tier-0 SR1 will not failover to the Tier-0 SR2. The reason behind this behavior is because a failover is triggered only if all northbound BGP sessions change to the “down” state.
Since Tier-0 SR1 still has an active BGP peering with a northbound router on the physical networking fabric, failover will not occur and traffic will be blackholed on the VRF that have only one BGP peer active.

Traditional Tier-1 gateways can be connected to Tier-0 VRF to provide a multi-tier routing architecture as demonstrated in figure 4-27. 


<p align="center">
    <img src="images/Figure4-27.png">
</p>
<p align="center">
Figure 4‑27: Multi-Tier routing architecture and VRF-lite

Stateful services can either run on a Tier-0 VRF gateway or a Tier-1 gateway except for VPN and Load Balancing as these features are not supported on a Tier-0 VRF. Tier-1 SR in charge of the stateful services for a particular VRF will be hosted on the same edge nodes as the Parent Tier-0 Gateway. Figure 4-28 represents stateful services running on traditional Tier-1 gateways SR. 


<p align="center">
    <img src="images/Figure4-28.png">
</p>
<p align="center">
Figure 4‑28: Stateful Services supported on Tier-1 

By default, data-plane traffic between VRF instances is isolated in
NSX-T. By configuring VRF Route Leaking, traffic can be exchanged
between VRF instances .

As a result, static routes must be configured on the Tier-0 VRF
instances to allow traffic to be exchanged. The next hop for these
static routes must not be a Tier-0 gateway (VRF or Parent). 
As a result, multi-tier routing architecture must be implemented to allow traffic to be exchanged between the VRF instances.

Figure 4-29 demonstrates a supported topology for VRF route leaking

Two static routes are necessary:

-   Static Route on Tier-0 VRF A
    -   Destination Subnet: 172.16.20.0/24
    -   Next Hop
        -   IP Address of Tier-1 DR in VRF B (e.g 100.64.80.3)
        -   Admin Distance: 1
        -   Scope: VRF-B
-   Static Route on Tier-0 VRF B
    -   Destination Subnet: 172.16.10.0/24
    -   Next Hop
        -   IP Address of Tier-1 DR in VRF A (e.g 100.64.80.1)
        -   Admin Distance: 1
        -   Scope: VRF-A



<p align="center">
    <img src="images/Figure4-29.png">
</p>
<p align="center">
Figure 4‑29: VRF Route leaking with static routes

VRF-lite also supports northbound VRF route leaking as traffic can be exchanged between a virtual workload on an VRF overlay segment and a bare metal server hosted in a different VRF on the physical networking fabric.
NSX-T VRF route leaking requires that the next hop for the static route must not be a Tier-0 gateway. Static routes pointing to the directly connected IP addresses uplink would not be a recommended design as the static route would fail if an outage would occur on that link or neighbor (Multiple static routes would be needed for redundancy). 

A loopback or virtual IP address host route (/32) can be advertised in the network in the destination VRF.  Since the host route is advertised by both top of rack switches, two ECMP routes will be installed in the Tier-0 VRF. 
Figure 4-30 demonstrates the design and Tier-0 VRF gateways will use all available healthy paths to the networking fabric to reach the server in VRF-B.


<p align="center">
    <img src="images/Figure4-30.png">
</p>
<p align="center">
Figure 4‑30: VRF leaking traffic with northbound destination


Two static routes are necessary:
-   Static Route on Tier-0 VRF A
    -   Destination Subnet: 10.10.10.0/24
    -   Next Hop
        -   192.168.1.1 (Loopback interface)
        -   Admin Distance: 1
        -   Scope: VRF-B
-   Static Route on Tier-0 VRF B
    -   Destination Subnet: 172.16.10.0/24
    -   Next Hop
        -   IP Address of Tier-1 DR in VRF A (e.g 100.64.80.1)
        -   Admin Distance: 1
        -   Scope: VRF-A

In case of a physical router outage, the next hop for the static route on the Tier-0 VRF A can still be reached using a different healthy BGP peer advertising that host route.


# 4.5 IPv6 Routing Capabilities

NSX-T Data Center also supports dual stack for the interfaces on a Tier-0 or Tier-1 Gateway. Users can leverage distributed services like distributed routing and distributed firewall for East-West traffic in a single tier topology or multi-tiered topology for IPv6 workloads now. Users can also leverage centralized services like Gateway Firewall for North-South traffic.

NSX-T Datacenter supports the following unicast IPv6 addresses:
•	Global Unicast: Globally unique IPv6 address and internet routable
•	Link-Local: Link specific IPv6 address and used as next hop for IPv6 routing protocols
•	Unique local: Site specific unique IPv6 addresses used for inter-site communication but not routable on internet. Based on RFC4193.

The following table shows a summarized view of supported IPv6 unicast and multicast address types on NSX-T Datacenter components.

  |**NSX-T Component**      |    **Unicast Addresses Supported**  |   **Multicast Address Supported**|  
  | -------------| -------------- | -----------------| 
  |**Tier-0 or Tier-1 Gateway Distributed Router (DR)**|   Global, Unique Local, Link Local          |  All Node address (FF02::1) <br> All Routers address (FF02:2) <br> Sollicited-node Multicast Address (FF02::1::FF:0/104)   | 
  |**Tier-0 or Tier-1 Gateway Services Router (SR)**|   Global, Unique Local, Link Local |  All Node address (FF02::1) <br> All Routers address (FF02:2) <br> Sollicited-node Multicast Address (FF02::1::FF:0/104    | 
  |**Inter-Tier Transit Link (Router link)**|   Global, Unique Local, Link Local |  All Node address (FF02::1) <br> All Routers address (FF02:2)| 
  |**Inter-Tier Transit Link (SR-DR link)**|  Link Local          |  All Node address (FF02::1) <br> All Routers address (FF02:2)             | 
<p align="center">
Table 4‑1: Type of IPv6 addresses supported on Tier-0 and Tier-1 Gateway components
</p>

Figure 4-31 shows a single tiered routing topology on the left side with a Tier-0 Gateway supporting dual stack on all interfaces and a multi-tiered routing topology on the right side with a Tier-0 Gateway and Tier-1 Gateway supporting dual stack on all interfaces. A user can either assign static IPv6 addresses to the workloads or use a DHCPv6 relay supported on gateway interfaces to get dynamic IPv6 addresses from an external DHCPv6 server.

For a multi-tier IPv6 routing topology, each Tier-0-to-Tier-1 peer connection is provided a /64 unique local IPv6 address from a pool i.e. fc5f:2b61:bd01::/48. A user has the flexibility to change this subnet range and use another subnet if desired. Similar to IPv4, this IPv6 address is auto plumbed by system in background.


<p align="center">
    <img src="images/Figure4-31.png">
</p>
<p align="center">
Figure 4‑31: Single tier and Multi-tier IPv6 routing topology


Tier-0 Gateway supports following IPv6 routing features:

-   Static routes with IPv6 Next-hop
-   MP-eBGP with IPv4 and IPv6 address families
-   Multi-hop eBGP
-   IBGP
-   ECMP support with static routes, EBGP and IBGP
-   Outbound and Inbound route influencing using Weight, Local Pref, AS
    Path prepend and MED.
-   IPv6 Route Redistribution
-   IPv6 Route Aggregation
-   IPv6 Prefix List and Route map
-   IPv6 Loopback Interfaces
-   
Tier-1 Gateway supports following IPv6 routing features:
-   Static routes with IPv6 Next-hop

IPv6 routing between Tier-0 and Tier-1 Gateway is auto plumbed similar to IPv4 routing. As soon as Tier-1 Gateway is connected to Tier-0 Gateway, the management plane configures a default route (::/0) on Tier-1 Gateway with next hop IPv6 address as Router link IP of Tier-0 Gateway (fc05:2b61:bd01:5000::1/64, as shown in figure 4-32). To provide reachability to subnets connected to the Tier-1 Gateway, the Management Plane (MP) configures routes on the Tier-0 Gateway for all the LIFs connected to Tier-1 Gateway with a next hop IPv6 address as Tier-1 Gateway Router link IP (fc05:2b61:bd01:5000::2/64, as shown in figure 4-32). 2001::/64 & 2002:/64 are seen as “Tier-1 Connected” routes on Tier-0.

Northbound, Tier-0 Gateway redistributes the Tier-0 connected and Tier-1 Connected routes in BGP and advertises these to its eBGP neighbor, the physical router.


<p align="center">
    <img src="images/Figure4-32.png">
</p>
<p align="center">
Figure 4‑32: IPv6 Routing in a Multi-tier topology

# 4.6 Services High Availability

NSX Edge nodes run in an Edge cluster, hosting centralized services, and providing connectivity to the physical infrastructure. Since the services are run on the SR component of a Tier-0 or Tier-1 gateway, the following concept is relevant to SR. This SR service runs on an Edge node and has two modes of operation – active/active or active/standby. When a Tier-1 gateway is configured to be hosted on an Edge cluster, an SR is automatically instantiated even if no services are configured or running on the Tier-1. When a Tier-1 SR is instantiated, the Tier-0 DR is removed from the hypervisors and located on the edge nodes only.

## 4.6.1 Active/Active

**Active/Active** - This is a high availability mode where SRs hosted on Edge nodes act as active forwarders. Stateless services such as layer 3 forwarding are IP based, so it does not matter which Edge node receives and forwards the traffic. All the SRs configured in active/active configuration mode are active forwarders. This high availability mode is only available on Tier-0 gateway.

Stateful services typically require tracking of connection state (e.g., sequence number check, connection state), thus traffic for a given session needs to go through the same Edge node. As of NSX-T 3.0, active/active HA mode does not support stateful services such as Gateway Firewall or stateful NAT. Stateless services, including reflexive NAT and stateless firewall, can leverage the active/active HA model. 

Left side of Figure 4-33 shows a Tier-0 gateway (configured in active/active high availability mode) with two external interfaces leveraging two different Edge nodes, EN1 and EN2. Right side of the diagram shows that the services router component (SR) of this Tier-0 gateway instantiated on both Edge nodes, EN1 and EN2. A Compute host, ESXi is also shown in the diagram that only has distributed component (DR) of Tier-0 gateway. 


<p align="center">
    <img src="images/Figure4-33.png">
</p>
<p align="center">
Figure 4‑33: Tier-0 gateway configured in Active/Active HA mode


Note that Tier-0 SR on Edge nodes, EN1 and EN2 have different IP addresses northbound toward physical routers and different IP addresses southbound towards Tier-0 DR. Management plane configures two default routes on Tier-0 DR with next hop as SR on EN1 (169.254.0.2) and SR on EN2 (169.254.0.3) to provide ECMP for overlay traffic coming from compute hosts.
North-South traffic from overlay workloads hosted on Compute hosts will be load balanced and sent to SR on EN1 or EN2, which will further do a routing lookup to send traffic out to the physical infrastructure.
A user does not have to configure these static default routes on Tier-0 DR. Automatic plumbing of default route happens in background depending upon the HA mode configuration. 

**Inter-SR Routing**

To provide redundancy for physical router failure, Tier-0 SRs on both Edge nodes must establish routing adjacency or exchange routing information with different physical router or TOR. These physical routers may or may not have the same routing information. For instance, a route 192.168.100.0/24 may only be available on physical router 1 and not on physical router 2.

For such asymmetric topologies, users can enable Inter-SR routing. This feature is only available on Tier-0 gateway configured in active/active high availability mode. Figure 4-34 shows an asymmetric routing topology with Tier-0 gateway on Edge node, EN1 and EN2 peering with physical router 1 and physical router 2, both advertising different routes. 

When Inter-SR routing is enabled by the user, an overlay segment is auto plumbed between SRs (similar to the transit segment auto plumbed between DR and SR) and each end gets an IP address assigned in 169.254.0.128/25 subnet by default.  An IBGP session is automatically created between Tier-0 SRs and northbound routes (EBGP and static routes) are exchanged on this IBGP session.


<p align="center">
    <img src="images/Figure4-34.png">
</p>
<p align="center">
Figure 4‑34: Inter-SR Routing

As explained in previous figure, Tier-0 DR has auto plumbed default routes with next hops as Tier-0 SRs and North-South traffic can go to either SR on EN1 or EN2. In case of asymmetric routing topologies, a particular Tier-0 SR may or may not have the route to a destination. In that case, traffic can follow the IBGP route to another SR that has the route to destination. 

Figure 4-34 shows a topology where Tier-0 SR on EN1 is learning a default WAN route 0.0.0.0/0 and a corporate prefix 192.168.100.0/24 from physical router 1 and physical router 2 respectively. If “External 1” interface on Tier-0 fails and the traffic from compute workloads destined to WAN lands on Tier-0 SR on EN1, this traffic can follow the default route (0.0.0.0/0) learned via IBGP from Tier-0 SR on EN2.Traffic is being sent to EN2 through the Geneve overlay. After a route lookup on Tier-0 SR on EN2, this N-S traffic can be sent to physical router 1 using “External interface 3”.

**Graceful Restart and BFD Interaction with Active/Active**

If an Edge node is connected to a TOR switch that does not have the dual supervisor or the ability to retain forwarding traffic when the control plane is restarting, enabling GR in eBGP TOR does not make sense. There is no value in preserving the forwarding table on either end or sending traffic to the failed or restarting device. In case of an active SR failure (i.e., the Edge node goes down), physical router failure, or path failure, forwarding will continue using another active SR or another TOR. BFD should be enabled with the physical routers for faster failure detection.

It is recommended to enable GR If the Edge node is connected to a dual supervisor system that supports forwarding traffic when the control plane is restarting. This will ensure that forwarding table data is preserved and forwarding will continue through the restarting supervisor or control plane. Enabling BFD with such a system would depend on the device-specific BFD implementation. If the BFD session goes down during supervisor failover, then BFD should not be enabled with this system. If the BFD implementation is distributed such that the BFD session would not go down in case of supervisor or control plane failure, then enable BFD as well as GR. 


## 4.6.2 Active/Standby

**Active/Standby** - This is a high availability mode where only one SR act as an active forwarder. This mode is required when stateful services are enabled. Services like NAT are in constant state of sync between active and standby SRs on the Edge nodes. This mode is supported on both Tier-1 and Tier-0 SRs. Preemptive and Non-Preemptive modes are available for both Tier-0 and Tier-1 SRs. Default mode for gateways configured in active/standby high availability configuration is non-preemptive. 

A user can select the preferred member (Edge node) when a gateway is configured in active/standby preemptive mode. When enabled, preemptive behavior allows a SR to resume active role on preferred edge node as soon as it recovers from a failure.

For Tier-1 Gateway, active/standby SRs have the same IP addresses northbound. Only the active SR will reply to ARP requests, while the standby SR interfaces operational state is set as down so that they will automatically drop packets.

For Tier-0 Gateway, active/standby SRs have different IP addresses northbound and both have eBGP sessions established on their uplinks. Both Tier-0 SRs (active and standby) receive routing updates from physical routers and advertise routes to the physical routers; however, the standby Tier-0 SR prepends its local AS three times in the BGP updates so that traffic from the physical routers prefer the active Tier-0 SR. 

Southbound IP addresses on active and standby Tier-0 SRs are the same and the operational state of standby SR southbound interface is down. Since the operational state of southbound Tier-0 SR interface is down, the Tier-0 DR does not send any traffic to the standby SR. Figure 4-35 shows active and standby Tier-0 SRs on Edge nodes “EN1” and “EN2”. 


<p align="center">
    <img src="images/Figure4-35.png" width="75%" height="75%">
</p>
<p align="center">
Figure 4‑35: Active and Standby Routing Control with eBGP

The placement of active and standby SR in terms of connectivity to TOR or northbound infrastructure becomes an important design choice, such that any component failure should not result in a failure of both active and standby service. Diversity of connectivity to TOR for bare metal edge nodes and host-specific availability consideration for hosts where Edge node VMs are hosted, becomes an important design choice. These choices are described in the design chapter.


### 4.6.2.1	Graceful Restart and BFD Interaction with Active

Active/standby services have an active/active control plane with active/standby data forwarding. In this redundancy model, eBGP is established on active and standby Tier-0s SR with their respective TORs. If the Edge node is connected to a system that does not have the dual supervisor or the ability to keep forwarding traffic when the control plane is restarting, enabling GR in eBGP does not make sense. There is no value in preserving the forwarding table on either end as well as no point sending traffic to the failed or restarting device. When the active Tier-0 SR goes down, the route advertised from standby Tier-0 becomes the best route and forwarding continues using the newly active SR. If the TOR switch supports BFD, it is recommended to run BFD on the both eBGP neighbors for faster failure detection.

It is recommended to enable GR If the Edge node is connected to a dual supervisor system that supports forwarding traffic when the control plane is restarting.  This will ensure that the forwarding table is table is preserved and forwarding will continue through the restarting supervisor or control plane. Enabling BFD with such system depends on BFD implementation of hardware vendor. If the BFD session goes down during supervisor failover, then BFD should not be enabled with this system; however, if the BFD implementation is distributed such that that the BFD session would not go down in case of supervisor or control plane failure, then enable BFD as well as GR. 


## 4.6.3 High Availbility Failover triggers

An active SR on an Edge node is declared down when one of the following
conditions is met:

-   Edge nodes in an Edge cluster exchange BFD keep lives on two
    interfaces of the Edge node, management and overlay tunnel
    interfaces. Failover will be triggered when a SR fails to receive
    keep lives on both interfaces.

-   All BGP sessions or northbound routing on the SR is down. This is
    only applicable on Tier-0 SR. When static routes are used on a Bare
    Metal Edge node with NSX-T 3.0, the failover will be triggered when
    the status of all PNIC carrying the uplinks is down.

-   Edge nodes also run BFD with compute hosts. When all the overlay
    tunnels are down to remote Edges and compute hypervisors, an SR
    would be declared down.

## 4.7 Edge Node 

Edge nodes are service appliances with pools of capacity, dedicated to running network and security services that cannot be distributed to the hypervisors. Edge node also provides connectivity to the physical infrastructure. Previous sections mentioned that centralized services will run on the SR component of Tier-0 or Tier-1 gateways. These features include:

-   Connectivity to physical infrastructure (static routing / BGP /
    MP-BGP)
-   VRF-lite
-   NAT
-   DHCP server
-   Metadata proxy
-   Gateway Firewall
-   Load Balancer
-   L2 Bridging
-   Service Interface
-   VPN

As soon as one of these services is configured or an external interface is defined on the Tier-0 gateway, a SR is instantiated on the Edge node. The Edge node is also a transport node just like compute nodes in NSX-T, and like a compute node it can connect to more than one transport zones. A specific Edge node can be connected to only one overlay transport zone and depending upon the topology, is connected to one or more VLAN transport zones for N-S connectivity. 

There are two transport zones on the Edge:

-   **Overlay Transport Zone**: Any traffic that originates from a VM
    participating in NSX-T domain may require reachability to external
    devices or networks. This is typically described as external
    North-South traffic. Traffic from VMs may also require some
    centralized service like NAT, load balancer etc. To provide
    reachability for N-S traffic and to consume centralized services,
    overlay traffic is sent from compute transport nodes to Edge nodes.
    Edge node needs to be configured with a single overlay transport
    zone so that it can decapsulate the overlay traffic received from
    compute nodes as well as encapsulate the traffic sent to compute
    nodes.

-   **VLAN Transport Zone**: Edge nodes connect to physical
    infrastructure using VLANs. Edge node needs to be configured for
    VLAN transport zone to provide external or N-S connectivity to the
    physical infrastructure. Depending upon the N-S topology, an edge
    node can be configured with one or more VLAN transport zones.

Edge node can have one or more N-VDS to provide desired connectivity.
Each N-VDS on the Edge node uses an uplink profile which can be same or
unique per N-VDS. Teaming policy defined in this uplink profile defines
how the N-VDS balances traffic across its uplinks. The uplinks can in
turn be individual pNICs or LAGs.

**Types of Edge Nodes**

Edge nodes are available in two form factors – VM and bare metal. Both
leverage the data plane development kit (DPDK) for faster packet
processing and high performance. There are different VM form factors
available. Each of them has a different resource footprint and can be
used to achieve different guidelines.

These are detailed in below table.

| **Size**            | **Memory** | **vCPU** | **Disk** | **Specific Usage Guidelines**                                                                                                                                                                               |
|---------------------|------------|----------|----------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Small**           | 4GB        | 2        | 200 GB   | PoC only, LB functionality is not available.                                                                                                                                                                |
| **Medium**          | 8GB        | 4        | 200 GB   | Suitable for production with centralized services like NAT, Gateway Firewall. Load balancer functionality can be leveraged for POC.                                                                         |
| **Large**           | 32GB       | 8        | 200 GB   | Suitable for production with centralized services like NAT, Gateway Firewall, load balancer etc.                                                                                                            |
| **Extra Large**     | 64GB       | 16       | 200GB    | Suitable for production with centralized services like NAT, Gateway Firewall, load balancer etc. Typically deployed, where higher performance is desired for services like Layer 7 Load balancer and VPN.   |
| **Bare metal Edge** | 32GB       | 8        | 200 GB   | Suitable for production with centralized services like NAT, Gateway Firewall, load balancer etc. Typically deployed, where higher performance at low packet size and sub-second N-S convergence is desired. |

The Bare Metal Edge resources specified above specify the minimum
resources needed. It is recommended to deploy an edge node on a bare
metal server with the following specifications for maximum performance:

-   Memory: 256GB

-   CPU Cores: 24

-   Disk Space: 200GB

When NSX-T Edge is installed as a VM, vCPUs are allocated to the Linux IP stack and DPDK. The number of vCPU assigned to a Linux IP stack or DPDK depends on the size of the Edge VM. A medium Edge VM has two vCPUs for Linux IP stack and two vCPUs dedicated for DPDK. This changes to four vCPUs for Linux IP stack and four vCPUs for DPDK in a large size Edge VM. Starting with NSX-T 3.0, several AMD CPUs are supported both for the virtualized and Bare Metal Edge node form factor. Specifications can be found here.

## 4.8 Multi-TEP support on Edge node

Staring with NSX-T 2.4 release, Edge nodes support multiple overlay tunnels (multi-TEP) configuration to load balance overlay traffic for overlay segments/logical switches. Multi-TEP is supported in both Edge VM and bare metal.  Figure 4-36 shows two TEPs configured on the bare metal Edge. Each overlay segment/logical switch is pinned to a specific tunnel end point IP, TEP IP1 or TEP IP2. Each TEP uses a different uplink, for instance, TEP IP1 uses Uplink1 that’s mapped to pNIC P1 and TEP IP2 uses Uplink2 that’s mapped to pNIC P2. This feature offers a better design choice by load balancing overlay traffic across both physical pNICs and also simplifies N-VDS design on the Edge.

Notice that a single N-VDS is used in this topology that carries both overlay and external traffic.
In-band management feature is leveraged for management traffic. Overlay traffic gets load balanced by using multi-TEP feature on Edge and external traffic gets load balanced using "Named Teaming policy" as described in section 3.1.3.1. 


<p align="center">
    <img src="images/Figure4-36.png">
</p>
<p align="center">
Figure 4‑36: Bare metal Edge -Same N-VDS for overlay and external traffic with Multi-TEP

-   TEP configuration must be done on one N-VDS only.
-   All TEPs must use same transport VLAN for overlay traffic.
-   All TEP IPs must be in same subnet and use same default gateway.

During a pNIC failure, Edge performs a TEP failover by migrating TEP IP and its MAC address to another uplink. For instance, if pNIC P1 fails, TEP IP1 along with its MAC address will be migrated to use Uplink2 that’s mapped to pNIC P2. In case of pNIC P1 failure, pNIC P2 will carry the traffic for both TEP IP1 and TEP IP2.

**A Case for a Better Design:** 

This version of the design guide introduced a simpler way to configure Edge connectivity, referred as “Single N-VDS Design”. The key reasons for adopting “Single N-VDS Design”:

This version of the design guide introduced a simpler way to configure
Edge connectivity, referred as “Single N-VDS Design”. The key reasons
for adopting “Single N-VDS Design”:

-   **Multi-TEP support for Edge** – Details of multi-TEP is described
    as above. Just like an ESXi transport node supporting multiple TEP,
    Edge node has a capability to support multiple TEP per uplink with
    following advantages:

    -   Removes critical topology restriction with bare metal – straight
        through LAG

    -   Allowing the use of multiple pNICs for the overlay traffic in
        both bare metal and VM form factor.

    -   An Edge VM supporting multiple TEP can have two uplinks from the
        same N-VDS, allowing utilization of both pNICs

-   **Multiple teaming policy per N-VDS** – [Default and Named Teaming
    Policy](#_Teaming_Policy)

    -   Allows specific uplink to be designated or pinned for a given
        VLAN

    -   Allowing uplinks to be active/standby or active-only to drive
        specific behavior of a given traffic types while co-existing
        other traffic type following entirely different paths

-   **Normalization of N-VDS configuration** – All form factors or Edge
    and deployments uses single N-VDS along with host. Single teaming
    policy for overlay – Load Balanced Source. Single policy for N-S
    peering – Named teaming Policy

### 4.8.1 Bare Metal Edge Node

NSX-T bare metal Edge runs on a physical server and is installed using an ISO file or PXE boot. 
Legacy BIOS mode is the only supported booting mode on NSXT-T 3.0. A bare metal Edge differs from the VM form factor Edge in terms of performance. It provides sub-second convergence, faster failover, and higher throughput at low packet size (discussed in performance Chapter 8). There are certain hardware requirements including CPU specifics and supported NICs can be found in the NSX Edge Bare Metal Requirements section of the NSX-T installation guide.

When a bare metal Edge node is installed, a dedicated interface is retained for management. If redundancy is desired, two NICs can be used for management plane high availability. These management interfaces can also be 1G. Bare metal Edge also supports in-band management where management traffic can leverage an interface being used for overlay or external (N-S) traffic.

Bare metal Edge node supports a maximum of 16 physical NICs for overlay traffic and external traffic to top of rack (TOR) switches. For each of these 16 physical NICs on the server, an internal interface is created following the naming scheme “fp-ethX”. These internal interfaces are assigned to the DPDK Fast Path. There is a flexibility in assigning these Fast Path interfaces (fp-eth) for overlay or external connectivity.

#### 4.8.1.1 Management Plane Configuration Choices with Bare Metal Node

This section covers all the available options in managing the bare metal node. There are four options as describe in below diagram:

**Out of Band Management with Single pNIC**
Left side of Figure 4-37 shows a bare metal edge node with 3 physical NICs. The dedicated pNIC for management and is used to send/receive management traffic. The management pNIC can be 1Gbps. There is not redundancy for management traffic in this topology. If P1 goes down, the management traffic will fail. However, Edge node will continue to function as this doesn’t affect data plane traffic.

**In Band Management – Data Plane (fast-path) NIC carrying Management Traffic**
This capability was added in NSX-T 2.4 release. It is not mandatory to have a dedicated physical interface to carry management traffic. This traffic can leverage one of the DPDK fast-path interfaces. On the right side of the Figure 4-37, P2 is selected to send management traffic. In-band management configuration is available via CLI on the Edge node. A user needs to provide following two parameters to configure in-band management.

-   VLAN for management traffic
-   MAC address of the DPDK Fast Path interface chosen for this
    management traffic.


<p align="center">
    <img src="images/Figure4-37.png">
</p>
<p align="center">
Figure 4‑37: Bare metal Edge Management Configuration Choices

Additionally, one can configure the management redundancy via LAG, however only one of the LAG members can be active at a time.

#### 4.8.1.2 Single N-VDS Bare Metal Configuration with 2 pNICs

Figure 4-38 shows 2 pNIC bare metal Edge using a single N-VDS design for data plane.
The left side of the diagram shows the bare metal Edge with four physical NICs where management traffic has dedicated two physical NICs (P1 & P2) configured in active/standby mode. 

A single N-VDS “Overlay and External N-VDS" is used in this topology that carries both overlay and External traffic. Overlay traffic from different overlay segments/logical switches gets pinned to TEP IP1 or TEP IP2 and gets load balanced across both uplinks, Uplink1 and Uplink2. Notice that, both TEP IPs use same transport VLAN i.e. VLAN 200 which is configured on both top of rack switches. 

Two VLANs segments, i.e. "External VLAN Segment 300" and "External VLAN Segment 400" are used to provide northbound connectivity to Tier-0 gateway. Same VLAN segment can also be used to connect Tier-0 Gateway to TOR-Left and TOR-Right, however it is not recommended because of inter-rack VLAN dependencies leading to spanning tree related convergence. External traffic from these VLAN segments is load balanced across uplinks using named teaming policy which pins a VLAN segment to a specific uplink. 

This topology provides redundancy for management, overlay and external traffic, in event of a pNIC failure on Edge node/TOR and TOR Failure.
 
The right side of the diagram shows two pNICs bare metal edge configured with the same N-VDS “Overlay and External N-VDS" for carrying overlay and external traffic as above that is also leveraging in-band management. 
4

<p align="center">
    <img src="images/Figure4-38.png">
</p>
<p align="center">
Figure 4‑38: Bare metal Edge configured for Multi-TEP - Single N-VDS for overlay and external traffic 
(With dedicated pNICs for Management and In-Band Management)

Both above topologies use the same transport node profile as shown in Figure 4-39.
This configuration shows a default teaming policy that uses both Uplink1 and Uplink2. This default policy is used for all the segments/logical switches created on this N-VDS.
Two additional teaming policies, “Vlan300-Policy” and “Vlan400-Policy” have been defined to override the default teaming policy and send traffic to “Uplink1” and “Uplink2” respectively. 

"External VLAN segment 300" is configured to use the named teaming policy “Vlan300-Policy” that sends traffic from this VLAN only on “Uplink1”. "External VLAN segment 400" is configured to use a named teaming policy “Vlan400-Policy” that sends traffic from this VLAN only on “Uplink2”. 

Based on these teaming policies, TOR-Left will receive traffic for VLAN 100 (Mgmt.), VLAN 200 (overlay) and VLAN 300 (Traffic from VLAN segment 300) and hence, should be configured for these VLANs. Similarly, TOR-Right will receive traffic for VLAN 100 (Mgmt.), VLAN 200 (overlay) and VLAN 400 (Traffic from VLAN segment 400). A sample configuration screenshot is shown below.


<p align="center">
    <img src="images/Figure4-39.png">
</p>
<p align="center">
Figure 4‑39: Bare metal Edge Transport Node Profile

Figure 4-40 shows a logical and physical topology where a Tier-0 gateway has four external interfaces. External interfaces 1 and 2 are provided by bare metal Edge node “EN1”, whereas External interfaces 3 and 4 are provided by bare metal Edge node “EN2”. Both the Edge nodes are in the same rack and connect to TOR switches in that rack. Both the Edge nodes are configured for Multi-TEP and use named teaming policy to send traffic from VLAN 300 to TOR-Left and traffic from VLAN 400 to TOR-Right. Tier-0 Gateway establishes BGP peering on all four external interfaces and provides 4-way ECMP.


<p align="center">
    <img src="images/Figure4-40.png">
</p>
<p align="center">
Figure 4‑40: 4-way ECMP using bare metal edges 

#### 4.8.1.3 Single N-VDS Bare Metal Configuration with Six pNICs

Figure 4-41 shows NSX-T bare metal Edge with six physical NICs. Management traffic has two dedicated pNICs configured in Active/Standby.  Two pNICs, P3 and P4 are dedicated for overlay traffic and two pNICs (P5 and P6) are dedicated for external traffic. 

A single N-VDS “Overlay and External N-VDS" is used in this topology that carries both overlay and External traffic. However, different uplinks are used to carry overlay and external traffic. Multi-TEP is configured to provide load balancing for the overlay traffic on Uplink1 (mapped to pNIC P3) and Uplink2 (mapped to pNIC P4). Notice that, both TEP IPs use same transport VLAN i.e. VLAN 200 which is configured on both top of rack switches.
 
Figure 4-41 also shows a configuration screenshot of named teaming policy defining two additional teaming policies, “Vlan300-Policy” and “Vlan400-Policy”. 
"External VLAN segment 300" is configured to use a named teaming policy “Vlan300-Policy” that sends traffic from this VLAN only on Uplink3 (mapped to pNIC P5). "External VLAN segment 400" is configured to use a named teaming policy “Vlan400-Policy” that sends traffic from this VLAN only on Uplink4 (mapped to pNIC P6). Hence, BGP traffic from Tier-0 on VLAN 300 always goes to TOR-Left and BGP traffic from Tier-0 on VLAN 400 always goes to TOR-Right.
 
This topology provides redundancy for management, overlay and external traffic. This topology also provides a simple, high bandwidth and deterministic design as there are dedicated physical NICs for different traffic types (overlay and External traffic).


<p align="center">
    <img src="images/Figure4-41.png">
</p>
<p align="center">
Figure 4‑41: Bare metal Edge with six pNICs - Same N-VDS for Overlay and External traffic


### 4.8.2 VM Edge Node

NSX-T VM Edge in VM form factor can be installed using an OVA, OVF, or ISO file. NSX-T Edge VM is only supported on ESXi host. 

An NSX-T Edge VM has four internal interfaces: eth0, fp-eth0, fp-eth1, and fp-eth2. Eth0 is reserved for management, while the rest of the interfaces are assigned to DPDK Fast Path. These interfaces are allocated for external connectivity to TOR switches and for NSX-T overlay tunneling. There is complete flexibility in assigning Fast Path interfaces (fp-eth) for overlay or external connectivity. As an example, fp-eth0 could be assigned for overlay traffic with fp-eth1, fp-eth2, or both for external traffic.

There is a default teaming policy per N-VDS that defines how the N-VDS balances traffic across its uplinks. This default teaming policy can be overridden for VLAN segments only using “named teaming policy”. To develop desired connectivity (e.g., explicit availability and traffic engineering), more than one N-VDS per Edge node may be required. Each N-VDS instance can have a unique teaming policy, allowing for flexible design choices. 



#### 4.8.2.1 Multiple N-VDS per Edge VM Configuration – NSX-T 2.4 or Older 

The “three N-VDS per Edge VM design” as commonly called has been deployed in production. This section briefly covers the design, so the reader does not miss the important decision which design to adopt based on NSX-T release target. 

The multiple N-VDS per Edge VM design recommendation is valid regardless of the NSX-T release. This design must be followed if the deployment target is NSX-T release 2.4 or older. The design recommendation is still completely applicable and viable to Edge VM deployment running NSX-T 2.5 release. In order to simplify consumption for the new design recommendation, the pre-2.5 release design has been moved to Appendix 5. The design choices that moved to appendix covers

-   2 pNICs bare metal design necessitating straight through LAG
    topology
-   Edge clustering design consideration for bare metal
-   4 pNICs bare metal design added to support existing deployment
-   Edge node design with 2 and 4 pNICs

It’s a mandatory to adopt this recommendation for NSX-T release up to 2.5.  The newer design as described in section 7.4.2.3 will not operate properly if adopted in release before NSX-T 2.5.   In addition, readers are highly encouraging to read the appendix section 5 to appreciate the new design recommendation.


<p align="center">
    <img src="images/Figure4-42.png">
</p>
<p align="center">
Figure 4‑42: Edge Node VM installed leveraging VDS port groups on a 2 pNIC host

Figure 4-42 shows an ESXi host with two physical NICs. Edges “VM1” is hosted on ESXi host leveraging the VDS port groups, each connected to both TOR switches. This figure also shows three N-VDS, named as “Overlay N-VDS”, “Ext 1 N-VDS”, and “Ext 2 N-VDS”. Three N-VDS are used in this design to ensure that overlay and external traffic use different vNIC of Edge VM. All three N-VDS use the same teaming policy i.e. Failover order with one active uplink.

**VLAN TAG Requirements**

Edge VM deployment shown in figure 4-42 remains valid and is ideal for deployments where only one VLAN is necessary on each vNIC of the Edge VM. However, it doesn’t cover all the deployment use cases. For instance, if a user cannot add service interfaces to connect VLAN backed workloads in above topology as that requires to allow one or more VLANs on the VDS DVPG (distributed virtual port group). If these DVPGs are configured to allow multiple VLANs, no change in DVPG configuration is needed when new service interfaces (workload VLAN segments) are added. 

When these DVPGs are configured to carry multiple VLANs, a VLAN tag is expected from Edge VM for traffic belonging to different VLANs.
 
VLAN tags can be applied to both overlay and external traffic at either N-VDS level or VSS/VDS level. On N-VDS, overlay and external traffic can be tagged using the following configuration:
-   Uplink Profile where the transport VLAN can be set which will tag
    overlay traffic only
-   VLAN segment connecting Tier-0 gateway to external devices- This
    configuration will apply a VLAN tag to the external traffic only.

Following are the three ways to configure VLAN tagging on VSS or VDS:
-   EST (External Switch Tagging)
-   VST (Virtual Switch Tagging)
-   VGT (virtual guest tagging)

For the details where each tagging can be applicable refer to following resources:

https://kb.vmware.com/s/article/1003806#vgtPoints

https://www.vmware.com/pdf/esx3_VLAN_wp.pdf 


Figure 4-43 shows an Edge node hosted on an ESXi host. In this example, VLAN tags are applied to both overlay and external traffic using uplink profile and VLAN segments connecting Tier-0 Gateway to physical infrastructure respectively. As a result, VDS port groups that provides connectivity to Edge VM receive VLAN tagged traffic. Hence, they should be configured to allow these VLANs in VGT (Virtual guest tagging) mode.
Uplink profile used for “Overlay N-VDS” has a transport VLAN defined as VLAN 200. This will ensure that the overlay traffic exiting vNIC2 has an 802.1Q VLAN 200 tag. Overlay traffic received on VDS port group “Transport PG” is VLAN tagged. That means that this Edge VM vNIC2 will have to be attached to a port group configured for Virtual Guest Tagging (VGT).
Tier-0 Gateway connects to the physical infrastructure using “External-1” and “External-2” interface leveraging VLAN segments “External VLAN Segment 300” and “External VLAN Segment 400” respectively. In this example, “External Segment 300” and “External Segment 400” are configured with a VLAN tag, 300 and 400 respectively. External traffic received on VDS port groups “Ext1 PG” and Ext2 PG” is VLAN tagged and hence, these port groups should be configured in VGT (Virtual guest tagging) mode and allow those specific VLANs.



<p align="center">
    <img src="images/Figure4-43.png">
</p>
<p align="center">
Figure 4‑43: VLAN tagging on Edge node

#### 4.8.2.2 Single N-VDS Based Configuration - Starting with NSX-T 2.5 release

Starting NSX-T 2.4 release, Edge nodes support Multi-TEP configuration to load balance overlay traffic for segments/logical switches. Similar to the bare metal Edge one N-VDS design, Edge VM also supports same N-VDS for overlay and external traffic. 

Even though this multi-TEP feature was available in NSX-T 2.4 release, the release that supports this design is NSX-T 2.5 release onward. It is mandatory to use the multiple N-VDS design for release NSX-T 2.4 or older. 

Figure 4-44 shows an Edge VM with one N-VDS i.e. “Overlay and External N-VDS”, to carry both overlay and external traffic. Multi-TEP is configured to provide load balancing for overlay traffic on “Uplink1” and “Uplink2”. “Uplink1” and “Uplink2” are mapped to use vNIC2 and vNIC3 respectively.  Based on this teaming policy, overlay traffic will be sent and received on both vNIC2 and vNIC3 of the Edge VM. Notice that, both TEP IPs use same transport VLAN i.e. VLAN 200 which is configured on both top of rack switches.

Similar to Figure 4-43, Tier-0 Gateway for BGP peering connects to the physical infrastructure leveraging VLAN segments “External VLAN Segment 300” and “External VLAN Segment 400” respectively. In this example, “External VLAN Segment 300” and “External VLAN Segment 400” are configured with a VLAN tag, 300 and 400 respectively. External traffic received on VDS port groups “Trunk1 PG” and Trunk2 PG” is VLAN tagged and hence, these port groups should be configured in VGT (Virtual guest tagging) mode and allow those specific VLANs.

Named teaming policy is also configured to load balance external traffic. Figure 4-28 also shows named teaming policy configuration used for this topology.  "External VLAN segment 300" is configured to use a named teaming policy “Vlan300-Policy” that sends traffic from this VLAN on “Uplink1” (vNIC2 of Edge VM). "External VLAN segment 400" is configured to use a named teaming policy “Vlan400-Policy” that sends traffic from this VLAN on “Uplink2” (vNIC3 of Edge VM). Based on this named teaming policy, North-South or external traffic from “External VLAN Segment 300” will always be sent and received on vNIC2 of the Edge VM. North-South or external traffic from “External VLAN Segment 400” will always be sent and received on vNIC3 of the Edge VM. 

Overlay or external traffic from Edge VM is received by the VDS DVPGs “Trunk1 PG” and “Trunk2 PG”. Teaming policy used on the VDS port group level defines how this overlay and external traffic coming from Edge node VM exits the hypervisor. For instance, “Trunk1 PG” is configured to use active uplink as “VDS-Uplink1” and standby uplink as “VDS-Uplink2”. “Trunk2 PG” is configured to use active uplink as “VDS-Uplink2” and standby uplink as “VDS-Uplink1”.

This configuration ensures that the traffic sent on “External VLAN Segment 300” i.e. VLAN 300 always uses vNIC2 of Edge VM to exit Edge VM. This traffic then uses “VDS-Uplink1” (based on “Trunk1 PG” configuration) and is sent to the left TOR switch. Similarly, traffic sent on VLAN 400 uses “VDS-Uplink2” and is sent to the TOR switch on the right.


<p align="center">
    <img src="images/Figure4-44.png">
</p>
<p align="center">
Figure 4‑44: VLAN tagging on Edge node

Starting with NSX-T release 2.5, single N-VDS deployment mode is recommended for both bare metal and Edge VM. Key benefits of single N-VDS deployment are:

-   Consistent deployment model for both Edge VM and bare metal Edge
    with one N-VDS carrying both overlay and external traffic.
-   Load balancing of overlay traffic with Multi-TEP configuration.
-   Ability to distribute external traffic to specific TORs for distinct
    point to point routing adjacencies.
-   No change in DVPG configuration when new service interfaces
    (workload VLAN segments) are added.
-   Deterministic North South traffic pattern.

Service interface were introduced earlier, following section focusses on how service interfaces work in the topology shown in Figure 4-45.

#### 4.8.2.3 VLAN Backed Service Interface on Tier-0 or Tier-1 Gateway

Service interface is an interface connecting VLAN backed segments/logical switch to provide connectivity to VLAN backed physical or virtual workloads. This interface acts as a gateway for these VLAN backed workloads and is supported both on Tier-0 and Tier-1 Gateways configured in active/standby HA configuration mode. 

Service interface is realized on Tier-0 SR or Tier-1 SR. This implies that traffic from a VLAN workload needs to go to Tier-0 SR or Tier-1 SR to consume any centralized service or to communicate with any other VLAN or overlay segments. Tier-0 SR or Tier-1 SR is always hosted on Edge node (bare metal or Edge VM).  

Figure 4-45 shows a VLAN segment “VLAN Seg-500” that is defined to provide connectivity to the VLAN workloads. “VLAN Seg-500” is configured with a VLAN tag of 500. Tier-0 gateway has a service interface “Service Interface-1” configured leveraging this VLAN segment and acts as a gateway for VLAN workloads connected to this VLAN segment. In this example, if the workload VM, VM1 needs to communicate with any other workload VM on overlay or VLAN segment, the traffic will be sent from the compute hypervisor (ESXi-2) to the Edge node (hosted on ESXi-1). This traffic is tagged with VLAN 500 and hence the DVPG receiving this traffic (“Trunk-1 PG” or “Trunk-2 PG”) must be configured in VST (Virtual Switch Tagging) mode. Adding more service interfaces on Tier-0 or Tier-1 is just a matter of making sure that the specific VLAN is allowed on DVPG (“Trunk-1 PG” or “Trunk-2 PG”).


<p align="center">
    <img src="images/Figure4-45.png">
</p>
<p align="center">
Figure 4‑45: VLAN tagging on Edge node with Service Interface

Note: Service interface can also be connected to overlay segments for standalone load balancer use cases. This is explained in Load balancer Chapter 6

### 4.8.3 Edge Cluster

An Edge cluster is a group of Edge transport nodes. It provides scale out, redundant, and high-throughput gateway functionality for logical networks. Scale out from the logical networks to the Edge nodes is achieved using ECMP. NSX-T 2.3 introduced the support for heterogeneous Edge nodes which facilitates easy migration from Edge node VM to bare metal Edge node without reconfiguring the logical routers on bare metal Edge nodes. There is a flexibility in assigning Tier-0 or Tier-1 gateways to Edge nodes and clusters. Tier-0 and Tier-1 gateways can be hosted on either same or different Edge clusters. 


<p align="center">
    <img src="images/Figure4-46.png">
</p>
<p align="center">
Figure 4‑46: Edge Cluster with Tier-0 and Tier-1 Services

Depending upon the services hosted on the Edge node and their usage, an Edge cluster could be dedicated simply for running centralized services (e.g., NAT). Figure 4-47 shows two clusters of Edge nodes. Edge Cluster 1 is dedicated for Tier-0 gateways only and provides external connectivity to the physical infrastructure. Edge Cluster 2 is responsible for NAT functionality on Tier-1 gateways. 


<p align="center">
    <img src="images/Figure4-47.png">
</p>
<p align="center">
Figure 4‑47: Multiple Edge Clusters with Dedicated Tier-0 and Tier-1 Services

There can be only one Tier-0 gateway per Edge node; however, multiple Tier-1 gateways can be hosted on one Edge node. 

A Tier-0 gateway supports a maximum of eight equal cost paths, thus a maximum of eight Edge nodes are supported for ECMP. Edge nodes in an Edge cluster run Bidirectional Forwarding Detection (BFD) on both tunnel and management networks to detect Edge node failure. The BFD protocol provides fast detection of failure for forwarding paths or forwarding engines, improving convergence. Edge VMs support BFD with minimum BFD timer of 500ms with three retries, providing a 1.5 second failure detection time. Bare metal Edges support BFD with minimum BFD TX/RX timer of 50ms with three retries which implies 150ms failure detection time. 


### 4.8.4 Failure Domain

Failure domain is a logical grouping of Edge nodes within an Edge Cluster. This feature is introduced in NSX-T 2.5 release and can be enabled on Edge cluster level via API configuration. Please refer to this API configuration available in Appendix 3.

As discussed in high availability section, a Tier-1 gateway with centralized services runs on Edge nodes in active/standby HA configuration mode. When a user assigns a Tier-1 gateway to an Edge cluster, NSX manager automatically chooses the Edge nodes in the cluster to run the active and standby Tier-1 SR. The auto placement of Tier-1 SRs on different Edge nodes considers several parameters like Edge capacity, active/standby HA state etc. 
Failure domains compliment auto placement algorithm and guarantee service availability in case of a rack failure. Active and standby instance of a Tier-1 SR always run in different failure domains.

Figure 4-48 shows an edge cluster comprised of four Edge nodes, EN1, EN2, EN3 and EN4. EN1 and EN2 connected to two TOR switches in rack 1 and EN3 and EN4 connected to two TOR switches in rack 2. Without failure domain, a Tier-1 SR could be auto placed in EN1 and EN2. If rack1 fails, both active and standby instance of this Tier-1 SR fail as well. 
EN1 and EN2 are configured to be a part of failure domain 1, while EN3 and EN4 are in failure domain 2. When a new Tier-1 SR is created and if active instance of that Tier-1 is hosted on EN1, then the standby Tier-1 SR will be instantiated in failure domain 2 (EN3 or EN4).


<p align="center">
    <img src="images/Figure4-48.png">
</p>
<p align="center">
Figure 4‑48: Failure Domains

To ensure that all Tier-1 services are active on a set of edge nodes, a user can also enforce that all active Tier-1 SRs are placed in one failure domain. This configuration is supported for Tier-1 gateway in preemptive mode only.



## 4.9 Other Network Services

### 4.9.1 Unicast Reverse Path Forwarding (uRPF)

A router forwards packets based on the value of the destination IP address field that is present in the IP header. The source IP address field is generally not used when forwarding a packet on a network (except when networks implement source-based routing). 

Unicast Reverse Path Forwarding is defined in RFC 2827 and 3704. Prevent packets with spoofed source IP address to be forwarded in the network. uURPF is generally enabled on a router per interface and not globally. A uRPF enabled router  A router will inspect the source IP address of theevery packet  of each packet received on an interface. It will validates that packets are coming from a legitimate source by inspecting the routing table. The This protection prevents spoofed source IP address attacks that are commonly used by sending packets with random source IP addresses. When a packet arrivearrives on an interface, aA The router will verify if the receiving that specific interface would be used to reach the source of the packet.t  It will discard the packets if the interfaces are different. 

This protection prevents spoofed source IP address attacks that are commonly used by sending packets with random source IP addresses.

Figure 4-49 diagrams a physical network with URPF enabled on the core router. 

1.  The core router receives a packet with a source IP address of
    10.1.1.1 on interface ethernet0/2.
2.  The core router has the URPF feature enabled on all its interfaces
    and will check in its routing table if the source IP address of the
    packet would be routed through interface ethernet 0/2. In this case,
    10.1.1.1 is the source IP address present in the IP header. The core
    router has a longest prefix match for 10.1.1.0/24 via interface
    ethernet 0/0.
3.  Since the packet does not come from the interface ethernet 0/0, the
    packet will be discarded.



<p align="center">
    <img src="images/Figure4-49.png">
</p>
<p align="center">
Figure 4‑49: uRPF

In NSX-T, URPF is enabled by default on external, internal and service interfaces. From a security standpoint, it is a best practice to keep uRPF enabled on these interfaces. uRPF is also recommended in architectures that leverage ECMP. On intra-tier and router link interfaces, a simplified anti-spoofing mechanism is implemented. It is checking that a packet is never sent back to the interface the packet was received on.

It is possible to disable uRPF in complex routing architecture where asymmetric routing exists.
As of NSX-T 3.0, it is possible to disable or enable URPF on the Policy UI except for downlink interfaces where the administrator would need to use the Manager UI or Policy API.


## 4.9.2 Network Address Translation

Users can enable NAT as a network service on NSX-T. This is a centralized service which can be enabled on both Tier-0 and Tier-1 gateways. 

Supported NAT rule types include:

-   **Source NAT (SNAT)**: Source NAT translates the source IP of the
    outbound packets to a known public IP address so that the
    application can communicate with the outside world without using its
    private IP address. It also keeps track of the reply.

-   **Destination NAT (DNAT)**: DNAT allows for access to internal
    private IP addresses from the outside world by translating the
    destination IP address when inbound communication is initiated. It
    also takes care of the reply. For both SNAT and DNAT, users can
    apply NAT rules based on 5 tuple match criteria.

-   **Reflexive NAT:** Reflexive NAT rules are stateless ACLs which must
    be defined in both directions. These do not keep track of the
    connection. Reflexive NAT rules can be used in cases where stateful
    NAT cannot be used due to asymmetric paths (e.g., user needs to
    enable NAT on active/active ECMP routers).

Table 4‑3 summarizes NAT rules and usage restrictions.

<table>
<tbody>
<tr class="odd">
<td><strong>NAT Rules Type</strong></td>
<td><strong>Type</strong></td>
<td><strong>Specific Usage Guidelines</strong></td>
</tr>
<tr class="even">
<td>Stateful</td>
<td><p>Source NAT (SNAT)</p>
<p>Destination NAT (DNAT)</p></td>
<td>Can be enabled on both Tier-0 and Tier-1 gateways</td>
</tr>
<tr class="odd">
<td>Stateless</td>
<td>Reflexive NAT</td>
<td>Can be enabled on Tier-0 gateway; generally used when the Tier-0 is in active/active mode.</td>
</tr>
</tbody>
</table>

Table 4‑3: NAT Usage Guideline

Table 4-4 summarizes the use cases and advantages of running NAT on
Tier-0 and Tier-1 gateways.

<table>
<tbody>
<tr class="odd">
<td><strong>Gateway Type</strong></td>
<td><strong>NAT Rule Type</strong></td>
<td><strong>Specific Usage Guidelines</strong></td>
</tr>
<tr class="even">
<td>Tier-0</td>
<td>Stateful</td>
<td><p>Recommended for PAS/PKS deployments.</p>
<p>E-W routing between different tenants remains completely distributed.</p></td>
</tr>
<tr class="odd">
<td>Tier-1</td>
<td>Stateful</td>
<td><p>Recommended for high throughput ECMP topologies.</p>
<p>Recommended for topologies with overlapping IP address space.</p></td>
</tr>
</tbody>
</table>

Table 4‑4: Tier-0 and Tier-1 NAT use cases

**NAT Service Router Placement**

As a centralized service, whenever NAT is enabled, a service component or SR must be instantiated on an Edge cluster. In order to configure NAT, specify the Edge cluster where the service should run; it is also possible the NAT service on a specific Edge node pair. If no specific Edge node is identified, the platform will perform auto placement of the services component on an Edge node in the cluster using a weighted round robin algorithm.


## 4.9.3 DHCP Services

NSX-T provides both DHCP relay and DHCP server functionality. DHCP relay can be enabled at the gateway level and can act as relay between non-NSX managed environment and DHCP servers. DHCP server functionality can be enabled to service DHCP requests from VMs connected to NSX-managed segments. DHCP server functionality is a stateful service and must be bound to an Edge cluster or a specific pair of Edge nodes as with NAT functionality.

## 4.9.4 Metadata Proxy Service

With a metadata proxy server, VM instances can retrieve instance-specific metadata from an OpenStack Nova API server. This functionality is specific to OpenStack use-cases only. Metadata proxy service runs as a service on an NSX Edge node. For high availability, configure metadata proxy to run on two or more NSX Edge nodes in an NSX Edge cluster.

## 4.9.5 Gateway Firewall Services

Gateway Firewall service can be enabled on the Tier-0 and Tier-1 gateway
for North-South firewalling. Table 4-5 summarizes Gateway Firewalling
usage criteria.


| **Gateway Firewall**                     |          **Specific Usage Guidelines**                                                                  |
|----------------------|--------------------------------------------------------------------------------------------|
| Stateful             | Can be enabled on both Tier-0 and Tier-1 gateways.                                         |
| Stateless            | Can be enabled on Tier-0 gateway; generally used when the Tier-0 is in active/active mode. |


Table 4‑5: Gateway Firewall Usage Guideline

Since Gateway Firewalling is a centralized service, it needs to run on
an Edge cluster or a set of Edge nodes. This service is described in
more detail in the [NSX-T Security Chapter](#_NSX-T_Security).




## 4.9.6 Proxy ARP

Proxy ARP is a method that consist of answering an ARP request on behalf of another host. This method is performed by a layer 3 networking device (usually a router). The purpose is to provide connectivity between 2 hosts when routing wouldn’t be possible for various reasons. 

Proxy ARP in an NSX-T infrastructure can be considered in environments where IP subnets are limited. Proof of concepts and VMware Enterprise PKS environments are usually using Proxy-ARP to simplify the network topology.

For production environment, it is recommended to implement proper routing between a physical fabric and the NSX-T Tier-0 by using either static routes or Border Gateway Protocol with BFD. If proper routing is used between the Tier-0 gateway and the physical fabric, BFD with its sub-second timers will converge faster. In case of failover with proxy ARP, the convergence relies on gratuitous ARP (broadcast) to update all hosts on the VLAN segment with the new MAC Address to use. If the Tier-0 gateway has proxy ARP enabled for 100 IP addresses, the newly active Tier-0 SR needs to send 100 Gratuitous ARP packets.

.

| Edge Node HA     |  **Specific Usage Guidelines**   |
|------------------|----------------------------------|
| Active / Standby | Supported                        |
| Active / Active  | Not supported                    |
 
Table 4‑6: Proxy ARP Support

By enabling proxy-ARP, hosts on the overlay segments and hosts on a VLAN segment can exchange network traffic together without implementing any change in the physical networking fabric. Proxy ARP is automatically enabled when a NAT rule or a load balancer VIP uses an IP address from the subnet of the Tier-0 gateway uplink.

Figure 4-50 presents the logical packet flow between a virtual machine connected to an NSX-T overlay segment and a virtual machine or physical appliance connected to a VLAN segment shared with the NSX-T Tier-0 uplinks.

In this example, the virtual machine connected to the overlay segment initiates networking traffic toward 20.20.20.100.



<p align="center">
    <img src="images/Figure4-50.png">
</p>
<p align="center">
Figure 4‑50: Proxy ARP  topology

1.  The virtual machine connected to the overlay segment with an IP
    address of 172.16.10.10 sends a packet to the physical appliance
    “SRV01” with an IP address of 20.20.20.100. the local DR hosted on
    the local hypervisor performs a routing lookup and sends the traffic
    to the SR.

2.  The SR hosted on an edge node translates the source IP of
    172.16.10.10 with a value of 20.20.20.10 and sends the traffic to
    the Tier-0.

3.  Tier-0 SR has proxy ARP enabled on its uplink interface and will
    send an ARP request (broadcast) on the vlan segment to map the IP
    address of 20.20.20.100 with the correct MAC address.

4.  The physical appliance “SRV01” answers to that ARP request with an
    ARP reply.

5.  Tier-0 SR sends the packet to the physical appliance with a source
    IP of 20.20.20.10 and a destination IP of 20.20.20.100.

6.  The physical appliance “SRV01” receives the packet and sends an ARP
    broadcast on the VLAN segment to map the IP address of the virtual
    machine (20.20.20.10) to the corresponding MAC address.

7.  Tier-0 receives the ARP request for 20.20.20.10 (broadcast) and has
    the proxy ARP feature enabled on its uplink interfaces. It replies
    to the ARP request with an ARP reply that contains the Tier-0 SR MAC
    address for the interface uplink.

8.  The physical appliance “SRV01” receives the ARP request and sends a
    packet on the vlan segment with a source IP of 20.20.20.100 and a
    destination IP of 20.20.20.10.

9.  The packet is being received by the Tier-0 SR and is being routed to
    the Tier-1 who does translate the Destination IP of 20.20.20.10 with
    a value of 172.16.10.10. Packet is sent to the overlay segment and
    the virtual machine receives it.

It is crucial to note that in this case, the traffic is initiated by the virtual machine which is connected to the overlay segment on the Tier-1. If the initial traffic was initiated by a server on the VLAN segment, a Destination NAT rule would have been required on the Tier-1/Tier-0 since the initial traffic would not match the SNAT rule that has been configured previously.

Figure 4-51 represents an outage on an active Tier-0 gateway with Proxy ARP enabled. The newly active Tier-0 gateway will send a gratuitous ARP to announce the new MAC address to be used by the hosts on the VLAN segment in order to reach the virtual machine connected to the overlay. It is critical to fathom that the newly active Tier-0 will send a Gratuitous ARP for each IP address that are configured for Proxy ARP.

<p align="center">
    <img src="images/Figure4-51.png">
</p>
<p align="center">
Figure 4‑51: Edge node failover and Proxy ARP.


## 4.10 Topology Consideration

This section covers a few of the many topologies that customers can build with NSX-T. NSX-T routing components - Tier-1 and Tier-0 gateways - enable flexible deployment of multi-tiered routing topologies. Topology design also depends on what services are enabled and where those services are provided at the provider or tenant level.

### 4.10.1 Supported Topologies

Figure 4-52 shows three topologies with Tier-0 gateway providing N-S traffic connectivity via multiple Edge nodes. The first topology is single-tiered where Tier-0 gateway connects directly to the segments and provides E-W routing between subnets. Tier-0 gateway provides multiple active paths for N-S L3 forwarding using ECMP. The second topology shows the multi-tiered approach where Tier-0 gateway provides multiple active paths for L3 forwarding using ECMP and Tier-1 gateways as first hops for the segments connected to them. Routing is fully distributed in this multi-tier topology. The third topology shows a multi-tiered topology with Tier-0 gateway configured in Active/Standby HA mode to provide some centralized or stateful services like NAT, VPN etc.

<p align="center">
    <img src="images/Figure4-52.png">
</p>
<p align="center">
Figure 4‑52: Single tier and multi-tier routing topologies.

As discussed in the two-tier routing section, centralized services can be enabled on Tier-1 or Tier-0 gateway level. Figure 4-53 shows two multi-tiered topologies. The first topology shows centralized services like NAT, load balancer on Tier-1 gateways while Tier-0 gateway provides multiple active paths for L3 forwarding using ECMP. The second topology shows centralized services configured on a Tier-1 and Tier-0 gateway. In NSX-T 2.4 release or earlier, some centralized services are only available on Tier-1 like load balancer and other only on Tier-0 like VPN. Starting NSX-T 2.5 release, below topology can be used where requirement is to use both Load balancer and VPN service on NSX-T. Note that VPN is available on Tier-1 gateways starting NSX-T 2.5 release.

<p align="center">
    <img src="images/Figure4-53.png">
</p>
<p align="center">
Figure 4‑53: Stateful and Stateless (ECMP) Services Topologies Choices at each tier


Figure 4-54 shows a topology with Tier-0 gateways connected back to back. “Tenant-1 Tier-0 Gateway” is configured for a stateful firewall while “Tenant-2 Tier-0 Gateway” has stateful NAT configured. Since stateful services are configured on both “Tenant-1 Tier-0 Gateway” and “Tenant-2 Tier-0 Gateway”, they are configured in Active/Standby high availability mode. The top layer of Tier-0 gateway, "Aggregate Tier-0 Gateway” provides ECMP for North-South traffic. 

Note that only external interfaces should be used to connect a Tier-0 gateway to another Tier-0 gateway. Static routing and BGP are supported to exchange routes between two Tier-0 gateways and full mesh connectivity is recommended for optimal traffic forwarding. 

This topology provides high N-S throughput with centralized stateful services running on different Tier-0 gateways. This topology also provides complete separation of routing tables on the tenant Tier-0 gateway level and allows services that are only available on Tier-0 gateways (like VPN until NSX-T 2.4 release) to leverage ECMP northbound. Note that VPN is available on Tier-1 gateways starting NSX-T 2.5 release. 

NSX-T 3.0 introduces new multi tenancy features such as EVPN and VRF-lite. These features are recommended and suitable for true multi-tenant architecture where stateful services need to be run on multiple layers or Tier-0
 
Full mesh connectivity is recommended for optimal traffic forwarding.


<p align="center">
    <img src="images/Figure4-54.png">
</p>
<p align="center">
Figure 4‑54: Multiple Tier-0 Topologies with Stateful and Stateless (ECMP) Services 

Figure 4-55 shows another topology with Tier-0 gateways connected back to back. “Corporate Tier-0 Gateway” on Edge cluster-1 provides connectivity to the corporate resources (172.16.0.0/16 subnet) learned via a pair of physical routers on the left. This Tier-0 has stateful Gateway Firewall enabled to allow access to restricted users only.

“WAN Tier-0 Gateway” on Edge-Cluster-2 provides WAN connectivity via WAN routers and is also configured for stateful NAT. 
“Aggregate Tier-0 gateway” on the Edge cluster-3 learns specific routes for corporate subnet (172.16.0.0/16) from “Corporate Tier-0 Gateway” and a default route from “WAN Tier-0 Gateway”. 

“Aggregate Tier-0 Gateway” provides ECMP for both corporate and WAN traffic originating from any segments connected to it or connected to a Tier-1 southbound. 
Full mesh connectivity is recommended for optimal traffic forwarding.


<p align="center">
    <img src="images/Figure4-55.png">
</p>
<p align="center">
Figure 4‑55: Multiple Tier-0 Topologies with Stateful and Stateless (ECMP) Services 

A Tier-1 gateway usually connects to a Tier-0 gateway but it is possible for some use cases to interconnect it to another Tier-1 gateway using service interfaces (SI) and downlink as depicted in figure 4-56. Static routing must be configured on both Tier-1 gateways in this case as dynamic routing is not supported. The Tier-0 gateway must be aware of the all the overlay segments prefixes to provide connectivity.


<p align="center">
    <img src="images/Figure4-56.png">
</p>
<p align="center">
Figure 4‑56: Supported Topology – T1 gateways interconnected to each other using Service Interface and Downlink 



### 4.10.2 Unsupported Topologies

While the deployment of logical routing components enables customers to deploy flexible multi-tiered routing topologies, Figure 4-57 presents topologies that are not supported. The topology on the left shows that a tenant Tier-1 gateway cannot be connected directly to another tenant Tier-1 gateway using downlinks exclusively. 

The rightmost topology highlights that a Tier-1 gateway cannot be connected to two different upstream Tier-0 gateways.

<p align="center">
    <img src="images/Figure4-57.png">
</p>
<p align="center">
Figure 4‑57: Unsupported Topologies
