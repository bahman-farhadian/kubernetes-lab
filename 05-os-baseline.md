# 05. OS Baseline

**Goal:** Bring every VM to a common, Kubernetes-ready OS state.

## Applies to
Every node in the inventory table you circled in [02-hardware-inventory.md](02-hardware-inventory.md).

## Steps

**1. Hostname and `/etc/hosts`** — run on every node, adjust per node:
```sh
sudo hostnamectl set-hostname k8s-ctrl-1   # match the name from 02-hardware-inventory.md
```
Append the matching block below to `/etc/hosts` on **every** node (same block everywhere).

Stacked etcd:
```
10.0.1.11  k8s-bastion
10.0.1.12  k8s-ctrl-1
10.0.1.13  k8s-ctrl-2
10.0.1.14  k8s-ctrl-3
10.0.1.21  k8s-work-1
10.0.1.22  k8s-work-2
10.0.1.23  k8s-work-3
10.0.1.31  k8s-monitor
```

External etcd (no `k8s-ctrl-3`; dedicated etcd instead):
```
10.0.1.11  k8s-bastion
10.0.1.12  k8s-ctrl-1
10.0.1.13  k8s-ctrl-2
10.0.1.15  k8s-etcd-1
10.0.1.16  k8s-etcd-2
10.0.1.17  k8s-etcd-3
10.0.1.21  k8s-work-1
10.0.1.22  k8s-work-2
10.0.1.23  k8s-work-3
10.0.1.31  k8s-monitor
```

Profile notes:
- **Heavy / GPU** — omit `k8s-monitor` (Prometheus/Grafana live on `k8s-bastion`; see [13-observability.md](13-observability.md)).
- **GPU** — add `10.0.1.24  k8s-work-4`.

**2. Disable swap** — Kubernetes refuses to start with swap on:
```sh
sudo swapoff -a
sudo sed -i '/\sswap\s/s/^/#/' /etc/fstab
```

**3. Kernel modules + sysctl** — required on control-plane and worker nodes (skip on bastion, monitor, and etcd-only nodes — they never run kubelet/containerd):
```sh
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
sudo modprobe overlay br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
sudo sysctl --system
```

**4. Time sync** — Debian 13 ships `systemd-timesyncd` enabled by default; just confirm it:
```sh
timedatectl status | grep "synchronized"
```

**5. Firewall** — this lab uses the port list from [03-network-plan.md](03-network-plan.md). If `nftables`/`ufw` is active, open those ports between nodes; otherwise leave the host firewall disabled and rely on network-level isolation (this is a lab, not exposed to the internet).

**6. Base packages + full upgrade**, then hold nothing here yet (no cluster packages installed in this step):
```sh
sudo apt update && sudo apt full-upgrade -y
sudo apt install -y curl gnupg ca-certificates apt-transport-https
```

## Prerequisites
- [04-prerequisites.md](04-prerequisites.md)

## Next
- [06-container-runtime.md](06-container-runtime.md)
