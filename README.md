# kubernetes-lab

Build a production-like Kubernetes cluster from scratch for hands-on learning, covering high availability, networking, storage, security, and cluster administration.

## Scope

This repo documents cluster deployment on top of a set of already-provisioned VMs. **VM/host provisioning is out of scope** — bring your own VMs (manual install, Ansible, a cloud provider, or any other method), reachable over SSH with a base OS installed. The node count and roles you need are defined in this documentation.

## Layout

One numbered spine. Circle a profile × scenario in [01-inventory.md](01-inventory.md) and one distro (Debian 13 or Ubuntu 26), then walk 02 → 09. The only branch is inside [05-deploy-kubernetes.md](05-deploy-kubernetes.md) (stacked or external bootstrap). Upgrades are 11, then 12, after the smoke test. GPU adds [14-gpu.md](14-gpu.md) after a healthy Heavy cluster. Site WAN/LAN addresses stay out of this repo — tables use example LAN IPs.

| Doc | Covers |
|---|---|
| [00-overview.md](00-overview.md) | Scope, profiles, scenarios, version pins |
| [01-inventory.md](01-inventory.md) | VM tables and the LAN plan, including the API load-balancer pair |
| [02-prepare.md](02-prepare.md) | Prerequisites and OS baseline |
| [03-firewall.md](03-firewall.md) | Gateway keepalived pair |
| [04-bastion.md](04-bastion.md) | Docker Compose: Nexus, Prometheus, Grafana |
| [05-deploy-kubernetes.md](05-deploy-kubernetes.md) | containerd, API HAProxy pair, bootstrap, join, Calico, kubectl, Helm |
| [06-ceph.md](06-ceph.md) | Ceph deploy |
| [07-ingress.md](07-ingress.md) | Traefik, published on the API pair |
| [08-observability.md](08-observability.md) | node_exporter |
| [09-smoke-test.md](09-smoke-test.md) | Smoke and stress |
| [10-security.md](10-security.md) | Hardening outline |
| [11-update-kubernetes.md](11-update-kubernetes.md) | Kubernetes, Calico, and external-etcd upgrades |
| [12-update-ceph.md](12-update-ceph.md) | Ceph upgrade |
| [13-troubleshooting.md](13-troubleshooting.md) | Notes filled in as a run hits them |
| [14-gpu.md](14-gpu.md) | GPU profile only |
| [15-deployment-log.md](15-deployment-log.md) | What was actually pinned, per run |

```mermaid
flowchart TD
    Plan["00–01 plan"]:::common
    Plan --> Prep["02–04 prepare"]:::common
    Prep --> K8s["05 deploy Kubernetes"]:::common
    K8s --> Ceph["06 Ceph"]:::common
    Ceph --> Rest["07–09 ingress, metrics, smoke"]:::common
    Rest --> Up["11 update Kubernetes\n12 update Ceph"]:::common
    Up --> GPU["14 GPU"]:::common
    Up --> Log["15 deployment log"]:::common
    GPU --> Log

    classDef common fill:#57606a,stroke:#32383f,color:#ffffff
```

Same palette as everywhere else — see the [color legend](00-overview.md#diagram-color-legend).

## Deployment profiles

Three independent instances of this lab, never joined together (they reuse the same IP plan) — see [00-overview.md](00-overview.md#deployment-profiles). Pick the matching table in [01-inventory.md](01-inventory.md); the procedure files do not change.

| Profile | Host | Notes |
|---|---|---|
| **Light** | Laptop | Two firewalls, two API proxies, bastion (jump + kubectl/Helm + Nexus + metrics), smaller nodes. Deploy this one first. |
| **Heavy** | Server | Same shape as Light, bigger nodes. |
| **GPU** | Server | Heavy + one GPU worker (`k8s-work-4`), passed-through NVIDIA, tainted. |

## Rollout plan

1. **Light (laptop) — current task.** On **Debian 13 first**, walk 02 → 09, then [11-update-kubernetes.md](11-update-kubernetes.md), then [12-update-ceph.md](12-update-ceph.md). Ubuntu 26 is the same path later, not at the same time. One etcd scenario per pass (stacked section of [05-deploy-kubernetes.md](05-deploy-kubernetes.md), later the external section).
2. **Heavy (server)** — same spine, Heavy tables in [01-inventory.md](01-inventory.md).
3. **GPU (server)** — Heavy plus `k8s-work-4` (joined in 05, OSD in 06) and [14-gpu.md](14-gpu.md).

Record the exact pinned component versions used on each run in [15-deployment-log.md](15-deployment-log.md).

## Scenarios

Two control-plane/etcd topologies — trade-offs in [00-overview.md](00-overview.md#scenarios):

- **Stacked etcd:** 3 control-plane nodes, etcd co-located on each. Bootstrap section in [05-deploy-kubernetes.md](05-deploy-kubernetes.md).
- **External etcd:** 2 control-plane nodes + 3 dedicated etcd nodes. The other bootstrap section in the same file.

## Status

02–09, 11, and 12 have runnable procedure (commands, configs, pinned-version installs). [10-security.md](10-security.md) and [13-troubleshooting.md](13-troubleshooting.md) stay outlines until a real run fills them in. [14-gpu.md](14-gpu.md) is an outline pending the first GPU run. Versions marked "verify current" are not promises — check them against upstream before running.
