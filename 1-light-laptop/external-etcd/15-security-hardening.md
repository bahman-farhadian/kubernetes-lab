# 15. Security Hardening

**Goal:** Move the cluster from "working" to "defensible."

## Covers
- RBAC review (avoid `cluster-admin` sprawl)
- Pod Security Admission (baseline/restricted namespaces)
- `NetworkPolicy` objects, enforced by Calico from [10-cni.md](10-cni.md) — e.g. default-deny per namespace, then explicit allows
- etcd encryption at rest (`EncryptionConfiguration` for Secrets) — the apiserver reaches the external etcd tier over TLS already set up in [08-bootstrap.md](08-bootstrap.md); this is about encrypting *values* at rest, on top of that transport security
- CIS Kubernetes Benchmark notes relevant to this lab
- Bastion SSH hardening (key-only auth, no root login) — extend the same hardening to the `k8s-etcd-*` nodes, since they're standalone VMs outside kubelet's reach
- Confirm every held package from earlier steps (`apt-mark showhold` on each node, `k8s-etcd-*` included) still matches [00-overview.md](../../00-overview.md)'s package hold policy

## Applies to
Light profile, external etcd.

## Prerequisites
- [14-smoke-test.md](14-smoke-test.md)

## Next
- [16-day2-operations.md](16-day2-operations.md)
