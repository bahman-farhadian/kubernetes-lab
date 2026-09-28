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

LAN addresses in these tables are **examples** for the procedure (`10.0.1.0/24`). Use your own numbering when you provision; do not commit site WAN/LAN addresses into this repo. Each firewall VM has a second NIC on WAN — WAN IPs stay off this table (see [03-network-plan.md](03-network-plan.md) / [06-firewall.md](06-firewall.md)).

### Scenario A — Stacked etcd

| # | VM Name | Role | LAN IP (example) | vCPU | RAM | Root Disk | Ceph OSD |
|---|---|---|---|---|---|---|---|
| 01 | k8s-fw-1 | Firewall (WAN + LAN) | 10.0.1.1 | 2 | 2 GB | 20 GB | - |
| 02 | k8s-fw-2 | Firewall (WAN + LAN) | 10.0.1.2 | 2 | 2 GB | 20 GB | - |
| 03 | k8s-bastion | Jump / HAProxy / Nexus / Prometheus / Grafana | 10.0.1.11 | 2 | 4 GB | 40 GB | - |
| 04 | k8s-ctrl-1 | Control Plane + etcd | 10.0.1.12 | 2 | 2 GB | 30 GB | - |
| 05 | k8s-ctrl-2 | Control Plane + etcd | 10.0.1.13 | 2 | 2 GB | 30 GB | - |
| 06 | k8s-ctrl-3 | Control Plane + etcd | 10.0.1.14 | 2 | 2 GB | 30 GB | - |
| 07 | k8s-work-1 | Worker / Storage | 10.0.1.21 | 4 | 4 GB | 20 GB | 40 GB |
| 08 | k8s-work-2 | Worker / Storage | 10.0.1.22 | 4 | 4 GB | 20 GB | 40 GB |
| 09 | k8s-work-3 | Worker / Storage | 10.0.1.23 | 4 | 4 GB | 20 GB | 40 GB |
| | **VM TOTALS (9 VMs)** | | | **24** | **26 GB** | **230 GB** | **120 GB** |
| | **HOST RESERVED** | | | **2** | **6 GB** | — | — |
| | **PC TOTALS** | | | **12** | **32 GB** | **1 TB** | — |

LAN VIP (keepalived, default gateway) is `10.0.1.254` in the examples — not a VM. Apiserver VIP `10.0.1.10` lives on the bastion ([09-load-balancer.md](09-load-balancer.md)).

