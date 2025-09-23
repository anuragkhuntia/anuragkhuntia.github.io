---
title: "Kubernetes Networking Across On-Prem Datacenters with BGP, ECMP, and BFD"
excerpt: "Designing multi-datacenter Kubernetes networking using BGP, ECMP, BFD, and Clos fabrics."
layout: single
classes: wide
permalink: /k8s-bgp-ecmp-bfd/
author: "Anurag Khuntia"
date: 2025-09-21
read_time: true
share: true
comments: false
related: true
categories: 
  - Kubernetes
  - Networking
  - BGP
  - ECMP
  - BFD
tags: 
  - kubernetes
  - networking
  - bgp
  - ecmp
  - bfd
  - clos
header:
  overlay_image: /assets/images/kubernets_onprem_fabric.png
  overlay_filter: 0.3
  caption: "Clos fabric with Leaf, Spine, Border Leaf, and Super Spine"
toc: true
toc_label: "Table of Contents"
toc_icon: "list"
---

## Introduction

Kubernetes powers mission-critical applications, many of which run in on-premises datacenters spanning multiple sites. 

Unlike public clouds that abstract multi-region networking, on-premises Kubernetes deployments require **native, scalable underlay fabrics** to route Pod and Service IPs efficiently.

Cloud providers simplify networking using managed services. However, in on-prem environments, architects must build **resilient, scalable, and observable network fabrics** that ensure Pod and Service reachability with fast, predictable failover.

---

## Core Networking Challenges

On-prem Kubernetes networking introduces a unique set of challenges:

1. **Scalable Routing:** Large clusters quickly outgrow static routes and L2 overlays.
2. **Fast Failover:** Without sub-second rerouting, workloads may become unreachable on failure.
3. **Multi-Datacenter Reachability:** Routing consistency across geographically separate sites is critical.
4. **Operational Simplicity:** Managing IPs for Clos topologies can introduce significant complexity.

---

## Clos Fabrics and Dynamic Routing

A **Clos (leaf-spine)** network provides the backbone for scalable, fault-tolerant datacenter networking.

### Leaf Switches
- Connect directly to Kubernetes nodes.
- Peer with spines or border devices using BGP.
- Serve as first-hop routers for traffic entering the fabric.

### Spine Switches
- Interconnect leaf switches.
- Provide multiple equal-cost paths (ECMP).
- Support redundancy and high bandwidth.

### Border Leaf Switches
- Bridge internal fabrics with WAN/Internet/DCI.
- Handle egress/ingress and inter-datacenter communication.

### Super Spines (Optional)
- Used in hyperscale environments.
- Connect multiple spine layers for additional ECMP and scale.

---

### ECMP (Equal-Cost Multi-Path)

ECMP spreads traffic across multiple paths of equal cost:
- Enables **high availability**.
- Improves **bandwidth utilization**.
- Simplifies **failure recovery**.

---

## IPv6 BGP Unnumbered: Scaling the Control Plane

Traditional IPv4 BGP requires point-to-point subnets, which doesn't scale well.

### IPv6 BGP Unnumbered Benefits:

- Uses **link-local IPv6 addresses** to establish BGP sessions.
- Eliminates the need for /30 or /31 subnets.
- IPv4 routes are still advertised normally.
- Reduces IP planning and automation complexity.

**Key Insight:** Use IPv6 for the control plane, IPv4 for the data plane.

---

## Running FRR on Kubernetes Nodes

Deploying **FRR (Free Range Routing)** on Kubernetes nodes turns them into routers:

- Advertises **Pod CIDRs** and **Service VIPs** directly into the fabric.
- Enables **BFD** for fast route withdrawal (<1s).
- Uses **loopback addresses** for ECMP stability.
- Avoids encapsulation; traffic is routed natively via L3.

This model makes workloads **visible and routable** throughout the Clos fabric.

---

## IP Addressing Model

| Segment | Example        | Purpose                          |
|---------|----------------|----------------------------------|
| Node    | 10.10.19.x     | Loopback IP for BGP router ID    |
| Pod     | 10.10.20.x     | Pod IP range (via Pod CIDRs)     |
| Service | 10.10.21.x     | VIPs (ClusterIP or LB services)  |

### Example:
- Node loopback: `10.10.19.10`
- Pod IP: `10.10.20.5`
- Service VIP: `10.10.21.40/32`

