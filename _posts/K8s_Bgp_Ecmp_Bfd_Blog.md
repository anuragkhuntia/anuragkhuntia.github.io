
# Kubernetes Networking Across On-Prem Datacenters with BGP, ECMP, and BFD

## Introduction

Kubernetes powers mission-critical applications, many of which run in on-premises datacenters spanning multiple sites. Unlike public clouds that abstract multi-region networking, on-premises Kubernetes deployments require **native, scalable underlay fabrics** to route Pod and Service IPs efficiently.

In public clouds, multi-zone and multi-region networking is hidden from the user. On-premises operators, however, must design networks that support:

- **Scalable routing** of Pod and Service IPs.  
- **Resilient failover** in case of link or node failures.  
- **Consistent connectivity** across multiple datacenters.  

This is achieved using a **Clos (spine–leaf) network fabric** with dynamic routing protocols and fast failover mechanisms:

| Technology | Purpose |
|------------|---------|
| **BGP (Border Gateway Protocol)** | Advertises Node, Pod, and Service IPs as first-class network entities |
| **ECMP (Equal-Cost Multi-Path)** | Load balances traffic across multiple spine-leaf paths |
| **BFD (Bidirectional Forwarding Detection)** | Provides sub-second failure detection for rapid failover |

By running **FRR (Free Range Routing)** on Kubernetes worker nodes, workloads are treated as peers, enabling **seamless scaling** and **consistent load distribution**. This combination allows on-prem Kubernetes clusters to achieve **cloud-like networking behavior**, making workloads reachable across datacenters as if they were in a single large fabric.

---

## Fabrics Overview

### 1. App Fabric
- Kubernetes clusters within individual datacenters.  
- Node, Pod, and Service routes advertised into the Clos fabric.  
- ECMP provides multi-path load balancing.  
- BFD triggers fast failover on node or link failures.

### 2. DCI Fabric
- Connects multiple App Fabrics across sites.  
- Propagates routes between datacenters.  
- Enables cross-site service reachability and workload scaling.

### 3. Edge Fabric
- Manages north-south ingress and egress traffic.  
- Applies policies and security before external exposure.  
- Connects to the internet, WANs, or partner clouds.

---

## Why IPv6 BGP Unnumbered?

Traditional IPv4 requires a `/31` or `/30` subnet for each point-to-point link. In large Clos fabrics with thousands of links, this is unmanageable.

**IPv6 BGP Unnumbered** addresses this:

- BGP sessions use **link-local IPv6 addresses** (auto-assigned).  
- No need to allocate or manage IPv4 subnets for P2P links.  
- BGP still exchanges **IPv4 routes** for Pod/Service IPs.  
- Reduces IP consumption and simplifies operations.

**In short:** IPv6 is used for the **control plane**, IPv4 for **workloads**.

---

## Addressing Model

| Segment | Example | Purpose |
|---------|---------|---------|
| **Node Segment** | 10.19.x.x | Worker node IP (also loopback/router ID) |
| **Pod Segment** | 10.20.x.x | Dynamic Pod IPs, advertised via kernel route redistribution |
| **Service Segment** | 10.21.x.x | ClusterIP/LoadBalancer IPs, assigned to loopback interface |

**Examples:**

- **Node IP:** `10.19.1.10` → Node1 loopback and BGP router ID  
- **Pod IP:** `10.20.1.5` → Pod running on Node1  
- **Service IP:** `10.21.40.40/32` → Database service reachable across fabrics

Segmentation separates nodes, workloads, and services, simplifying routing policies and scaling.

---

## Why FRR on Kubernetes Nodes?

Default Kubernetes networking uses overlays (VXLAN, Geneve, CNI plugins):

- Adds encapsulation overhead.  
- Pods are invisible to the fabric.  
- Limited scalability across datacenters.  

**Solution:** Run **FRR** on each node:

- Establishes BGP peering with top-of-rack (ToR) leaf switches.  
- Advertises Pod CIDRs and Service VIPs directly.  
- Uses **loopbacks as stable next-hops** for resilient multipath routing.  
- Supports **BFD** for sub-second failure detection.

