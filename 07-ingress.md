# 07. Ingress

**Goal:** Expose services outside the cluster.

**Choice:** Traefik — ingress-nginx is being sunset upstream, so this lab standardizes on Traefik instead. No cert-manager for this lab (internal-only); revisit if you expose services externally.

## Steps

**1. On `k8s-bastion`** (Helm and kubeconfig from [05-deploy-kubernetes.md](05-deploy-kubernetes.md)), install a pinned Traefik chart version (check [github.com/traefik/traefik-helm-chart](https://github.com/traefik/traefik-helm-chart) for the current version list):
```sh
helm repo add traefik https://traefik.github.io/charts && helm repo update
helm search repo traefik/traefik --versions | head   # pick an exact chart version
TRAEFIK_CHART_VERSION="<version from the list above>"
kubectl create namespace traefik
helm install traefik traefik/traefik -n traefik --version "${TRAEFIK_CHART_VERSION}"
```
Helm has no `apt-mark hold` equivalent — the pin *is* the control: only ever `helm upgrade` this release with an explicit `--version` you chose deliberately, never omit it.

**2. Expose it** — since there's no cloud LoadBalancer here, use a `NodePort` (or `hostNetwork`) Service and point HAProxy on the API pair (`k8s-lb-1` / `k8s-lb-2`) at it. Add the frontends to the same `haproxy.cfg` that already binds `10.0.1.10:6443`:
```sh
kubectl get svc -n traefik   # note the NodePort for 80/443
```
On **both** load-balancer nodes, add frontend/backend pairs for ports 80 and 443, backending to `<worker-ip>:<NodePort>` for each worker, then `sudo systemctl reload haproxy`. `ip_nonlocal_bind` from the API section already covers these binds.

**3. Verify** with a throwaway `IngressRoute`/`Ingress` and `curl` to the load-balancer VIP (`http://10.0.1.10/`), not to the bastion.

## Prerequisites
- [06-ceph.md](06-ceph.md)

## Next
- [08-observability.md](08-observability.md)
