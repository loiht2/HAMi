# Fork Usage & Differences from Upstream

This document describes what this fork (`loiht2/HAMi`) changes compared to the upstream [`Project-HAMi/HAMi`](https://github.com/Project-HAMi/HAMi), and how to install it via Helm on a fresh cluster.

**Branch:** `feat/dra-monitor-container-dirs` (consistent across all repositories)

---

## Table of Contents

1. [Overview of Changes](#overview-of-changes)
2. [Repository Structure](#repository-structure)
3. [Related Repositories](#related-repositories)
4. [What Changed: Code](#what-changed-code)
5. [What Changed: Helm Chart](#what-changed-helm-chart)
6. [Prerequisites](#prerequisites)
7. [Helm Installation](#helm-installation)
8. [Verifying the Deployment](#verifying-the-deployment)
9. [Available Metrics](#available-metrics)

---

## Overview of Changes

This fork extends HAMi's DRA (Dynamic Resource Allocation) subsystem to provide **accurate real-time GPU memory metrics per container** using NVML, in addition to the HAMi-core tracked values.

| Feature | Upstream | This Fork |
|---------|----------|-----------|
| DRA driver deployment | Requires separate HAMi-DRA chart | Embedded as local subchart in `charts/hami/` |
| DRA enabled by default | `dra.enabled: false` | `dra.enabled: true` |
| Monitor deployment mode | Single Deployment | Node-level DaemonSet (accesses shared memory per node) |
| Real GPU memory usage (NVML) | Not available | `vGPU_device_memory_usage_real_in_MiB` via NVML |
| Memory metrics in MiB | Not available | All 6 memory metrics exposed in MiB |
| Container GPU driver | `containerDriver: true` (bundled) | `containerDriver: false` (uses host driver) |
| Images | `ghcr.io/projecthami/k8s-dra-driver:v0.0.1-dev` | `docker.io/loihoangthanh1411/hami-dra:v1.3` |

---

## Repository Structure

This fork includes three modified sub-repositories that work together:

```
HAMi/                                   # ← This repo: parent Helm chart
├── charts/hami/
│   ├── Chart.yaml                      # hami-dra dependency: local file://
│   ├── values.yaml                     # DRA enabled, custom images v1.5
│   └── charts/
│       ├── hami-dra/                   # Embedded HAMi-DRA subchart source
│       └── hami-dra-0.1.0.tgz         # Built by helm dependency build
```

---

## Related Repositories

All repositories use the same branch: **`feat/dra-monitor-container-dirs`**

| Repository | Role | Key Docs |
|------------|------|----------|
| [`loiht2/HAMi`](https://github.com/loiht2/HAMi) | Parent Helm chart; embeds HAMi-DRA as subchart | This README |
| [`loiht2/HAMi-core-fix-memory`](https://github.com/loiht2/HAMi-core-fix-memory) | LD_PRELOAD CUDA interceptor; fixes container memory tracking; adds `memory_monitor_watcher` thread for NVML data | [Change Coverage](https://github.com/loiht2/HAMi-core-fix-memory/blob/feat/dra-monitor-container-dirs/docs/change-coverage.md) |
| [`loiht2/HAMi-DRA`](https://github.com/loiht2/HAMi-DRA) | DRA webhook + **HAMi-DRA-monitor** DaemonSet; reads shared memory and exposes Prometheus metrics | [Monitor Docs](https://github.com/loiht2/HAMi-DRA/blob/feat/dra-monitor-container-dirs/docs/MONITOR.md) |
| [`loiht2/k8s-dra-driver`](https://github.com/loiht2/k8s-dra-driver) | DRA kubelet plugin; creates `containers/{podUID}_{containerName}/` cache dirs for HAMi-core | [DRA Monitor Integration](https://github.com/loiht2/k8s-dra-driver/blob/feat/dra-monitor-container-dirs/docs/dra-monitor-integration.md) |

---

## What Changed: Code

### 1. HAMi-core — Memory Fix & `memory_monitor_watcher` Thread

**Repository:** [`loiht2/HAMi-core-fix-memory`](https://github.com/loiht2/HAMi-core-fix-memory) (branch: `feat/dra-monitor-container-dirs`)

Three key changes were made to HAMi-core (the LD_PRELOAD CUDA interceptor library):

**a) Fix container vs host GPU memory tracking** — The NVML process memory query was comparing container-namespace PIDs against NVML-reported host PIDs, causing zero memory readings. Fixed by matching against `hostpid` instead of `pid`. Also added `get_gpu_memory_real_usage()` which returns `max(NVML, tracked)` for conservative OOM detection.

**b) Fix memory underflow guards** — `cuMemGetInfo_v2` and `nvmlDeviceGetMemoryInfo` could underflow when `usage > limit`. Now returns `free = 0` instead of wrapping or erroring.

**c) Add `memory_monitor_watcher` thread** — A new background thread in `src/multiprocess/multiprocess_utilization_watcher.c` polls `nvmlDeviceGetComputeRunningProcesses()` every 1 second and writes results into `procs[].monitorused[]` in shared memory. This runs **unconditionally** (regardless of SM limit config), ensuring the HAMi-DRA-monitor always has real NVML memory data to read.

**Files changed:** `src/allocator/allocator.c`, `src/cuda/memory.c`, `src/multiprocess/multiprocess_memory_limit.c`, `src/multiprocess/multiprocess_memory_limit.h`, `src/multiprocess/multiprocess_utilization_watcher.c`, `src/nvml/hook.c`

See the full [Change Coverage Document](https://github.com/loiht2/HAMi-core-fix-memory/blob/feat/dra-monitor-container-dirs/docs/change-coverage.md) for detailed per-file analysis.

### 2. HAMi-DRA-monitor — Real Memory Metrics via DaemonSet

**Repository:** [`loiht2/HAMi-DRA`](https://github.com/loiht2/HAMi-DRA) (branch: `feat/dra-monitor-container-dirs`)

The HAMi-DRA-monitor is deployed as a **separate node-level DaemonSet** (not a sidecar) that:

- Mounts the host's `/usr/local/vgpu/containers/` directory
- Scans HAMi-core shared memory (`.cache`) files for each `{podUID}_{containerName}` directory
- Reads `monitorused[]` from shared memory for real NVML-reported GPU memory
- Exposes per-container Prometheus metrics including `vGPU_device_memory_usage_real_in_MiB`

The DaemonSet mode is required because HAMi-core shared memory files exist at host paths on each GPU node — a single Deployment would only see shared memory from one node.

See the full [Monitor Documentation](https://github.com/loiht2/HAMi-DRA/blob/feat/dra-monitor-container-dirs/docs/MONITOR.md) for metrics reference and configuration.

| Metric | Description |
|--------|-------------|
| `vGPU_device_memory_usage_real_in_MiB` | Real GPU memory usage per container from NVML (matches `nvidia-smi`) |
| `vGPU_device_memory_usage_in_MiB` | HAMi-core tracked usage in MiB |
| `vGPU_device_memory_limit_in_MiB` | vGPU memory limit in MiB |
| `vGPU_device_memory_buffer_size_MiB` | Buffer size in MiB |
| `vGPU_device_memory_context_size_MiB` | Context size in MiB |
| `vGPU_device_memory_module_size_MiB` | Module size in MiB |

### 3. k8s-dra-driver — Cache Directory Alignment

**Repository:** [`loiht2/k8s-dra-driver`](https://github.com/loiht2/k8s-dra-driver) (branch: `feat/dra-monitor-container-dirs`)

The DRA kubelet plugin was changed to create cache directories using the same naming convention as the traditional device plugin:

```
/usr/local/vgpu/containers/{podUID}_{containerName}/
```

Instead of the previous `claims/{claimUID}/` layout. A `resolveContainerInfo()` method resolves the pod UID and container name from the ResourceClaim, including support for template-based ResourceClaims via `pod.Status.ResourceClaimStatuses`.

See the full [DRA Monitor Integration Document](https://github.com/loiht2/k8s-dra-driver/blob/feat/dra-monitor-container-dirs/docs/dra-monitor-integration.md) for architecture details.

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
      tag: "v1.5"
      pullPolicy: IfNotPresent

  drivers:
    nvidia:
      containerDriver: false      # was: true — use host GPU driver
      image:
        registry: docker.io
        repository: loihoangthanh1411/hami-dra
        tag: "v1.5"
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

### Container-level Metrics (from HAMi-core shared memory)

These metrics are read from HAMi-core shared memory by the HAMi-DRA-monitor DaemonSet running on each node.

**Base labels:** `podnamespace`, `podname`, `ctrname`, `vdeviceid`, `deviceuuid`
**Extended labels** (on `usage_real` and `limit` only): `pod_uid`, `image`, `image_id`, `device_type`

| Metric | Unit | Source |
|--------|------|--------|
| `Device_memory_desc_of_container` | bytes | HAMi-core shm (cudaMalloc tracking) |
| `Device_utilization_desc_of_container` | % | HAMi-core shm (SM utilization) |
| `Device_last_kernel_of_container` | seconds | HAMi-core shm (since last kernel) |
| `vGPU_device_memory_usage_in_MiB` | MiB | HAMi-core shm (cudaMalloc tracked, rounded) |
| `vGPU_device_memory_usage_real_in_MiB` | MiB | **NVML** (process RSS — matches `nvidia-smi`, rounded) |
| `vGPU_device_memory_limit_in_MiB` | MiB | HAMi-core shm (rounded) |
| `vGPU_device_memory_buffer_size_MiB` | MiB | HAMi-core shm (rounded) |
| `vGPU_device_memory_context_size_MiB` | MiB | HAMi-core shm (rounded) |
| `vGPU_device_memory_module_size_MiB` | MiB | HAMi-core shm (rounded) |

See the full [Monitor Documentation](https://github.com/loiht2/HAMi-DRA/blob/feat/dra-monitor-container-dirs/docs/MONITOR.md) for complete metric reference, label descriptions, and Prometheus integration.

### Pod-level Allocation Metrics (from DRA cache)

| Metric | Unit | Source |
|--------|------|--------|
| `vGPUDeviceCoreAllocated` | cores | DRA ResourceClaim allocation |
| `vGPUDeviceMemoryAllocated` | MB | DRA ResourceClaim allocation |

### Node-level GPU Metrics (from DRA cache)

| Metric | Unit | Source |
|--------|------|--------|
| `GPUDeviceMemoryLimit` | MB | DRA ResourceSlice |
| `GPUDeviceCoreLimit` | cores | DRA ResourceSlice |
| `GPUDeviceMemoryAllocated` | MB | DRA ResourceSlice |
| `GPUDeviceCoreAllocated` | cores | DRA ResourceSlice |

> **Note:** The bytes-unit variants (e.g. `vGPU_device_memory_usage_in_bytes`) are commented out in the source code and can be re-enabled if needed.

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
