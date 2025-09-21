# **Kubernetes Networking Across On-Prem Datacenters with BGP, ECMP, and BFD**

## **Introduction**

Kubernetes powers mission-critical applications, many of which run in
on-premises datacenters spanning multiple sites. Unlike public clouds
that abstract multi-region networking, on-premises Kubernetes
deployments require **native, scalable underlay fabrics** to route Pod
and Service IPs efficiently.

While cloud providers hide networking complexity behind managed
services, on-premises environments expose the full challenge: designing
**resilient, scalable, and observable network fabrics** that allow Pods
and Services to be directly reachable, with predictable failover
behavior.

## **Core Networking Challenges**

On-premises Kubernetes networks face unique challenges:

1.  **Scalable Routing:** As the number of nodes, Pods, and Services
    > grows, traditional L2 overlays and static routes cannot scale
    > efficiently.

2.  **Fast Failover:** Links or nodes can fail. Without sub-second
    > detection and rerouting, workloads may become unreachable.

3.  **Multi-Datacenter Connectivity:** Workloads often span multiple
    > datacenters, requiring consistent routing policies and service
    > reachability across sites.

4.  **Operational Simplicity:** Large IP management overhead can
    > complicate configuration, especially for point-to-point links in
    > Clos fabrics.

## **Clos Fabrics and Dynamic Routing**

A **Clos network** (spine-leaf architecture) forms the backbone of
scalable, predictable datacenter networking:

1.  **Leaf Switches**:

    -   Connect directly to servers, including Kubernetes nodes.

    -   Handle BGP peering with upstream layers or external networks.

    -   Forward traffic to spine switches for inter-leaf communication.

2.  **Spine Switches**:

    -   Interconnect leaf switches, creating a high-speed backbone.

    -   Provide multiple equal-cost paths between leaves, enabling
        > **ECMP** (Equal-Cost Multi-Path).

    -   Ensure low latency, high bandwidth, and redundancy within the
        > datacenter fabric.

3.  **Border Leaf Switches**:

    -   Act as the datacenter's gateway to external networks (WAN,
        > Internet, or other datacenters).

    -   Peer with edge routers or upstream providers using protocols
        > like BGP.

    -   Aggregate traffic from the leaf-spine fabric for efficient
        > external routing.

4.  **Super Spine (Optional for Large-Scale Fabrics)**:

    -   Sits above the spine layer in very large datacenters to scale
        > horizontally.

    -   Connects multiple spine blocks, reducing oversubscription and
        > latency.

    -   Provides additional ECMP paths, enabling multi-pod or
        > multi-fabric connectivity for massive deployments.

5.  **ECMP (Equal-Cost Multi-Path)**:

    -   Balances traffic across multiple spine-leaf paths.

    -   Reduces congestion, provides redundancy, and optimizes bandwidth
        > utilization.

Dynamic routing protocols, particularly **BGP**, enable nodes and
services to advertise their presence directly into the fabric.

## **IPv6 BGP Unnumbered: Scaling Control Planes**

Traditional IPv4 BGP requires dedicated subnets for each point-to-point
link. In large fabrics, this results in massive subnet consumption and
operational overhead.

**IPv6 BGP Unnumbered** solves this problem:

-   Uses **link-local IPv6 addresses** for BGP session establishment.

-   No need to allocate /31 or /30 subnets per link.

-   IPv4 routes for Pods and Services are still exchanged.

-   Simplifies automation and reduces errors.

**Key takeaway:** IPv6 handles the control plane; IPv4 remains for
workloads.

### **Why FRR on Kubernetes Nodes?**

Running **FRR (Free Range Routing)** on nodes converts each Kubernetes
worker into a mini-router:

-   Advertises **Pod CIDRs** and **Service VIPs** directly to the
    > fabric.

-   Supports **BFD** for rapid failure detection (\<1 second).

-   Uses **loopbacks** as stable next-hops for ECMP.

-   Avoids overlay encapsulation overhead, enabling **native L3
    > routing** visibility.

This architecture ensures that **all workloads are first-class
citizens** on the network, visible to switches and routers for direct
routing, allowing the datacenter fabric to see Pods and Services as
**native IP prefixes**, not just encapsulated traffic.

## **Addressing Model**

  -----------------------------------------------------------------------
  **Segment**       **Example**         **Purpose**
  ----------------- ------------------- ---------------------------------
  Node              10.10.19.x          Loopback/router ID for nodes

  Pod               10.10.20.x          Dynamic Pod IPs

  Service           10.10.21.x          ClusterIP or LoadBalancer VIPs
  -----------------------------------------------------------------------

-   **10.10.19.x → Node Segment\
    > **

    -   Each Kubernetes node (baremetal or VM) has an IP from this
        > range.

    -   These are the primary IPs used for node-level communication and
        > management.

    -   These are used as unique loopback IP and **BGP router-IDs**.

    -   Advantage: Loopbacks remain stable even if physical interfaces
        > change, ensuring routing consistency.

    -   Example: 10.10.19.10 for Node1.

-   **10.10.20.x → Pod Segment\
    > **

    -   Each Pod running inside Kubernetes receives an IP from this
        > range.

    -   Allocated per-node as **Pod CIDRs** (e.g., Node1 gets
        > 10.10.20.0/24).

    -   Example: 10.10.20.5 → Pod running on Node1.

