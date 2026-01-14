# Network Performance Testing

This directory contains Kubernetes resources for testing RDMA/InfiniBand network performance and NCCL collective operations across all worker nodes.

## Table of Contents
- [Purpose](#purpose)
- [Prerequisites](#prerequisites)
- [Universal Deployment](#universal-deployment)
- [Usage](#usage)
- [Test Architecture](#test-architecture)
- [What Gets Tested](#what-gets-tested)
- [Test Results](#test-results)
- [Container Image](#container-image)
- [Expected Results](#expected-results)
- [Troubleshooting](#troubleshooting)
- [Notes](#notes)

## Purpose

Run these tests **AFTER** baremetal deployment completes to validate:
- InfiniBand/RDMA connectivity and bandwidth using `ib_write_bw`
- NCCL collective operations performance (all-reduce, all-gather, etc.)
- Multi-node GPU communication performance

## Prerequisites

- Baremetal deployment completed successfully
- MOFED drivers installed on all worker nodes
- GPU operator deployed (for NCCL tests)
- SR-IOV networks configured

## Universal Deployment

**This test suite automatically adapts to any cluster configuration:**
- **Auto-discovers** all available SR-IOV networks (no hardcoded network names)
- **Works with any** Mellanox NIC model (deviceID auto-detected during deployment)
- **Supports any** interface naming scheme (ens3f0np0, p2, eth0, etc.)
- **Scales** from 2 to 20+ worker nodes automatically

The test coordinator Job:
1. Discovers all SR-IOV networks in the cluster
2. Dynamically creates test worker pods with correct network attachments
3. Runs comprehensive all-to-all performance tests
4. Generates interactive HTML report with network topology visualization
5. Serves results via HTTPS web interface

No manual configuration needed - just apply and run!

## Usage

### Quick Test

```bash
# Apply the network performance test resources
oc apply -k manifests/99-network-perf-tests

# Watch the test coordinator
oc logs -n default job/network-perf-test -f

# View web report (URL shown in job logs)
# The report is available via HTTPS route at:
# https://network-perf-report-default.apps.<cluster-domain>
```

### Viewing Results

**Option 1: Web Report (Recommended)**

After tests complete, an interactive HTML report is automatically generated and served via HTTPS:

```bash
# Get the report URL
oc get route network-perf-report -n default -o jsonpath='{.spec.host}'

# Open in browser: https://<route-url>
```

The web report includes:
- Interactive SVG network topology diagram
- All performance test results (RDMA, TCP, NCCL)
- Hardware specifications and routing information
- Color-coded bandwidth metrics

**Option 2: View Raw Logs**

```bash
# View full test results in logs
oc logs -n default job/network-perf-test
```

### Cleanup

**By default, worker pods remain running** after tests complete to allow:
- Inspecting pod state and logs
- Running additional manual tests
- Accessing the web report

**Manual cleanup (required):**

```bash
# Delete worker pods
oc delete daemonset network-perf-test-worker -n default

# Delete the test coordinator Job and web service
oc delete -k manifests/99-network-perf-tests
```

**To enable automatic cleanup:**

If you want worker pods deleted automatically after tests, edit `manifests/99-network-perf-tests/network-perf-test.yaml`:

```yaml
env:
- name: CLEANUP_WORKERS
  value: "true"  # Auto-delete worker pods when tests complete
```

## Test Architecture

### Multi-Port HCA Optimization

The test suite automatically detects multi-port HCAs and optimizes testing to avoid contention:

**Dual-port HCAs (phys_port_cnt=2):**
- Uses `ib_write_bw -O` flag to test both ports simultaneously with a single process
- Only ONE parallel test runs per RDMA device (deduplication)

**4-port (or higher) HCAs (phys_port_cnt=4+):**
- Cannot use `-O` flag (only supports 2-port mode)
- Only ONE VF is tested per RDMA device in parallel mode (deduplication)
- Prevents multiple parallel processes from competing on the same physical HCA

**Single-port VFs (phys_port_cnt=1):**
- Each VF is tested independently
- No deduplication needed (each VF is a separate device)

**Example scenarios:**
- **2 VFs from same 2-port HCA**: One test with `-O` flag (both ports)
- **4 VFs from same 4-port HCA**: One test (first VF only, no `-O`)
- **2 VFs from different HCAs**: Two separate tests

### Pods Without RDMA Devices

If worker pods don't have RDMA/InfiniBand devices:
- RDMA tests are automatically skipped
- TCP bandwidth tests still run normally
- NCCL tests still run (if GPUs are available)

This allows testing mixed environments with both RDMA-capable and non-RDMA nodes.

## What Gets Tested

### 1. InfiniBand/RoCE Bandwidth Tests (ib_write_bw)

**Comprehensive all-to-all testing:**
- Tests from **one source pod to all other target pods** (pod1→pod2, pod1→pod3, pod1→pod4, pod1→pod5, etc.)
- Tests **each SR-IOV NIC separately** (net1, net2, net3, etc.)
- Runs **three types of tests per NIC per pair**:
  - **RoCE (Host Memory)**: Standard RDMA write using host memory buffers
  - **RoCE with CUDA**: RDMA transfers using CUDA memory buffers via `--use_cuda`
  - **TCP**: Standard TCP socket performance as baseline comparison
- **Test duration**: 20 seconds per test
- **Message size**: 1 MB (1048576 bytes)
- Reports bandwidth in MB/s and Gb/s

**Example for 2 nodes with 2 NICs:**
- Unidirectional pairs: 1 (pod1→pod2)
- NICs per pair: 2 (net1, net2)
- Tests per NIC: 3 (RoCE, RoCE+CUDA, TCP)
- Total tests: 1 × 2 × 3 = **6 bandwidth tests**

**Example for 5 nodes with 2 NICs:**
- Unidirectional pairs: 4 (pod1→pod2, pod1→pod3, pod1→pod4, pod1→pod5)
- NICs per pair: 2
- Tests per NIC: 3 (RoCE, RoCE+CUDA, TCP)
- Total tests: 4 × 2 × 3 = **24 bandwidth tests**

### 2. NCCL Performance Tests

**Single-Node GPU Tests** (per worker pod):
- **all_reduce_perf**: Tests collective reduction operations
- **all_gather_perf**: Tests collective gather operations
- **broadcast_perf**: Tests broadcast operations
- **reduce_scatter_perf**: Tests reduce-scatter operations
- Reports bandwidth for different message sizes (8B to 128MB)

**Multi-Node GPU Tests** (using torchrun):
- Tests actual network bandwidth for multi-GPU workloads
- Uses all available SR-IOV NICs automatically (net1, net2, etc.)
- Enables GPUDirect RDMA for optimal GPU-to-GPU transfers
- Tests message sizes: 4MB, 16MB, 64MB, 128MB
- Uses PyTorch distributed NCCL backend

**NCCL Environment Variables** (automatically set on worker pods):
- `NCCL_SOCKET_IFNAME=net1,net2` - Use all SR-IOV NICs
- `NCCL_IB_HCA=mlx5_2,mlx5_3` - Use all RDMA devices
- `NCCL_NET_GDR_LEVEL=5` - Enable GPUDirect RDMA
- `NCCL_DEBUG=WARN` - Minimal debug output

## Test Results

**Interactive Web Report:**

After tests complete, access the interactive HTML report via HTTPS:

```bash
# Get the report URL
oc get route network-perf-report -n default -o jsonpath='{.spec.host}'

# Open in browser
```

The web report includes:
- **Network Topology**: Interactive SVG diagram showing all worker nodes, NICs, and connections
- **Performance Tables**: RDMA (RoCE/RoCE+CUDA), TCP, and NCCL results with color-coded metrics
- **Hardware Info**: GPU, CPU, RAM, NIC specifications
- **Routing Info**: Network routes for each SR-IOV interface
- **Aggregate Bandwidth**: Combined throughput across all NICs

**Raw Logs:**

```bash
# View full test results in logs
oc logs -n default job/network-perf-test

# Summary section shows:
# - IB bandwidth between each node pair
# - TCP baseline performance
# - Single-node and multi-node NCCL performance metrics
# - Any failures or issues detected
```

## Container Image

### Option 1: Custom Pre-Built Image (Recommended)

Build the custom image with **all tools pre-installed** for faster pod startup:

```bash
# Build the image (from manifests/99-network-perf-tests directory)
podman build -t quay.io/youruser/nccl-perf-test:universal -f Dockerfile .

# Push to your registry
podman push quay.io/youruser/nccl-perf-test:universal

# Update the DaemonSet image in network-perf-test.yaml:
# image: quay.io/youruser/nccl-perf-test:universal
```

**Custom image includes:**
- NCCL tests pre-compiled for **all NVIDIA GPU architectures** (V100, A100, H100, H200, etc.)
- perftest (ib_write_bw, ib_read_bw, etc.)
- All network and analysis tools (iproute2, jq, bc, numactl, etc.)
- GPU detection script (nvidia-smi, lspci)

**Benefits:**
- **Much faster startup**: No 2-3 minute compilation wait
- **Universal**: Works on any NVIDIA GPU (V100 to H200)
- **Consistent**: Same tools across all nodes

### Option 2: Base Image (Default)

The tests default to `nvcr.io/nvidia/pytorch:24.01-py3` and install tools at runtime:
- Takes 2-3 minutes per pod to install and compile
- Still works, just slower to start

## Expected Results

### RoCE/InfiniBand Bandwidth

**RoCE (Host Memory):**
- HDR InfiniBand (100 Gbps): 90-100 Gb/s (~11-12 GB/s)
- NDR InfiniBand (200 Gbps): 180-200 Gb/s (~22-25 GB/s)

**RoCE with CUDA:**
- Should be similar to host memory RoCE performance
- May be lower if PCIe bottleneck exists or CUDA buffer copies add overhead
- With proper setup (PCIe Gen4/5): 85-95 Gb/s for 100G, 170-190 Gb/s for 200G

**TCP Baseline:**
- Significantly lower than RoCE performance (typically 10-40 Gb/s on 100G links)
- Performance varies based on PCIe generation, CPU, and system configuration
- Useful for comparison and diagnosing RoCE issues

**Per-NIC Results:**
- Each SR-IOV NIC (net1, net2, etc.) should show similar performance
- Asymmetric results may indicate cabling, switch, or NIC issues

### NCCL Performance
- Varies based on message size and operation type
- Check bus bandwidth utilization
- Compare with NVIDIA specifications for your GPU model

### Test Duration
- With 2 nodes, 2 NICs: ~5-7 minutes total
- With 5 nodes, 2 NICs: ~35-45 minutes total
- With 20 nodes, 2 NICs: ~240-260 minutes total (4-4.5 hours)

## Troubleshooting

### No RDMA devices found
- Check MOFED driver status: `oc exec <pod> -- ibstat`
- Verify SR-IOV networks are attached: `oc exec <pod> -- ip a`
- Check SR-IOV policies are applied to worker nodes

### NCCL tests fail
- Verify GPU operator deployed successfully
- Check GPU availability: `oc exec <pod> -- nvidia-smi`
- Review NCCL compilation logs in worker pod

### Low bandwidth results
- Check for network congestion or errors: `oc exec <pod> -- ibstat`
- Verify RDMA is enabled on SR-IOV networks
- Check for CPU throttling or resource limits

### RoCE with CUDA tests fail
- Verify GPU is available: `oc exec <pod> -- nvidia-smi`
- Check CUDA runtime is working properly
- Verify MOFED driver with GPU support is installed
- Check for CUDA errors in test logs

## Notes

- Tests run in the `default` namespace
- **Worker pods remain running** by default after tests complete (manual cleanup required)
- Job persists until manually deleted to allow reviewing results at any time
- **Web report available via HTTPS** route (automatically generated after tests)
- Uses DaemonSet to deploy test pods on **all worker nodes** (scales to any cluster size)
- Requires nodes to be labeled with `node-role.kubernetes.io/worker`
- Tests use **all SR-IOV networks** configured during deployment (automatically detected)
- Tests from **source pod (pod 1) to all target pods** unidirectionally
- Each pair tests **all NICs separately** (net1, net2, etc.)
- Each NIC runs **three test modes**: RoCE (host memory), RoCE with CUDA, and TCP
- Test duration: **30 seconds per test** for stable results
- **Multi-node NCCL tests** run using `torchrun` with all available NICs
- **NCCL environment variables** pre-configured for optimal multi-GPU performance
- For large clusters (20+ nodes), tests may take several hours to complete
- HTML report generator uses Python modules mounted via ConfigMap

