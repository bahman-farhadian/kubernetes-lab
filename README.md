# kubernetes-lab

Build a production-like Kubernetes cluster from scratch for hands-on learning, covering high availability, networking, storage, security, and cluster administration.

## Scope

This repo documents cluster deployment on top of a set of already-provisioned VMs. **VM/host provisioning is out of scope** — bring your own VMs (manual install, Ansible, a cloud provider, or any other method), reachable over SSH with a base OS installed. The node count and roles you need are defined in this documentation.

## Layout

One numbered spine at the repo root. Circle a profile × scenario in [02-hardware-inventory.md](02-hardware-inventory.md) and one distro (Debian 13 or Ubuntu 26), then walk 04 → 19. The only real branch is step 10 (two bootstrap files). GPU adds [20-gpu-node.md](20-gpu-node.md) after a healthy Heavy cluster. Site WAN/LAN addresses stay out of this repo — tables use example LAN IPs.

| Doc | Covers |
|---|---|
| [00-overview.md](00-overview.md) | Scope, profiles, Debian 13 **or** Ubuntu 26, firewall pair, version-pinning, diagram color legend |
| [01-scenarios.md](01-scenarios.md) | Stacked vs. external etcd trade-offs |
| [02-hardware-inventory.md](02-hardware-inventory.md) | VM sizing for every profile × scenario combination |
| [03-network-plan.md](03-network-plan.md) | LAN examples, firewall edge, ports, DNS |
| [04-prerequisites.md](04-prerequisites.md) … [19-troubleshooting.md](19-troubleshooting.md) | Procedure (same commands for Light/Heavy/GPU; profile notes where the node list differs) |
| [06-firewall.md](06-firewall.md) | keepalived WAN + LAN VIP (`k8s-fw-1` / `k8s-fw-2`) |
| [07-nexus.md](07-nexus.md) | Bastion Docker Engine + Compose (Nexus, Prometheus, Grafana) |
| [10-bootstrap-stacked.md](10-bootstrap-stacked.md) / [10-bootstrap-external.md](10-bootstrap-external.md) | The scenario fork — open **one** |
| [20-gpu-node.md](20-gpu-node.md) | GPU profile only: NVIDIA driver, toolkit, taint, device plugin |
| [21-deployment-log.md](21-deployment-log.md) | Template for recording exactly what got deployed, when |

```mermaid
flowchart TD
    Root["00–03 shared"]:::common
    Root --> P04["04–09"]:::common
    P04 --> S10["10-bootstrap-stacked"]:::scenarioA
    P04 --> E10["10-bootstrap-external"]:::scenarioB
    S10 --> Rest["11–19"]:::common
    E10 --> Rest
    Rest --> GPU["20-gpu-node\nGPU profile only"]:::common
    Rest --> Log["21-deployment-log"]:::common
    GPU --> Log

    classDef common fill:#57606a,stroke:#32383f,color:#ffffff
    classDef scenarioA fill:#8250df,stroke:#4b1f91,color:#ffffff
    classDef scenarioB fill:#d29922,stroke:#7d5c05,color:#1a1a1a
```

Purple = stacked etcd, amber = external etcd, gray = shared. Same palette as everywhere else — see the full [color legend](00-overview.md#diagram-color-legend).

## Deployment profiles

Three independent instances of this lab, never joined together (they reuse the same IP plan) — see [00-overview.md](00-overview.md#deployment-profiles). Pick the matching table in [02-hardware-inventory.md](02-hardware-inventory.md); the procedure files do not change.

| Profile | Host | Notes |
|---|---|---|
| **Light** | Laptop | Two firewalls, combined bastion (jump + HAProxy + Nexus + metrics), smaller nodes. Deploy this one first. |
| **Heavy** | Server | Same shape as Light, bigger nodes. |
| **GPU** | Server | Heavy + one GPU worker (`k8s-work-4`), passed-through NVIDIA, tainted. |

## Rollout plan

1. **Light (laptop) — current task.** Walk 04–19 twice, once per etcd scenario (open [10-bootstrap-stacked.md](10-bootstrap-stacked.md) then [10-bootstrap-external.md](10-bootstrap-external.md)), including the upgrade exercise in [18-day2-operations.md](18-day2-operations.md). One distro per instance (Debian 13 **or** Ubuntu 26).
2. **Heavy (server)** — same spine, Heavy tables in [02-hardware-inventory.md](02-hardware-inventory.md).
3. **GPU (server)** — Heavy plus `k8s-work-4` (joined in 11, OSD in 13) and [20-gpu-node.md](20-gpu-node.md).

Record the exact pinned component versions used on each run in [21-deployment-log.md](21-deployment-log.md).

## Scenarios

Two supported control-plane/etcd topologies — trade-offs in [01-scenarios.md](01-scenarios.md):

- **Stacked etcd:** 3 control-plane nodes, etcd co-located on each — [10-bootstrap-stacked.md](10-bootstrap-stacked.md).
- **External etcd:** 2 control-plane nodes + 3 dedicated etcd nodes — [10-bootstrap-external.md](10-bootstrap-external.md).

## Status

Steps 04–16 and 18 have real, runnable procedure (commands, configs, pinned-version installs). [17-security-hardening.md](17-security-hardening.md) and [19-troubleshooting.md](19-troubleshooting.md) stay outlines until a real run fills them in. [20-gpu-node.md](20-gpu-node.md) is an outline pending the first GPU run. Versions/URLs marked "verify current" throughout are deliberately not hardcoded — check them against upstream before running, don't trust them as pinned.
