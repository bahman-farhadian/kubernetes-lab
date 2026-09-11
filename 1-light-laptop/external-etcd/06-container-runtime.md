# 06. Container Runtime

**Goal:** Install and configure the container runtime on control-plane and worker nodes.

## Applies to
`k8s-ctrl-1/2` and `k8s-work-1/2/3`. Not required on the bastion, `k8s-monitor`, or the `k8s-etcd-*` nodes — etcd runs as a native systemd service, no kubelet/containerd involved (see [08-bootstrap.md](08-bootstrap.md)).

## Steps

**1. Install a pinned containerd version from Debian's own repo** (keeps this manual to one apt source per concern — no extra `docker.com` repo), same version on every control-plane/worker node:
```sh
sudo apt update
apt-cache madison containerd   # list exact available versions — pick one
CONTAINERD_VERSION="<version from the list above>"
sudo apt install -y containerd=${CONTAINERD_VERSION}
```
> Verify the version kubeadm expects for your chosen Kubernetes minor (step 08) is satisfied: `containerd --version`. If Debian 13's bundled version is too old for the Kubernetes release you pick, use Docker's official `containerd.io` apt repo instead — check [download.docker.com](https://download.docker.com) for the current Debian 13/Trixie instructions before adding it.

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
- [05-os-baseline.md](05-os-baseline.md)

## Next
- [07-load-balancer.md](07-load-balancer.md)
