# 06. Container Runtime

**Goal:** Install and configure the container runtime on control-plane, etcd (if applicable), and worker nodes.

## Covers
- containerd installation
- `SystemdCgroup` and cgroup driver alignment with kubelet
- CRI socket configuration
- Registry/mirror configuration if needed

## Applies to
Control-plane and worker nodes, both scenarios. Not required on etcd-only nodes (Scenario B).

## Prerequisites
- [05-os-baseline.md](05-os-baseline.md)

## Next
- [07-load-balancer.md](07-load-balancer.md)
