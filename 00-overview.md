# 00. Overview

**Goal:** Define what this lab builds, what it deliberately leaves out, and how to read the rest of the docs.

## Covers
- Purpose: build a production-like, highly-available Kubernetes cluster for hands-on learning.
- Learning focus: HA control plane, networking, storage (Ceph), security, day-2 operations.
- Out of scope: VM/host provisioning. This repo assumes the VMs already exist (created manually, via Ansible, via a cloud provider, or any other method) and are reachable over SSH with a base OS installed.
- Two supported topologies, chosen up front in [Scenarios](#scenarios).
- Three deployment profiles (below), each its own independent instance of this lab, never joined together.
- Style: manual, step-by-step, real commands — short enough to read end to end and understand what's happening, so you can later drive the same cluster with kubeadm scripted, Kubespray, or any other automation with eyes open.

## Deployment profiles

Named so a run and its docs/logs can be referred to unambiguously — always say which profile, not just "the cluster":

| Profile | Host | Nodes |
|---|---|---|
| **Stacked** | This host | 2 firewalls, 2 API load balancers, 1 bastion, 3 control-plane, 3 workers. Deploy this first. Guest RAM 22 GB. |
| **Stacked + GPU** | This host | Stacked plus `k8s-work-4` at 2 GB. Guest RAM 24 GB. |
| **External etcd** | This host | 2 control-plane nodes and 3 etcd VMs. Guest RAM 23 GB, or 25 GB with the GPU worker. |

Circle the matching table in [01-inventory.md](01-inventory.md), then walk [02-prepare.md](02-prepare.md) through [09-smoke-test.md](09-smoke-test.md) in number order. Inside [05-deploy-kubernetes.md](05-deploy-kubernetes.md), open one bootstrap (stacked or external). Upgrades come after that stack is up: [11-update-kubernetes.md](11-update-kubernetes.md), then [12-update-ceph.md](12-update-ceph.md). See [README.md](README.md#layout). **Deploy stacked first, then external etcd, one distro at a time.** The host is 12 logical CPUs and 31 GB RAM. Guest RAM stays at or under 26 GB, including stacked with the GPU worker (24 GB). The old ~128 GB GPU plan is not used. See [README.md](README.md#rollout-plan). Record the exact pinned versions used for each profile's run in [15-deployment-log.md](15-deployment-log.md) — the steps use version *variables* (`KUBE_DEPLOY_VERSION`, `CEPH_DEPLOY_VERSION`, …), and which concrete value you picked for a given profile/run is exactly the kind of thing that's easy to lose track of otherwise.

## Fixed decisions

These are settled for the whole manual — later steps assume them rather than re-justifying them:

| Area | Choice |
|---|---|
| OS | **Debian 13 ("Trixie") or Ubuntu 26 ("Resolute Raccoon")** on every VM of a given lab instance — do not mix distros inside one cluster. Work **one distro at a time** (Debian first, then Ubuntu). Apt commands are the same shape; where a repo URL or package name differs, the step says so. |
| Edge | Three separate roles, same as an on-prem production cluster. Firewalls `k8s-fw-1` / `k8s-fw-2`: keepalived, WAN VIP + LAN gateway VIP. They do **not** run HAProxy. Load balancers `k8s-lb-1` / `k8s-lb-2`: keepalived + HAProxy, their own VRID, floating VIP for `:6443` and later `:80`/`:443`. Bastion: one admin VM (jump, kubectl, Helm, Nexus, Prometheus, Grafana). It is not in the request path and it does **not** run HAProxy. WAN numbering stays out of this repo. |
| Bootstrap tool | `kubeadm` (not k3s/RKE2/Kubespray) |
| Container runtime | **containerd** on every Kubernetes node. **Docker Engine** exists only on `k8s-bastion`, as a single-node daemon for the Compose support stack. Do not install Docker on ctrl/workers. |
| CNI | Calico (NetworkPolicy support, needed in [10-security.md](10-security.md)) |
| Storage | Ceph, installed as native `apt` packages on the OS (not Rook) + Ceph-CSI inside the cluster |
| Ingress | Traefik (ingress-nginx is being sunset upstream) |
| Bastion support plane | Nexus + Prometheus + Grafana as **Docker Compose** on `k8s-bastion` ([04-bastion.md](04-bastion.md)). `kubectl` and Helm 3 are installed there during [05-deploy-kubernetes.md](05-deploy-kubernetes.md). API HAProxy is **not** on this VM. `node_exporter` stays systemd on every VM. |
| Cache | Nexus (in that Compose stack) proxies apt and container registries so each object is fetched from the internet once. |
| GPU (GPU profile only) | `k8s-work-4` is a VM with the GPU passed straight through to it (PCI passthrough); NVIDIA driver + container toolkit + plain Kubernetes device plugin inside the guest, no GPU Operator/MIG/time-slicing; node is tainted so only pods that explicitly tolerate it can be scheduled there — [14-gpu.md](14-gpu.md) |
| Container-count philosophy | Kubernetes and Ceph are systemd + containerd on cluster nodes (kubeadm model). Lab-support apps that must stay *outside* the cluster (Nexus, Prometheus, Grafana) run as Compose on a non-k8s VM whose host daemon is Docker. Do not put those apps in-cluster, and do not put Docker next to kubelet. |

## Version pinning and the upgrade exercise

Every package this manual installs for the cluster to function (`containerd`, `kubelet`, `kubeadm`, `kubectl`, `haproxy`, `keepalived`, `ceph-*`, `etcd-*`, `docker-ce` on the bastion, `prometheus-node-exporter`, …) is installed at an **exact pinned version** (`apt install pkg=<version>`, never a bare `apt install pkg`) and `apt-mark hold`ed right after. Compose images (`sonatype/nexus3`, `prom/prometheus`, `grafana/grafana`) are pinned by **tag** in `compose.yaml`, not by apt. A plain `apt upgrade`/`unattended-upgrades` run must never be able to silently bump a component that could break the cluster or change its behavior underneath you.

This pinning is deliberate for a second reason, not just safety: **Kubernetes, Ceph, and Calico are each deployed one version behind current stable** ([05-deploy-kubernetes.md](05-deploy-kubernetes.md) for Kubernetes and Calico, [06-ceph.md](06-ceph.md) for Ceph), specifically so there's a real version to upgrade *to*. [11-update-kubernetes.md](11-update-kubernetes.md) and [12-update-ceph.md](12-update-ceph.md) walk that, component by component, using the same unhold → install exact new pinned version → verify → re-hold cycle (Calico has no apt package/hold; the same shape applies via its operator manifest version). Staying current matters here, not just as an exercise: an untouched cluster silently ages out of its security-support window. That upgrade walkthrough is as much the point of this lab as the initial bootstrap is.

**Checked 2026-09** (Kubernetes has no LTS track — it ships a new minor roughly every 4 months and supports the 3 most recent; Ceph ships a new stable release roughly once a year, in support until the next-next one ships; Calico ships patch releases frequently within a minor):

| Component | Deploy version | Upgrade-to version | Source |
|---|---|---|---|
| Kubernetes | v1.36 (latest patch 1.36.4) | v1.37 (current stable, released 2026-08-26) | [kubernetes.io/releases](https://kubernetes.io/releases/) |
| Ceph | Squid v19.x (latest 19.2.6) — **EOL 2026-10-31**, don't linger on it | Tentacle v20.x (latest 20.2.4) | [docs.ceph.com/en/latest/releases](https://docs.ceph.com/en/latest/releases/) |
| Calico | v3.31.7 (one minor behind) | v3.32.2 (current stable) | [github.com/projectcalico/calico/releases](https://github.com/projectcalico/calico/releases) |

Re-check all three before you actually run the steps — this table is a snapshot, not a promise. Debian 13/Trixie's own `ceph-common` (`18.2.7+ds-1+deb13u1`, Reef) is already past Reef's upstream end of life, and Ceph's official [OS recommendations](https://docs.ceph.com/en/latest/start/os-recommendations/) rate Debian 13 tier "C" (packages exist, untested by the Ceph project) — [06-ceph.md](06-ceph.md) has the fallback plan if `download.ceph.com`'s Trixie repo doesn't cooperate.

## Scenarios

Pick one before provisioning. It changes the node count and which bootstrap section you open in [05-deploy-kubernetes.md](05-deploy-kubernetes.md).

| | Stacked etcd | External etcd |
|---|---|---|
| Control-plane VMs | 3 (etcd on each) | 2 (apiserver only) |
| Dedicated etcd VMs | 0 | 3 |
| Failure isolation | etcd and apiserver fail together | independent |
| Bootstrap section | Stacked | External |

Stacked is the simpler path and the one to build first. External costs more VMs and uses a native `etcd-server` package rather than kubeadm's static-pod etcd. Official kubeadm external etcd is 3 control planes plus 3 etcd nodes running etcd as a static pod; this lab keeps 2 apiservers and distro etcd on purpose.

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
