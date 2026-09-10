# 15. Day-2 Operations

**Goal:** Operate the cluster after initial bootstrap.

## Covers
- Control-plane and node upgrades (`kubeadm upgrade`)
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
