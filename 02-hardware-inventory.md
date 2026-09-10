# 02. Hardware Inventory

**Goal:** Record the VM inventory and resource budget for whichever host and scenario you're deploying to.

> VM creation itself is out of scope for this repo (see [00-overview.md](00-overview.md)). These tables describe what the VMs should look like once provisioned, regardless of how you provision them.

## Two independent lab environments

This repo covers two separate hosts, each running its own **independent** instance of the lab — not one cluster spanning both. Both reuse the same `10.0.1.0/24` addressing (see [03-network-plan.md](03-network-plan.md)), which only works because they're never online as the same cluster at the same time. Build/rebuild each host's cluster on its own; don't try to join a laptop node to a server cluster or vice versa.

- **Laptop** — capped at 50% CPU share, smaller footprint, the original scratch environment.
- **Server** — dedicated hardware, 250 GB NVMe for OS/root disks + 1 TB NVMe for Ceph OSDs, larger nodes, and one worker with a GPU.

## Laptop

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
| | **VM TOTALS** | | | **20** | **21 GB** | **210 GB** | **120 GB** |

> `k8s-monitor` is new versus the original plan: Prometheus + Grafana run outside the cluster (see [00-overview.md](00-overview.md)), and putting them on the bastion would couple monitoring uptime to the LB/jump host. If you'd rather not spend a whole VM on it, fold it back into `k8s-bastion` and bump that VM to 2 vCPU / 3 GB instead — [13-observability.md](13-observability.md) doesn't care which host it lands on.

### Scenario B — External etcd

> Draft — fill in vCPU/RAM/disk once you've decided how much to shave off the control-plane nodes to make room for the dedicated etcd VMs.

| # | VM Name | Role | IP Address | vCPU | RAM | Root Disk | Ceph OSD |
|---|---|---|---|---|---|---|---|
| 01 | k8s-bastion | Bastion / LB | 10.0.1.11 | 1 | 1 GB | 20 GB | - |
| 02 | k8s-ctrl-1 | Control Plane | 10.0.1.12 | TBD | TBD | TBD | - |
| 03 | k8s-ctrl-2 | Control Plane | 10.0.1.13 | TBD | TBD | TBD | - |
| 04 | k8s-etcd-1 | etcd | 10.0.1.15 | TBD | TBD | TBD | - |
| 05 | k8s-etcd-2 | etcd | 10.0.1.16 | TBD | TBD | TBD | - |
| 06 | k8s-etcd-3 | etcd | 10.0.1.17 | TBD | TBD | TBD | - |
| 07 | k8s-work-1 | Worker / Storage | 10.0.1.21 | 4 | 4 GB | 20 GB | 40 GB |
| 08 | k8s-work-2 | Worker / Storage | 10.0.1.22 | 4 | 4 GB | 20 GB | 40 GB |
| 09 | k8s-work-3 | Worker / Storage | 10.0.1.23 | 4 | 4 GB | 20 GB | 40 GB |
| 10 | k8s-monitor | Prometheus + Grafana | 10.0.1.31 | 1 | 2 GB | 20 GB | - |
| | **VM TOTALS** | | | TBD | TBD | TBD | 120 GB |

## Server

Host budget: 250 GB NVMe for root disks, 1 TB NVMe dedicated to Ceph OSDs — sized as given, no CPU-share cap.

### Scenario A — Stacked etcd

| # | VM Name | Role | IP Address | vCPU | RAM | Root Disk (250G NVMe) | Ceph OSD (1T NVMe) |
|---|---|---|---|---|---|---|---|
| 01 | k8s-bastion | Bastion / LB / Monitoring | 10.0.1.11 | 2 | 4 GB | 20 GB | - |
| 02 | k8s-ctrl-1 | Control Plane + etcd | 10.0.1.12 | 2 | 8 GB | 30 GB | - |
| 03 | k8s-ctrl-2 | Control Plane + etcd | 10.0.1.13 | 2 | 8 GB | 30 GB | - |
| 04 | k8s-ctrl-3 | Control Plane + etcd | 10.0.1.14 | 2 | 8 GB | 30 GB | - |
| 05 | k8s-work-1 | Worker / Storage | 10.0.1.21 | 4 | 24 GB | 20 GB | 200 GB |
| 06 | k8s-work-2 | Worker / Storage | 10.0.1.22 | 4 | 24 GB | 20 GB | 200 GB |
| 07 | k8s-work-3 | Worker / Storage | 10.0.1.23 | 4 | 24 GB | 20 GB | 200 GB |
| 08 | k8s-work-4 | Worker / Storage / GPU (NVIDIA) | 10.0.1.24 | 2 | 24 GB | 20 GB | 200 GB |
| | **VM TOTALS** | | | **22** | **124 GB** | **190 GB** | **800 GB** |
| | **HOST RESERVED** | | | **2** | **4 GB** | **60 GB** | **200 GB** |
| | **PC TOTALS** | | | **24** | **128 GB** | **250 GB** | **1 TB** |

> No separate `k8s-monitor` VM here, unlike the laptop plan — this host's budget is fully committed (VM totals + host reserved already equal PC totals, no slack). Prometheus + Grafana run directly on `k8s-bastion` instead, which is sized generously enough (2 vCPU / 4 GB) to absorb it; see [13-observability.md](13-observability.md).
>
> `k8s-work-4` is a VM with a GPU passed straight through to it (PCI passthrough). That passthrough/IOMMU configuration happens at the hypervisor level and is out of scope here (see [00-overview.md](00-overview.md)) — this repo assumes the GPU is already visible inside the VM. Inside the cluster, the node is tainted to reserve it for GPU workloads only. In-guest driver, device-plugin, and taint setup: [17-gpu-node.md](17-gpu-node.md).

### Scenario B — External etcd

Same pattern as the laptop's Scenario B: shave the control-plane nodes down to 2 and add 3 dedicated `k8s-etcd-*` VMs, carved out of the same budget above. Not sized yet — fill in if/when you need it.

## Prerequisites
- [01-scenarios.md](01-scenarios.md)

## Next
- [03-network-plan.md](03-network-plan.md)
