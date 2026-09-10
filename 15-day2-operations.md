# 15. Day-2 Operations

**Goal:** Operate the cluster after initial bootstrap.

## Covers
- **Upgrading held packages** — every package this manual held (`containerd`, `kubelet`/`kubeadm`/`kubectl`, `haproxy`, `ceph-*`, `prometheus*`, `grafana`) is upgraded deliberately, one node at a time: `sudo apt-mark unhold <pkg>` → drain/cordon if it's a k8s node → `apt install <pkg>=<version>` → verify healthy → `apt-mark hold <pkg>` again. Never a blanket `apt upgrade`.
- Control-plane and node upgrades (`kubeadm upgrade`), respecting the kubelet/kubeadm skew policy
- etcd backup and restore (member list differs by scenario — see [stacked](08-stacked-etcd-bootstrap.md)/[external](08-external-etcd-bootstrap.md))
- Adding/removing control-plane, etcd, and worker nodes
- Draining and cordoning nodes for maintenance
- Certificate rotation

## Applies to
Both scenarios (backup/restore procedure differs slightly by topology).

## Prerequisites
- [14-security-hardening.md](14-security-hardening.md)

## Next
- [16-troubleshooting.md](16-troubleshooting.md)
