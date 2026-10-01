# 12. Update Ceph

**Goal:** Upgrade Ceph from the release deployed in [06-ceph.md](06-ceph.md) to the next pinned release. Do this only after that cluster is healthy and after [11-update-kubernetes.md](11-update-kubernetes.md).

Deployed on `CEPH_DEPLOY_RELEASE`/`CEPH_DEPLOY_VERSION` ([06-ceph.md](06-ceph.md)) — Squid (v19.x), current as of 2026-09 but scheduled to reach end of life 2026-10-31, so don't sit on it indefinitely; this exercise upgrades to Tentacle (v20.x). Order matters: **mons (one at a time) → mgrs → OSDs (one node at a time)**. Check the target release's own upgrade notes on [docs.ceph.com](https://docs.ceph.com/en/latest/releases/) first — some releases require an extra step (e.g. `ceph osd require-osd-release <name>`) once every daemon is upgraded, not assumed here since it depends which two releases you're moving between.

**1. Point at the new release's repo** (all OSD nodes):
```sh
CEPH_UPGRADE_RELEASE=tentacle   # v20.x — current stable as of 2026-09 (20.2.4); reverify at docs.ceph.com/en/latest/releases
sudo sed -i "s/debian-${CEPH_DEPLOY_RELEASE}/debian-${CEPH_UPGRADE_RELEASE}/" /etc/apt/sources.list.d/ceph.list
sudo apt update
apt-cache madison ceph-common
CEPH_UPGRADE_VERSION="20.2.4-1~$(lsb_release -sc)"   # confirm this exact string against the madison output above
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
Repeat on the other workers (including `k8s-work-4` on GPU).

**6. Clear `noout` and do a final check:**
```sh
sudo ceph osd unset noout
sudo ceph versions   # every daemon should report the new version
sudo ceph -s          # HEALTH_OK
```

## Prerequisites
- [06-ceph.md](06-ceph.md) healthy
- [11-update-kubernetes.md](11-update-kubernetes.md)

## Next
- [13-troubleshooting.md](13-troubleshooting.md)
