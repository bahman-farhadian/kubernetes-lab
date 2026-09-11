# 02. Hardware Inventory

**Goal:** Record the VM inventory and resource budget for whichever host and scenario you're deploying to.

> VM creation itself is out of scope for this repo (see [00-overview.md](00-overview.md)). These tables describe what the VMs should look like once provisioned, regardless of how you provision them.

## Three independent lab environments

This repo covers three deployment profiles — **Light**, **Heavy**, **GPU** (named in [00-overview.md](00-overview.md#deployment-profiles)) — across two physical hosts, each running its own **independent** instance of the lab, never one cluster spanning hosts. All three reuse the same `10.0.1.0/24` addressing (see [03-network-plan.md](03-network-plan.md)), which only works because no two are ever online at the same time. Build/rebuild each profile's cluster on its own; don't try to join a Light node to a Heavy/GPU cluster or vice versa.

- **Light** (laptop) — capped at 50% CPU share, smaller footprint, the original scratch environment. **Deploy this one first.**
- **Heavy** (server) — dedicated hardware, larger nodes, same node shape as Light otherwise.
- **GPU** (server) — Heavy plus `k8s-work-4`, a worker with a passed-through NVIDIA GPU.

Rollout order and current status: [README.md](README.md#rollout-plan).

## Laptop — Light profile

Host budget: capped at 50% CPU share on the host. Adjust to your actual host's available cores/RAM/disk before sizing VMs.

### Scenario A — Stacked etcd

| # | VM Name | Role | IP Address | vCPU | RAM | Root Disk | Ceph OSD |
|---|---|---|---|---|---|---|---|
| 01 | k8s-bastion | Bastion / LB | 10.0.1.11 | 1 | 1 GB | 20 GB | - |
| 02 | k8s-ctrl-1 | Control Plane + etcd | 10.0.1.12 | 2 | 2 GB | 30 GB | - |
| 03 | k8s-ctrl-2 | Control Plane + etcd | 10.0.1.13 | 2 | 2 GB | 30 GB | - |
| 04 | k8s-ctrl-3 | Control Plane + etcd | 10.0.1.14 | 2 | 2 GB | 30 GB | - |
| 05 | k8s-work-1 | Worker / Storage | 10.0.1.21 | 4 | 4 GB | 20 GB | 40 GB |
| 06 | k8s-work-2 | Worker / Storage | 10.0.1.22 | 4 | 4 GB | 20 GB | 40 GB |
| 07 | k8s-work-3 | Worker / Storage | 10.0.1.23 | 4 | 4 GB | 20 GB | 40 GB |
| 08 | k8s-monitor | Prometheus + Grafana | 10.0.1.31 | 1 | 2 GB | 20 GB | - |
| | **VM TOTALS** | | | **20** | **21 GB** | **190 GB** | **120 GB** |
| | **HOST RESERVED** | | | **2** | **11 GB** | — | — |
| | **PC TOTALS** | | | **12** | **32 GB** | **1 TB** | — |

- **RAM** is a hard cap on the laptop: VM total + host reserved = 32 GB exactly, no slack.
- **vCPU** is deliberately oversubscribed (20 vCPU allocated against a 12-vCPU, 50%-capped share) — safe for vCPU, unlike RAM, since the hypervisor time-slices cores that aren't all pegged at once.
- **Disk** is one physical pool on the laptop (unlike the server's split NVMe): Root (190 GB) + Ceph OSD (120 GB) = 310 GB used out of the 1 TB disk, leaving ~690 GB for the host OS, snapshots, etc.
- `k8s-monitor` is new versus the original plan: Prometheus + Grafana run outside the cluster (see [00-overview.md](00-overview.md)), and putting them on the bastion would couple monitoring uptime to the LB/jump host. If you'd rather not spend a whole VM on it, fold it back into `k8s-bastion` and bump that VM to 2 vCPU / 3 GB instead — each directory's `13-observability.md` (e.g. [1-light-laptop/stacked-etcd/13-observability.md](1-light-laptop/stacked-etcd/13-observability.md)) doesn't care which host it lands on.

### Scenario B — External etcd

Control plane drops to 2 nodes (apiserver/scheduler/controller-manager only, no etcd); etcd moves to 3 dedicated VMs. Same reasoning as [01-scenarios.md](01-scenarios.md): apiserver is stateless so it doesn't need an odd/quorum count, only etcd does.

| # | VM Name | Role | IP Address | vCPU | RAM | Root Disk | Ceph OSD |
|---|---|---|---|---|---|---|---|
| 01 | k8s-bastion | Bastion / LB | 10.0.1.11 | 1 | 1 GB | 20 GB | - |
| 02 | k8s-ctrl-1 | Control Plane | 10.0.1.12 | 2 | 2 GB | 30 GB | - |
| 03 | k8s-ctrl-2 | Control Plane | 10.0.1.13 | 2 | 2 GB | 30 GB | - |
| 04 | k8s-etcd-1 | etcd | 10.0.1.15 | 1 | 2 GB | 20 GB | - |
| 05 | k8s-etcd-2 | etcd | 10.0.1.16 | 1 | 2 GB | 20 GB | - |
| 06 | k8s-etcd-3 | etcd | 10.0.1.17 | 1 | 2 GB | 20 GB | - |
| 07 | k8s-work-1 | Worker / Storage | 10.0.1.21 | 4 | 4 GB | 20 GB | 40 GB |
| 08 | k8s-work-2 | Worker / Storage | 10.0.1.22 | 4 | 4 GB | 20 GB | 40 GB |
| 09 | k8s-work-3 | Worker / Storage | 10.0.1.23 | 4 | 4 GB | 20 GB | 40 GB |
| 10 | k8s-monitor | Prometheus + Grafana | 10.0.1.31 | 1 | 2 GB | 20 GB | - |
| | **VM TOTALS** | | | **21** | **25 GB** | **220 GB** | **120 GB** |
| | **HOST RESERVED** | | | **2** | **7 GB** | — | — |
| | **PC TOTALS** | | | **12** | **32 GB** | **1 TB** | — |

Same reconciliation rules as Scenario A: RAM caps exactly at 32 GB (25 + 7), vCPU oversubscribes on purpose (21 allocated vs. the 12-vCPU share), and Root+OSD (220 + 120 = 340 GB) share the one 1 TB disk with ~660 GB left over. Three extra etcd VMs (each with their own small root disk) is why Scenario B's totals run a bit higher than Scenario A's despite dropping a control-plane node.

## Server — Heavy and GPU profiles

Host budget: 250 GB NVMe for root disks, 1 TB NVMe dedicated to Ceph OSDs — sized as given, no CPU-share cap. Heavy and GPU are the **same base cluster**; GPU is Heavy plus one extra VM (`k8s-work-4`) and one extra step, planned as `18-gpu-node.md` in each `3-gpu-server/` scenario directory (see [3-gpu-server/stacked-etcd/README.md](3-gpu-server/stacked-etcd/README.md)). Deploy Heavy, confirm it's healthy, then decide whether to add the GPU worker on top rather than bootstrapping GPU from scratch.

### Heavy — Scenario A (Stacked etcd)

| # | VM Name | Role | IP Address | vCPU | RAM | Root Disk (250G NVMe) | Ceph OSD (1T NVMe) |
|---|---|---|---|---|---|---|---|
| 01 | k8s-bastion | Bastion / LB / Monitoring | 10.0.1.11 | 2 | 4 GB | 20 GB | - |
| 02 | k8s-ctrl-1 | Control Plane + etcd | 10.0.1.12 | 2 | 8 GB | 30 GB | - |
| 03 | k8s-ctrl-2 | Control Plane + etcd | 10.0.1.13 | 2 | 8 GB | 30 GB | - |
| 04 | k8s-ctrl-3 | Control Plane + etcd | 10.0.1.14 | 2 | 8 GB | 30 GB | - |
| 05 | k8s-work-1 | Worker / Storage | 10.0.1.21 | 4 | 24 GB | 20 GB | 200 GB |
| 06 | k8s-work-2 | Worker / Storage | 10.0.1.22 | 4 | 24 GB | 20 GB | 200 GB |
| 07 | k8s-work-3 | Worker / Storage | 10.0.1.23 | 4 | 24 GB | 20 GB | 200 GB |
| | **VM TOTALS** | | | **20** | **100 GB** | **170 GB** | **600 GB** |

Leaves 4 vCPU / 28 GB / 80 GB root / 400 GB OSD of the box unused — that headroom is exactly `k8s-work-4`, reserved for the GPU profile below rather than spent here.

No separate `k8s-monitor` VM, unlike Light — Prometheus + Grafana run directly on `k8s-bastion` instead, sized generously enough (2 vCPU / 4 GB) to absorb it; see each directory's `13-observability.md`.

### GPU — Scenario A (Heavy + GPU worker)

Everything in Heavy above, plus:

| # | VM Name | Role | IP Address | vCPU | RAM | Root Disk (250G NVMe) | Ceph OSD (1T NVMe) |
|---|---|---|---|---|---|---|---|
| 08 | k8s-work-4 | Worker / Storage / GPU (NVIDIA) | 10.0.1.24 | 2 | 24 GB | 20 GB | 200 GB |
| | **VM TOTALS (Heavy + this row)** | | | **22** | **124 GB** | **190 GB** | **800 GB** |
| | **HOST RESERVED** | | | **2** | **4 GB** | **60 GB** | **200 GB** |
| | **PC TOTALS** | | | **24** | **128 GB** | **250 GB** | **1 TB** |

The GPU profile is the only one that fully commits the box's budget (VM totals + host reserved = PC totals, no slack) — one more reason to bring Heavy up cleanly first.

`k8s-work-4` is a VM with a GPU passed straight through to it (PCI passthrough). That passthrough/IOMMU configuration happens at the hypervisor level and is out of scope here (see [00-overview.md](00-overview.md)) — this repo assumes the GPU is already visible inside the VM. Inside the cluster, the node is tainted to reserve it for GPU workloads only. In-guest driver, device-plugin, and taint setup: planned as `18-gpu-node.md` in each `3-gpu-server/` scenario directory — see [3-gpu-server/stacked-etcd/README.md](3-gpu-server/stacked-etcd/README.md).

### Heavy/GPU — Scenario B (External etcd)

Sized against the **GPU** profile's full budget (bastion + 3 workers + the GPU worker, same as the GPU — Scenario A table above), since that's the box's full commitment — control plane drops to 2 nodes and etcd moves to 3 dedicated VMs carved out of what the third stacked control-plane node used to cost:

| # | VM Name | Role | IP Address | vCPU | RAM | Root Disk (250G NVMe) | Ceph OSD (1T NVMe) |
|---|---|---|---|---|---|---|---|
| 01 | k8s-bastion | Bastion / LB / Monitoring | 10.0.1.11 | 2 | 4 GB | 20 GB | - |
| 02 | k8s-ctrl-1 | Control Plane | 10.0.1.12 | 2 | 8 GB | 30 GB | - |
| 03 | k8s-ctrl-2 | Control Plane | 10.0.1.13 | 2 | 8 GB | 30 GB | - |
| 04 | k8s-etcd-1 | etcd | 10.0.1.15 | 1 | 2 GB | 10 GB | - |
| 05 | k8s-etcd-2 | etcd | 10.0.1.16 | 1 | 2 GB | 10 GB | - |
| 06 | k8s-etcd-3 | etcd | 10.0.1.17 | 1 | 2 GB | 10 GB | - |
| 07 | k8s-work-1 | Worker / Storage | 10.0.1.21 | 4 | 24 GB | 20 GB | 200 GB |
| 08 | k8s-work-2 | Worker / Storage | 10.0.1.22 | 4 | 24 GB | 20 GB | 200 GB |
| 09 | k8s-work-3 | Worker / Storage | 10.0.1.23 | 4 | 24 GB | 20 GB | 200 GB |
| 10 | k8s-work-4 | Worker / Storage / GPU (NVIDIA) | 10.0.1.24 | 2 | 24 GB | 20 GB | 200 GB |
| | **VM TOTALS** | | | **23** | **122 GB** | **190 GB** | **800 GB** |
| | **HOST RESERVED** | | | **1** | **6 GB** | **60 GB** | **200 GB** |
| | **PC TOTALS** | | | **24** | **128 GB** | **250 GB** | **1 TB** |

Every column reconciles exactly (VM totals + host reserved = PC totals) — no oversubscription anywhere, same as the GPU — Scenario A table. Drop row 10 (`k8s-work-4`) for a Scenario-B "Heavy only" cluster — that alone frees 2 vCPU / 24 GB / 20 GB root / 200 GB OSD, which you can add back to `k8s-bastion`/the control-plane nodes or just leave idle.

etcd nodes are deliberately smaller than the control-plane nodes here (1 vCPU / 2 GB / 10 GB each): etcd is disk/network-sensitive more than CPU-hungry at this scale, and splitting it out is what let the 2 remaining control-plane nodes keep the same 2 vCPU / 8 GB as the stacked Scenario A version.

## Prerequisites
- [01-scenarios.md](01-scenarios.md)

## Next
- [03-network-plan.md](03-network-plan.md)
