# 13. Observability

**Goal:** Get metrics and logs flowing before relying on the cluster for anything.

## Covers
- `metrics-server` for `kubectl top` / HPA
- Prometheus + Grafana (e.g. kube-prometheus-stack) for cluster and Ceph metrics
- Log aggregation approach (e.g. Loki, or a simpler baseline for this lab)
- Alerting basics (etcd quorum loss, disk pressure, node not ready)

## Applies to
Both scenarios.

## Prerequisites
- [12-ingress.md](12-ingress.md)

## Next
- [14-security-hardening.md](14-security-hardening.md)
