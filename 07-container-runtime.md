# 07. Container Runtime

**Goal:** Install and configure the container runtime on control-plane and worker nodes.

## Applies to
Every `k8s-ctrl-*` and `k8s-work-*` in the inventory table you circled in [02-hardware-inventory.md](02-hardware-inventory.md). Not required on the firewalls, bastion, `k8s-monitor`, or `k8s-etcd-*`.

## Steps

**1. Install a pinned containerd version from Debian's own repo** (keeps this manual to one apt source per concern — no extra `docker.com` repo), same version on every control-plane/worker node:
```sh
sudo apt update
apt-cache madison containerd   # list exact available versions — pick one
CONTAINERD_VERSION="<version from the list above>"
sudo apt install -y containerd=${CONTAINERD_VERSION}
```
> Verify the version kubeadm expects for your chosen Kubernetes minor (step 09) is satisfied: `containerd --version`. If the distro's bundled version is too old, use Docker's official `containerd.io` apt repo instead — check [download.docker.com](https://download.docker.com) for Debian 13 or Ubuntu 26 before adding it.

**2. Generate default config and switch to the systemd cgroup driver** (must match kubelet's cgroup driver):
```sh
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl enable containerd
```

**3. Confirm the CRI socket** kubeadm will use:
```sh
sudo ctr version
ls -l /run/containerd/containerd.sock
```

**4. Hold the package** so `apt upgrade` can't silently change the runtime under a live cluster:
```sh
sudo apt-mark hold containerd
```

## Prerequisites
- [06-firewall.md](06-firewall.md)

## Next
- [08-load-balancer.md](08-load-balancer.md)
