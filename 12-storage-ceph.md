# 12. Storage — Ceph (native) + Ceph-CSI

**Goal:** Bootstrap Ceph as native `apt` packages/systemd services on the worker nodes (no Rook operator pods — see [00-overview.md](00-overview.md)), then let Kubernetes consume it via the lean Ceph-CSI driver.

Mon + mgr + OSD are co-located on `k8s-work-1/2/3` — 3 mons for quorum, one OSD per node using the dedicated Ceph disk from [02-hardware-inventory.md](02-hardware-inventory.md). This is unrelated to the etcd tier (`k8s-etcd-*` in the external scenario) — Ceph's own mon/quorum is entirely separate from Kubernetes' etcd.

Distro apt only ever carries one Ceph release per OS release, which leaves nothing to upgrade *to* later. Debian 13 as of 2026-09 bundles Reef (`18.2.7+ds-1+deb13u1`), which upstream already retired in March 2026. So this uses Ceph's own apt repo instead (`$(lsb_release -sc)` picks `trixie` or the Ubuntu 26 codename), pinned one release behind current stable (same reasoning as the Kubernetes pin in step 09), so [17-day2-operations.md](17-day2-operations.md) has a real upgrade: **Squid (v19.x) → Tentacle (v20.x)**.

> **OS caveat, checked 2026-09:** Ceph's [OS recommendations](https://docs.ceph.com/en/latest/start/os-recommendations/) rate Debian 13 tier "C" (packages exist, untested). Ubuntu 26 may or may not be listed yet — check that page before you add the repo. Try the repo below first; if `apt update` fails, Debian's fallback is its bundled Reef (`apt-cache policy ceph-common`) as a stopgap (already past upstream EOL). On Ubuntu, do not fall back to an unmaintained distro Ceph; fix the Ceph repo or stop.

## Steps — native Ceph cluster (run on `k8s-work-1/2/3`)

**GPU profile:** `k8s-work-4` also gets an OSD (same `ceph-volume` step). Mons stay on work-1/2/3 for quorum of 3.

**1. Add Ceph's repo and install a pinned release** (all OSD nodes — check [docs.ceph.com/en/latest/releases](https://docs.ceph.com/en/latest/releases/) for current/supported releases before running, and see the Debian 13 caveat above):
```sh
CEPH_DEPLOY_RELEASE=squid   # v19.x — current stable as of 2026-09 (19.2.6), scheduled EOL 2026-10-31; reverify at docs.ceph.com/en/latest/releases
curl -fsSL https://download.ceph.com/keys/release.asc | sudo gpg --dearmor -o /usr/share/keyrings/ceph.gpg
echo "deb [signed-by=/usr/share/keyrings/ceph.gpg] https://download.ceph.com/debian-${CEPH_DEPLOY_RELEASE}/ $(lsb_release -sc) main" \
  | sudo tee /etc/apt/sources.list.d/ceph.list
sudo apt update

apt-cache madison ceph-common   # list exact available versions for this release — pick one; 19.2.6 was latest as of 2026-09
CEPH_DEPLOY_VERSION="19.2.6-1~$(lsb_release -sc)"   # confirm this exact string against the madison output above

sudo apt install -y ceph-mon=${CEPH_DEPLOY_VERSION} ceph-mgr=${CEPH_DEPLOY_VERSION} \
  ceph-osd=${CEPH_DEPLOY_VERSION} ceph-common=${CEPH_DEPLOY_VERSION}
sudo apt-mark hold ceph-mon ceph-mgr ceph-osd ceph-common
```
Use the same `CEPH_DEPLOY_VERSION` on all three (four, GPU) nodes. On `k8s-work-4` you only need `ceph-osd` + `ceph-common` if you prefer not to run a fourth mon/mgr.

