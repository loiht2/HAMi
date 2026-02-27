# Fork Usage & Differences from Upstream

This document describes what this fork (`loiht2/HAMi`) changes compared to the upstream [`Project-HAMi/HAMi`](https://github.com/Project-HAMi/HAMi), and how to install it via Helm on a fresh cluster.

---

## Table of Contents

1. [Overview of Changes](#overview-of-changes)
2. [Repository Structure](#repository-structure)
3. [What Changed: Code](#what-changed-code)
4. [What Changed: Helm Chart](#what-changed-helm-chart)
5. [Prerequisites](#prerequisites)
6. [Helm Installation](#helm-installation)
7. [Verifying the Deployment](#verifying-the-deployment)
8. [Available Metrics](#available-metrics)

---

## Overview of Changes

This fork extends HAMi's DRA (Dynamic Resource Allocation) subsystem to provide **accurate real-time GPU memory metrics per container** using NVML, in addition to the HAMi-core tracked values.

| Feature | Upstream | This Fork |
|---------|----------|-----------|
| DRA driver deployment | Requires separate HAMi-DRA chart | Embedded as local subchart in `charts/hami/` |
| DRA enabled by default | `dra.enabled: false` | `dra.enabled: true` |
| Monitor deployment mode | Single Deployment | Node-level DaemonSet (accesses shared memory per node) |
| Real GPU memory usage (NVML) | Not available | `vGPU_device_memory_usage_real_in_MiB` via NVML |
| Memory metrics in MiB | Not available | All 6 memory metrics exposed in MiB and bytes |
| Container GPU driver | `containerDriver: true` (bundled) | `containerDriver: false` (uses host driver) |
| Images | `ghcr.io/projecthami/k8s-dra-driver:v0.0.1-dev` | `docker.io/loihoangthanh1411/hami-dra:v1.3` |

---

## Repository Structure

This fork includes three modified sub-repositories that are checked in or referenced:

```
HAMi/
├── charts/hami/                        # ← Modified: HAMi parent Helm chart
│   ├── Chart.yaml                      # ← hami-dra dependency: local file://
│   ├── values.yaml                     # ← DRA enabled, custom images v1.3
│   └── charts/
│       ├── hami-dra/                   # ← Embedded HAMi-DRA subchart source
│       └── hami-dra-0.1.0.tgz         # ← Built by helm dependency build
├── HAMi-DRA/                           # ← Modified subchart source (submodule)
│   └── charts/hami-dra/
└── k8s-dra-driver/                     # ← Modified DRA driver + monitor (submodule)
    └── cmd/
        ├── hami-dra/                   # ← DRA driver binary
        └── hami-dra-monitor/           # ← Monitor with real memory metrics
```

---

## What Changed: Code

### 1. HAMi-core — `memory_monitor_watcher` Thread

**Repository:** [`loiht2/HAMi-core`](https://github.com/loiht2/HAMi-core) (branch: `feat/fix-memory-monitor`)

A background thread was added to `HAMi-core` (the LD_PRELOAD CUDA interceptor library) that periodically writes the **real GPU memory usage per container** into shared memory using NVML.

- **File:** `src/memory_monitor.c` (new)
- **Function:** `memory_monitor_watcher()` — polls `nvmlDeviceGetComputeRunningProcesses()` every 1 second, matches PIDs inside the container's cgroup, and writes total usage into `monitorused[]` in the existing shared memory struct (`sharedRegionT`)
- **Impact:** All existing HAMi-core shared memory consumers (monitor, scheduler) can now read real NVML-reported usage without spawning `nvidia-smi`

### 2. HAMi-DRA Monitor — `DeviceMemoryMonitor()` and MiB Metrics

**Repository:** [`loiht2/HAMi-DRA`](https://github.com/loiht2/HAMi-DRA) (branch: `feat/dra-monitor-container-dirs`)

The monitor daemon was extended to:

- **`DeviceMemoryMonitor(idx)`**: reads `shm.monitorused` from the HAMi-core shared memory file at `/tmp/hami/vgpu-<containerID>` and exports it as `vGPU_device_memory_usage_real_in_MiB`
- **MiB metrics**: added MiB (mebibyte) versions of all 6 existing byte-scale memory metrics for human readability in dashboards

**New metrics added:**

| Metric | Description |
|--------|-------------|
| `vGPU_device_memory_usage_real_in_MiB` | Real GPU memory usage per container from NVML (matches `nvidia-smi`) |
| `vGPU_device_memory_usage_real_in_bytes` | Same, in bytes |
| `vGPU_device_memory_usage_in_MiB` | HAMi-core tracked usage in MiB |
| `vGPU_device_memory_limit_in_MiB` | vGPU memory limit in MiB |
| `vGPU_device_memory_buffer_size_MiB` | Buffer size in MiB |
| `vGPU_device_memory_context_size_MiB` | Context size in MiB |
| `vGPU_device_memory_module_size_MiB` | Module size in MiB |

### 3. NodeLevel Monitor (DaemonSet instead of Deployment)

The monitor was changed from a single `Deployment` to a **node-level DaemonSet** (`nodeLevel.enabled: true`). This is required because HAMi-core shared memory files exist at the host path `/tmp/hami/` on each GPU node — a single Deployment would only see shared memory from the node it runs on.

---

## What Changed: Helm Chart

### `charts/hami/Chart.yaml`

The `hami-dra` subchart dependency was changed from a remote Helm repository to a **local embedded chart**:

```yaml
# Before (upstream):
dependencies:
  - name: hami-dra
    version: "0.1.0"
    repository: "https://project-hami.github.io/HAMi-DRA/"

# After (this fork):
dependencies:
  - name: hami-dra
    version: "0.1.0"
    repository: "file://charts/hami-dra"
```

This means `helm install` works directly from the cloned repo with no external Helm repo access needed.

### `charts/hami/values.yaml`

Key value changes:

```yaml
dra:
  enabled: true                   # was: false

hami-dra:
  webhook:
    image:
      tag: "0.1.0"                # corrects upstream's wrong "v0.1.0" tag
      pullPolicy: IfNotPresent

  monitor:
    enabled: true
    nodeLevel:
      enabled: true               # was: false — DaemonSet per node
      hookPath: /usr/local/vgpu   # where HAMi-core installs vgpu libraries
    image:
      registry: docker.io
      repository: loihoangthanh1411/hami-dra-monitor
      tag: "v1.3"
      pullPolicy: IfNotPresent

  drivers:
    nvidia:
      containerDriver: false      # was: true — use host GPU driver
      image:
        registry: docker.io
        repository: loihoangthanh1411/hami-dra
        tag: "v1.3"
        pullPolicy: IfNotPresent
```

---

## Prerequisites

- Kubernetes cluster with DRA feature gates enabled (`DynamicResourceAllocation=true`)
- NVIDIA GPU nodes with NVIDIA driver installed directly on the host (not containerized)
- HAMi-core (`libvgpu.so`) installed on GPU nodes at `/usr/local/vgpu/` (this is done by the existing HAMi device-plugin or manually)
- `cert-manager` installed in the cluster (required for webhook TLS certificates)
- Helm 3.x

Verify DRA is enabled:

```bash
kubectl get node -o jsonpath='{.items[0].metadata.annotations}' | grep dra
# or check kube-apiserver flags for --feature-gates=DynamicResourceAllocation=true
```

Verify cert-manager:

```bash
kubectl get pods -n cert-manager
```

---

## Helm Installation

### Step 1: Clone the fork

```bash
git clone https://github.com/loiht2/HAMi.git
cd HAMi
```

### Step 2: Build Helm dependencies

The `hami-dra` subchart is embedded as a local source. Build the packaged `.tgz` before installing:

```bash
cd charts/hami
helm dependency build .
cd ../..
```

Expected output:
```
Saving 1 charts
Deleting outdated charts
```

### Step 3: Install

```bash
helm install hami charts/hami/ \
  --namespace hami-system \
  --create-namespace
```

### Step 4: Wait for pods

```bash
kubectl get pods -n hami-system -w
```

Expected steady state (assuming 3 GPU nodes):

```
NAME                                       READY   STATUS    NODE
hami-dra-driver-kubelet-plugin-<hash>      1/1     Running   gpu-1
hami-dra-driver-kubelet-plugin-<hash>      1/1     Running   gpu-2
hami-dra-driver-kubelet-plugin-<hash>      1/1     Running   gpu-3
hami-hami-dra-monitor-<hash>               1/1     Running   gpu-1
hami-hami-dra-monitor-<hash>               1/1     Running   gpu-2
hami-hami-dra-monitor-<hash>               1/1     Running   gpu-3
hami-hami-dra-webhook-<hash>               1/1     Running   gpu-3
```

> Nodes without a GPU will show `Init:0/1` (driver) or `ContainerCreating` (monitor) — this is expected because no NVIDIA devices exist on those nodes.

### Upgrading

To upgrade after pulling new changes:

```bash
cd charts/hami
helm dependency build .
cd ../..
helm upgrade hami charts/hami/ -n hami-system
```

### Uninstalling

```bash
helm uninstall hami -n hami-system
```

---

## Verifying the Deployment

### Check images are correct

```bash
kubectl get pods -n hami-system \
  -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{range .spec.containers[*]}{.image}{"\t"}{end}{"\n"}{end}'
```

Expected output:
```
hami-dra-driver-kubelet-plugin-...    docker.io/loihoangthanh1411/hami-dra:v1.3
hami-hami-dra-monitor-...             docker.io/loihoangthanh1411/hami-dra-monitor:v1.3
hami-hami-dra-webhook-...             ghcr.io/project-hami/hami-dra-webhook:0.1.0
```

### Check metrics endpoint

```bash
# Get monitor service ClusterIP
kubectl get svc hami-hami-dra-monitor -n hami-system

# Fetch metrics (run from within the cluster or use kubectl port-forward)
kubectl port-forward -n hami-system svc/hami-hami-dra-monitor 8080:8080 &
curl -s http://localhost:8080/metrics | grep "# HELP vGPU_device_memory"
```

---

## Available Metrics

When a GPU workload is running, the monitor exposes the following metrics on port `8080`:

### Memory Metrics (12 total)

| Metric | Unit | Source |
|--------|------|--------|
| `vGPU_device_memory_usage_in_bytes` | bytes | HAMi-core shm (cudaMalloc tracking) |
| `vGPU_device_memory_usage_in_MiB` | MiB | Same |
| `vGPU_device_memory_usage_real_in_bytes` | bytes | **NVML** (process RSS — matches `nvidia-smi`) |
| `vGPU_device_memory_usage_real_in_MiB` | MiB | **NVML** (process RSS — matches `nvidia-smi`) |
| `vGPU_device_memory_limit_in_bytes` | bytes | HAMi-core shm |
| `vGPU_device_memory_limit_in_MiB` | MiB | HAMi-core shm |
| `vGPU_device_memory_buffer_size_bytes` | bytes | HAMi-core shm |
| `vGPU_device_memory_buffer_size_MiB` | MiB | HAMi-core shm |
| `vGPU_device_memory_context_size_bytes` | bytes | HAMi-core shm |
| `vGPU_device_memory_context_size_MiB` | MiB | HAMi-core shm |
| `vGPU_device_memory_module_size_bytes` | bytes | HAMi-core shm |
| `vGPU_device_memory_module_size_MiB` | MiB | HAMi-core shm |

### Why `usage_real` > `usage`

The gap between `usage_real` and `usage` represents GPU memory that is allocated outside of `cudaMalloc` — specifically:

- **CUDA context** (~100–300 MiB): loaded by the driver when any CUDA call is first made
- **Framework buffers** (~500 MiB–1+ GiB): PyTorch, TensorFlow and other frameworks pre-allocate memory pools

`usage` (tracked by HAMi-core via `cudaMalloc` intercept) reflects only what the model explicitly allocates for tensors.  
`usage_real` (from NVML) reflects the total resident GPU memory for all processes in the container.

Example observed values for a PyTorch model:

```
vGPU_device_memory_usage_in_MiB       = 1766   (cudaMalloc tracked)
vGPU_device_memory_usage_real_in_MiB  = 2586   (NVML — matches nvidia-smi)
delta                                  = 820    (CUDA context + PyTorch caching allocator)
```
