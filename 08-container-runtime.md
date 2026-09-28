# 08. Container Runtime

**Goal:** Install and configure the container runtime on control-plane and worker nodes.

## Applies to
Every `k8s-ctrl-*` and `k8s-work-*` in the inventory table you circled in [02-hardware-inventory.md](02-hardware-inventory.md). Not required on the firewalls, bastion, or `k8s-etcd-*`.

## Steps

**1. Install a pinned containerd version from Debian's own repo** (keeps this manual to one apt source per concern — no extra `docker.com` repo), same version on every control-plane/worker node:
```sh
sudo apt update
apt-cache madison containerd   # list exact available versions — pick one
CONTAINERD_VERSION="<version from the list above>"
sudo apt install -y containerd=${CONTAINERD_VERSION}
```
> Verify the version kubeadm expects for your chosen Kubernetes minor (step 10) is satisfied: `containerd --version`. If the distro's bundled version is too old, use Docker's official `containerd.io` apt repo instead — check [download.docker.com](https://download.docker.com) for Debian 13 or Ubuntu 26 before adding it. Prefer installing via the Nexus apt proxy from [07-nexus.md](07-nexus.md) so the `.deb` is cached.

**2. Generate default config and switch to the systemd cgroup driver** (must match kubelet's cgroup driver):
```sh
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
```

**2b. Registry mirrors** — send docker.io / registry.k8s.io / quay.io to Nexus on the bastion (see [07-nexus.md](07-nexus.md) step 5). Then:
```sh
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
- [07-nexus.md](07-nexus.md)

## Next
- [09-load-balancer.md](09-load-balancer.md)
