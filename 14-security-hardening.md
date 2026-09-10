# 14. Security Hardening

**Goal:** Move the cluster from "working" to "defensible."

## Covers
- RBAC review (avoid `cluster-admin` sprawl)
- Pod Security Admission (baseline/restricted namespaces)
- `NetworkPolicy` objects, enforced by Calico from [10-cni.md](10-cni.md) — e.g. default-deny per namespace, then explicit allows
- etcd encryption at rest (`EncryptionConfiguration` for Secrets)
- CIS Kubernetes Benchmark notes relevant to this lab
- Bastion SSH hardening (key-only auth, no root login)
- Confirm every held package from earlier steps (`apt-mark showhold` on each node) still matches [00-overview.md](00-overview.md)'s package hold policy

## Applies to
Both scenarios.

## Prerequisites
- [13-observability.md](13-observability.md)

## Next
- [15-day2-operations.md](15-day2-operations.md)
