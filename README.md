# kubernetes-lab

Build a production-like Kubernetes cluster from scratch for hands-on learning, covering high availability, networking, storage, security, and cluster administration.

## Scope

This repo documents cluster deployment on top of a set of already-provisioned VMs. **VM/host provisioning is out of scope** — bring your own VMs (manual install, Ansible, a cloud provider, or any other method), reachable over SSH with a base OS installed. The node count and roles you need are defined in this documentation.

## Deployment profiles

Three independent instances of this lab, never joined together (they reuse the same IP plan) — see [00-overview.md](00-overview.md#deployment-profiles) for the full explanation:

| Profile | Host | Notes |
|---|---|---|
| **Light** | Laptop | Capped CPU share, smaller nodes. Deploy this one first. |
| **Heavy** | Server | Same shape as Light, bigger nodes. |
| **GPU** | Server | Heavy + one GPU worker (`k8s-work-4`), passed-through NVIDIA, tainted. |

All three follow the same deployment steps below; GPU adds step 17. Sizing for each: [02-hardware-inventory.md](02-hardware-inventory.md).

## Rollout plan

1. **Light (laptop) — current task.** Deploy first; it's the smallest, cheapest place to shake out mistakes before repeating the same steps on the server.
2. **Heavy (server)** — once Light is healthy end to end (through [15-day2-operations.md](15-day2-operations.md)'s upgrade exercise).
3. **GPU (server)** — add `k8s-work-4` and [17-gpu-node.md](17-gpu-node.md) on top of a healthy Heavy deployment, rather than bootstrapping GPU from scratch.

Record the exact pinned component versions used on each run in [18-deployment-log.md](18-deployment-log.md).

## Scenarios

Two supported control-plane/etcd topologies — pick one before you provision VMs. Details and trade-offs: [01-scenarios.md](01-scenarios.md).

- **Scenario A — Stacked etcd**: 3 control-plane nodes, etcd co-located on each.
- **Scenario B — External etcd**: 2 control-plane nodes + 3 dedicated etcd nodes.

## Deployment steps

Each step is its own file in the repo root, numbered in the order you follow them. Steps common to both scenarios carry only a number; step 08 is where the two paths fork, so it carries a `stacked-etcd`/`external-etcd` prefix — use whichever file matches the scenario you picked, then continue with the numbered steps.

| Step | Doc | Applies to |
|---|---|---|
| 00 | [Overview](00-overview.md) | Both |
| 01 | [Deployment Scenarios](01-scenarios.md) | Both |
| 02 | [Hardware Inventory](02-hardware-inventory.md) | Both |
| 03 | [Network Plan](03-network-plan.md) | Both |
| 04 | [Prerequisites](04-prerequisites.md) | Both |
| 05 | [OS Baseline](05-os-baseline.md) | Both |
| 06 | [Container Runtime](06-container-runtime.md) | Both |
| 07 | [Bastion / Load Balancer](07-load-balancer.md) | Both |
| 08 | [Bootstrap — Stacked etcd](08-stacked-etcd-bootstrap.md) | Scenario A |
| 08 | [Bootstrap — External etcd](08-external-etcd-bootstrap.md) | Scenario B |
| 09 | [Join Worker Nodes](09-join-nodes.md) | Both |
| 10 | [CNI](10-cni.md) | Both |
| 11 | [Storage — Ceph](11-storage-ceph.md) | Both |
| 12 | [Ingress](12-ingress.md) | Both |
| 13 | [Observability](13-observability.md) | Both |
| 14 | [Security Hardening](14-security-hardening.md) | Both |
| 15 | [Day-2 Operations](15-day2-operations.md) | Both |
| 16 | [Troubleshooting](16-troubleshooting.md) | Both |
| 17 | [GPU Worker (NVIDIA)](17-gpu-node.md) | GPU profile only |
| 18 | [Deployment Log](18-deployment-log.md) | All — appendix, filled in as you deploy |

```mermaid
flowchart TD
    S00["00 Overview"]:::common --> S01["01 Scenarios"]:::common
    S01 --> S02["02 Hardware Inventory"]:::common --> S03["03 Network Plan"]:::common
    S03 --> S04["04 Prerequisites"]:::common --> S05["05 OS Baseline"]:::common
    S05 --> S06["06 Container Runtime"]:::common --> S07["07 Bastion / LB"]:::common
    S07 --> S08A["08 Bootstrap\nStacked etcd"]:::scenarioA
    S07 --> S08B["08 Bootstrap\nExternal etcd"]:::scenarioB
    S08A --> S09["09 Join Worker Nodes"]:::common
    S08B --> S09
    S09 --> S10["10 CNI"]:::common --> S11["11 Storage (Ceph)"]:::common
    S11 --> S12["12 Ingress"]:::common --> S13["13 Observability"]:::common
    S13 --> S14["14 Security Hardening"]:::common --> S15["15 Day-2 Operations"]:::common
    S15 --> S16["16 Troubleshooting"]:::common
    S16 -. GPU profile only .-> S17["17 GPU Worker\n(NVIDIA)"]:::common

    classDef common fill:#57606a,stroke:#32383f,color:#ffffff
    classDef scenarioA fill:#8250df,stroke:#4b1f91,color:#ffffff
    classDef scenarioB fill:#d29922,stroke:#7d5c05,color:#1a1a1a
```

Purple = Scenario A (stacked etcd), amber = Scenario B (external etcd), gray = shared steps. See the full [color legend](00-overview.md#diagram-color-legend).

## Status

Steps 00–15 and 17 have real, runnable procedure (commands, configs, pinned-version installs). [16-troubleshooting.md](16-troubleshooting.md) stays an outline until issues actually come up during a run-through. Versions/URLs marked "verify current" throughout are deliberately not hardcoded — check them against upstream before running, don't trust them as pinned. Actual deployment progress is tracked in [Rollout plan](#rollout-plan) above, not here.
