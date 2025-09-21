---
title: "Kubernetes Networking Across On-Prem Datacenters with BGP, ECMP, and BFD"
layout: single
author: "Anurag Khuntia"
date: 2025-09-21
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
  overlay_image: /assets/images/clos-network.png
  overlay_filter: 0.3
  caption: "Clos fabric with Leaf, Spine, Border Leaf, and Super Spine"
toc: true
toc_label: "Table of Contents"
toc_icon: "list"
excerpt: "How Clos fabrics, BGP, ECMP, and BFD enable scalable, cloud-like networking for Kubernetes in on-prem datacenters."
---

## Introduction

Kubernetes is no longer limited to the cloud. Enterprises are increasingly deploying it in **on-premises datacenters** where networking becomes a first-class challenge. Unlike public clouds that hide complexity behind managed load balancers and VPC abstractions, on-prem environments require you to design the **underlay fabric** yourself.  

The question is: how do we make these fabrics **scalable, predictable, and fault-tolerant** while keeping Pods and Services directly reachable? The answer lies in combining **Clos architectures** with **BGP, ECMP, and BFD**.

---

## Core Networking Challenges

Running Kubernetes on-premises introduces a unique set of networking challenges. As clusters scale to hundreds of nodes and thousands of Pods, traditional Layer-2 overlays break down. Routing must scale linearly, and failures must be handled in milliseconds to avoid disrupting applications.  

The main problems can be summarized as:

- **Scalable Routing:** Static routes and VLAN overlays cannot scale to the size of modern clusters.  
- **Fast Failover:** Without rapid failure detection, workloads can become unreachable for seconds or even minutes.  
- **Multi-Datacenter Connectivity:** Applications increasingly span multiple sites and require consistent routing policies.  
- **Operational Simplicity:** Managing thousands of point-to-point IPs in Clos fabrics quickly becomes an operational nightmare.  

---

## Clos Fabrics and Dynamic Routing

The foundation of a modern datacenter network is the **Clos (spine-leaf) fabric**. This architecture eliminates bottlenecks and provides predictable bandwidth and latency.  

### Leaf Switches
Leaf switches connect directly to servers — in this case, Kubernetes worker nodes. They also establish BGP sessions to exchange routing information. All east-west traffic between nodes is sent “up” to the spines and then “down” to the destination leaf.  

### Spine Switches
Spine switches interconnect all leaves, forming a non-blocking fabric. Since every leaf connects to every spine, there are always multiple equal-cost paths between any two endpoints. This guarantees **low latency, predictable bandwidth, and natural redundancy**.  

### Border Leaf Switches
For traffic leaving the datacenter, border leaves act as gateways. They peer with WAN routers, the internet, or partner networks, aggregating traffic from the fabric and enforcing external routing policies.  

### Super Spine
In very large datacenters, a **super spine** layer is introduced above the spines. This allows fabrics to scale horizontally by interconnecting multiple spine blocks, further reducing oversubscription.  

### ECMP in Action
The magic that ties it together is **Equal-Cost Multi-Path (ECMP)**. Instead of sending traffic down a single link, ECMP spreads flows evenly across all available spine-leaf paths. The result is balanced utilization and seamless failover if a link or switch goes down.  

---

## IPv6 BGP Unnumbered: Scaling Control Planes

One operational headache of traditional IPv4 BGP is that it requires a dedicated subnet for every point-to-point link. In a large fabric, this burns through address space and complicates IPAM.  

**IPv6 BGP Unnumbered** solves this elegantly. It uses IPv6 link-local addresses to establish BGP sessions, meaning you no longer need to allocate `/31` or `/30` subnets for each connection. Pods and Services still use IPv4 — but the control plane runs over IPv6, making it leaner and easier to automate.  

> **Key Takeaway:** Let IPv6 run the **control plane**, and IPv4 carry the **workloads**.  
{: .notice--success}

---

## Why FRR on Kubernetes Nodes?

