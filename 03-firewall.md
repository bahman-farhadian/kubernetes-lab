# 03. Firewall pair

**Goal:** Put a two-node keepalived firewall in front of the LAN so the cluster has a redundant default gateway and a single WAN entry. The API VIP (`10.0.1.10`) is a second keepalived pair, `k8s-lb-1` / `k8s-lb-2`, in [05-deploy-kubernetes.md](05-deploy-kubernetes.md). Do not put that VIP on these firewalls, and do not reuse this VRID for it — both pairs speak VRRP on the same LAN.

**Applies to:** `k8s-fw-1` and `k8s-fw-2` only. Not a Kubernetes node — no kubelet, no containerd.

WAN addresses, WAN VIP, and VRID are **site-local**. This file uses variables. Do not copy a production/lab WAN numbering plan into the repo.

```sh
WAN_IF=eth0                 # NIC that faces upstream
LAN_IF=eth1                 # NIC on the cluster LAN (10.0.1.0/24 in the example tables)
WAN_VIP="<your WAN VIP>"    # keepalived address on $WAN_IF
LAN_VIP=10.0.1.254          # example LAN gateway — must match [02-prepare.md](02-prepare.md)
VRID=60                     # unique per lab instance on a shared L2; pick another if 60 is taken
```

`k8s-fw-1` is MASTER (priority 100), `k8s-fw-2` is BACKUP (priority 90). Same `VRID` on both instances is fine because they run on different interfaces.

## Steps

**1. Enable forwarding** (both firewall nodes):
```sh
cat <<EOF | sudo tee /etc/sysctl.d/k8s-fw.conf
net.ipv4.ip_forward = 1
net.ipv4.conf.all.forwarding = 1
EOF
sudo sysctl --system
```

**2. Install a pinned keepalived** (same version on both nodes):
```sh
sudo apt update
apt-cache madison keepalived
KEEPALIVED_VERSION="<version from the list above>"
sudo apt install -y keepalived=${KEEPALIVED_VERSION}
sudo apt-mark hold keepalived
```

**3. Configure keepalived** — `/etc/keepalived/keepalived.conf` on `k8s-fw-1` (MASTER). Substitute interface names and `$WAN_VIP`. Do not put a real passphrase in git; set `auth_pass` locally (8 chars, keepalived truncates anyway):

```
vrrp_instance WAN {
    state MASTER
    interface ${WAN_IF}
    virtual_router_id ${VRID}
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass <local-only>
    }
    virtual_ipaddress {
        ${WAN_VIP}
    }
}

vrrp_instance LAN {
    state MASTER
    interface ${LAN_IF}
    virtual_router_id ${VRID}
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass <local-only>
    }
    virtual_ipaddress {
        10.0.1.254/24
    }
}
```

On `k8s-fw-2`: same file with `state BACKUP` and `priority 90` in both instances. Then:

```sh
sudo systemctl enable --now keepalived
ip -br addr show   # MASTER should show both VIPs; BACKUP should not
```

**4. Forward LAN → WAN** (both nodes; nftables). This is a lab NAT, not a hardened edge policy. A minimal install does not ship the `nft` binary — the package is `nftables` on Debian 13 and Ubuntu 26. Docker stays off these VMs: Docker's install docs do not support an `nft` ruleset on a host that runs Docker Engine, which is why NAT lives here and Docker lives on the bastion.

```sh
sudo apt install -y nftables
sudo nft add table ip nat
sudo nft add chain ip nat postrouting '{ type nat hook postrouting priority 100; }'
sudo nft add rule ip nat postrouting oifname "$WAN_IF" masquerade
sudo nft add table ip filter
sudo nft add chain ip filter forward '{ type filter hook forward priority 0; policy drop; }'
sudo nft add rule ip filter forward ct state established,related accept
sudo nft add rule ip filter forward iifname "$LAN_IF" oifname "$WAN_IF" accept
sudo nft add rule ip filter forward iifname "$WAN_IF" oifname "$LAN_IF" ct state established,related accept
```

Persist with `nft list ruleset | sudo tee /etc/nftables.conf` and `sudo systemctl enable nftables` (package name is `nftables` on both Debian 13 and Ubuntu 26).

**5. Point the cluster at the LAN VIP** — on every **non-firewall** VM, replace the temporary default route from [02-prepare.md](02-prepare.md) with `10.0.1.254`. Do this only after step 3 shows the VIP on the MASTER.

```sh
IFACE=$(ip -br addr show | awk '/10\.0\.1\./ {print $1; exit}')
echo "LAN iface: $IFACE"    # confirm before changing the default route
sudo ip route replace default via 10.0.1.254 dev "$IFACE"
ip route | grep default      # via 10.0.1.254
ping -c1 10.0.1.254
curl -sI https://deb.debian.org | head -1   # or https://archive.ubuntu.com — outbound via the pair
```

Persist it or the next reboot goes back to the temporary gateway (or nowhere). ifupdown: on the stanza that already has this VM's `10.0.1.0/24` address, set `gateway 10.0.1.254` and delete any other `gateway` line. NetworkManager: `nmcli con modify <connection> ipv4.gateway 10.0.1.254 && nmcli con up <connection>`.

Failover check: `sudo systemctl stop keepalived` on MASTER; VIPs must appear on BACKUP within a couple of seconds; restore keepalived on MASTER afterward.

## Request path

```mermaid
flowchart LR
    Ext["Upstream / WAN"] --> WANVIP["WAN VIP"]:::bastion
    WANVIP --> FW1["k8s-fw-1"]:::bastion
    WANVIP --> FW2["k8s-fw-2"]:::bastion
    FW1 --> LANVIP["LAN VIP\n(default gateway)"]:::bastion
    FW2 --> LANVIP
    LANVIP --> LAN["Cluster LAN"]:::worker

    classDef bastion fill:#1f6feb,stroke:#0c2d6b,color:#ffffff
    classDef worker fill:#2da44e,stroke:#164c24,color:#ffffff
```

## Prerequisites
- [02-prepare.md](02-prepare.md)

## Next
- [04-bastion.md](04-bastion.md)
