# 03. Network Plan

**Goal:** Define addressing, DNS, and connectivity assumptions used by every later step.

## Covers
- Subnet layout (matches the IPs in [02-hardware-inventory.md](02-hardware-inventory.md))
- Hostname / DNS resolution strategy (static `/etc/hosts` vs. a real DNS server)
- VIP for the apiserver load balancer (used in [07-load-balancer.md](07-load-balancer.md))
- Firewall/port requirements between bastion, control-plane, etcd, and worker nodes
- External connectivity (does the cluster need outbound internet for pulling images?)

## Network diagram

Generic path shared by both scenarios — the etcd tier is dedicated in Scenario B and absent (folded into control plane) in Scenario A; see [01-scenarios.md](01-scenarios.md) for the per-scenario topology.

```mermaid
flowchart LR
    Ext["External client"] --> Bastion["k8s-bastion\nVIP / LB"]:::bastion
    Bastion --> CP["Control plane\n(2-3 nodes)"]:::controlPlane
    CP -.-> Etcd["etcd\n(stacked or dedicated)"]:::etcd
    CP --> Work["Worker nodes\n(k8s-work-1..3)"]:::worker
    Work --> Storage["Ceph OSDs"]:::storage

    classDef bastion fill:#1f6feb,stroke:#0c2d6b,color:#ffffff
    classDef controlPlane fill:#8250df,stroke:#4b1f91,color:#ffffff
    classDef etcd fill:#d29922,stroke:#7d5c05,color:#1a1a1a
    classDef worker fill:#2da44e,stroke:#164c24,color:#ffffff
    classDef storage fill:#0d9488,stroke:#0f766e,color:#ffffff
```

## Prerequisites
- [02-hardware-inventory.md](02-hardware-inventory.md)

## Next
- [04-prerequisites.md](04-prerequisites.md)
