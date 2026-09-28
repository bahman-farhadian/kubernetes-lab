# 15. Security Hardening

**Goal:** Move the cluster from "working" to "defensible."

## Covers
- RBAC review (avoid `cluster-admin` sprawl)
- Pod Security Admission (baseline/restricted namespaces)
- `NetworkPolicy` objects, enforced by Calico from [10-cni.md](10-cni.md) — e.g. default-deny per namespace, then explicit allows
- etcd encryption at rest (`EncryptionConfiguration` for Secrets). External etcd already has transport TLS from [08-bootstrap-external.md](08-bootstrap-external.md); this is about encrypting *values* at rest on top of that
- CIS Kubernetes Benchmark notes relevant to this lab
- Bastion SSH hardening (key-only auth, no root login) — extend the same hardening to `k8s-etcd-*` when using external etcd, since they're standalone VMs outside kubelet's reach
- Confirm every held package from earlier steps (`apt-mark showhold` on each node, `k8s-etcd-*` included if present) still matches [00-overview.md](00-overview.md)'s package hold policy

## Prerequisites
- [14-smoke-test.md](14-smoke-test.md)

## Next
- [16-day2-operations.md](16-day2-operations.md)
