# 00. Overview

**Goal:** Define what this lab builds, what it deliberately leaves out, and how to read the rest of the docs.

## Covers
- Purpose: build a production-like, highly-available Kubernetes cluster for hands-on learning.
- Learning focus: HA control plane, networking, storage (Ceph), security, day-2 operations.
- Out of scope: VM/host provisioning. This repo assumes the VMs already exist (created manually, via Ansible, via a cloud provider, or any other method) and are reachable over SSH with a base OS installed.
- Two supported topologies, chosen up front in [01-scenarios.md](01-scenarios.md).
- Style: manual, step-by-step, real commands — short enough to read end to end and understand what's happening, so you can later drive the same cluster with kubeadm scripted, Kubespray, or any other automation with eyes open.

## Fixed decisions

These are settled for the whole manual — later steps assume them rather than re-justifying them:

| Area | Choice |
|---|---|
| OS | Debian 13 ("Trixie") on every VM |
| Bootstrap tool | `kubeadm` (not k3s/RKE2/Kubespray) |
| Container runtime | containerd only |
| CNI | Calico (NetworkPolicy support, needed in [14-security-hardening.md](14-security-hardening.md)) |
| Storage | Ceph, installed as native `apt` packages on the OS (not Rook) + Ceph-CSI inside the cluster |
| Ingress | Traefik (ingress-nginx is being sunset upstream) |
| Monitoring | node_exporter on every node + Prometheus/Grafana as native OS packages on a dedicated monitor node — outside the cluster, so cluster problems don't take monitoring down with them |
| Container-count philosophy | Prefer a host-installed daemon over an in-cluster operator/pod wherever both exist (this is why Ceph and monitoring live outside Kubernetes) |

## Package hold policy

Every package this manual installs for the cluster to function (`containerd`, `kubelet`, `kubeadm`, `kubectl`, `haproxy`, `ceph-*`, `prometheus*`, `grafana`, …) gets `apt-mark hold`ed right after install. A plain `apt upgrade`/`unattended-upgrades` run must never be able to silently bump a component that could break the cluster or change its behavior underneath you — upgrades to held packages are deliberate and go through [15-day2-operations.md](15-day2-operations.md), one node at a time. The `hold` command is repeated in each step next to the install it applies to.

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

The flow palette's scenario colors intentionally match the role palette: Scenario A is purple because its etcd is embedded in the (purple) control plane; Scenario B is amber because its etcd is the (amber) standalone tier.

## Prerequisites
- None — start here.
