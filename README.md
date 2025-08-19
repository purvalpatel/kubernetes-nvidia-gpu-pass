# GPU Passthrough in Kubernetes with NVIDIA GeForce RTX 4050

![Kubernetes GPU Support](https://img.shields.io/badge/GPU-Kubernetes%20Supported-brightgreen)

This documentation provides a complete guide to setting up GPU passthrough for Kubernetes pods using consumer-grade NVIDIA GPUs (specifically tested with GeForce RTX 4050) through time-slicing.

## Table of Contents
- [Cluster Configuration](#cluster-configuration)
- [Prerequisites](#prerequisites)
- [Setup Instructions](#setup-instructions)
  - [Package Installation](#1-package-installation)
  - [Node Labeling](#2-node-labeling)
  - [NVIDIA Runtime Configuration](#3-nvidia-runtime-configuration)
  - [Device Plugin Deployment](#4-device-plugin-deployment)
- [Verification](#verification)
- [Deployment Example](#deployment-example)
- [Troubleshooting](#troubleshooting)
- [Known Limitations](#known-limitations)
- [Useful Commands](#useful-commands)

## Cluster Configuration

| Node Type      | Hardware Specification          | GPU Support |
|----------------|----------------------------------|-------------|
| Master Node    | CPU-only                        | None        |
| Worker Node    | With NVIDIA H100                | Yes         |

> **Note**: Consumer-grade GPUs like the RTX 4050 don't support NVIDIA's MIG (Multi-Instance GPU) technology but can be shared through time-slicing.

## Prerequisites

- Kubernetes cluster (v1.20+ recommended)
- NVIDIA drivers installed on GPU node (tested with 525+)
- Containerd as container runtime
- `kubectl` configured to access your cluster

## Setup Instructions

### Package Installation

On the GPU worker node:

```bash
sudo apt-get update
sudo apt-get install -y \
    nvidia-container-toolkit \
    nvidia-container-toolkit-base \
    containerd \
    nvidia-ctk
```

### Label GPU Node (Master Node)
```bash
kubectl get nodes
kubectl label node <worker-node-name> gpu=true
```

Configure NVIDIA Runtime (GPU Worker Node)

### Generate default config:
```bash
sudo containerd config default | sudo tee /etc/containerd/config.toml
```

### Edit configuration:
```bash
sudo nano /etc/containerd/config.toml
```
Add/modify these sections:
```toml
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes]
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.nvidia]
    runtime_type = "io.containerd.runc.v2"
    [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.nvidia.options]
      BinaryName = "/usr/bin/nvidia-container-runtime"
      systemdCgroup = true

[plugins."io.containerd.grpc.v1.cri".containerd]
  default_runtime_name = "nvidia"
```
restart services:
```
sudo systemctl restart containerd
sudo systemctl restart kubelet
```
### Deploy NVIDIA Device Plugin (Master Node)
```
kubectl apply -f https://raw.githubusercontent.com/NVIDIA/k8s-device-plugin/v0.14.5/nvidia-device-plugin.yml
```

### Verification:
check GPU resources:
```
kubectl describe node <worker-node-name> | grep -A10 Allocatable
```
<img width="416" height="190" alt="image" src="https://github.com/user-attachments/assets/7a8ee12e-1bf1-4dc4-91b4-74e7356164ab" />

Verify device plugin:
```
kubectl get pods -n kube-system | grep nvidia-device-plugin
```

## Example Deployment
gpu-deployment.yaml
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gpu-sample-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: gpu-test
  template:
    metadata:
      labels:
        app: gpu-test
    spec:
      containers:
        - name: cuda-container
          image: nvidia/cuda:12.2.0-base-ubuntu22.04
          command: ["sleep", "infinity"]
          resources:
            limits:
              nvidia.com/gpu: 1
      nodeSelector:
        gpu: "true"
```

Deploy and test:
```
kubectl apply -f gpu-deployment.yaml
kubectl exec -it $(kubectl get pod -l app=gpu-test -o jsonpath="{.items[0].metadata.name}") -- nvidia-smi
```

## Troubleshooting
    Check device plugin logs:

```bash
kubectl logs -n kube-system $(kubectl get pods -n kube-system -l name=nvidia-device-plugin-ds -o jsonpath="{.items[0].metadata.name}")
```

Verify driver installation:
```bash
nvidia-smi  # On worker node
```
Test container runtime directly:
```bash
sudo ctr run --rm -t --runtime io.containerd.runc.v2 \
  --env NVIDIA_VISIBLE_DEVICES=all \
  docker.io/nvidia/cuda:12.2.0-base-ubuntu22.04 test nvidia-smi
```

Common Fixes
```bash
# Reconfigure runtime
sudo nvidia-ctk runtime configure --runtime=containerd
sudo systemctl restart containerd kubelet
```

Reinstall device plugin
```
kubectl delete -f https://raw.githubusercontent.com/NVIDIA/k8s-device-plugin/v0.14.5/nvidia-device-plugin.yml
kubectl apply -f https://raw.githubusercontent.com/NVIDIA/k8s-device-plugin/v0.14.5/nvidia-device-plugin.yml
```

Useful Commands
```bash
# Monitor GPU usage in pod
kubectl exec -it <pod-name> -- watch -n 1 nvidia-smi

# Check GPU allocation
kubectl describe node <node-name> | grep -A10 "Allocated resources"

# Get GPU pod information
kubectl get pods -o wide | grep gpu
```
