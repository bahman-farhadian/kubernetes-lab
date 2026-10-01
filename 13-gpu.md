# 13. GPU node

**Goal:** After a healthy Heavy cluster, add `k8s-work-4` as a tainted NVIDIA worker. GPU is Heavy plus this step, not a from-scratch build.

**Status:** Outline — fill in with real commands on the first GPU run. PCI passthrough / IOMMU is hypervisor-level and out of scope (see [00-overview.md](00-overview.md)); this repo assumes the GPU is already visible inside the VM.

Do this only for the **GPU** profile, after [12-troubleshooting.md](12-troubleshooting.md) on a cluster that already matches the Heavy table in [01-inventory.md](01-inventory.md). `k8s-work-4` should already have been joined in [05-deploy-kubernetes.md](05-deploy-kubernetes.md) and given an OSD in [07-ceph.md](07-ceph.md).

## Covers
- Confirm the GPU is visible in the guest (`lspci`, `nvidia-smi` after the driver)
- NVIDIA driver + `nvidia-container-toolkit` inside the VM
- Wire the toolkit into containerd, restart containerd + kubelet
- Label + **taint** `k8s-work-4` (`nvidia.com/gpu=present:NoSchedule`) so ordinary pods cannot land there
- NVIDIA Kubernetes device plugin DaemonSet (plain device plugin — no GPU Operator / MIG / time-slicing, per [00-overview.md](00-overview.md#fixed-decisions))
- A one-pod smoke test that *tolerates* the taint and requests `nvidia.com/gpu: 1`

## Prerequisites
- Heavy-shaped cluster already through [10-smoke-test.md](10-smoke-test.md)
- [01-inventory.md](01-inventory.md) GPU table (Scenario A or B)

## Next
- [14-deployment-log.md](14-deployment-log.md)
