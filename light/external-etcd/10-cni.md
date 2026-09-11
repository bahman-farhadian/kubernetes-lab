# 10. CNI (Pod Networking)

**Goal:** Install a CNI plugin so nodes go `Ready` and pods get networking.

**Choice:** Calico — supports `NetworkPolicy` (used in [14-security-hardening.md](14-security-hardening.md)) and matches the `192.168.0.0/16` pod CIDR set in step 08.

## Steps

**1. Install the Tigera operator + Calico CRDs** (from `k8s-ctrl-1`, or wherever `~/.kube/config` is; check [projectcalico.org](https://projectcalico.org) for the current release tag before running — pin it, don't track `master`):
```sh
CALICO_VERSION=v3.29.0   # verify this is still current before running
kubectl create -f "https://raw.githubusercontent.com/projectcalico/calico/${CALICO_VERSION}/manifests/tigera-operator.yaml"
```

**2. Apply the Calico custom resources**, with the pod CIDR matching `kubeadm init`:
```sh
curl -fsSL -o custom-resources.yaml \
  "https://raw.githubusercontent.com/projectcalico/calico/${CALICO_VERSION}/manifests/custom-resources.yaml"
grep -A1 'cidr:' custom-resources.yaml   # confirm it reads 192.168.0.0/16 (default) before applying — edit it first if you used a different pod CIDR in step 08
kubectl create -f custom-resources.yaml
```

**3. Verify:**
```sh
kubectl get pods -n calico-system -w
kubectl get nodes    # all should flip to Ready once Calico pods are Running
```

## Applies to
Light profile, either etcd scenario.

## Prerequisites
- [09-join-nodes.md](09-join-nodes.md)

## Next
- [11-storage-ceph.md](11-storage-ceph.md)
