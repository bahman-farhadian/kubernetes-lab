# 00. Overview

**Goal:** Define what this lab builds, what it deliberately leaves out, and how to read the rest of the docs.

## Covers
- Purpose: build a production-like, highly-available Kubernetes cluster for hands-on learning.
- Learning focus: HA control plane, networking, storage (Ceph), security, day-2 operations.
- Out of scope: VM/host provisioning. This repo assumes the VMs already exist (created manually, via Ansible, via a cloud provider, or any other method) and are reachable over SSH with a base OS installed.
- Two supported topologies, chosen up front in [01-scenarios.md](01-scenarios.md).
- Three deployment profiles (below), each its own independent instance of this lab, never joined together.
- Style: manual, step-by-step, real commands — short enough to read end to end and understand what's happening, so you can later drive the same cluster with kubeadm scripted, Kubespray, or any other automation with eyes open.

## Deployment profiles

Named so a run and its docs/logs can be referred to unambiguously — always say which profile, not just "the cluster":

| Profile | Host | Nodes |
|---|---|---|
| **Light** | Laptop | 2 firewalls, bastion, 3 control-plane, 3 workers, monitor |
| **Heavy** | Server | 2 firewalls, bastion, 3 control-plane, 3 workers — same shape as Light, bigger nodes, no separate monitor VM |
| **GPU** | Server | Heavy + `k8s-work-4` (NVIDIA, passed-through, tainted) |

