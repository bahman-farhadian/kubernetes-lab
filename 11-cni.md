# 11. CNI (Pod Networking)

**Goal:** Install a CNI plugin so nodes go `Ready` and pods get networking.

**Choice:** Calico — supports `NetworkPolicy` (used in [16-security-hardening.md](16-security-hardening.md)) and matches the `192.168.0.0/16` pod CIDR set in step 09. Same reasoning as Kubernetes/Ceph applies here too: deployed one release behind current stable, so [17-day2-operations.md](17-day2-operations.md) has a real Calico upgrade to walk through, not just a pin-and-forget.

## Steps

**1. Install the Tigera operator + Calico CRDs** (from `k8s-ctrl-1`, or wherever `~/.kube/config` is; check [github.com/projectcalico/calico/releases](https://github.com/projectcalico/calico/releases) for the current tag before running — pin it, don't track `master`):
```sh
CALICO_DEPLOY_VERSION=v3.31.7   # checked 2026-09: one minor behind current stable v3.32.x — reverify at the releases page above
kubectl create -f "https://raw.githubusercontent.com/projectcalico/calico/${CALICO_DEPLOY_VERSION}/manifests/tigera-operator.yaml"
```

**2. Apply the Calico custom resources**, with the pod CIDR matching `kubeadm init`:
```sh
curl -fsSL -o custom-resources.yaml \
  "https://raw.githubusercontent.com/projectcalico/calico/${CALICO_DEPLOY_VERSION}/manifests/custom-resources.yaml"
grep -A1 'cidr:' custom-resources.yaml   # confirm it reads 192.168.0.0/16 (default) before applying — edit it first if you used a different pod CIDR in step 09
kubectl create -f custom-resources.yaml
```

**3. Verify:**
```sh
kubectl get pods -n calico-system -w
kubectl get nodes    # all should flip to Ready once Calico pods are Running
```

## Prerequisites
- [10-join-nodes.md](10-join-nodes.md)

## Next
- [12-storage-ceph.md](12-storage-ceph.md)