Instead of hiding Pods behind overlays, each Kubernetes node can be turned into a **mini-router** by running [FRR (Free Range Routing)](https://frrouting.org/). This approach has several benefits:  

- Nodes advertise **Pod CIDRs** and **Service VIPs** directly to the fabric.  
- **BFD** enables sub-second detection of failures, far faster than default BGP timers.  
- Loopback addresses serve as **stable next-hops** for ECMP, even if interfaces change.  
- Native Layer-3 routing eliminates overlay complexity and makes Pods first-class citizens on the network.  

This means the fabric sees Pods and Services as **real IP prefixes**, not encapsulated packets hidden inside tunnels.  

---

## Addressing Model

To keep the design clean, we use separate IP ranges for nodes, Pods, and Services:  

| Segment | Example      | Purpose                          |
|---------|-------------|----------------------------------|
| Node    | 10.10.19.x  | Node loopbacks & router IDs      |
| Pod     | 10.10.20.x  | Pod IPs, allocated per-node CIDRs|
| Service | 10.10.21.x  | ClusterIP or LoadBalancer VIPs   |

For example, Node1 may use `10.10.19.10` as its router ID, a Pod on that node may get `10.10.20.5`, and a database Service may be advertised globally as `10.10.21.40/32`. This clear separation makes routing policies easier to reason about.  

---

## Sample FRR Configuration

Here’s a simplified FRR configuration for a node:

```conf
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

interface lo
 ip address 10.10.19.10/32     # Node loopback
 ip address 10.10.21.66/32     # Service VIP
exit

router bgp 65496
 bgp router-id 10.10.19.10
 bgp bestpath as-path multipath-relax

 neighbor TOR peer-group
 neighbor TOR remote-as internal
 neighbor TOR bfd
 neighbor TOR timers 1 3

 neighbor eno5 interface peer-group TOR

 address-family ipv4 unicast
  network 10.10.19.10/32
  network 10.10.21.66/32
  redistribute kernel
  redistribute connected
 exit-address-family
exit
This example shows how the node uses a loopback for stability, advertises service IPs, and peers directly with ToR (leaf) switches.

Fabrics Overview

The datacenter network can be visualized as three interconnected fabrics:

App Fabrics — the per-datacenter Clos networks where Kubernetes clusters live.

DCI Fabric — connects multiple App Fabrics across geographies, allowing workloads to stretch across sites.

Edge Fabric — manages ingress and egress traffic to the outside world, applying policies before exposure.

BGP, ECMP, and BFD in Action

With the architecture in place, here’s how routing works in practice.

Every node establishes two BGP sessions with its upstream leaf switches. ECMP ensures that traffic is balanced across both uplinks. BFD runs alongside these sessions, monitoring health in real-time. If one path fails, FRR immediately withdraws the route, and traffic shifts seamlessly to the surviving link.

Service IPs are advertised via loopbacks, making them highly available and routable. Leaf switches distribute these routes across the spines, while DCI extends them across datacenters. Edge devices ensure external clients can reach the right node regardless of failures.

This combination of ECMP load-balancing and BFD-driven fast failover guarantees resiliency at scale.

Extended Theory

Why choose BGP in Kubernetes fabrics? The reasons go beyond just scalability.

BGP naturally prevents routing loops, even with multiple paths. It can handle thousands of routes without collapsing under complexity, and its policies allow fine-grained control over how traffic flows. Combined with ECMP, it makes the most of every available link, while BFD ensures that failures are handled in milliseconds.

Together, these protocols bring cloud-scale networking discipline into on-prem datacenters.

Best Practices

From experience, here are a few practical tips:

Use automation (Ansible, operators) to roll out FRR configs consistently.

Keep IP allocations consistent across fabrics to avoid conflicts.

Actively monitor BGP and BFD sessions — they are your lifeline.

Secure routing with prefix filters and strict policies.

Regularly test failover scenarios to validate assumptions.

Conclusion

By integrating Kubernetes nodes directly into a Clos fabric with FRR, BGP, ECMP, and BFD, enterprises gain a cloud-like networking experience on-prem:

Pods and Services become natively routable entities.

Failures are handled in sub-seconds.

IPv6 unnumbered simplifies the control plane.

Workloads can scale across multiple datacenters with confidence.