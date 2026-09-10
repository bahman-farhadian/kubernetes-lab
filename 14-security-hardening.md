# 14. Security Hardening

**Goal:** Move the cluster from "working" to "defensible."

## Covers
- RBAC review (avoid `cluster-admin` sprawl)
- Pod Security Admission (baseline/restricted namespaces)
- NetworkPolicies (built on CNI support from [10-cni.md](10-cni.md))
- etcd encryption at rest
- CIS Kubernetes Benchmark notes relevant to this lab
- Bastion SSH hardening (key-only auth, no root login)

## Applies to
Both scenarios.

## Prerequisites
- [13-observability.md](13-observability.md)

## Next
- [15-day2-operations.md](15-day2-operations.md)