-   **10.10.21.x → Service Segment\
    > **

    -   Represents **virtual IPs** (ClusterIP or LoadBalancer) assigned
        > to services.

    -   These are advertised into the fabric as **/32 routes**, which
        > means each service IP is unique and routable.

    -   Example: 10.10.21.40/32 → A database service reachable across
        > fabrics.

This segmentation separates **nodes, workloads, and services**, making
routing policies easier to design and scale.

**Sample FRR Configuration**

frr version 8.1\
frr defaults traditional\
hostname cp1\
log syslog informational\
no ipv6 forwarding\
service integrated-vtysh-config\
\
interface eno5\
ipv6 nd ra-interval 6\
no ipv6 nd suppress-ra\
exit\
\
interface eno7\
ipv6 nd ra-interval 6\
no ipv6 nd suppress-ra\
exit\
\
interface lo\
ip address 10.10.19.10/32 \# Node loopback (router-id)\
ip address 10.10.21.66/32 \# Service IP advertised via loopback\
ip address 10.10.21.227/32 \# Additional service IPs on loopback\
exit\
\
router bgp 65496\
bgp router-id 10.10.19.10\
bgp bestpath as-path multipath-relax\
\
neighbor TOR peer-group\
neighbor TOR remote-as internal\
neighbor TOR bfd\
neighbor TOR timers 1 3\
\
neighbor eno5 interface peer-group TOR\
neighbor eno7 interface peer-group TOR\
\
bgp fast-convergence\
\
address-family ipv4 unicast\
network 10.30.19.6/32\
network 10.30.21.66/32\
network 10.30.21.227/32\
redistribute kernel\
redistribute connected\
exit-address-family\
exit

**Fabrics Overview**

**App Fabrics**

-   Kubernetes clusters in individual datacenters.

-   Node, Pod, Service routes advertised into Clos fabric.

-   ECMP provides multi-path load balance.

-   BFD triggers fast failover on node/link faults.

**DCI Fabric**

-   Connects multiple App Fabrics geographically.

-   Propagates routes between datacenters.

-   Enables cross-site service reachability and workload scaling.

**Edge Fabric**

-   Manages north-south ingress/egress.

-   Applies policy and security before external exposure.

-   Connects to the internet, WANs, or partner clouds.

![](media/image3.png){width="6.5in" height="3.5277777777777777in"}

## **BGP, ECMP, and BFD in Action**

### **Multi-BGP Neighbor Setup**

-   Each node peers with two upstream leaf switches.

-   ECMP allows traffic to utilize both uplinks efficiently.

-   **BFD** ensures that failures are detected sub-second, allowing FRR
    > to withdraw routes immediately.

### **Service Advertisement Flow**

-   Node advertises Service VIP to fabric via BGP.

-   Leaf switches propagate the route across spines (ECMP paths).

-   DCI fabric shares the route across datacenters.

-   Edge fabric routes external traffic to the correct node.

-   Failures are handled seamlessly, with remaining nodes continuing to
    > advertise VIPs.

**Why Service IPs Are Added to Loopback**

-   Service IPs are virtual and unbound from physical interfaces.

-   Assigning them to the node\'s loopback makes them stable BGP hosts.

-   Advertising these as /32 routes ensures accurate reachability.

-   This supports HA and ECMP forwarding of service traffic across the
    > fabric.

**Why Pod IPs Are Advertised Automatically**

-   Pods obtain dynamic IPs within node-specific CIDRs.

-   These IPs map to kernel-managed routes automatically.

-   FRR redistributes these kernel routes, advertising pod presence
    > dynamically.

-   This automation reduces manual config and promotes real-time fabric
    > update.

## **Extended Theory**

### **BGP Benefits in K8s**

-   **Loop-free routing:** BGP prevents routing loops even with multiple
    > paths.

-   **Scalability:** Thousands of nodes and services can be advertised
    > without creating massive L2 overlays.

-   **Policy Control:** Route-maps and filters can enforce service
    > placement policies.

### **ECMP Considerations**

-   Provides **load distribution** across multiple paths.

-   Works best with **stable loopback next-hops**.

-   May require tuning for **hash algorithms** to avoid uneven traffic
    > flows.

### **BFD Insights**

-   Detects link or node failure in **milliseconds**.

-   Reduces downtime by quickly withdrawing unreachable routes.

-   Works in combination with BGP to trigger failover without waiting
    > for BGP timers.

## **Best Practices** 

-   Use **automation** (Ansible, operators) for FRR configs.

-   Maintain **IP consistency** across datacenters to avoid routing
    > conflicts.

-   Monitor **BGP and BFD sessions** actively.

-   Secure routing with **prefix filters** and **network policies**.

-   Test failover scenarios regularly.

## **Conclusion**

By extending Kubernetes into the **Clos fabric** using **FRR, BGP, ECMP,
and BFD**, enterprises can:

-   Treat Pods and Services as **natively routable entities**.

-   Achieve **sub-second failover** with BFD.

-   Use **IPv6 BGP unnumbered** to simplify operations.

-   Scale workloads **horizontally across multiple datacenters**.

This design delivers **cloud-like networking behavior** for on-prem
deployments---allowing Kubernetes services to remain globally reachable,
highly available, and scalable across fabrics.