This segmentation separates infrastructure and workload IPs for cleaner routing policies.

---

## Sample FRR Configuration

```bash
frr version 8.1
frr defaults traditional
hostname cp1
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
 ip address 10.10.19.10/32     # Node loopback
 ip address 10.10.21.66/32     # Service VIP
 ip address 10.10.21.227/32    # Additional VIP
exit

router bgp 65496
 bgp router-id 10.10.19.10
 bgp bestpath as-path multipath-relax

 neighbor TOR peer-group
 neighbor TOR remote-as internal
 neighbor TOR bfd
 neighbor TOR timers 1 3

 neighbor eno5 interface peer-group TOR
 neighbor eno7 interface peer-group TOR

 bgp fast-convergence

 address-family ipv4 unicast
  network 10.10.19.10/32
  network 10.10.21.66/32
  network 10.10.21.227/32
  redistribute kernel
  redistribute connected
 exit-address-family
exit

```

## Fabrics Overview

**App Fabrics**

- Kubernetes clusters in individual datacenters.
- Node, Pod, Service routes advertised into Clos fabric.
- ECMP provides multi-path load balance.
- BFD triggers fast failover on node/link faults.

**DCI Fabric**

- Connects multiple App Fabrics geographically.
- Propagates routes between datacenters.
- Enables cross-site service reachability and workload scaling.

**Edge Fabric**

- Manages north-south ingress/egress.
- Applies policy and security before external exposure.
- Connects to the internet, WANs, or partner clouds.

![Clos fabric with Leaf, Spine, Border Leaf, and Super Spine](/assets/images/kubernets_onprem_fabric.png){width="6.5in" height="3.5277in"}

## BGP, ECMP, and BFD in Action

### Multi-BGP Neighbor Setup

- Each node peers with two upstream leaf switches.
- ECMP allows traffic to utilize both uplinks efficiently.
- **BFD** ensures failures are detected sub-second, allowing FRR to withdraw routes immediately.

### Service Advertisement Flow

- Node advertises Service VIP to fabric via BGP.
- Leaf switches propagate the route across spines (ECMP paths).
- DCI fabric shares the route across datacenters.
- Edge fabric routes external traffic to the correct node.
- Failures are handled seamlessly, with remaining nodes continuing to advertise VIPs.

### Why Service IPs Are Added to Loopback

- Service IPs are virtual and unbound from physical interfaces.
- Assigning them to the node's loopback makes them stable BGP hosts.
- Advertising these as /32 routes ensures accurate reachability.
- Supports HA and ECMP forwarding of service traffic across the fabric.

### Why Pod IPs Are Advertised Automatically

- Pods obtain dynamic IPs within node-specific CIDRs.
- These IPs map to kernel-managed routes automatically.
- FRR redistributes kernel routes, advertising pod presence dynamically.
- Automation reduces manual config and promotes real-time fabric update.

## Extended Theory

### BGP Benefits in K8s

- **Loop-free routing:** BGP prevents routing loops even with multiple paths.
- **Scalability:** Thousands of nodes and services can be advertised without massive L2 overlays.
- **Policy Control:** Route-maps and filters can enforce service placement policies.

### ECMP Considerations

- Provides **load distribution** across multiple paths.
- Works best with **stable loopback next-hops**.
- May require tuning for **hash algorithms** to avoid uneven traffic flows.

### BFD Insights

- Detects link or node failure in **milliseconds**.
- Reduces downtime by quickly withdrawing unreachable routes.
- Works alongside BGP to trigger failover without waiting for BGP timers.

## Best Practices

- Use **automation** (Ansible, operators) for FRR configs.
- Maintain **IP consistency** across datacenters to avoid routing conflicts.
- Monitor **BGP and BFD sessions** actively.
- Secure routing with **prefix filters** and **network policies**.
- Test failover scenarios regularly.

## Conclusion

By extending Kubernetes into the **Clos fabric** using **FRR, BGP, ECMP, and BFD**, enterprises can:

- Treat Pods and Services as **natively routable entities**.
- Achieve **sub-second failover** with BFD.
- Use **IPv6 BGP unnumbered** to simplify operations.
- Scale workloads **horizontally across multiple datacenters**.

This design delivers **cloud-like networking behavior** for on-prem deployments — allowing Kubernetes services to remain globally reachable, highly available, and scalable across fabrics.

