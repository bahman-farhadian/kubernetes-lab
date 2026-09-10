# 12. Ingress

**Goal:** Expose services outside the cluster.

**Choice:** Traefik — ingress-nginx is being sunset upstream, so this lab standardizes on Traefik instead. No cert-manager for this lab (internal-only); revisit if you expose services externally.

## Steps

**1. Install a pinned Traefik chart version via Helm** (check [github.com/traefik/traefik-helm-chart](https://github.com/traefik/traefik-helm-chart) for the current version list):
```sh
helm repo add traefik https://traefik.github.io/charts && helm repo update
helm search repo traefik/traefik --versions | head   # pick an exact chart version
TRAEFIK_CHART_VERSION="<version from the list above>"
kubectl create namespace traefik
helm install traefik traefik/traefik -n traefik --version "${TRAEFIK_CHART_VERSION}"
```
Helm has no `apt-mark hold` equivalent — the pin *is* the control: only ever `helm upgrade` this release with an explicit `--version` you chose deliberately, never omit it.

**2. Expose it** — since there's no cloud LoadBalancer here, use a `NodePort` (or `hostNetwork`) Service and point the HAProxy on `k8s-bastion` at it, the same way it already fronts the apiserver in [07-load-balancer.md](07-load-balancer.md):
```sh
kubectl get svc -n traefik   # note the NodePort for 80/443
```
Add a second HAProxy frontend/backend pair on the bastion for ports 80/443, backending to `<worker-ip>:<NodePort>` for each worker.

**3. Verify** with a throwaway `IngressRoute`/`Ingress` and `curl` through the bastion.

## Applies to
Both scenarios.

## Prerequisites
- [11-storage-ceph.md](11-storage-ceph.md)

## Next
- [13-observability.md](13-observability.md)
