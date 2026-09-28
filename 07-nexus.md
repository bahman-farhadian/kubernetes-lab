# 07. Bastion Docker (Nexus, Prometheus, Grafana)

**Goal:** On `k8s-bastion` only, install a **single-node Docker Engine** and run **three separate Compose projects** — one directory and one `compose.yaml` each:

| App | Directory | Host ports |
|---|---|---|
| Nexus | `/opt/nexus` | 8081 (UI/apt), 8082 (docker connector) |
| Prometheus | `/opt/prometheus` | 9090 |
| Grafana | `/opt/grafana` | 3000 |

Do **not** put all three in one Compose file. Upgrade and restart stay independent.

kubelet/containerd/Ceph/keepalived/HAProxy stay **systemd** on their own VMs. Do not install Docker on control-plane or worker nodes.

The first `docker compose pull` for these three images still hits the internet (via the firewall pair). After Nexus is up, apt and **cluster** image pulls go through the cache.

**Applies to:** `k8s-bastion` only.

## Why Docker here and systemd on the cluster

Docker Engine on this VM is the host daemon for lab-support processes. It is not Kubernetes' container runtime. Mixing Docker with kubelet on the same node is out of scope.

## Steps — Docker Engine

**1. Install a pinned Docker Engine** from Docker's apt repo (check [docs.docker.com](https://docs.docker.com/engine/install/) for Debian 13 / Ubuntu 26 before adding the repo):

```sh
apt-cache madison docker-ce docker-ce-cli containerd.io docker-compose-plugin
DOCKER_CE_VERSION="<version from madison>"
sudo apt install -y docker-ce=${DOCKER_CE_VERSION} docker-ce-cli=${DOCKER_CE_VERSION} \
  containerd.io=<matching> docker-compose-plugin=<matching>
sudo apt-mark hold docker-ce docker-ce-cli containerd.io docker-compose-plugin
sudo systemctl enable --now docker
sudo docker info
```

`containerd.io` here is Docker's runtime on the **bastion only**. Cluster nodes use distro `containerd` from [08-container-runtime.md](08-container-runtime.md).

```sh
sudo mkdir -p /opt/nexus /opt/prometheus /opt/grafana
```

Pin image tags (confirm on Docker Hub; examples below were current-ish in 2026-09).

## Steps — `/opt/nexus/compose.yaml`

```yaml
services:
  nexus:
    image: sonatype/nexus3:3.96.3
    restart: unless-stopped
    ports:
      - "8081:8081"
      - "8082:8082"
    volumes:
      - nexus-data:/nexus-data
    ulimits:
      nofile: 65536

volumes:
  nexus-data:
```

```sh
cd /opt/nexus && sudo docker compose up -d
curl -s -o /dev/null -w "%{http_code}\n" http://127.0.0.1:8081
```

First start can take a minute. Change the admin password in the UI (`http://k8s-bastion:8081`). **Do not commit it.**

## Steps — `/opt/prometheus`

`/opt/prometheus/prometheus.yml` — scrape every node in the [05-os-baseline.md](05-os-baseline.md) hosts file (example LAN). External etcd: drop `.14`, add `.15/.16/.17`. GPU: add `.24`. Targets stay `DOWN` until [15-observability.md](15-observability.md) installs `node_exporter`.

```yaml
global:
  scrape_interval: 15s
scrape_configs:
  - job_name: node
    static_configs:
      - targets:
          - 10.0.1.1:9100
          - 10.0.1.2:9100
          - 10.0.1.11:9100
          - 10.0.1.12:9100
          - 10.0.1.13:9100
          - 10.0.1.14:9100
          - 10.0.1.21:9100
          - 10.0.1.22:9100
          - 10.0.1.23:9100
```

`/opt/prometheus/compose.yaml`:

```yaml
services:
  prometheus:
    image: prom/prometheus:v2.55.1
    restart: unless-stopped
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus-data:/prometheus

volumes:
  prometheus-data:
```

```sh
cd /opt/prometheus && sudo docker compose up -d
curl -s -o /dev/null -w "%{http_code}\n" http://127.0.0.1:9090/-/ready
```

## Steps — `/opt/grafana/compose.yaml`

Grafana cannot use Compose DNS `prometheus` because it is a different project. Reach Prometheus on the host:

```yaml
services:
  grafana:
    image: grafana/grafana:11.5.2
    restart: unless-stopped
    ports:
      - "3000:3000"
    extra_hosts:
      - "host.docker.internal:host-gateway"
    volumes:
      - grafana-data:/var/lib/grafana
    environment:
      GF_SECURITY_ADMIN_USER: admin
      # GF_SECURITY_ADMIN_PASSWORD from a local env file — do not commit it

volumes:
  grafana-data:
```

```sh
cd /opt/grafana && sudo docker compose up -d
curl -s -o /dev/null -w "%{http_code}\n" http://127.0.0.1:3000
```

Datasource URL from inside Grafana: `http://host.docker.internal:9090`. From a browser on the LAN: `http://k8s-bastion:3000`.

HAProxy stays **systemd** in [09-load-balancer.md](09-load-balancer.md).

## Steps — Nexus proxy repositories

Create apt proxies for the distro of *this* cluster only, plus docker proxies. Enable a LAN HTTP docker connector on **8082**.

| Name (example) | Format | Remote URL |
|---|---|---|
| `apt-debian` / `apt-debian-security` | apt | Debian mirrors (Debian 13 instance only) |
| `apt-ubuntu` | apt | Ubuntu archive (Ubuntu 26 instance only) |
| `apt-kubernetes` | apt | `https://pkgs.k8s.io/core:/stable:/v1.36/deb/` |
| `apt-ceph` | apt | `https://download.ceph.com/debian-squid/` |
| `docker-dockerio` | docker | `https://registry-1.docker.io` |
| `docker-k8s` | docker | `https://registry.k8s.io` |
| `docker-quay` | docker | `https://quay.io` |

## Steps — point the cluster at Nexus

**Apt** on every other node — URI host `http://k8s-bastion:8081/repository/<name>/`. Keep upstream `signed-by` keyrings.

**containerd mirrors** on ctrl/workers — [08-container-runtime.md](08-container-runtime.md).

## Prerequisites
- [06-firewall.md](06-firewall.md)

## Next
- [08-container-runtime.md](08-container-runtime.md)