One procedure spine (`04`–`18`) covers all three; circle the matching table in [02-hardware-inventory.md](02-hardware-inventory.md) and, at step 09, open [09-bootstrap-stacked.md](09-bootstrap-stacked.md) or [09-bootstrap-external.md](09-bootstrap-external.md). GPU adds [19-gpu-node.md](19-gpu-node.md). See [README.md](README.md#layout) and [01-scenarios.md](01-scenarios.md). **Deploy in this order: Light first (both scenarios), then Heavy, then GPU** — see [README.md](README.md#rollout-plan). Record the exact pinned versions used for each profile's run in [20-deployment-log.md](20-deployment-log.md) — the steps use version *variables* (`KUBE_DEPLOY_VERSION`, `CEPH_DEPLOY_VERSION`, …), and which concrete value you picked for a given profile/run is exactly the kind of thing that's easy to lose track of otherwise.

## Fixed decisions

These are settled for the whole manual — later steps assume them rather than re-justifying them:

| Area | Choice |
|---|---|
| OS | **Debian 13 ("Trixie") or Ubuntu 26 ("Resolute Raccoon")** on every VM of a given lab instance — do not mix distros inside one cluster. Apt commands are the same shape; where a repo URL or package name differs, the step says so. |
| Edge | Two-node keepalived firewall (`k8s-fw-1` / `k8s-fw-2`): WAN VIP + LAN VIP (default gateway). VRID is per instance. WAN numbering is site-local and is **not** recorded in this repo. The bastion HAProxy VIP is a separate LAN address — no keepalived on the bastion. |
| Bootstrap tool | `kubeadm` (not k3s/RKE2/Kubespray) |
| Container runtime | containerd only |
| CNI | Calico (NetworkPolicy support, needed in [16-security-hardening.md](16-security-hardening.md)) |
| Storage | Ceph, installed as native `apt` packages on the OS (not Rook) + Ceph-CSI inside the cluster |
| Ingress | Traefik (ingress-nginx is being sunset upstream) |
| Monitoring | node_exporter on every node + Prometheus/Grafana as native OS packages, outside the cluster so cluster problems don't take monitoring down with them (dedicated `k8s-monitor` VM on Light; folded into `k8s-bastion` on Heavy/GPU — budget-dependent, see [02-hardware-inventory.md](02-hardware-inventory.md)) |
| GPU (GPU profile only) | `k8s-work-4` is a VM with the GPU passed straight through to it (PCI passthrough); NVIDIA driver + container toolkit + plain Kubernetes device plugin inside the guest, no GPU Operator/MIG/time-slicing; node is tainted so only pods that explicitly tolerate it can be scheduled there — [19-gpu-node.md](19-gpu-node.md) |
| Container-count philosophy | Prefer a host-installed daemon over an in-cluster operator/pod wherever both exist (this is why Ceph and monitoring live outside Kubernetes) |

## Version pinning and the upgrade exercise

Every package this manual installs for the cluster to function (`containerd`, `kubelet`, `kubeadm`, `kubectl`, `haproxy`, `keepalived`, `ceph-*`, `etcd-*`, `prometheus*`, `grafana`, …) is installed at an **exact pinned version** (`apt install pkg=<version>`, never a bare `apt install pkg`) and `apt-mark hold`ed right after. A plain `apt upgrade`/`unattended-upgrades` run must never be able to silently bump a component that could break the cluster or change its behavior underneath you.

This pinning is deliberate for a second reason, not just safety: **Kubernetes, Ceph, and Calico are each deployed one version behind current stable** (steps 09/10 for Kubernetes, step 11 for Calico, step 12 for Ceph), specifically so there's a real version to upgrade *to*. [17-day2-operations.md](17-day2-operations.md) walks that cluster through the upgrade, component by component, using the same unhold → install exact new pinned version → verify → re-hold cycle for every node (Calico has no apt package/hold, but the same "deploy old, upgrade deliberately" shape applies via its operator manifest version). Staying current matters here, not just as an exercise: an untouched cluster silently ages out of its security-support window. That upgrade walkthrough is as much the point of this lab as the initial bootstrap is.

**Checked 2026-09** (Kubernetes has no LTS track — it ships a new minor roughly every 4 months and supports the 3 most recent; Ceph ships a new stable release roughly once a year, in support until the next-next one ships; Calico ships patch releases frequently within a minor):

| Component | Deploy version | Upgrade-to version | Source |
|---|---|---|---|
| Kubernetes | v1.36 (latest patch 1.36.4) | v1.37 (current stable, released 2026-08-26) | [kubernetes.io/releases](https://kubernetes.io/releases/) |
| Ceph | Squid v19.x (latest 19.2.6) — **EOL 2026-10-31**, don't linger on it | Tentacle v20.x (latest 20.2.4) | [docs.ceph.com/en/latest/releases](https://docs.ceph.com/en/latest/releases/) |
| Calico | v3.31.7 (one minor behind) | v3.32.2 (current stable) | [github.com/projectcalico/calico/releases](https://github.com/projectcalico/calico/releases) |

Re-check all three before you actually run the steps — this table is a snapshot, not a promise. Debian 13/Trixie's own `ceph-common` (`18.2.7+ds-1+deb13u1`, Reef) is already past Reef's upstream end of life, and Ceph's official [OS recommendations](https://docs.ceph.com/en/latest/start/os-recommendations/) rate Debian 13 tier "C" (packages exist, untested by the Ceph project) — [12-storage-ceph.md](12-storage-ceph.md) has the fallback plan if `download.ceph.com`'s Trixie repo doesn't cooperate.

## Diagram color legend

Every Mermaid diagram in this repo reuses the same two palettes (GitHub can't share Mermaid `classDef`s across files, so each diagram repeats the same hex values verbatim — keep them in sync if you change one).

**Role palette** (architecture/topology diagrams — bastion, control-plane, etcd, worker, storage):

| Role | Color |
|---|---|
| Bastion / LB | 🔵 `#1f6feb` |
| Control plane | 🟣 `#8250df` |
| etcd | 🟠 `#d29922` |
| Worker | 🟢 `#2da44e` |
| Storage (Ceph) | 🟦 `#0d9488` |

**Flow palette** (step-sequence diagrams — README.md pipeline):

| Meaning | Color |
|---|---|
| Common step (both scenarios) | ⚪ `#57606a` |
| Scenario A — stacked etcd | 🟣 `#8250df` (matches Control plane) |
| Scenario B — external etcd | 🟠 `#d29922` (matches etcd) |

The flow palette's scenario colors intentionally match the role palette: Scenario A is purple because its etcd is embedded in the (purple) control plane; Scenario B is amber because its etcd is the (amber) standalone tier. "Scenario A"/"Scenario B" and "stacked"/"external" are the same two things — diagrams may label nodes either way, but the colors don't change.

## Prerequisites
- None — start here.