- **RAM** is a hard cap on the laptop: VM total + host reserved = 32 GB exactly, no slack.
- **vCPU** is deliberately oversubscribed (24 vCPU allocated against a 12-vCPU, 50%-capped share) — safe for vCPU, unlike RAM, since the hypervisor time-slices cores that aren't all pegged at once.
- **Disk** is one physical pool on the laptop (unlike the server's split NVMe): Root (230 GB) + Ceph OSD (120 GB) = 350 GB used out of the 1 TB disk, leaving ~650 GB for the host OS, snapshots, etc.
- Jump, HAProxy, Nexus (apt/container cache), and Prometheus/Grafana share `k8s-bastion` — one VM instead of bastion + monitor. Nexus blob store is why the root disk is 40 GB ([07-nexus.md](07-nexus.md), [15-observability.md](15-observability.md)).

### Scenario B — External etcd

Control plane drops to 2 nodes (apiserver/scheduler/controller-manager only, no etcd); etcd moves to 3 dedicated VMs. Same reasoning as [01-scenarios.md](01-scenarios.md): apiserver is stateless so it doesn't need an odd/quorum count, only etcd does.

| # | VM Name | Role | LAN IP (example) | vCPU | RAM | Root Disk | Ceph OSD |
|---|---|---|---|---|---|---|---|
| 01 | k8s-fw-1 | Firewall (WAN + LAN) | 10.0.1.1 | 2 | 2 GB | 20 GB | - |
| 02 | k8s-fw-2 | Firewall (WAN + LAN) | 10.0.1.2 | 2 | 2 GB | 20 GB | - |
| 03 | k8s-bastion | Jump / HAProxy / Nexus / Prometheus / Grafana | 10.0.1.11 | 2 | 4 GB | 40 GB | - |
| 04 | k8s-ctrl-1 | Control Plane | 10.0.1.12 | 2 | 2 GB | 30 GB | - |
| 05 | k8s-ctrl-2 | Control Plane | 10.0.1.13 | 2 | 2 GB | 30 GB | - |
| 06 | k8s-etcd-1 | etcd | 10.0.1.15 | 1 | 2 GB | 20 GB | - |
| 07 | k8s-etcd-2 | etcd | 10.0.1.16 | 1 | 2 GB | 20 GB | - |
| 08 | k8s-etcd-3 | etcd | 10.0.1.17 | 1 | 2 GB | 20 GB | - |
| 09 | k8s-work-1 | Worker / Storage | 10.0.1.21 | 4 | 4 GB | 20 GB | 40 GB |
| 10 | k8s-work-2 | Worker / Storage | 10.0.1.22 | 4 | 4 GB | 20 GB | 40 GB |
| 11 | k8s-work-3 | Worker / Storage | 10.0.1.23 | 4 | 4 GB | 20 GB | 40 GB |
| | **VM TOTALS (11 VMs)** | | | **25** | **30 GB** | **260 GB** | **120 GB** |
| | **HOST RESERVED** | | | **2** | **2 GB** | — | — |
| | **PC TOTALS** | | | **12** | **32 GB** | **1 TB** | — |

Same reconciliation rules as Scenario A: RAM caps exactly at 32 GB (30 + 2), vCPU oversubscribes. Three extra etcd VMs is why Scenario B runs heavier than A despite dropping a control-plane node. 30 GB of VMs on a 32 GB laptop is tight — do not run this at the same time as anything else.

## Server — Heavy and GPU profiles

Host budget: 250 GB NVMe for root disks, 1 TB NVMe dedicated to Ceph OSDs — sized as given, no CPU-share cap. Heavy and GPU are the **same base cluster**; GPU is Heavy plus one extra VM (`k8s-work-4`) and one extra step, [20-gpu-node.md](20-gpu-node.md). Deploy Heavy, confirm it's healthy, then decide whether to add the GPU worker on top rather than bootstrapping GPU from scratch.

### Heavy — Scenario A (Stacked etcd)

| # | VM Name | Role | LAN IP (example) | vCPU | RAM | Root Disk (250G NVMe) | Ceph OSD (1T NVMe) |
|---|---|---|---|---|---|---|---|
| 01 | k8s-fw-1 | Firewall (WAN + LAN) | 10.0.1.1 | 2 | 2 GB | 20 GB | - |
| 02 | k8s-fw-2 | Firewall (WAN + LAN) | 10.0.1.2 | 2 | 2 GB | 20 GB | - |
| 03 | k8s-bastion | Jump / HAProxy / Nexus / Prometheus / Grafana | 10.0.1.11 | 2 | 4 GB | 40 GB | - |
| 04 | k8s-ctrl-1 | Control Plane + etcd | 10.0.1.12 | 2 | 8 GB | 30 GB | - |
| 05 | k8s-ctrl-2 | Control Plane + etcd | 10.0.1.13 | 2 | 8 GB | 30 GB | - |
| 06 | k8s-ctrl-3 | Control Plane + etcd | 10.0.1.14 | 2 | 8 GB | 30 GB | - |
| 07 | k8s-work-1 | Worker / Storage | 10.0.1.21 | 4 | 24 GB | 20 GB | 200 GB |
| 08 | k8s-work-2 | Worker / Storage | 10.0.1.22 | 4 | 24 GB | 20 GB | 200 GB |
| 09 | k8s-work-3 | Worker / Storage | 10.0.1.23 | 4 | 24 GB | 20 GB | 200 GB |
| | **VM TOTALS (9 VMs)** | | | **24** | **104 GB** | **230 GB** | **600 GB** |

The firewall pair spends the 4 vCPU that used to sit idle for GPU. `k8s-work-4` still fits in RAM/disk; GPU stacked oversubscribes **2 vCPU** on this box.

Same combined bastion as Light (jump + HAProxy + Nexus + Prometheus/Grafana). Root disk 40 GB is for the Nexus blob store. See [07-nexus.md](07-nexus.md) and [15-observability.md](15-observability.md).

### GPU — Scenario A (Heavy + GPU worker)

Everything in Heavy above, plus:

| # | VM Name | Role | LAN IP (example) | vCPU | RAM | Root Disk (250G NVMe) | Ceph OSD (1T NVMe) |
|---|---|---|---|---|---|---|---|
| 10 | k8s-work-4 | Worker / Storage / GPU (NVIDIA) | 10.0.1.24 | 2 | 24 GB | 20 GB | 200 GB |
| | **VM TOTALS (10 VMs)** | | | **26** | **128 GB** | **250 GB** | **800 GB** |
| | **HOST RESERVED** | | | — | — | — | **200 GB** |
| | **PC TOTALS** | | | **24** | **128 GB** | **250 GB** | **1 TB** |

RAM and disk still close; vCPU oversubscribes 26 vs 24. Bring Heavy up cleanly first.

`k8s-work-4` is a VM with a GPU passed straight through to it (PCI passthrough). That passthrough/IOMMU configuration happens at the hypervisor level and is out of scope here (see [00-overview.md](00-overview.md)) — this repo assumes the GPU is already visible inside the VM. Inside the cluster, the node is tainted to reserve it for GPU workloads only. In-guest driver, device-plugin, and taint setup: [20-gpu-node.md](20-gpu-node.md).

### Heavy/GPU — Scenario B (External etcd)

Sized against the **GPU** profile's full budget (bastion + 3 workers + the GPU worker, same as the GPU — Scenario A table above), since that's the box's full commitment — control plane drops to 2 nodes and etcd moves to 3 dedicated VMs carved out of what the third stacked control-plane node used to cost:

| # | VM Name | Role | LAN IP (example) | vCPU | RAM | Root Disk (250G NVMe) | Ceph OSD (1T NVMe) |
|---|---|---|---|---|---|---|---|
| 01 | k8s-fw-1 | Firewall (WAN + LAN) | 10.0.1.1 | 2 | 2 GB | 20 GB | - |
| 02 | k8s-fw-2 | Firewall (WAN + LAN) | 10.0.1.2 | 2 | 2 GB | 20 GB | - |
| 03 | k8s-bastion | Jump / HAProxy / Nexus / Prometheus / Grafana | 10.0.1.11 | 2 | 4 GB | 40 GB | - |
| 04 | k8s-ctrl-1 | Control Plane | 10.0.1.12 | 2 | 8 GB | 30 GB | - |
| 05 | k8s-ctrl-2 | Control Plane | 10.0.1.13 | 2 | 8 GB | 30 GB | - |
| 06 | k8s-etcd-1 | etcd | 10.0.1.15 | 1 | 2 GB | 10 GB | - |
| 07 | k8s-etcd-2 | etcd | 10.0.1.16 | 1 | 2 GB | 10 GB | - |
| 08 | k8s-etcd-3 | etcd | 10.0.1.17 | 1 | 2 GB | 10 GB | - |
| 09 | k8s-work-1 | Worker / Storage | 10.0.1.21 | 4 | 24 GB | 20 GB | 200 GB |
| 10 | k8s-work-2 | Worker / Storage | 10.0.1.22 | 4 | 24 GB | 20 GB | 200 GB |
| 11 | k8s-work-3 | Worker / Storage | 10.0.1.23 | 4 | 24 GB | 20 GB | 200 GB |
| 12 | k8s-work-4 | Worker / Storage / GPU (NVIDIA) | 10.0.1.24 | 2 | 24 GB | 20 GB | 200 GB |
| | **VM TOTALS (12 VMs)** | | | **27** | **126 GB** | **250 GB** | **800 GB** |
| | **HOST RESERVED** | | | — | **2 GB** | — | **200 GB** |
| | **PC TOTALS** | | | **24** | **128 GB** | **250 GB** | **1 TB** |

vCPU oversubscribes (27 vs 24); RAM/disk still close. Drop `k8s-work-4` for a Scenario-B "Heavy only" cluster.

etcd nodes are deliberately smaller than the control-plane nodes here (1 vCPU / 2 GB / 10 GB each): etcd is disk/network-sensitive more than CPU-hungry at this scale, and splitting it out is what let the 2 remaining control-plane nodes keep the same 2 vCPU / 8 GB as the stacked Scenario A version.

## Prerequisites
- [01-scenarios.md](01-scenarios.md)

## Next
- [03-network-plan.md](03-network-plan.md)
