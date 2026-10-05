# 07. Ingress

**Goal:** Expose services outside the cluster.

**Choice:** Traefik — ingress-nginx is being sunset upstream, so this lab standardizes on Traefik instead. No cert-manager for this lab (internal-only); revisit if you expose services externally.

## Steps

**1. Traefik** — from `k8s-bastion`. The pin is the `--version` flag. Never upgrade without it.
```sh
helm repo add traefik https://traefik.github.io/charts && helm repo update
helm search repo traefik/traefik --versions | head          # copy one chart version
TRAEFIK_CHART_VERSION="<version from the list above>"
kubectl create namespace traefik
helm install traefik traefik/traefik -n traefik --version "${TRAEFIK_CHART_VERSION}"
kubectl get svc -n traefik                                  # note the NodePort for 80 and 443
```

**2. Publish 80 and 443** — on both `k8s-lb-1` and `k8s-lb-2`, add frontends to the HAProxy file that already binds `10.0.1.10:6443`. Backends are `<worker-ip>:<NodePort>`.
```sh
sudo systemctl reload haproxy    # pick up the new frontends; nonlocal bind is already set
```

**3. Check the VIP** — not the bastion address.
```sh
curl -sv --max-time 5 http://10.0.1.10/    # the API pair, port 80, after a test Ingress exists
```

## Prerequisites
- [06-ceph.md](06-ceph.md)

## Next
- [08-observability.md](08-observability.md)
