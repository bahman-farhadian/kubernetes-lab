# 15. Day-2 Operations

**Goal:** Operate the cluster after initial bootstrap — most importantly, prove the pin-and-hold policy actually works by deliberately upgrading the whole cluster, one held component at a time, from the version deployed in step 08 to a newer pinned version.

**Rule for every held package** (`containerd`, `kubelet`/`kubeadm`/`kubectl`, `haproxy`, `etcd-*`, `ceph-*`, `prometheus*`, `grafana`): `sudo apt-mark unhold <pkg>` → drain/cordon if it's a k8s node → `apt install <pkg>=<exact-new-version>` (never a bare `apt install`/`apt upgrade`) → verify healthy → `sudo apt-mark hold <pkg>` again. A package never spends more than the length of one upgrade step unheld.

## Steps — Kubernetes minor upgrade

Deployed on `KUBE_DEPLOY_VERSION` (step 08). Upgrading one minor at a time, in this order: **first control-plane node → remaining control-plane node → workers** — never skip a minor, per [kubeadm's version skew policy](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/kubeadm-upgrade/). This scenario's apiserver is stateless (external etcd), but `kubeadm upgrade` still walks the same control-plane component set on each node.

**1. Point at the new minor's repo and pick a pinned patch** (on `k8s-ctrl-1` first):
```sh
KUBE_UPGRADE_MINOR=v1.34   # exactly one minor above KUBE_DEPLOY_MINOR — never skip a minor
curl -fsSL "https://pkgs.k8s.io/core:/stable:/${KUBE_UPGRADE_MINOR}/deb/Release.key" \
  | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/${KUBE_UPGRADE_MINOR}/deb/ /" \
  | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt update
apt-cache madison kubeadm
KUBE_UPGRADE_VERSION="1.34.x-1.1"   # pick from the list above
```

**2. `k8s-ctrl-1`** — upgrade `kubeadm` first, apply the cluster upgrade, then `kubelet`/`kubectl`:
```sh
sudo apt-mark unhold kubeadm
sudo apt install -y kubeadm=${KUBE_UPGRADE_VERSION}
sudo apt-mark hold kubeadm
sudo kubeadm upgrade plan
sudo kubeadm upgrade apply v${KUBE_UPGRADE_VERSION%%-*}

sudo apt-mark unhold kubelet kubectl
sudo apt install -y kubelet=${KUBE_UPGRADE_VERSION} kubectl=${KUBE_UPGRADE_VERSION}
sudo apt-mark hold kubelet kubectl
sudo systemctl daemon-reload && sudo systemctl restart kubelet
```

**3. `k8s-ctrl-2`** — the only other control-plane node in this scenario:
```sh
sudo apt-mark unhold kubeadm
sudo apt install -y kubeadm=${KUBE_UPGRADE_VERSION}
sudo apt-mark hold kubeadm
sudo kubeadm upgrade node

sudo apt-mark unhold kubelet kubectl
sudo apt install -y kubelet=${KUBE_UPGRADE_VERSION} kubectl=${KUBE_UPGRADE_VERSION}
sudo apt-mark hold kubelet kubectl
sudo systemctl daemon-reload && sudo systemctl restart kubelet
```

**4. Each worker** — cordon and drain *first* (unlike control-plane nodes, workers run your actual pods):
```sh
kubectl cordon k8s-work-1
kubectl drain k8s-work-1 --ignore-daemonsets --delete-emptydir-data

# on k8s-work-1 itself:
sudo apt-mark unhold kubeadm
sudo apt install -y kubeadm=${KUBE_UPGRADE_VERSION}
sudo apt-mark hold kubeadm
sudo kubeadm upgrade node
sudo apt-mark unhold kubelet
sudo apt install -y kubelet=${KUBE_UPGRADE_VERSION}
sudo apt-mark hold kubelet
sudo systemctl daemon-reload && sudo systemctl restart kubelet

kubectl uncordon k8s-work-1
```
Repeat per worker, one at a time — never drain two simultaneously on a 3-node worker pool.

**5. Verify:**
```sh
kubectl get nodes -o wide   # every node on the new version, all Ready
```

## Steps — etcd cluster upgrade

Unlike the internal-etcd scenario (where `kubeadm upgrade` also bumps etcd's static-pod image), this etcd is a plain apt package independent of Kubernetes' own version — it needs its own upgrade, same "one node at a time, verify quorum" discipline as Ceph mons below.

**1. Pick a new pinned version** (on `k8s-etcd-1`):
```sh
sudo apt update
apt-cache madison etcd-server
ETCD_UPGRADE_VERSION="<version from the list above>"
```

**2. Upgrade one node at a time, verifying quorum before moving to the next:**
```sh
sudo apt-mark unhold etcd-server etcd-client
sudo apt install -y etcd-server=${ETCD_UPGRADE_VERSION} etcd-client=${ETCD_UPGRADE_VERSION}
sudo apt-mark hold etcd-server etcd-client
sudo systemctl restart etcd

etcdctl --endpoints=https://10.0.1.15:2379,https://10.0.1.16:2379,https://10.0.1.17:2379 \
  --cacert=/etc/etcd/pki/ca.pem --cert=/etc/etcd/pki/k8s-etcd-1.pem --key=/etc/etcd/pki/k8s-etcd-1-key.pem \
  endpoint health --cluster
```
Repeat on `k8s-etcd-2`, then `k8s-etcd-3`. A mixed-version quorum mid-rollout is expected and safe, same as Ceph mons below.

## Steps — Ceph release upgrade

Deployed on `CEPH_DEPLOY_RELEASE`/`CEPH_DEPLOY_VERSION` (step 11). Order matters: **mons (one at a time) → mgrs → OSDs (one node at a time)**. Check the target release's own upgrade notes on [docs.ceph.com](https://docs.ceph.com/en/latest/releases/) first — some releases require an extra step (e.g. `ceph osd require-osd-release <name>`) once every daemon is upgraded, not assumed here since it depends which two releases you're moving between.

**1. Point at the new release's repo** (all 3 nodes):
```sh
CEPH_UPGRADE_RELEASE=squid   # the release you're upgrading to
sudo sed -i "s/debian-${CEPH_DEPLOY_RELEASE}/debian-${CEPH_UPGRADE_RELEASE}/" /etc/apt/sources.list.d/ceph.list
sudo apt update
apt-cache madison ceph-common
CEPH_UPGRADE_VERSION="19.2.x-1~$(lsb_release -sc)"   # pick from the list above
```

**2. Prevent rebalancing churn** while daemons briefly restart (from any node):
```sh
sudo ceph osd set noout
```

**3. Upgrade mons, one node at a time** — verify quorum before moving to the next:
```sh
sudo apt-mark unhold ceph-mon ceph-common
sudo apt install -y ceph-mon=${CEPH_UPGRADE_VERSION} ceph-common=${CEPH_UPGRADE_VERSION}
sudo apt-mark hold ceph-mon ceph-common
sudo systemctl restart ceph-mon@k8s-work-1
sudo ceph -s   # confirm quorum intact before touching k8s-work-2
```
Repeat on `k8s-work-2`, then `k8s-work-3`. A mixed-version mon quorum during this rolling upgrade is expected and safe.

**4. Upgrade mgrs, one node at a time:**
```sh
sudo apt-mark unhold ceph-mgr
sudo apt install -y ceph-mgr=${CEPH_UPGRADE_VERSION}
sudo apt-mark hold ceph-mgr
sudo systemctl restart ceph-mgr@k8s-work-1
```
Repeat on the other two; `ceph -s` shows the active mgr fail over to a standby momentarily — expected.

**5. Upgrade OSDs, one node at a time:**
```sh
sudo apt-mark unhold ceph-osd
sudo apt install -y ceph-osd=${CEPH_UPGRADE_VERSION}
sudo apt-mark hold ceph-osd
sudo systemctl restart ceph-osd@$(ls /var/lib/ceph/osd | grep -oP 'ceph-\K[0-9]+')
sudo ceph -s   # HEALTH_OK (or HEALTH_WARN with noout set) before moving to the next node
```
Repeat on the other two workers.

**6. Clear `noout` and do a final check:**
```sh
sudo ceph osd unset noout
sudo ceph versions   # every daemon should report the new version
sudo ceph -s          # HEALTH_OK
```

## Also covers
- etcd backup and restore: `etcdctl snapshot save` directly against any `k8s-etcd-*` node (no `kubectl exec` needed, unlike the internal-etcd scenario — this etcd is a plain systemd service)
- Adding/removing control-plane, etcd, and worker nodes
- Certificate rotation (etcd's own CA from [08-bootstrap.md](08-bootstrap.md) step 2, separate from Kubernetes' PKI)

## Applies to
Light profile, external etcd.

## Prerequisites
- [14-security-hardening.md](14-security-hardening.md)

## Next
- [16-troubleshooting.md](16-troubleshooting.md)
