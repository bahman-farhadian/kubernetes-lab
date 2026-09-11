# 17. Troubleshooting

**Goal:** Collect known issues and fixes encountered while building this lab.

## Covers
- etcd quorum loss recovery (on the standalone `k8s-etcd-*` nodes — no `kubectl exec` available, work directly via `etcdctl`/`systemctl`)
- Node stuck `NotReady`
- Ceph OSD/placement group issues
- kubeadm join/token failures
- LB/VIP failover issues on the bastion

## Applies to
Light profile, external etcd.

## Prerequisites
- Populated as issues come up during deployment.

## Next
- [18-deployment-log.md](../../18-deployment-log.md) — record what you actually deployed
