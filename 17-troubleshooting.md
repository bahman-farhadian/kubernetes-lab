# 17. Troubleshooting

**Goal:** Collect known issues and fixes encountered while building this lab.

## Covers
- etcd quorum loss recovery
  - **Stacked:** `kubectl exec` into the etcd static pod
  - **External:** on the standalone `k8s-etcd-*` nodes — no `kubectl exec` available, work directly via `etcdctl`/`systemctl`
- Node stuck `NotReady`
- Ceph OSD/placement group issues
- kubeadm join/token failures
- LB/VIP failover issues on the bastion

## Prerequisites
- Populated as issues come up during deployment.

## Next
- GPU profile: [18-gpu-node.md](18-gpu-node.md)
- Everyone: [19-deployment-log.md](19-deployment-log.md) — record what you actually deployed
