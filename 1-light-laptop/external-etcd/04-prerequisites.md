# 04. Prerequisites

**Goal:** Confirm every VM is in the expected starting state before any cluster-building step begins.

> Continues from the root docs — read [00-overview.md](../../00-overview.md) through [03-network-plan.md](../../03-network-plan.md) first if you haven't. This directory is the **Light** profile, **external etcd** scenario — 2 control-plane nodes + 3 dedicated etcd nodes, per [01-scenarios.md](../../01-scenarios.md).

## Covers
- Base OS: **Debian 13 ("Trixie")**, minimal/netinst install, fully updated (`apt update && apt full-upgrade`) before you start
- Every VM reachable by hostname/IP from your workstation (or via the bastion as a jump host) over SSH, key-based auth, with a `sudo`-capable non-root user
- `curl`, `gnupg`, `ca-certificates` present (needed to add the Kubernetes/Ceph/Grafana apt repos in later steps)
- The IP plan in [02-hardware-inventory.md](../../02-hardware-inventory.md) (Laptop — Light profile, Scenario B table) matches what's actually assigned to each VM, including the 3 `k8s-etcd-*` nodes
- A working DNS resolver or internet mirror reachable from every node (apt repos, container images)

## Verify before continuing
```sh
# from your workstation, once per node
ssh <user>@<node-ip> 'cat /etc/os-release | grep VERSION_ID; sudo -n true && echo "sudo OK"'
```
Confirm `VERSION_ID="13"` and passwordless (or prompting) `sudo` works for every node in [02-hardware-inventory.md](../../02-hardware-inventory.md) before moving on.

## Prerequisites
- [03-network-plan.md](../../03-network-plan.md)

## Next
- [05-os-baseline.md](05-os-baseline.md)
