# 08. Bootstrap — External etcd (Scenario B)

**Goal:** Stand up an independent etcd cluster, then initialize the control plane against it.

## Covers
- TLS certificate generation for the etcd cluster (etcd CA, peer, server, client certs)
- etcd cluster bootstrap across `k8s-etcd-1/2/3`
- Verifying etcd quorum before touching the control plane
- `kubeadm init` on `k8s-ctrl-1` with `--external-etcd-*` flags pointing at the etcd cluster
- `kubeadm join --control-plane` on `k8s-ctrl-2`

## Bootstrap sequence

```mermaid
sequenceDiagram
    participant E as k8s-etcd-1..3
    participant C1 as k8s-ctrl-1
    participant C2 as k8s-ctrl-2
    E->>E: bootstrap etcd cluster + TLS
    Note over E: quorum verified before touching control plane
    C1->>E: kubeadm init --external-etcd-*
    C2->>C1: kubeadm join --control-plane
```

```mermaid
flowchart TB
    C1["k8s-ctrl-1\nCP"]:::controlPlane
    C2["k8s-ctrl-2\nCP"]:::controlPlane
    E1["k8s-etcd-1"]:::etcd
    E2["k8s-etcd-2"]:::etcd
    E3["k8s-etcd-3"]:::etcd
    C1 & C2 --> E1 & E2 & E3
    E1 --- E2 --- E3 --- E1

    classDef controlPlane fill:#8250df,stroke:#4b1f91,color:#ffffff
    classDef etcd fill:#d29922,stroke:#7d5c05,color:#1a1a1a
```

## Applies to
Scenario B only.

## Prerequisites
- [07-load-balancer.md](07-load-balancer.md)
- [02-hardware-inventory.md](02-hardware-inventory.md) — Scenario B table

## Next
- [09-join-nodes.md](09-join-nodes.md)