**2. Generate cluster identity + config** (once, e.g. on `k8s-work-1`):
```sh
FSID=$(uuidgen)
echo "$FSID"   # save this — you'll need it again for Ceph-CSI's clusterID
sudo tee /etc/ceph/ceph.conf <<EOF
[global]
fsid = ${FSID}
mon initial members = k8s-work-1, k8s-work-2, k8s-work-3
mon host = 10.0.1.21, 10.0.1.22, 10.0.1.23
public network = 10.0.1.0/24
auth cluster required = cephx
auth service required = cephx
auth client required = cephx
osd pool default size = 3
EOF
```
Copy this `ceph.conf` to `/etc/ceph/ceph.conf` on all OSD nodes.

**3. Generate keyrings and monmap** (once, on `k8s-work-1`, then copy the resulting files to the other two):
```sh
sudo ceph-authtool --create-keyring /tmp/ceph.mon.keyring --gen-key -n mon. --cap mon 'allow *'
sudo ceph-authtool --create-keyring /etc/ceph/ceph.client.admin.keyring --gen-key \
  -n client.admin --cap mon 'allow *' --cap osd 'allow *' --cap mds 'allow *' --cap mgr 'allow *'
sudo ceph-authtool /tmp/ceph.mon.keyring --import-keyring /etc/ceph/ceph.client.admin.keyring

sudo monmaptool --create --fsid "$FSID" \
  --add k8s-work-1 10.0.1.21 --add k8s-work-2 10.0.1.22 --add k8s-work-3 10.0.1.23 \
  /tmp/monmap
```
Copy `/tmp/ceph.mon.keyring`, `/tmp/monmap`, and `/etc/ceph/ceph.client.admin.keyring` to the same paths on `k8s-work-2`/`k8s-work-3`.

**4. Bootstrap each mon** (all 3 mon nodes, same commands, substitute the local hostname):
```sh
sudo -u ceph mkdir -p /var/lib/ceph/mon/ceph-k8s-work-1
sudo -u ceph ceph-mon --mkfs -i k8s-work-1 --monmap /tmp/monmap --keyring /tmp/ceph.mon.keyring
sudo systemctl enable --now ceph-mon@k8s-work-1
```

**5. Bootstrap mgr** (all 3 mon nodes). Create the directory **before** writing the keyring — `-o` will not create parent dirs:
```sh
sudo mkdir -p /var/lib/ceph/mgr/ceph-k8s-work-1
sudo chown ceph:ceph /var/lib/ceph/mgr/ceph-k8s-work-1
sudo ceph auth get-or-create mgr.k8s-work-1 mon 'allow profile mgr' osd 'allow *' mds 'allow *' \
  -o /var/lib/ceph/mgr/ceph-k8s-work-1/keyring
sudo chown ceph:ceph /var/lib/ceph/mgr/ceph-k8s-work-1/keyring
sudo systemctl enable --now ceph-mgr@k8s-work-1
```

