# 17. GPU Worker — NVIDIA (k8s-work-4)

**Goal:** Make the passed-through NVIDIA GPU on `k8s-work-4` usable by pods, via the plain Kubernetes device plugin (single workload per GPU — no MIG/time-slicing/GPU Operator, per [00-overview.md](00-overview.md)) — and reserve the node for GPU workloads only, via a taint, so ordinary pods can't land there and waste it.

`k8s-work-4` is a VM with the physical GPU passed straight through to it (hypervisor-level PCI passthrough, out of scope here — see below); everything in this doc runs inside that guest.

## Applies to
GPU profile only, `k8s-work-4`. PCI passthrough/IOMMU config at the hypervisor is out of scope (see [00-overview.md](00-overview.md)) — this assumes the GPU already shows up inside the VM.

## Steps

**1. Confirm the GPU is visible in the guest:**
```sh
lspci -nnk | grep -i nvidia
```
If nothing shows up, the problem is host-level passthrough, not anything below.

**2. Install the NVIDIA driver.** Debian ships one in `contrib`/`non-free-firmware` — enable those components first if they aren't already, then:
```sh
sudo apt update
apt-cache madison nvidia-driver   # list exact available versions — pick one
NVIDIA_DRIVER_VERSION="<version from the list above>"
sudo apt install -y nvidia-driver=${NVIDIA_DRIVER_VERSION} firmware-misc-nonfree
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
apt-cache madison nvidia-container-toolkit   # list exact available versions — pick one
NVIDIA_CTK_VERSION="<version from the list above>"
sudo apt install -y nvidia-container-toolkit=${NVIDIA_CTK_VERSION}
sudo apt-mark hold nvidia-container-toolkit
```

**4. Wire it into containerd** (installed back in [06-container-runtime.md](06-container-runtime.md)):
```sh
sudo nvidia-ctk runtime configure --runtime=containerd
sudo systemctl restart containerd
```

**5. Label *and taint* the node** (from `k8s-ctrl-1`) — the label lets the device plugin (and later, GPU pods) target this node; the taint is what actually reserves it, repelling every ordinary pod so the GPU isn't wasted on generic scheduling:
```sh
kubectl label node k8s-work-4 gpu=nvidia
kubectl taint node k8s-work-4 nvidia.com/gpu=present:NoSchedule
```
From here on, only pods that explicitly carry the matching toleration below can land on `k8s-work-4` — that's the point.

**6. Deploy the NVIDIA Kubernetes device plugin** as a DaemonSet — it needs the toleration itself just to reach the now-tainted node (check [github.com/NVIDIA/k8s-device-plugin](https://github.com/NVIDIA/k8s-device-plugin) for the current release tag before pinning):
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
          operator: Equal
          value: present
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

**7. Verify** the taint and allocatable GPU are in place:
```sh
kubectl describe node k8s-work-4 | grep -A2 "Taints:"        # expect nvidia.com/gpu=present:NoSchedule
kubectl describe node k8s-work-4 | grep -A3 "Allocatable:"   # expect nvidia.com/gpu: 1
```
Then run a throwaway pod that opts in with the matching `nodeSelector` + `toleration` — without both, it will never be scheduled on `k8s-work-4`:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-smoke-test
spec:
  nodeSelector: {gpu: nvidia}
  tolerations:
    - key: nvidia.com/gpu
      operator: Equal
      value: present
      effect: NoSchedule
  containers:
    - name: cuda-test
      image: nvcr.io/nvidia/cuda:12.6.0-base-ubuntu24.04   # verify this tag still exists before running
      command: ["nvidia-smi"]
      resources:
        limits: {nvidia.com/gpu: 1}
```
`kubectl logs gpu-smoke-test` should show the card. Every real GPU workload later needs this same `nodeSelector`/`toleration`/`resources.limits` trio — that's the deliberate cost of tainting the node.

## Prerequisites
- [06-container-runtime.md](06-container-runtime.md)
- [09-join-nodes.md](09-join-nodes.md) — node must already be joined and `Ready`
- [10-cni.md](10-cni.md)

## Next
- [18-deployment-log.md](18-deployment-log.md) — record what you actually deployed
