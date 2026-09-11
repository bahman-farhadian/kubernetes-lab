# 18. Deployment Log

**Goal:** Record the exact pinned versions actually used each time a profile is deployed or upgraded. The steps in this repo use version *variables* (`KUBE_DEPLOY_VERSION`, `CEPH_DEPLOY_VERSION`, …) rather than hardcoded numbers — this doc is where the concrete values you picked for a given run actually live, since [00-overview.md](00-overview.md) can't record them for you.

## Applies to
All profiles and both etcd scenarios. Add one new row per deploy and per upgrade — don't overwrite previous rows, the history is the point (it's what makes each directory's `16-day2-operations.md` upgrade exercise checkable afterward, e.g. [1-light-laptop/internal-etcd/16-day2-operations.md](1-light-laptop/internal-etcd/16-day2-operations.md): what did we run, what did we upgrade to, when).

## How to use this doc

Copy the table below for each profile ([Light](README.md#rollout-plan) / Heavy / GPU) the first time you deploy it, then append a row every time you deploy or upgrade that profile.

### Profile: _(Light / Heavy / GPU)_ — Scenario _(A / B)_

| Date | Event | KUBE version | Containerd | Calico | Ceph release/version | Ceph-CSI chart | Traefik chart | node_exporter / Prometheus / Grafana | NVIDIA driver / toolkit / plugin (GPU only) | Notes |
|---|---|---|---|---|---|---|---|---|---|---|
| YYYY-MM-DD | Initial deploy | | | | | | | | | |
| YYYY-MM-DD | K8s upgrade | | | | | | | | | |
| YYYY-MM-DD | Ceph upgrade | | | | | | | | | |

- **Date** — when the step actually ran, not when it was planned.
- **Event** — "Initial deploy" (steps 08–11), "K8s upgrade" or "Ceph upgrade" (each directory's `16-day2-operations.md`), or anything else worth a line (cert rotation, node replaced, etc.).
- Leave columns blank if that component wasn't touched in this event — e.g. a Ceph upgrade row only needs the Ceph column filled in.

## Prerequisites
- Populated as each profile is actually deployed — nothing to fill in before then.
