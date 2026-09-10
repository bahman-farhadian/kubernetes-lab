# 07. Bastion / Load Balancer

**Goal:** Stand up the apiserver-facing load balancer (and jump host) on `k8s-bastion` before bootstrapping the control plane.

## Covers
- HAProxy (or similar) frontend for the kube-apiserver VIP defined in [03-network-plan.md](03-network-plan.md)
- Health checks against control-plane nodes
- Optional: keepalived for VIP failover if the bastion itself becomes a target for HA
- Bastion's role as an SSH jump host into the private VM network

## Applies to
Both scenarios — the bastion always load-balances across the control-plane nodes (3 in Scenario A, 2 in Scenario B).

## Request path

```mermaid
flowchart LR
    Client["kubectl / clients"] --> VIP["VIP :6443"]:::bastion
    VIP --> HAP["HAProxy on k8s-bastion"]:::bastion
    HAP --> C1["k8s-ctrl-1"]:::controlPlane
    HAP --> C2["k8s-ctrl-2"]:::controlPlane
    HAP -.-> C3["k8s-ctrl-3\n(Scenario A only)"]:::controlPlane

    classDef bastion fill:#1f6feb,stroke:#0c2d6b,color:#ffffff
    classDef controlPlane fill:#8250df,stroke:#4b1f91,color:#ffffff
    classDef etcd fill:#d29922,stroke:#7d5c05,color:#1a1a1a
    classDef worker fill:#2da44e,stroke:#164c24,color:#ffffff
    classDef storage fill:#0d9488,stroke:#0f766e,color:#ffffff
```

## Prerequisites
- [06-container-runtime.md](06-container-runtime.md) (bastion itself doesn't need a container runtime, but control-plane targets must be reachable)

## Next
- Scenario A: [08-stacked-etcd-bootstrap.md](08-stacked-etcd-bootstrap.md)
- Scenario B: [08-external-etcd-bootstrap.md](08-external-etcd-bootstrap.md)
