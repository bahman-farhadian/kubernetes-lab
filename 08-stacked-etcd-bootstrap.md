# 08. Bootstrap — Stacked etcd (Scenario A)

**Goal:** Initialize the HA control plane with `kubeadm`, etcd stacked on each control-plane node.

## Covers
- `kubeadm init` on `k8s-ctrl-1` with the apiserver VIP/LB endpoint from [07-load-balancer.md](07-load-balancer.md)
- Certificate distribution to `k8s-ctrl-2` / `k8s-ctrl-3`
- `kubeadm join --control-plane` on the remaining control-plane nodes
- Verifying etcd cluster health across all 3 members

## Bootstrap sequence

```mermaid
sequenceDiagram
    participant C1 as k8s-ctrl-1
    participant C2 as k8s-ctrl-2
    participant C3 as k8s-ctrl-3
    C1->>C1: kubeadm init (stacked etcd)
    C1->>C2: distribute certs
    C1->>C3: distribute certs
    C2->>C1: kubeadm join --control-plane
    C3->>C1: kubeadm join --control-plane
    Note over C1,C3: etcd quorum verified across all 3 members
```

```mermaid
flowchart TB
    C1["k8s-ctrl-1\nCP + etcd"]:::controlPlane
    C2["k8s-ctrl-2\nCP + etcd"]:::controlPlane
    C3["k8s-ctrl-3\nCP + etcd"]:::controlPlane
    C1 --- C2 --- C3 --- C1

    classDef controlPlane fill:#8250df,stroke:#4b1f91,color:#ffffff
    classDef etcd fill:#d29922,stroke:#7d5c05,color:#1a1a1a
```

## Applies to
Scenario A only.

## Prerequisites
- [07-load-balancer.md](07-load-balancer.md)
- [02-hardware-inventory.md](02-hardware-inventory.md) — Scenario A table

## Next
- [09-join-nodes.md](09-join-nodes.md)
