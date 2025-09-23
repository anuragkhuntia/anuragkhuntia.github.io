---
layout: single
title: "Kubernetes Networking Across On-Prem Datacenters with BGP, ECMP, and BFD"
excerpt: "Exploring scalable Kubernetes networking for multi-site on-prem clusters."
header:
  image: /assets/images/datacenter-k8s.jpeg
  teaser: /assets/images/datacenter-k8s.jpeg
sidebar:
  nav: "main"
toc: true
toc_label: "Contents"
breadcrumbs: true
---

## Introduction

Kubernetes powers mission-critical applications in enterprises, many of which still operate in **on-premises datacenters** spanning multiple sites.  

Unlike cloud environments that abstract away networking complexity, on-prem deployments must deal directly with **routing, failover, and scalability**. Designing a **resilient, observable, and highly available network fabric** is essential for seamless connectivity across clusters.  

In this blog, we’ll explore how modern datacenter concepts—**Clos fabrics, BGP dynamic routing, ECMP multipath forwarding, and BFD fast failover**—enable Kubernetes to integrate natively into enterprise networks.


---

## Core Networking Challenges in On-Prem Kubernetes
On-premises Kubernetes faces unique challenges:

- **Scalable Routing:** Static routes and L2 overlays don’t scale beyond a few hundred nodes.  
- **Rapid Failover:** Without fast detection, link/node failures cause downtime.  
- **Multi-Datacenter Connectivity:** Services must remain reachable across sites.  
- **Operational Simplicity:** Managing IPs and routes manually is error-prone—automation is key.  

---

## Clos Fabrics and Dynamic Routing
A **Clos (spine-leaf) fabric** underpins most modern datacenters:

- **Leaf switches** connect to servers (Kubernetes nodes), peering with BGP.  
- **Spine switches** interconnect leaves, providing high bandwidth and ECMP paths.  
- **Border leaves** connect to WANs, other DCs, or the Internet.  
- **Super spines** (in large DCs) connect multiple spine blocks.  

Using **BGP**, nodes and services advertise their IPs into the fabric. **ECMP** spreads traffic across multiple paths, ensuring redundancy and load balancing.

---

## Scaling Control Planes with IPv6 BGP Unnumbered
Traditional IPv4 BGP requires per-link subnets, which wastes IP space.  
**IPv6 BGP unnumbered** solves this:

- Uses **link-local IPv6** addresses for BGP sessions.  
- No need for `/31` or `/30` subnets.  
- Still exchanges **IPv4 Pod and Service routes**.  

➡️ Key takeaway: **IPv6 for control plane, IPv4 for workloads.**nsight:** Use IPv6 for the control plane, IPv4 for the data plane.

---

## Running FRR on Kubernetes Nodes
[FRR (Free Range Routing)](https://frrouting.org) is an open-source routing suite supporting BGP, OSPF, IS-IS, etc.  

On Kubernetes nodes, FRR turns each worker into a **mini-router**:

- Advertises **Pod CIDRs** and **Service VIPs** to the fabric.  
- Supports **BFD** for sub-second failover.  
- Uses **loopbacks** as stable next-hops for ECMP.  
- Eliminates overlay tunneling, enabling **native L3 visibility**. 

---

## IP Addressing Model

A clean IP plan keeps things consistent:

- **Node Segment (10.10.19.x):** Loopback per node, used as BGP Router-ID.  
- **Pod Segment (10.10.20.x):** Dynamic IPs from per-node CIDRs, auto-advertised.  
- **Service Segment (10.10.21.x):** ClusterIP /32 VIPs assigned to loopbacks, advertised across the fabric.  

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

![Clos fabric with Leaf, Spine, Border Leaf, and Super Spine](/assets/images/kubernets_onprem_fabric.png)

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

