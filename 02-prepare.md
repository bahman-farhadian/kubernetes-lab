# 02. Prepare the VMs

**Goal:** Confirm every VM is in the expected starting state before any cluster-building step begins.

> Continues from the root docs — read [00-overview.md](00-overview.md) through [01-inventory.md](01-inventory.md) first if you haven't. Circle **one** profile × scenario in [01-inventory.md](01-inventory.md) **and one distro** (Debian 13 or Ubuntu 26). That table is the node list for every later step. Do not mix distros in one cluster.

## Covers
- Base OS: **Debian 13 ("Trixie")** *or* **Ubuntu 26 ("Resolute Raccoon")**, minimal install, fully updated (`apt update && apt full-upgrade`) before you start
- Every VM reachable by hostname/IP from your workstation (or via the bastion as a jump host) over SSH, key-based auth, with a `sudo`-capable non-root user — including `k8s-fw-1` / `k8s-fw-2`
- `curl`, `gnupg`, `ca-certificates` present (needed to add the Kubernetes/Ceph/Grafana apt repos in later steps)
- The LAN plan in the table you circled matches what's assigned (including firewall LAN IPs, `k8s-etcd-*` if external, `k8s-work-4` if GPU). WAN addresses on the firewalls are site-local and stay out of this repo. There is no `k8s-monitor` VM.
- Each firewall VM has two NICs (WAN + LAN)
- A working path to the internet **once the firewall pair is up** (step 06); before that, you may need a temporary default route to finish `apt` on the firewalls themselves

## Verify before continuing
```sh
# from your workstation, once per node
ssh <user>@<node-ip> 'grep -E "^(ID|VERSION_ID)=" /etc/os-release; sudo -n true && echo "sudo OK"'
ssh <user>@<node-ip> 'cat /sys/class/dmi/id/product_uuid; cat /etc/machine-id'
```
Confirm `ID=debian` and `VERSION_ID="13"`, **or** `ID=ubuntu` and `VERSION_ID="26.04"` / `"26.10"` (whichever Ubuntu 26 ships as), and `sudo` works, for every node in that table.

`product_uuid` and `/etc/machine-id` must be different on every VM. kubeadm uses them to identify nodes and refuses the install when two match ([install-kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#verify-mac-address)). A from-scratch install already has unique values. A clone does not: `sudo rm -f /etc/machine-id && sudo systemd-machine-id-setup`, and set a new SMBIOS UUID in the hypervisor before boot.

## OS baseline

**Goal:** Bring every VM to a common, Kubernetes-ready OS state.

## Applies to
Every node in the inventory table you circled in [01-inventory.md](01-inventory.md).

## Steps

**1. Hostname and `/etc/hosts`** — run on every node, adjust per node:
```sh
sudo hostnamectl set-hostname k8s-ctrl-1   # match the name from 01-inventory.md
```
Append the matching block below to `/etc/hosts` on **every** node (same block everywhere). These are the **example** LAN addresses from [01-inventory.md](01-inventory.md) — substitute yours if they differ. Do not add WAN addresses here.

Stacked etcd:
```
10.0.1.1    k8s-fw-1
10.0.1.2    k8s-fw-2
10.0.1.8    k8s-lb-1
10.0.1.9    k8s-lb-2
10.0.1.254  k8s-fw-vip
10.0.1.10   k8s-apiserver
10.0.1.11   k8s-bastion
10.0.1.12   k8s-ctrl-1
10.0.1.13   k8s-ctrl-2
10.0.1.14   k8s-ctrl-3
10.0.1.21   k8s-work-1
10.0.1.22   k8s-work-2
10.0.1.23   k8s-work-3
```

External etcd (no `k8s-ctrl-3`; dedicated etcd instead):
```
10.0.1.1    k8s-fw-1
10.0.1.2    k8s-fw-2
10.0.1.8    k8s-lb-1
10.0.1.9    k8s-lb-2
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
```

Profile notes:
- **GPU** — add `10.0.1.24  k8s-work-4`.
- There is no `k8s-monitor` VM. Prometheus/Grafana and Nexus run on `k8s-bastion`.

**2. Disable swap** — Kubernetes refuses to start with swap on:
```sh
sudo swapoff -a
sudo sed -i '/\sswap\s/s/^/#/' /etc/fstab
```

**2b. Leave the install-time default route in place.** Step 6 needs outbound apt, and the LAN VIP (`10.0.1.254`) does not exist until keepalived is up in [03-firewall.md](03-firewall.md). Pointing the default route at it here black-holes that apt run. Step 06 replaces the temporary gateway after the VIP answers.

**3. Kernel modules + sysctl** — required on control-plane and worker nodes (skip on firewalls, the API load-balancer pair, the bastion, and etcd-only nodes — they never run kubelet/containerd). Firewalls get forwarding in step 06 instead:
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

**5. Host packet filter** — on cluster nodes, if `nftables`/`ufw` is active, open the LAN port list from [01-inventory.md](01-inventory.md); otherwise leave it disabled and rely on the firewall pair plus network isolation. Do not confuse this with `k8s-fw-*` (those VMs are [03-firewall.md](03-firewall.md)).

**6. Base packages + full upgrade**, then hold nothing here yet (no cluster packages installed in this step):
```sh
sudo apt update && sudo apt full-upgrade -y
sudo apt install -y curl gnupg ca-certificates apt-transport-https
```

## Prerequisites
- [01-inventory.md](01-inventory.md)

## Next
- [03-firewall.md](03-firewall.md)
