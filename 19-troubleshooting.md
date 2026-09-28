# 19. Troubleshooting

**Goal:** Collect known issues and fixes encountered while building this lab.

## Covers
- etcd quorum loss recovery
  - **Stacked:** `kubectl exec` into the etcd static pod
  - **External:** on the standalone `k8s-etcd-*` nodes — no `kubectl exec` available, work directly via `etcdctl`/`systemctl`
- Node stuck `NotReady`
- Ceph OSD/placement group issues
- kubeadm join/token failures
- LB/VIP issues on the bastion (apiserver `10.0.1.10`)
- Firewall keepalived WAN/LAN VIP failover (`k8s-fw-1` / `k8s-fw-2`)

## Prerequisites
- Populated as issues come up during deployment.

## Next
- GPU profile: [20-gpu-node.md](20-gpu-node.md)
- Everyone: [21-deployment-log.md](21-deployment-log.md) — record what you actually deployed
