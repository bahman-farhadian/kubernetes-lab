# 00. Overview

**Goal:** Define what this lab builds, what it deliberately leaves out, and how to read the rest of the docs.

## Covers
- Purpose: build a production-like, highly-available Kubernetes cluster for hands-on learning.
- Learning focus: HA control plane, networking, storage (Ceph), security, day-2 operations.
- Out of scope: VM/host provisioning. This repo assumes the VMs already exist (created manually, via Ansible, via a cloud provider, or any other method) and are reachable over SSH with a base OS installed.
- Two supported topologies, chosen up front in [01-scenarios.md](01-scenarios.md).

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
