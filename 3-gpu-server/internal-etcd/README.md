# GPU — Internal (Stacked) etcd

**Status:** Not started. Per the [rollout plan](../../README.md#rollout-plan), this profile is built only after **Heavy** (`../../2-heavy-server/internal-etcd/`) is healthy — GPU is Heavy plus one extra worker (`k8s-work-4`) and one extra step, not a from-scratch build.

## How to build this out

1. Copy every file from `../../2-heavy-server/internal-etcd/` into this directory (once that one is real, not this stub).
2. Add `k8s-work-4` in step 09 (join) and step 11 (it also gets a Ceph OSD, per the **GPU — Scenario A** table in [02-hardware-inventory.md](../../02-hardware-inventory.md#gpu--scenario-a-heavy--gpu-worker)).
3. Add a new final step, `17-gpu-node.md`, covering: confirming GPU passthrough in the guest, NVIDIA driver + `nvidia-container-toolkit`, wiring into containerd, labeling + **tainting** `k8s-work-4` (`nvidia.com/gpu=present:NoSchedule`) so ordinary pods can't land there, and the NVIDIA Kubernetes device plugin DaemonSet (plain device plugin — no GPU Operator/MIG/time-slicing, per [00-overview.md](../../00-overview.md#deployment-profiles)). PCI passthrough/IOMMU itself is out of scope (hypervisor-level).
4. Delete this README once the real files are in place.

## Reference
- Profile/scenario definitions: [00-overview.md](../../00-overview.md#deployment-profiles), [01-scenarios.md](../../01-scenarios.md)
- Hardware: [02-hardware-inventory.md](../../02-hardware-inventory.md)
- Network: [03-network-plan.md](../../03-network-plan.md)
