---
layout: single
title: "Cosmolet: Technical Deep Dive"
excerpt: "Exploring automated BGP service IP advertisement for on-prem Kubernetes clusters using FRR."
header:
  image: /assets/images/cosmolet-bgp.jpeg
  teaser: /assets/images/cosmolet-bgp.jpeg
categories: [Kubernetes, Networking]
tags: [Kubernetes, BGP, FRR, Bare-Metal, LoadBalancer, On-Prem]
sidebar:
  nav: "main"
toc: true
toc_label: "Contents"
breadcrumbs: true
---

# Cosmolet: Deep Dive into Kubernetes BGP Service Advertisement

When running Kubernetes on bare-metal, one of the biggest challenges is integrating Kubernetes networking with the existing data center fabric.  
While cloud environments provide managed LoadBalancers, on-prem clusters typically depend on external devices, static routes, or CNI-specific BGP integrations that often break the abstraction between Kubernetes and the underlay.

**Cosmolet** aims to bridge this gap.

It’s a lightweight, CNI-agnostic Kubernetes controller that automates the advertisement of Kubernetes Service IPs directly to your network using **FRR (Free Range Routing)**.  
Cosmolet runs as a **DaemonSet** across all cluster nodes, continuously monitoring the cluster state and syncing service reachability with the routing fabric — without the need for external proxies or vendor-specific solutions.

---

## Design Philosophy

Cosmolet’s core principle is **simplicity without compromise**:
- It should work in any Kubernetes environment — regardless of the CNI.
- It should integrate naturally with standard BGP deployments.
- It should remain transparent, observable, and secure.

This philosophy results in an agent that directly interacts with the **Linux networking stack** and **FRR**, making BGP-based service advertisement both predictable and portable.

---

## Operating Modes

Cosmolet operates in **two distinct modes** — `Connected` and `Dynamic` — to accommodate different FRR configurations and deployment styles.  
It automatically determines the mode at runtime based on FRR’s configuration parameters.

### Connected Mode
If FRR is configured with: bgpd distributed connected
then Cosmolet runs in **Connected Mode**.  
In this mode:
- FRR automatically advertises all locally connected routes.
- Cosmolet’s responsibility is to **synchronize service IPs with the loopback interface**.
- It adds or removes `/32` (or `/128` for IPv6) routes corresponding to Kubernetes service IPs.
- No direct BGP configuration commands are issued — FRR handles advertisement natively.

Connected mode is ideal for modern, distributed FRR setups commonly found in large-scale fabrics (e.g., Cumulus, SONiC, or containerized FRR instances).

### Dynamic Mode
If FRR does **not** have the `distributed connected` parameter, Cosmolet switches to **Dynamic Mode**.  
In this mode:
- Cosmolet takes direct control of BGP advertisement.
- It uses the FRR vtysh CLI or API to programmatically add or remove network statements under the BGP configuration context.
- Each node independently advertises only those service IPs it’s responsible for.

This mode ensures compatibility with simpler or traditional FRR setups where automatic route distribution isn’t configured.

---

## Service Discovery and Health Validation

Cosmolet continuously watches Kubernetes `Service` and `Endpoint` objects using native client-go informers.  
For every eligible service (`LoadBalancer` or `ClusterIP`), it checks whether:
- The service has healthy endpoints.
- The service IP is assigned or reachable on the node.

If both conditions are met, the service IP is added to the loopback and advertised to the fabric.

If all endpoints of a service become unhealthy, Cosmolet **withdraws the advertisement** — preventing blackhole routes and ensuring that BGP reflects real application availability.

This **health-aware advertisement** model forms the foundation of reliable BGP-based service publishing in Kubernetes.

---

## Loopback IP Synchronization

Loopback management is central to Cosmolet’s operation.

For each service IP that should be advertised, the controller runs:
```bash
ip addr add 10.30.21.232/32 dev lo 
```
and when no longer valid:
```bash
ip addr del 10.30.21.232/32 dev lo 
```
Cosmolet automatically reconciles the loopback state on every iteration:

- Active IPs → retained
- Stale IPs → removed
- Excluded IPs → preserved (based on config.yaml)

This ensures that the node’s loopback interface always reflects the real set of advertised service IPs.

---
## FRR Integration and Control Flow

FRR (Free Range Routing) acts as the BGP stack beneath Cosmolet.
Cosmolet communicates with FRR in one of two ways:

Dynamic Mode:
Direct vtysh command execution:
```bash
vtysh -c "configure terminal" \
      -c "router bgp 65001" \
      -c "network 10.30.21.232/32"
```
or its removal:
```bash
vtysh -c "configure terminal" \
      -c "router bgp 65001" \
      -c "no network 10.30.21.232/32"
```
Connected Mode:
FRR automatically detects connected loopback IPs, so no direct CLI execution is needed. Cosmolet’s role ends after adding the IP to lo.

This modular integration ensures that Cosmolet can adapt to both fully distributed and standalone FRR deployments, without code or configuration changes.

---
## High Availability and Scalability

Cosmolet runs as a DaemonSet — ensuring that every node independently manages its local advertisements.
This design offers:

- Horizontal scalability: Each node runs autonomously.
- No single point of failure: BGP advertisements are localized.
- Leader election support (optional): Used for centralized coordination or cluster-wide statistics.

This decentralized architecture matches Kubernetes’ fault-tolerance model — if a node or pod fails, its routes are automatically withdrawn by FRR’s BGP session teardown.

---
## Security and Privileges

Cosmolet follows a minimal-privilege principle:

- Runs as a non-root container.
- Uses Linux capabilities (NET_ADMIN) only where required.
- No dependency on sudo or privileged mode.
- RBAC limited to read-only access on Service and Endpoints resources.

This ensures compatibility with hardened cluster policies and secure multi-tenant setups.

---

## Deployment Approaches

Cosmolet supports multiple deployment methods:

- Helm chart for production use with configurable BGP, FRR, and namespace settings.
- YAML manifests for quick testing or custom integration.
- Custom DaemonSet templates for embedding within existing network operators.

Since it always runs as a DaemonSet, deployment is per-node and aligns with FRR’s operational model.

---

## Closing Thoughts

Cosmolet simplifies one of the most complex aspects of bare-metal Kubernetes — integrating service IP routing with real BGP networks.
Its two-mode architecture ensures that it can adapt to both modern and legacy FRR topologies, making it an effective choice for clusters that span racks, fabrics, or hybrid environments.

By bringing BGP awareness natively into the Kubernetes control plane, Cosmolet enables operators to run cloud-like networking on bare metal — predictably, efficiently, and transparently.