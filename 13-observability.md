# 13. Observability

**Goal:** Get metrics and logs flowing before relying on the cluster for anything.

**Approach:** host-level monitoring only, entirely outside the cluster (per [00-overview.md](00-overview.md)) — `node_exporter` on every node, Prometheus + Grafana on one monitoring host: the dedicated `k8s-monitor` VM on the **laptop**, or `k8s-bastion` itself on the **server** (no budget headroom for a separate VM there — see [02-hardware-inventory.md](02-hardware-inventory.md)). Steps below say `k8s-monitor`; substitute `k8s-bastion` if you're on the server. This deliberately does not cover in-cluster object metrics (kube-state-metrics) or logs — just CPU/memory/disk on every node, which is what was asked for; add `metrics-server` later if `kubectl top`/HPA is needed.

## Steps

**1. `node_exporter` on every node** — bastion, control-plane, etcd (Scenario B), workers, and `k8s-monitor` itself:
```sh
sudo apt update
sudo apt install -y prometheus-node-exporter
sudo apt-mark hold prometheus-node-exporter
sudo systemctl enable --now prometheus-node-exporter
```
Verify locally: `curl -s localhost:9100/metrics | head`.

**2. Prometheus on `k8s-monitor`:**
```sh
sudo apt install -y prometheus
sudo apt-mark hold prometheus
```
Add every node to the scrape config, `/etc/prometheus/prometheus.yml`:
```yaml
scrape_configs:
  - job_name: node
    static_configs:
      - targets:
          - 10.0.1.11:9100   # k8s-bastion
          - 10.0.1.12:9100   # k8s-ctrl-1
          - 10.0.1.13:9100   # k8s-ctrl-2
          - 10.0.1.14:9100   # k8s-ctrl-3 (Scenario A) / or k8s-etcd-1..3 (Scenario B)
          - 10.0.1.21:9100   # k8s-work-1
          - 10.0.1.22:9100   # k8s-work-2
          - 10.0.1.23:9100   # k8s-work-3
          - 10.0.1.31:9100   # k8s-monitor itself (laptop) — omit this line on the server; add 10.0.1.24:9100 (k8s-work-4) instead
```
```sh
sudo systemctl restart prometheus
sudo systemctl enable prometheus
```
Verify: `curl -s localhost:9090/api/v1/targets | grep health` — all `"up"`.

**3. Grafana on `k8s-monitor`** — not in Debian's main repo, add Grafana's own apt repo (check [grafana.com/docs](https://grafana.com/docs/grafana/latest/setup-grafana/installation/debian/) for the current Debian 13/Trixie instructions before adding):
```sh
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://apt.grafana.com/gpg.key | sudo gpg --dearmor -o /etc/apt/keyrings/grafana.gpg
echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" \
  | sudo tee /etc/apt/sources.list.d/grafana.list
sudo apt update
sudo apt install -y grafana
sudo apt-mark hold grafana
sudo systemctl enable --now grafana-server
```

**4. Wire up Grafana:** add Prometheus (`http://localhost:9090`) as a data source, then import a community "Node Exporter Full" dashboard (search its ID on [grafana.com/dashboards](https://grafana.com/dashboards) — verify it's still maintained before importing) for the CPU/memory/disk view across all nodes.

> "Storage" here is host-level disk usage via `node_exporter`'s filesystem collector, not Ceph cluster internals (pool usage, PG state, OSD latency). Ceph ships its own `prometheus` `mgr` module (`ceph mgr module enable prometheus`) if you want that scraped later — out of scope for this pass since it wasn't asked for.

## Applies to
Both scenarios.

## Prerequisites
- [12-ingress.md](12-ingress.md)

## Next
- [14-security-hardening.md](14-security-hardening.md)
