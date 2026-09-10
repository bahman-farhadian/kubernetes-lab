# 11. Storage — Ceph (Rook)

**Goal:** Deploy Rook-Ceph using the raw OSD disks attached to the worker nodes.

## Covers
- Rook operator installation
- CephCluster CR using the OSD devices/disks defined in [02-hardware-inventory.md](02-hardware-inventory.md)
- StorageClass creation (block/RBD, and CephFS if needed)
- Verifying Ceph cluster health (`ceph -s` via toolbox pod)

## Ceph cluster layout

```mermaid
flowchart TB
    subgraph Ceph["Rook-Ceph cluster"]
        direction LR
        W1["k8s-work-1"]:::worker --> O1["OSD 40GB"]:::storage
        W2["k8s-work-2"]:::worker --> O2["OSD 40GB"]:::storage
        W3["k8s-work-3"]:::worker --> O3["OSD 40GB"]:::storage
    end

    classDef worker fill:#2da44e,stroke:#164c24,color:#ffffff
    classDef storage fill:#0d9488,stroke:#0f766e,color:#ffffff
```

## Applies to
Both scenarios (storage layout is identical — only control-plane/etcd topology differs between scenarios).

## Prerequisites
- [10-cni.md](10-cni.md)

## Next
- [12-ingress.md](12-ingress.md)
