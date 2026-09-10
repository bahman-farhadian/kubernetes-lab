# 17. GPU Worker — NVIDIA (k8s-work-4)

**Goal:** Make the passed-through NVIDIA GPU on `k8s-work-4` usable by pods, via the plain Kubernetes device plugin (single workload per GPU — no MIG/time-slicing/GPU Operator, per [00-overview.md](00-overview.md)).

## Applies to
Server environment only, `k8s-work-4`. PCI passthrough/IOMMU config at the hypervisor is out of scope (see [00-overview.md](00-overview.md)) — this assumes the GPU already shows up inside the VM.

## Steps

**1. Confirm the GPU is visible in the guest:**
```sh
lspci -nnk | grep -i nvidia
```
If nothing shows up, the problem is host-level passthrough, not anything below.

**2. Install the NVIDIA driver.** Debian ships one in `contrib`/`non-free-firmware` — enable those components first if they aren't already, then:
```sh
sudo apt update
sudo apt install -y nvidia-driver firmware-misc-nonfree
sudo apt-mark hold nvidia-driver
sudo reboot
```
After reboot: `nvidia-smi` should list the card.

**3. Install nvidia-container-toolkit** (NVIDIA's own apt repo; check [github.com/NVIDIA/nvidia-container-toolkit](https://github.com/NVIDIA/nvidia-container-toolkit) for the current Debian 13/Trixie instructions before adding it — Trixie may not have a dedicated repo yet, in which case use the closest supported Debian/Ubuntu release repo):
```sh
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey \
  | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit.gpg
curl -fsSL https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list \
  | sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit.gpg] https://#' \
  | sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
sudo apt update
sudo apt install -y nvidia-container-toolkit
sudo apt-mark hold nvidia-container-toolkit
```

**4. Wire it into containerd** (installed back in [06-container-runtime.md](06-container-runtime.md)):
```sh
sudo nvidia-ctk runtime configure --runtime=containerd
sudo systemctl restart containerd
```

**5. Label the node** so the device plugin only schedules there (from `k8s-ctrl-1`):
```sh
kubectl label node k8s-work-4 gpu=nvidia
```

**6. Deploy the NVIDIA Kubernetes device plugin** as a DaemonSet, pinned to labeled nodes (check [github.com/NVIDIA/k8s-device-plugin](https://github.com/NVIDIA/k8s-device-plugin) for the current release tag before pinning):
```sh
DEVICE_PLUGIN_VERSION=v0.17.0   # verify this is still current before running
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: nvidia-device-plugin
  namespace: kube-system
spec:
  selector:
    matchLabels: {name: nvidia-device-plugin-ds}
  template:
    metadata:
      labels: {name: nvidia-device-plugin-ds}
    spec:
      nodeSelector: {gpu: nvidia}
      tolerations:
        - key: nvidia.com/gpu
          operator: Exists
          effect: NoSchedule
      containers:
        - name: nvidia-device-plugin-ctr
          image: nvcr.io/nvidia/k8s-device-plugin:${DEVICE_PLUGIN_VERSION}
          securityContext:
            allowPrivilegeEscalation: false
            capabilities: {drop: ["ALL"]}
          volumeMounts:
            - name: device-plugin
              mountPath: /var/lib/kubelet/device-plugins
      volumes:
        - name: device-plugin
          hostPath: {path: /var/lib/kubelet/device-plugins}
EOF
```

**7. Verify:**
```sh
kubectl describe node k8s-work-4 | grep -A3 "Allocatable:"   # expect nvidia.com/gpu: 1
```
Then run a throwaway pod requesting `resources.limits: {nvidia.com/gpu: 1}` and confirm `nvidia-smi` works inside it.

## Prerequisites
- [06-container-runtime.md](06-container-runtime.md)
- [09-join-nodes.md](09-join-nodes.md) — node must already be joined and `Ready`
- [10-cni.md](10-cni.md)

## Next
- None — this is the last step for environments with a GPU worker.
