# InfraBric Deployer

**Automated Baremetal NVIDIA GPU + RDMA/RoCE Infrastructure Deployment**

InfraBrig Deployer provides a complete, production-ready ArgoCD-based GitOps deployment for baremetal clusters with NVIDIA GPUs and RDMA/InfiniBand networking.

## Features

### 🚀 Core Capabilities

- **Fully Automated Deployment**: Zero-touch deployment of GPU and RDMA infrastructure
- **Self-Healing**: Automated detection and repair of stuck operators and resources
- **Universal GPU Support**: Pre-compiled images supporting V100, A100, H100, H200, and all NVIDIA architectures
- **RoCE/RDMA Auto-Discovery**: Automatic detection and configuration of RDMA-capable NICs
- **SR-IOV Automation**: Dynamic SR-IOV policy and network generation

### 🧩 Components

1. **[Node Preparation](manifests/00-node-preparation/README.md)** - Automated node labeling and GPU/MOFED exclusion from master nodes
2. **[NFD (Node Feature Discovery)](manifests/15-nfd/README.md)** - Hardware feature detection
3. **[SR-IOV Operator](manifests/30-sriov-operator/README.md)** - Network virtualization for RDMA
4. **[NVIDIA Network Operator](manifests/28-nvidia-network-operator/README.md)** - MOFED driver and RDMA configuration
5. **[NVIDIA GPU Operator](manifests/30-gpu-operator-nfd/README.md)** - GPU driver and device plugin
6. **[RoCE Discovery](manifests/35-roce-discovery/README.md)** - Automatic RDMA NIC detection and configuration
7. **[Self-Healing Monitor](manifests/27-gitops/README.md)** - Continuous health monitoring and auto-repair
8. **[Network Performance Testing](manifests/99-network-perf-tests/README.md)** - IB bandwidth and NCCL validation tools

### 🧹 Utilities

- **[Cluster Cleanup](manifests/00-cleanup/README.md)** - Complete operator removal for fresh deployments
- **[Network Performance Tests](manifests/99-network-perf-tests/README.md)** - RDMA bandwidth and NCCL testing

---

## Quick Start

### Prerequisites

- OpenShift cluster (baremetal)
- NVIDIA GPUs installed
- InfiniBand/RoCE-capable NICs
- Cluster-admin permissions

### Deployment

See the **[Baremetal Deployment Guide](rig/baremetal/bootstrap/README.md)** for detailed installation steps.

**Quick version:**

1. **Fork this repository** and update the Git URLs:

   > **Why fork?** ArgoCD continuously monitors the Git repository and **automatically applies every change** to your cluster. If you use the upstream repository directly, any changes pushed by others will be immediately deployed to your production cluster without your review or approval. Forking gives you full control over what gets deployed and when.

   ```bash
   # Update repoURL in all app.yaml files under apps/
   find apps/ -name "app.yaml" -exec sed -i '' 's|github.com/bbenshab/Infrabric-deployer|github.com/YOUR_USERNAME/Infrabric-deployer|g' {} \;
   ```

2. **Deploy prerequisites** (GitOps operator + health monitor):
   ```bash
   oc apply -f rig/baremetal/bootstrap/00-prerequisites.yaml

   # Wait for GitOps operator to be ready (~2-3 minutes)
   oc wait --for=condition=ready pod -l control-plane=gitops-operator -n openshift-operators --timeout=300s

   # Wait for ArgoCD instance to be ready (~1 minute)
   oc wait --for=condition=ready pod -l app.kubernetes.io/name=openshift-gitops-server -n openshift-gitops --timeout=180s
   ```

   **Expected output** - ArgoCD pods should be running:
   ```
   $ oc get pods -n openshift-gitops
   NAME                                                         READY   STATUS    RESTARTS   AGE
   cluster-67f5c4874b-9crvc                                     1/1     Running   0          8m35s
   gitops-plugin-598cff7645-h67nt                               1/1     Running   0          8m35s
   openshift-gitops-application-controller-0                    1/1     Running   0          8m34s
   openshift-gitops-applicationset-controller-69964ffd4-6v7fj   1/1     Running   0          8m34s
   openshift-gitops-dex-server-6dc958b54-tsqwd                  1/1     Running   0          8m33s
   openshift-gitops-redis-68469975f8-ndwnz                      1/1     Running   0          8m34s
   openshift-gitops-repo-server-75db857f8-5fqzb                 1/1     Running   0          8m34s
   openshift-gitops-server-674758b98b-nlg5c                     1/1     Running   0          8m34s
   ```

   > **Note:** You might also see `argocd-health-monitor-*` pods with `Status: Completed`. This is completely normal - they run as a CronJob every 2 minutes to monitor ArgoCD health.

