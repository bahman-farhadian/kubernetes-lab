# 01. Deployment Scenarios

**Goal:** Pick a control-plane/etcd topology before provisioning VMs, since it changes the node count and the bootstrap steps you'll follow later.

Each scenario has its own directory under every profile — `stacked-etcd/` for Scenario A, `external-etcd/` for Scenario B (see [README.md](README.md#layout)). The two names are interchangeable throughout this repo.

## Scenario A — Stacked etcd (`stacked-etcd/`)
- etcd runs co-located on each control-plane node (standard `kubeadm` HA topology).
- 3 control-plane nodes (odd count, required for etcd quorum).
- Simpler: fewer VMs, one bootstrap path, etcd and apiserver fail together per node.
- Bootstrap doc: `08-bootstrap.md` in whichever profile's `stacked-etcd/` directory you're deploying, e.g. [1-light-laptop/stacked-etcd/08-bootstrap.md](1-light-laptop/stacked-etcd/08-bootstrap.md).

## Scenario B — External (dedicated) etcd (`external-etcd/`)
- etcd runs on its own VMs, independent of the control-plane nodes.
- 3 dedicated etcd nodes (odd count, still required for quorum) + 2 control-plane nodes (apiserver is stateless, so it doesn't need quorum and can run on fewer nodes).
- More VMs and moving parts, but control-plane and etcd fail independently, and each can be scaled/replaced on its own.
- Bootstrap doc: `08-bootstrap.md` in whichever profile's `external-etcd/` directory you're deploying, e.g. [1-light-laptop/external-etcd/08-bootstrap.md](1-light-laptop/external-etcd/08-bootstrap.md).

## Topology diagrams

```mermaid
flowchart TB
    subgraph A["Scenario A — Stacked etcd"]
        direction TB
        AB["k8s-bastion\nLB"]:::bastion
        AC1["k8s-ctrl-1\nCP + etcd"]:::controlPlane
        AC2["k8s-ctrl-2\nCP + etcd"]:::controlPlane
        AC3["k8s-ctrl-3\nCP + etcd"]:::controlPlane
        AW["k8s-work-1..3\nWorker + OSD"]:::worker
        AB --> AC1 & AC2 & AC3
        AC1 --- AC2 --- AC3
        AC1 & AC2 & AC3 --> AW
    end

    classDef bastion fill:#1f6feb,stroke:#0c2d6b,color:#ffffff
    classDef controlPlane fill:#8250df,stroke:#4b1f91,color:#ffffff
    classDef etcd fill:#d29922,stroke:#7d5c05,color:#1a1a1a
    classDef worker fill:#2da44e,stroke:#164c24,color:#ffffff
    classDef storage fill:#0d9488,stroke:#0f766e,color:#ffffff
```

```mermaid
flowchart TB
    subgraph B["Scenario B — External etcd"]
        direction TB
        BB["k8s-bastion\nLB"]:::bastion
        BC1["k8s-ctrl-1\nCP"]:::controlPlane
        BC2["k8s-ctrl-2\nCP"]:::controlPlane
        BE1["k8s-etcd-1"]:::etcd
        BE2["k8s-etcd-2"]:::etcd
        BE3["k8s-etcd-3"]:::etcd
        BW["k8s-work-1..3\nWorker + OSD"]:::worker
        BB --> BC1 & BC2
        BC1 & BC2 --> BE1 & BE2 & BE3
        BE1 --- BE2 --- BE3
        BC1 & BC2 --> BW
    end

    classDef bastion fill:#1f6feb,stroke:#0c2d6b,color:#ffffff
    classDef controlPlane fill:#8250df,stroke:#4b1f91,color:#ffffff
    classDef etcd fill:#d29922,stroke:#7d5c05,color:#1a1a1a
    classDef worker fill:#2da44e,stroke:#164c24,color:#ffffff
    classDef storage fill:#0d9488,stroke:#0f766e,color:#ffffff
```

Color key: 🔵 bastion/LB, 🟣 control plane, 🟠 etcd, 🟢 worker. Same palette as every other diagram in this repo — see [00-overview.md](00-overview.md#diagram-color-legend).

## Trade-off summary

| | Scenario A (stacked) | Scenario B (external etcd) |
|---|---|---|
| Control-plane VMs | 3 | 2 |
| Dedicated etcd VMs | 0 | 3 |
| Total control-plane + etcd VMs | 3 | 5 |
| Failure isolation (etcd vs apiserver) | Coupled | Independent |
| Operational complexity | Lower | Higher |

## Prerequisites
- [00-overview.md](00-overview.md)

## Next
- Node inventory and resource budget for both scenarios: [02-hardware-inventory.md](02-hardware-inventory.md)
