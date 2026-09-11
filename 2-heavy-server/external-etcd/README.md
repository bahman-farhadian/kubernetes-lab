# Heavy — External etcd

**Status:** Not started. Per the [rollout plan](../../README.md#rollout-plan), this profile is built only after **Light** (`../../1-light-laptop/external-etcd/`) has been deployed and validated end to end, including the upgrade exercise in its `15-day2-operations.md`.

## How to build this out

The procedure is the same as `1-light-laptop/external-etcd/` — same steps, same commands, same package-hold discipline — because the hardware shape is the same (bastion, 2 control-plane, 3 dedicated etcd, 3 workers), just bigger. When you're ready:

1. Copy every file from `../../1-light-laptop/external-etcd/` into this directory.
2. Re-point hardware/sizing references at the **Heavy/GPU — Scenario B** table in [02-hardware-inventory.md](../../02-hardware-inventory.md#heavygpu--scenario-b-external-etcd) instead of the Light table (IP addresses are identical — only vCPU/RAM/disk sizing and the note about `k8s-monitor` not existing on this host differ; Prometheus/Grafana live on `k8s-bastion` instead, per `13-observability.md`'s note in that doc). Drop the `k8s-work-4` references — that's the GPU profile's job, not Heavy's.
3. Carry over anything Light's real run taught you (a flag that needed adjusting, a version that turned out unavailable, etc.) — don't blindly copy stale corrections.
4. Delete this README once the real files are in place.

## Reference
- Profile/scenario definitions: [00-overview.md](../../00-overview.md#deployment-profiles), [01-scenarios.md](../../01-scenarios.md)
- Hardware: [02-hardware-inventory.md](../../02-hardware-inventory.md)
- Network: [03-network-plan.md](../../03-network-plan.md)
