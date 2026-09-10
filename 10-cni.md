# 10. CNI (Pod Networking)

**Goal:** Install a CNI plugin so nodes go `Ready` and pods get networking.

## Covers
- CNI choice (e.g. Cilium or Calico) and why
- Pod CIDR alignment with `kubeadm init` flags from step 08 ([stacked](08-stacked-etcd-bootstrap.md)/[external](08-external-etcd-bootstrap.md))
- Verifying inter-node and inter-pod connectivity
- Network policy support (used later in [14-security-hardening.md](14-security-hardening.md))

## Applies to
Both scenarios.

## Prerequisites
- [09-join-nodes.md](09-join-nodes.md)

## Next
- [11-storage-ceph.md](11-storage-ceph.md)
