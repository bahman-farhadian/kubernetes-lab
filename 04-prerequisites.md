# 04. Prerequisites

**Goal:** Confirm every VM is in the expected starting state before any cluster-building step begins.

> Continues from the root docs — read [00-overview.md](00-overview.md) through [03-network-plan.md](03-network-plan.md) first if you haven't. Circle **one** profile × scenario in [02-hardware-inventory.md](02-hardware-inventory.md) **and one distro** (Debian 13 or Ubuntu 26). That table is the node list for every later step. Do not mix distros in one cluster.

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
```
Confirm `ID=debian` and `VERSION_ID="13"`, **or** `ID=ubuntu` and `VERSION_ID="26.04"` / `"26.10"` (whichever Ubuntu 26 ships as), and `sudo` works, for every node in that table.

## Prerequisites
- [03-network-plan.md](03-network-plan.md)

## Next
- [05-os-baseline.md](05-os-baseline.md)