3. **Configure private repository access** (ONLY if your fork is private):

   > **Public repository?** Skip this step - ArgoCD can access public repositories without credentials.

   If your forked repository is private, configure Git credentials by following the guide:

   **[Private Repository Configuration Guide](manifests/01-private-repo-config/README.md)**

4. **Deploy the bootstrap**:
   ```bash
   oc apply -k rig/baremetal/bootstrap
   ```

5. **Monitor deployment**:

   > **Important:** ArgoCD applications showing "Healthy" status does NOT mean all pods and drivers are fully deployed. The NVIDIA GPU Operator is the last component to deploy and takes the longest time. Always verify the actual pod status in the `nvidia-gpu-operator` namespace before considering the cluster ready.

   ```bash
   # Watch ArgoCD applications status
   watch oc get applications -n openshift-gitops

   # Monitor GPU operator pods (last component to deploy)
   watch oc get pods -n nvidia-gpu-operator

   # Cluster is ready when all GPU operator pods are Running:
   # - nvidia-driver-daemonset pods (one per GPU node)
   # - nvidia-dcgm-exporter pods
   # - nvidia-device-plugin-daemonset pods
   # - gpu-feature-discovery pods
   # - nvidia-operator-validator pods

   # Monitor self-healing health monitor
   oc logs -n openshift-gitops -l job-name=argocd-health-monitor --tail=100
   ```

---

## Directory Structure

```
Infrabric-deployer/
├── apps/                          # ArgoCD Application manifests
│   ├── 00-node-preparation/       # Node labeling and preparation
│   ├── 15-nfd/                    # Node Feature Discovery
│   ├── 20-operators-baremetal/    # Core infrastructure operators
│   ├── 27-gitops/                 # GitOps/ArgoCD configuration
│   ├── 28-nvidia-network-operator/ # NVIDIA MOFED and RDMA
│   ├── 29-nvidia-mofed-ready/     # MOFED readiness gate
│   ├── 30-gpu-operator-nfd/       # NVIDIA GPU operator
│   ├── 30-grafana/                # Monitoring (optional)
│   ├── 30-sriov-operator/         # SR-IOV network operator
│   └── 35-roce-discovery/         # RoCE NIC auto-discovery
├── manifests/                     # Kubernetes resource manifests
│   ├── 00-cleanup/                # Cluster cleanup job
│   ├── 00-node-preparation/       # Node prep job and scripts
│   ├── 15-nfd/                    # NFD configuration
│   ├── 20-operators/              # Helm charts for operators
│   ├── 27-gitops/                 # Self-healing monitor CronJob
│   ├── 28-nvidia-network-operator/ # NicClusterPolicy and version discovery
│   ├── 29-nvidia-mofed-ready/     # MOFED readiness check
│   ├── 30-gpu-operator-nfd/       # ClusterPolicy for GPU
│   ├── 30-grafana/                # Grafana dashboards
│   ├── 30-sriov-operator/         # SriovOperatorConfig
│   ├── 35-roce-discovery/         # RoCE discovery DaemonSet and jobs
│   └── 99-network-perf-tests/     # Network performance validation
└── rig/                           # Environment configurations
    ├── baremetal/                 # Baremetal deployment (OpenShift)
    │   ├── bootstrap/             # Initial deployment resources
    │   ├── infra-operators-values.yaml # Operator configuration values
    │   ├── kustomization.yaml     # Kustomize overlay
    │   └── namespace.yaml         # Target namespace
    ├── aws/                       # AWS deployment (coming soon)
    └── ibm-cloud/                 # IBM Cloud deployment (coming soon)

```

---

## Deployment Workflow

### ArgoCD Sync Waves

The deployment uses ArgoCD sync waves for proper ordering:

- **Wave -2**: Prerequisites (self-healing monitor, RBAC)
- **Wave -1**: Node preparation
- **Wave 0**: NFD
- **Wave 1**: SR-IOV operator
- **Wave 2**: NVIDIA Network Operator (MOFED)
- **Wave 3**: MOFED readiness gate
- **Wave 4**: GPU operator

---

## Testing & Validation

### Network Performance Tests

Test RDMA bandwidth, TCP performance, and NCCL collective operations. See **[Network Performance Tests README](manifests/99-network-perf-tests/README.md)** for details.

