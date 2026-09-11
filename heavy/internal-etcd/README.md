# Heavy — Internal (Stacked) etcd

**Status:** Not started. Per the [rollout plan](../../README.md#rollout-plan), this profile is built only after **Light** (`../../light/internal-etcd/`) has been deployed and validated end to end, including the upgrade exercise in its `15-day2-operations.md`.

## How to build this out

The procedure is the same as `light/internal-etcd/` — same steps, same commands, same package-hold discipline — because the hardware shape is the same (bastion, 3 control-plane, 3 workers), just bigger. When you're ready:

1. Copy every file from `../../light/internal-etcd/` into this directory.
2. Re-point hardware/sizing references at the **Heavy** table in [02-hardware-inventory.md](../../02-hardware-inventory.md#server--heavy-and-gpu-profiles) instead of the Light table (IP addresses are identical — only vCPU/RAM/disk sizing and the note about `k8s-monitor` not existing on this host differ; Prometheus/Grafana live on `k8s-bastion` instead, per `13-observability.md`'s note in that doc).
3. Carry over anything Light's real run taught you (a flag that needed adjusting, a version that turned out unavailable, etc.) — don't blindly copy stale corrections.
4. Delete this README once the real files are in place.

## Reference
- Profile/scenario definitions: [00-overview.md](../../00-overview.md#deployment-profiles), [01-scenarios.md](../../01-scenarios.md)
- Hardware: [02-hardware-inventory.md](../../02-hardware-inventory.md)
- Network: [03-network-plan.md](../../03-network-plan.md)
