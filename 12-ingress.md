# 12. Ingress

**Goal:** Expose services outside the cluster.

**Choice:** Traefik — ingress-nginx is being sunset upstream, so this lab standardizes on Traefik instead. No cert-manager for this lab (internal-only); revisit if you expose services externally.

## Steps

**1. Install Traefik via Helm** (check [github.com/traefik/traefik-helm-chart](https://github.com/traefik/traefik-helm-chart) for the current chart version before pinning):
```sh
helm repo add traefik https://traefik.github.io/charts && helm repo update
kubectl create namespace traefik
helm install traefik traefik/traefik -n traefik
```

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
