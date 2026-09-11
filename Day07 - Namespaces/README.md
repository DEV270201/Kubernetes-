# Day 07 - Namespaces

## What I Did Today

Today I covered namespaces: what they are, how they compare to clusters, when to use one over the other, and how IP addressing actually works across namespaces inside a cluster.

---

## The Core Distinction

Before getting into namespaces, the question worth answering first is: why not just create a new cluster for every team or environment?

### What a Cluster Actually Costs You

A cluster is not just a logical boundary. It is a physical infrastructure reality:

- Its own control plane (API server, scheduler, controller-manager, etcd). On managed services like EKS, GKE, or AKS, you pay a control plane fee per cluster.
- Its own set of nodes (VMs) that need to be sized, patched, and monitored.
- Its own networking setup, ingress controllers, DNS, and CNI plugins.
- Its own cluster-wide installs: monitoring stack, logging, service mesh, cert-manager, and so on. All duplicated per cluster.
- Separate kubectl contexts and separate cloud IAM/auth wiring.

Ten teams on ten clusters means ten control planes, ten Prometheus installs, and ten sets of nodes that are probably underutilized because no single team fills a whole cluster efficiently.

### What a Namespace Costs You

Basically nothing. A namespace is a label and scope inside etcd. Creating one is instant, free, and does not touch infrastructure at all.

---

## Cluster vs Namespace

| | Cluster | Namespace |
|---|---|---|
| **Isolation strength** | Hard (separate control plane, node-level boundaries possible) | Soft (same control plane, same nodes, logical only) |
| **Cost** | High — real infrastructure | Near zero |
| **Shared resource pool** | No — capacity is siloed per cluster | Yes — pods from different namespaces share the same nodes |
| **Blast radius of a control-plane outage** | Contained to one cluster | Affects everyone on that cluster |
| **Setup time** | Minutes to provision infrastructure | Instant |

---

## When Namespaces Are the Right Call

- Multiple teams or apps that trust each other and belong to the same org. They want to share the efficiency of one node pool but still need name-collision avoidance, scoped RBAC, and resource quotas without paying for separate infrastructure.
- Environment separation within the same trust boundary, for example `dev` and `staging` on one cluster. (Production is usually still a separate cluster for blast-radius reasons.)
- You want centralized cluster-wide tooling (one Prometheus, one ingress controller, one cert-manager) that manages everything, rather than reinstalling it per team.

Namespaces give you:

- **Name collision avoidance:** a `frontend` Service in `team-a`'s namespace and a `frontend` Service in `team-b`'s namespace can coexist without conflict.
- **Scoped RBAC:** Team A can only touch objects in their namespace.
- **Resource quotas:** limit Team A to 4 CPUs and Team B to 8 CPUs from the shared pool.

## When You Actually Need a Separate Cluster

- Hard security or compliance boundaries, for example PCI workloads or genuinely untrusted tenants who should not share a kernel or a control plane.
- Different cloud regions or providers.
- You want to upgrade or tear down one environment (like a test cluster) without any risk to others.
- A team large enough to consume an entire cluster's capacity on its own. The shared pool benefit disappears at that point.

---

## Short Version

A cluster is the unit of **infrastructure isolation** (expensive, heavy, strong boundary).

A namespace is the unit of **organizational isolation** within infrastructure you are already sharing (free, light, soft boundary).

Reach for a namespace by default. Only spin up a new cluster when you need a boundary that a namespace genuinely cannot provide: security isolation, physical separation, or an independent lifecycle.

---

## How IP Addressing Works Across Namespaces

A natural follow-up question: can two pods in different namespaces have the same IP?

No. Within a single cluster, namespaces do not get their own IP range. Pod IPs and Service ClusterIPs are unique across the entire cluster regardless of namespace. But services/deployments/pods can definitely have the same name across different namespaces

### Why DNS Namespacing Exists Then

Since IPs are unique cluster-wide and not human-friendly, and two Services in different namespaces can share the same name (like `frontend` in both `team-a` and `team-b`), Kubernetes uses fully qualified domain names to disambiguate:

```
frontend.team-a.svc.cluster.local
frontend.team-b.svc.cluster.local
```

The name collision is what is scoped by namespace, not the IP.

### Where IPs Can Actually Repeat

- **Different clusters:** Cluster 1 and Cluster 2 can have overlapping pod CIDRs (both using `10.244.0.0/16` for example) since they are independent networks. This is a common gotcha when setting up cross-cluster networking or a service mesh later. Non-overlapping CIDRs need to be planned upfront if clusters ever need to connect.
- **NodePort / hostPort:** These use the node's IP with a port. Two pods in different namespaces could each request the same `hostPort`, but they would collide on the same node. Kubernetes prevents this by not scheduling two pods with the same `hostPort` onto the same node.

---
