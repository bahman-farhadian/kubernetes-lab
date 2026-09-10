# 02. Hardware Inventory

**Goal:** Record the VM inventory and resource budget for whichever scenario you picked in [01-scenarios.md](01-scenarios.md).

> VM creation itself is out of scope for this repo (see [00-overview.md](00-overview.md)). These tables describe what the VMs should look like once provisioned, regardless of how you provision them.

## Host budget (example: laptop, capped at 50% CPU share for the host)

Adjust to your actual host's available cores/RAM/disk before sizing VMs.

## Scenario A — Stacked etcd

| # | VM Name | Role | IP Address | vCPU | RAM | Root Disk | Ceph OSD |
|---|---|---|---|---|---|---|---|
| 01 | k8s-bastion | Bastion / LB | 10.0.1.11 | 1 | 1 GB | 20 GB | - |
| 02 | k8s-ctrl-1 | Control Plane + etcd | 10.0.1.12 | 2 | 2 GB | 30 GB | - |
| 03 | k8s-ctrl-2 | Control Plane + etcd | 10.0.1.13 | 2 | 2 GB | 30 GB | - |
| 04 | k8s-ctrl-3 | Control Plane + etcd | 10.0.1.14 | 2 | 2 GB | 30 GB | - |
| 05 | k8s-work-1 | Worker / Storage | 10.0.1.21 | 4 | 4 GB | 20 GB | 40 GB |
| 06 | k8s-work-2 | Worker / Storage | 10.0.1.22 | 4 | 4 GB | 20 GB | 40 GB |
| 07 | k8s-work-3 | Worker / Storage | 10.0.1.23 | 4 | 4 GB | 20 GB | 40 GB |
| | **VM TOTALS** | | | **19** | **19 GB** | **190 GB** | **120 GB** |

## Scenario B — External etcd

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
| | **VM TOTALS** | | | TBD | TBD | TBD | 120 GB |

## Prerequisites
- [01-scenarios.md](01-scenarios.md)

## Next
- [03-network-plan.md](03-network-plan.md)
