# 09. Join Worker Nodes

**Goal:** Join the worker nodes to the control plane bootstrapped in the previous step.

## Covers
- `kubeadm join` on `k8s-work-1/2/3`
- Verifying node `Ready` status (will stay `NotReady` until CNI is installed)
- Node labels/taints if any are needed for this lab

## Join flow

```mermaid
flowchart LR
    CP["Control plane\n(from step 08)"]:::controlPlane --> W1["k8s-work-1"]:::worker
    CP --> W2["k8s-work-2"]:::worker
    CP --> W3["k8s-work-3"]:::worker

    classDef controlPlane fill:#8250df,stroke:#4b1f91,color:#ffffff
    classDef worker fill:#2da44e,stroke:#164c24,color:#ffffff
```

## Applies to
Both scenarios.

## Prerequisites
- [08-stacked-etcd-bootstrap.md](08-stacked-etcd-bootstrap.md) or [08-external-etcd-bootstrap.md](08-external-etcd-bootstrap.md)

## Next
- [10-cni.md](10-cni.md)
