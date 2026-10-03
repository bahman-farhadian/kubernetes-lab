# 01. Inventory and network

**Goal:** VM list, resource budget, and the LAN plan for whichever profile and scenario you are deploying.

> VM creation is out of scope ([00-overview.md](00-overview.md)). These tables are what the VMs look like once they exist.

## Two lab environments

**Heavy** and **GPU** are separate clusters on the server. They reuse `10.0.1.0/24`, so only one is on a given L2 at a time. Do not join a Heavy node to the GPU cluster.

A 32 GB laptop cannot hold this layout. The API pair alone is two extra VMs, and stacked already wants 28 GB before the hypervisor. There is no laptop profile.

- **Heavy** (server) — deploy this first.
- **GPU** (server) — Heavy plus `k8s-work-4`.

Rollout: [README.md](README.md#rollout-plan).

LAN addresses here are **examples**. Use your own when you provision. Do not commit site WAN or LAN addresses into this repo. Each firewall has a second NIC on WAN; WAN addresses stay off these tables.

## Network

- **Gateway:** `k8s-fw-1` / `k8s-fw-2`, keepalived only, LAN VIP `10.0.1.254`. Every node defaults through this address. No HAProxy here. [03-firewall.md](03-firewall.md).
- **Load balancer:** `k8s-lb-1` / `k8s-lb-2`, keepalived + HAProxy, VIP `10.0.1.10`. Clients use it for the API (`:6443`) and, after ingress, for `:80`/`:443`. Own VRID, different from the firewall LAN VRID. Node-to-node traffic does not pass through this pair. [05-deploy-kubernetes.md](05-deploy-kubernetes.md).
- **Bastion** `10.0.1.11`: jump host, Nexus, Prometheus, Grafana, `kubectl`, Helm. One VM is normal for this role. No HAProxy and no VIP on it.
- **DNS:** static `/etc/hosts` on every node ([02-prepare.md](02-prepare.md)). No cluster DNS server for node names.
- **Kubernetes ranges** (must not overlap the LAN): pod CIDR `192.168.0.0/16`, service CIDR `10.96.0.0/12`.
- **Ports:** `6443` (apiserver, via the API VIP), `80`/`443` (ingress, same VIP, added in [07-ingress.md](07-ingress.md)), `8081`/`8082` (Nexus), `2379-2380` (etcd), `10250` (kubelet), `179`/`4789` (Calico), `9100` (node_exporter), `9090`/`3000` (Prometheus/Grafana on the bastion). VRRP is protocol 112, once for the firewall pair and once for the API pair.
- First Nexus fill goes out through the firewall NAT. After [04-bastion.md](04-bastion.md), apt and image pulls can use the bastion cache.

```mermaid
flowchart LR
    Ext["Upstream / WAN"] --> WANVIP["WAN VIP"]:::bastion
    WANVIP --> FW["k8s-fw-1 / k8s-fw-2\ngateway only"]:::bastion
    FW --> LAN["LAN"]:::worker
    Client["kubectl / ingress"] --> LB["k8s-lb-1 / k8s-lb-2\nVIP :6443 :80 :443"]:::bastion
    LB --> CP["Control plane"]:::controlPlane
    LAN --> CP
    CP -.-> Etcd["etcd"]:::etcd
    CP --> Work["Workers"]:::worker
    Work --> Storage["Ceph OSDs"]:::storage

    classDef bastion fill:#1f6feb,stroke:#0c2d6b,color:#ffffff
    classDef controlPlane fill:#8250df,stroke:#4b1f91,color:#ffffff
    classDef etcd fill:#d29922,stroke:#7d5c05,color:#1a1a1a
    classDef worker fill:#2da44e,stroke:#164c24,color:#ffffff
    classDef storage fill:#0d9488,stroke:#0f766e,color:#ffffff
```

The bastion is on the LAN and is not drawn above: it is not on the gateway path or the API path.

### VMs and connections — stacked

Eleven VMs. Addresses are the example LAN. External etcd drops `k8s-ctrl-3` and adds `k8s-etcd-1/2/3` (`.15`–`.17`). GPU adds `k8s-work-4` (`.24`).

```mermaid
flowchart TB
    WAN["Upstream WAN"]

    FW1["k8s-fw-1\nLAN .1 + WAN NIC"]:::bastion
    FW2["k8s-fw-2\nLAN .2 + WAN NIC"]:::bastion
    GW[".254 gateway VIP\nnot a VM"]:::bastion

    LB1["k8s-lb-1\n.8"]:::bastion
    LB2["k8s-lb-2\n.9"]:::bastion
    API[".10 load-balancer VIP\nnot a VM"]:::bastion

    BAST["k8s-bastion\n.11 admin"]:::bastion

    C1["k8s-ctrl-1\n.12 CP + etcd"]:::controlPlane
    C2["k8s-ctrl-2\n.13 CP + etcd"]:::controlPlane
    C3["k8s-ctrl-3\n.14 CP + etcd"]:::controlPlane

    W1["k8s-work-1\n.21 worker + OSD"]:::worker
    W2["k8s-work-2\n.22 worker + OSD"]:::worker
    W3["k8s-work-3\n.23 worker + OSD"]:::worker

    WAN --- FW1
    WAN --- FW2
    FW1 <-->|"VRRP"| FW2
    FW1 --- GW
    FW2 --- GW
    GW -.->|"default route"| BAST

    LB1 <-->|"VRRP"| LB2
    LB1 --- API
    LB2 --- API
    API -->|"TCP 6443"| C1
    API -->|"TCP 6443"| C2
    API -->|"TCP 6443"| C3

    C1 <-->|"etcd"| C2
    C2 <-->|"etcd"| C3
    C3 <-->|"etcd"| C1
    C1 -->|"kubelet"| W1
    C1 -->|"kubelet"| W2
    C1 -->|"kubelet"| W3
    W1 <-->|"Calico / Ceph"| W2
    W2 <-->|"Calico / Ceph"| W3

    classDef bastion fill:#1f6feb,stroke:#0c2d6b,color:#ffffff
    classDef controlPlane fill:#8250df,stroke:#4b1f91,color:#ffffff
    classDef etcd fill:#d29922,stroke:#7d5c05,color:#1a1a1a
    classDef worker fill:#2da44e,stroke:#164c24,color:#ffffff
    classDef storage fill:#0d9488,stroke:#0f766e,color:#ffffff
```

Every box except the two VIPs is a VM. Both firewalls have a WAN NIC and a LAN NIC. Every other VM has one LAN NIC on `10.0.1.0/24` and uses `.254` as its default gateway. `k8s-bastion` is on that LAN for SSH, Nexus, and metrics only — no line to the API VIP. After ingress, the same `.10` VIP also accepts TCP 80 and 443 and HAProxy sends those to the workers.

## Server — Heavy and GPU

Host budget as previously sized: 24 vCPU, 128 GB RAM, 250 GB NVMe for root disks, 1 TB NVMe for Ceph. The API pair adds 2 vCPU, 2 GB, and 20 GB of root on every server table. GPU stacked therefore wants 130 GB RAM and 270 GB of root NVMe; a chassis that is still 128 GB / 250 GB root is 2 GB of RAM and 20 GB of disk short. vCPU oversubscribe on GPU stacked goes from 2 to 4.

### Heavy — Scenario A (Stacked etcd)

| # | VM Name | Role | LAN IP (example) | vCPU | RAM | Root Disk | Ceph OSD |
|---|---|---|---|---|---|---|---|
| 01 | k8s-fw-1 | Firewall (WAN + LAN) | 10.0.1.1 | 2 | 2 GB | 20 GB | - |
| 02 | k8s-fw-2 | Firewall (WAN + LAN) | 10.0.1.2 | 2 | 2 GB | 20 GB | - |
| 03 | k8s-lb-1 | API HAProxy (VRRP master) | 10.0.1.8 | 1 | 1 GB | 10 GB | - |
| 04 | k8s-lb-2 | API HAProxy (VRRP backup) | 10.0.1.9 | 1 | 1 GB | 10 GB | - |
| 05 | k8s-bastion | Jump / kubectl + Helm / Nexus / Prometheus / Grafana | 10.0.1.11 | 2 | 4 GB | 40 GB | - |
| 06 | k8s-ctrl-1 | Control Plane + etcd | 10.0.1.12 | 2 | 8 GB | 30 GB | - |
| 07 | k8s-ctrl-2 | Control Plane + etcd | 10.0.1.13 | 2 | 8 GB | 30 GB | - |
| 08 | k8s-ctrl-3 | Control Plane + etcd | 10.0.1.14 | 2 | 8 GB | 30 GB | - |
| 09 | k8s-work-1 | Worker / Storage | 10.0.1.21 | 4 | 24 GB | 20 GB | 200 GB |
| 10 | k8s-work-2 | Worker / Storage | 10.0.1.22 | 4 | 24 GB | 20 GB | 200 GB |
| 11 | k8s-work-3 | Worker / Storage | 10.0.1.23 | 4 | 24 GB | 20 GB | 200 GB |
| | **VM TOTALS (11 VMs)** | | | **26** | **106 GB** | **250 GB** | **600 GB** |

The bastion is jump, kubectl, Helm, Nexus, Prometheus, and Grafana. No HAProxy. Root disk 40 GB is the Nexus blob store.

### GPU — Scenario A (Heavy + GPU worker)

Everything in Heavy above, plus:

| # | VM Name | Role | LAN IP (example) | vCPU | RAM | Root Disk | Ceph OSD |
|---|---|---|---|---|---|---|---|
| 12 | k8s-work-4 | Worker / Storage / GPU (NVIDIA) | 10.0.1.24 | 2 | 24 GB | 20 GB | 200 GB |
| | **VM TOTALS (12 VMs)** | | | **28** | **130 GB** | **270 GB** | **800 GB** |
| | **PC AS PREVIOUSLY STATED** | | | **24** | **128 GB** | **250 GB** | **1 TB** |

`k8s-work-4` is a VM with the GPU passed through. Hypervisor IOMMU setup is out of scope. The node is tainted. Driver and device plugin: [14-gpu.md](14-gpu.md).

### Heavy/GPU — Scenario B (External etcd)

| # | VM Name | Role | LAN IP (example) | vCPU | RAM | Root Disk | Ceph OSD |
|---|---|---|---|---|---|---|---|
| 01 | k8s-fw-1 | Firewall (WAN + LAN) | 10.0.1.1 | 2 | 2 GB | 20 GB | - |
| 02 | k8s-fw-2 | Firewall (WAN + LAN) | 10.0.1.2 | 2 | 2 GB | 20 GB | - |
| 03 | k8s-lb-1 | API HAProxy (VRRP master) | 10.0.1.8 | 1 | 1 GB | 10 GB | - |
| 04 | k8s-lb-2 | API HAProxy (VRRP backup) | 10.0.1.9 | 1 | 1 GB | 10 GB | - |
| 05 | k8s-bastion | Jump / kubectl + Helm / Nexus / Prometheus / Grafana | 10.0.1.11 | 2 | 4 GB | 40 GB | - |
| 06 | k8s-ctrl-1 | Control Plane | 10.0.1.12 | 2 | 8 GB | 30 GB | - |
| 07 | k8s-ctrl-2 | Control Plane | 10.0.1.13 | 2 | 8 GB | 30 GB | - |
| 08 | k8s-etcd-1 | etcd | 10.0.1.15 | 1 | 2 GB | 10 GB | - |
| 09 | k8s-etcd-2 | etcd | 10.0.1.16 | 1 | 2 GB | 10 GB | - |
| 10 | k8s-etcd-3 | etcd | 10.0.1.17 | 1 | 2 GB | 10 GB | - |
| 11 | k8s-work-1 | Worker / Storage | 10.0.1.21 | 4 | 24 GB | 20 GB | 200 GB |
| 12 | k8s-work-2 | Worker / Storage | 10.0.1.22 | 4 | 24 GB | 20 GB | 200 GB |
| 13 | k8s-work-3 | Worker / Storage | 10.0.1.23 | 4 | 24 GB | 20 GB | 200 GB |
| 14 | k8s-work-4 | Worker / Storage / GPU (NVIDIA) | 10.0.1.24 | 2 | 24 GB | 20 GB | 200 GB |
| | **VM TOTALS (14 VMs)** | | | **29** | **128 GB** | **270 GB** | **800 GB** |
| | **PC AS PREVIOUSLY STATED** | | | **24** | **128 GB** | **250 GB** | **1 TB** |

RAM matches 128 GB with nothing reserved. Root disks need 270 GB, 20 GB past the 250 GB NVMe. Drop `k8s-work-4` for a Heavy-only external cluster (27 vCPU / 104 GB / 250 GB root / 600 GB OSD).

etcd nodes stay 1 vCPU / 2 GB / 10 GB: at this size etcd is disk and network sensitive, not CPU hungry.

## Prerequisites
- [00-overview.md](00-overview.md)

## Next
- [02-prepare.md](02-prepare.md)
