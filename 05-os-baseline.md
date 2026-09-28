# 05. OS Baseline

**Goal:** Bring every VM to a common, Kubernetes-ready OS state.

## Applies to
Every node in the inventory table you circled in [02-hardware-inventory.md](02-hardware-inventory.md).

## Steps

**1. Hostname and `/etc/hosts`** — run on every node, adjust per node:
```sh
sudo hostnamectl set-hostname k8s-ctrl-1   # match the name from 02-hardware-inventory.md
```
Append the matching block below to `/etc/hosts` on **every** node (same block everywhere). These are the **example** LAN addresses from [02-hardware-inventory.md](02-hardware-inventory.md) — substitute yours if they differ. Do not add WAN addresses here.

Stacked etcd:
```
10.0.1.1    k8s-fw-1
10.0.1.2    k8s-fw-2
10.0.1.254  k8s-fw-vip
10.0.1.10   k8s-apiserver
10.0.1.11   k8s-bastion
10.0.1.12   k8s-ctrl-1
10.0.1.13   k8s-ctrl-2
10.0.1.14   k8s-ctrl-3
10.0.1.21   k8s-work-1
10.0.1.22   k8s-work-2
10.0.1.23   k8s-work-3
10.0.1.31   k8s-monitor
```

External etcd (no `k8s-ctrl-3`; dedicated etcd instead):
```
10.0.1.1    k8s-fw-1
10.0.1.2    k8s-fw-2
10.0.1.254  k8s-fw-vip
10.0.1.10   k8s-apiserver
10.0.1.11   k8s-bastion
10.0.1.12   k8s-ctrl-1
10.0.1.13   k8s-ctrl-2
10.0.1.15   k8s-etcd-1
10.0.1.16   k8s-etcd-2
10.0.1.17   k8s-etcd-3
10.0.1.21   k8s-work-1
10.0.1.22   k8s-work-2
10.0.1.23   k8s-work-3
10.0.1.31   k8s-monitor
```

Profile notes:
- **Heavy / GPU** — omit `k8s-monitor` (Prometheus/Grafana live on `k8s-bastion`; see [14-observability.md](14-observability.md)).
- **GPU** — add `10.0.1.24  k8s-work-4`.

**2. Disable swap** — Kubernetes refuses to start with swap on:
```sh
sudo swapoff -a
sudo sed -i '/\sswap\s/s/^/#/' /etc/fstab
```

**2b. Default gateway** — on every **non-firewall** VM, default route via the LAN VIP (`10.0.1.254` in the examples). The VIP is not up until [06-firewall.md](06-firewall.md); set the route now so it works as soon as keepalived starts.

**3. Kernel modules + sysctl** — required on control-plane and worker nodes (skip on firewalls, bastion, monitor, and etcd-only nodes — they never run kubelet/containerd). Firewalls get forwarding in step 06 instead:
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

**4. Time sync** — Debian 13 and Ubuntu 26 ship `systemd-timesyncd` enabled by default; just confirm it:
```sh
timedatectl status | grep "synchronized"
```

**5. Host packet filter** — on cluster nodes, if `nftables`/`ufw` is active, open the LAN port list from [03-network-plan.md](03-network-plan.md); otherwise leave it disabled and rely on the firewall pair plus network isolation. Do not confuse this with `k8s-fw-*` (those VMs are step 06).

**6. Base packages + full upgrade**, then hold nothing here yet (no cluster packages installed in this step):
```sh
sudo apt update && sudo apt full-upgrade -y
sudo apt install -y curl gnupg ca-certificates apt-transport-https
```

## Prerequisites
- [04-prerequisites.md](04-prerequisites.md)

## Next
- [06-firewall.md](06-firewall.md)