```bash
# Run performance tests
oc apply -k manifests/99-network-perf-tests

# View results
oc logs -n default job/network-perf-test -f

# Expected results:
# - RoCE (Host Memory): ~90-100 Gb/s (100G InfiniBand)
# - RoCE with CUDA: ~85-95 Gb/s (GPUDirect RDMA)
# - TCP Baseline: ~10-40 Gb/s (standard TCP/IP)
# - NCCL all_reduce: ~400 GB/s (GPU memory bandwidth)

# Cleanup after testing
oc delete -k manifests/99-network-perf-tests
oc delete daemonset network-perf-test-worker -n default
```

**Note:** The test coordinator Job dynamically creates a DaemonSet at runtime (not managed by kustomize). You must manually delete the DaemonSet after testing, as `oc delete -k` only removes the Job and ConfigMap template.

---

## 🧹 Cluster Cleanup

Before redeploying or when you need to start fresh, use the automated cleanup job to remove all operator resources. See **[Cleanup README](manifests/00-cleanup/README.md)** for full documentation.

### Quick Cleanup

```bash
# Apply the cleanup Job directly
oc apply -k manifests/00-cleanup

# Watch the cleanup process
oc logs -n openshift-operators job/cleanup-operators -f

# Wait for completion (auto-deletes after 10 minutes)
oc wait --for=condition=complete job/cleanup-operators -n openshift-operators --timeout=600s
```

### What Gets Cleaned Up

**IMPORTANT:** ArgoCD/GitOps is **NOT** cleaned up automatically since it manages the cleanup job itself. See "Manual ArgoCD Cleanup" below for final steps.

The cleanup job removes:
- **ArgoCD Applications:** All operator Applications (preserves gitops and root-app)
- **Operators:** GPU Operator, NVIDIA Network Operator, NFD, SR-IOV
- **CSVs:** From ALL namespaces (solves stuck CSV issues)
- **Resources:** Subscriptions, InstallPlans, ClusterPolicies, NicClusterPolicies
- **Network Performance Testing:** Jobs, DaemonSets, ConfigMaps, and RBAC
- **Namespaces:** nvidia-gpu-operator, nvidia-network-operator, openshift-nfd, openshift-sriov-network-operator, helm-charts
- **Stuck Resources:** Automatically removes finalizers from stuck namespaces and resources
- **CRDs:** Operator-related custom resource definitions (GPU, Network, NFD, SR-IOV)
- **RBAC:** Operator ClusterRoles and ClusterRoleBindings

### Verify Cleanup

```bash
# Check no operator CSVs remain
oc get csv -A | grep -E "(gpu|nvidia|nfd|sriov)" || echo "Clean ✓"

# Check no operator namespaces remain (GitOps should still exist)
oc get ns | grep -E "(nvidia|nfd|sriov|helm-charts)" || echo "Clean ✓"

# Check no operator CRDs remain (ArgoCD CRDs should still exist)
oc get crd | grep -E "(nvidia|mellanox|nfd|sriov)" || echo "Clean ✓"
```

### Manual ArgoCD Cleanup

After all operator resources are cleaned up, ArgoCD can be removed manually as the final step:

```bash
# Delete the ArgoCD namespace
oc delete namespace openshift-gitops

# Delete ArgoCD CRDs
oc delete crd -l app.kubernetes.io/part-of=argocd

# Delete ArgoCD operator subscription (if installed via OLM)
oc delete subscription openshift-gitops-operator -n openshift-operators --ignore-not-found

# Verify ArgoCD is fully removed
oc get ns openshift-gitops 2>/dev/null && echo "Still exists" || echo "Removed ✓"
oc get crd | grep argoproj || echo "CRDs removed ✓"
```

**Why manual cleanup?**
- ArgoCD manages the cleanup job itself
- Deleting ArgoCD while it's running would terminate the cleanup job prematurely
- This ensures all operator resources are fully cleaned before removing the orchestrator

**Note:** The cleanup job is safe to run multiple times (idempotent) and auto-deletes after 10 minutes.

---

## Custom Images

### Network Performance Testing Image

Pre-built universal image supporting all NVIDIA GPUs:

```
quay.io/bbenshab/perf-test:universal
```

**Includes:**
- NCCL tests (all_reduce, all_gather, etc.) pre-compiled for all GPU architectures
- perftest (ib_write_bw, ib_read_bw)
- All analysis tools (jq, bc, numactl, lspci, nvidia-smi)

**Build your own:**
```bash
cd manifests/99-network-perf-tests
podman build -t your-registry/perf-test:latest -f Dockerfile .
podman push your-registry/perf-test:latest
```