**6. Bootstrap-OSD keyring** — `ceph-volume` authenticates as `client.bootstrap-osd`. Without this file, OSD create fails ([Ceph manual deployment](https://docs.ceph.com/en/latest/install/manual-deployment/)):
```sh
sudo mkdir -p /var/lib/ceph/bootstrap-osd
sudo ceph auth get-or-create client.bootstrap-osd \
  mon 'profile bootstrap-osd' mgr 'allow r' \
  -o /var/lib/ceph/bootstrap-osd/ceph.keyring
```
Copy `/var/lib/ceph/bootstrap-osd/ceph.keyring` (and `/etc/ceph/ceph.conf` if not already there) to every OSD node (`k8s-work-1/2/3`, plus `k8s-work-4` on GPU).

**7. Bring up the OSD** on each worker's dedicated Ceph disk (confirm the device name with `lsblk` first — don't assume `/dev/sdb`):
```sh
sudo apt install -y lvm2   # ceph-volume needs it
sudo ceph-volume lvm create --data /dev/sdb
```

**8. Verify:**
```sh
sudo ceph -s   # expect 3 mons in quorum, 3 osds up/in (4 osds on GPU)
```

## Steps — expose storage to Kubernetes via Ceph-CSI

**9. Create a pool and a scoped client for CSI:**
```sh
sudo ceph osd pool create kubernetes
sudo rbd pool init kubernetes
sudo ceph auth get-or-create client.kubernetes \
  mon 'profile rbd' osd 'profile rbd pool=kubernetes' mgr 'profile rbd pool=kubernetes' \
  -o /etc/ceph/ceph.client.kubernetes.keyring
sudo ceph auth print-key client.kubernetes   # save this key for the Secret below
```

**10. Helm 3, then a pinned Ceph-CSI RBD chart** — Debian's `apt` package named `helm` is Emacs, not this. Install Helm from the official tarball on the host that has `kubectl` (usually `k8s-ctrl-1`). Check [github.com/helm/helm/releases](https://github.com/helm/helm/releases) and pin an exact tag:
```sh
helm version   # skip the next block if Helm 3 is already on PATH
HELM_VERSION="v3.22.0"   # latest Helm 3 as of 2026-09 — confirm against the releases page above
curl -fsSL "https://get.helm.sh/helm-${HELM_VERSION}-linux-amd64.tar.gz" -o /tmp/helm.tgz
tar -xzf /tmp/helm.tgz -C /tmp
sudo install -m 0755 /tmp/linux-amd64/helm /usr/local/bin/helm
helm version
```

Deploy Ceph-CSI (check [github.com/ceph/ceph-csi](https://github.com/ceph/ceph-csi) for the current chart version list):
```sh
helm repo add ceph-csi https://ceph.github.io/csi-charts && helm repo update
helm search repo ceph-csi/ceph-csi-rbd --versions | head   # pick an exact chart version
CEPH_CSI_CHART_VERSION="<version from the list above>"
kubectl create namespace ceph-csi-rbd
helm install ceph-csi-rbd ceph-csi/ceph-csi-rbd -n ceph-csi-rbd --version "${CEPH_CSI_CHART_VERSION}" \
  --set csiConfig[0].clusterID="${FSID}" \
  --set csiConfig[0].monitors[0]=10.0.1.21:6789 \
  --set csiConfig[0].monitors[1]=10.0.1.22:6789 \
  --set csiConfig[0].monitors[2]=10.0.1.23:6789
```

**11. Secret + StorageClass:**
```sh
kubectl create secret generic csi-rbd-secret -n ceph-csi-rbd \
  --from-literal=userID=kubernetes --from-literal=userKey=<key from step 9>

cat <<EOF | kubectl apply -f -
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ceph-rbd
provisioner: rbd.csi.ceph.com
parameters:
  clusterID: "${FSID}"
  pool: kubernetes
  csi.storage.k8s.io/provisioner-secret-name: csi-rbd-secret
  csi.storage.k8s.io/provisioner-secret-namespace: ceph-csi-rbd
  csi.storage.k8s.io/node-stage-secret-name: csi-rbd-secret
  csi.storage.k8s.io/node-stage-secret-namespace: ceph-csi-rbd
reclaimPolicy: Delete
allowVolumeExpansion: true
EOF
```

**12. Verify** with a test PVC — confirm it reaches `Bound`, then delete it.

## Ceph cluster layout

```mermaid
flowchart TB
    subgraph Ceph["Native Ceph cluster (mon+mgr+osd)"]
        direction LR
        W1["k8s-work-1"]:::worker --> O1["OSD"]:::storage
        W2["k8s-work-2"]:::worker --> O2["OSD"]:::storage
        W3["k8s-work-3"]:::worker --> O3["OSD"]:::storage
    end

    classDef worker fill:#2da44e,stroke:#164c24,color:#ffffff
    classDef storage fill:#0d9488,stroke:#0f766e,color:#ffffff
```

## Prerequisites
- [11-cni.md](11-cni.md)

## Next
- [13-ingress.md](13-ingress.md)
