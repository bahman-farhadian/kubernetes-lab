# 05. OS Baseline

**Goal:** Bring every VM (bastion, control-plane, etcd, workers) to a common, Kubernetes-ready OS state.

## Covers
- Hostname and `/etc/hosts` (or DNS) consistency across all nodes
- Disable swap
- Kernel modules (`overlay`, `br_netfilter`) and required `sysctl` settings
- Time synchronization (chrony/systemd-timesyncd)
- Disable/adjust firewall per [03-network-plan.md](03-network-plan.md)
- OS package updates and required base packages

## Applies to
All nodes, both scenarios.

## Prerequisites
- [04-prerequisites.md](04-prerequisites.md)

## Next
- [06-container-runtime.md](06-container-runtime.md)