See **[Network Performance Tests README](manifests/99-network-perf-tests/README.md)** for details.

---

## Configuration

### RoCE/SR-IOV Subnet Modes

Configure subnet allocation in `manifests/35-roce-discovery/job-generator.yaml`:

- **Separate subnets** (default): Each NIC gets its own subnet (10.0.101.0/24, 10.0.102.0/24)
- **Shared subnet**: All NICs share one subnet (10.0.100.0/24)

See **[RoCE Discovery README](manifests/35-roce-discovery/README.md)** for details.

### GPU ClusterPolicy

Customize GPU operator settings in `manifests/30-gpu-operator-nfd/clusterpolicy.yaml`:
- Driver version (auto-detected by default)
- Device plugin configuration
- GPU Feature Discovery settings

See **[GPU Operator README](manifests/30-gpu-operator-nfd/README.md)** for details.

### Network Operator

Customize MOFED/RDMA settings in `manifests/28-nvidia-network-operator/nicclusterpolicy.yaml`:
- OFED version (auto-detected by default)
- Secondary network configuration
- RDMA shared device plugin

See **[NVIDIA Network Operator README](manifests/28-nvidia-network-operator/README.md)** for details.

---

## Troubleshooting

### Check Deployment Status

```bash
# View all ArgoCD applications
watch oc get applications -n openshift-gitops

# Check health monitor logs
oc logs -n openshift-gitops -l job-name=argocd-health-monitor --tail=100

# Verify operators are running
oc get csv -A | grep -E "(gpu|nvidia|nfd|sriov)"
```

### OLM CSV Stuck in Pending State

**Symptom:** After cleanup or fresh deployment, operator ClusterServiceVersion (CSV) remains in `Pending` phase for several minutes with status `RequirementsNotMet`.

**Example:**
```bash
oc get csv -n openshift-operators
# NAME                                DISPLAY                  VERSION   REPLACES   PHASE
# openshift-gitops-operator.v1.19.0   Red Hat OpenShift GitOps 1.19.0              Pending
```

**Root Cause:** This occurs when OLM recreates a CSV before its required CRDs are fully deleted from the cluster. The timing issue typically happens during cleanup when:
1. Cleanup Job deletes operator Subscription → OLM starts CSV cleanup
2. OLM has already recreated the CSV
3. Cleanup Job deletes CRDs 10 seconds later
4. CSV is now stuck waiting for CRDs that won't be created because InstallPlan already shows "Complete"

**Diagnostic Commands:**

```bash
# 1. Check CSV status
oc get csv <csv-name> -n <namespace> -o jsonpath='{.status.phase}'
# Output: Pending

# 2. Check detailed CSV conditions
oc describe csv <csv-name> -n <namespace>
# Look for:
#   Message: one or more requirements couldn't be found
#   Reason: RequirementsNotMet

# 3. Check InstallPlan status
oc get installplan -n <namespace>
# If InstallPlan shows "Complete" but CRDs are missing, CSV is stuck
```

**Resolution:**

The **self-healing health monitor** automatically detects and fixes this issue by deleting stuck CSVs after 5 minutes. If you need immediate resolution:

```bash
# Delete the stuck CSV
oc delete csv <csv-name> -n <namespace>

# Wait 30-60 seconds, then verify:
oc get csv -n <namespace>
# Phase should progress: Installing → Succeeded
```

**Prevention:** The health monitor runs every 2 minutes and automatically handles this scenario. During initial deployment, the auto-approval Job includes retry logic to minimize this issue.

### Common Issues

- **Stuck namespaces**: Use the [cleanup job](manifests/00-cleanup/README.md) to remove finalizers
- **Failed InstallPlans**: Health monitor auto-approves and retries
- **GPU pods on master nodes**: [Node preparation job](manifests/00-node-preparation/README.md) adds exclusion labels
- **MOFED not ready**: [MOFED readiness gate](manifests/29-nvidia-mofed-ready/README.md) waits for driver pods
- **MOFED CrashLoopBackOff**: See [NVIDIA Network Operator troubleshooting](manifests/28-nvidia-network-operator/README.md#troubleshooting)

See individual manifest READMEs for detailed troubleshooting.

---

## Contributing

This is a specialized deployment for NVIDIA GPU + RDMA infrastructure. Contributions welcome for:
- Additional GPU architectures
- Network performance optimizations
- Enhanced monitoring and observability
- Bug fixes and improvements

---

## License

This project is provided as-is for baremetal GPU cluster deployments.