This allows **Pods and Services** to be seen as **native IP prefixes**, eliminating encapsulation overhead.

---

## Sample FRR Configuration

```bash
frr version 8.1
frr defaults traditional
hostname cp01
log syslog informational
no ipv6 forwarding
service integrated-vtysh-config

interface eno5
 ipv6 nd ra-interval 6
 no ipv6 nd suppress-ra
exit

interface eno7
 ipv6 nd ra-interval 6
 no ipv6 nd suppress-ra
exit

interface lo
 ip address 10.30.19.6/32     # Node loopback (router-id)
 ip address 10.30.21.66/32    # Service IP
 ip address 10.30.21.227/32   # Additional service IPs
exit

router bgp 65496
 bgp router-id 10.30.19.6
 bgp bestpath as-path multipath-relax

 neighbor TOR peer-group
 neighbor TOR remote-as internal
 neighbor TOR bfd
 neighbor TOR timers 1 3

 neighbor eno5 interface peer-group TOR
 neighbor eno7 interface peer-group TOR

 bgp fast-convergence

 address-family ipv4 unicast
  network 10.30.19.6/32
  network 10.30.21.66/32
  network 10.30.21.227/32
  redistribute kernel
  redistribute connected
 exit-address-family
exit
```

---

## Why Service IPs Are Added to Loopback

- Service IPs are virtual and **not bound to physical interfaces**.  
- Assigning them to loopback makes them **stable BGP hosts**.  
- Advertising as `/32` ensures **accurate reachability**.  
- Supports HA and **ECMP forwarding**.

---

## Why Pod IPs Are Advertised Automatically

- Pods receive **dynamic IPs** from node-specific CIDRs.  
- Kernel-managed routes are redistributed by FRR automatically.  
- Reduces manual configuration and ensures **real-time fabric updates**.

---

## Multi-BGP Neighbor Leaf Switch Setup

- Each node peers with **two leaf switches** using separate interfaces (e.g., `eno5`, `eno7`).  
- **BFD enabled** for sub-second failure detection.  
- Peer groups simplify configuration and policy consistency.  
- **ECMP balances traffic**, avoiding session flaps.  
- **Stable loopback IP** used as next-hop for resilience.

```bash
router bgp 65496
 bgp router-id 10.30.19.6

 neighbor TOR peer-group
 neighbor TOR remote-as internal
 neighbor TOR bfd
 neighbor TOR timers 1 3

 neighbor eno5 interface peer-group TOR
 neighbor eno7 interface peer-group TOR

 bgp fast-convergence
```

---

## Service Advertisement Flow

**Example:** Service VIP `10.21.40.40/32` in App Fabric #1

1. **Node-Level:**  
   Node hosting the service advertises `10.21.40.40/32` via FRR.  Next-hop is the **node loopback** (e.g., `10.255.1.1`).

2. **App Fabric Distribution:**  
   Leaf switch installs the route.  ECMP propagates it across spines.

3. **DCI Propagation:**  
   Prefix exported into DCI fabric. Service becomes reachable from other datacenters.

4. **Edge Access:**  
   External clients reach service via Edge → DCI → App Fabric → Node.

5. **Failure Handling:**  
   If Node1 fails, BFD tears down session and FRR withdraws route. Service remains reachable via other nodes advertising the same VIP.

---

## Best Practices

- Automate FRR deployment using **Ansible** or operators.  
- Use **IPv6 BGP unnumbered sessions** for scalability.  
- Coordinate **IP space and routing policies** with network/security teams.  
- Monitor **BGP and BFD states** for alerts.  
- Secure pods with **Kubernetes Network Policies**.

---

## Conclusion

By extending Kubernetes into the Clos fabric with **FRR, BGP, ECMP, and BFD**, enterprises can:

- Treat Pods and Services as **natively routable entities**.  
- Achieve **sub-second failover** with BFD.  
- Simplify operations using **IPv6 BGP unnumbered**.  
- Scale workloads **horizontally across multiple datacenters**.  

This design delivers **cloud-like networking behavior** on-prem, ensuring services are globally reachable, highly available, and scalable.
