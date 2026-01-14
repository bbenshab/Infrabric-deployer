# Operator Cleanup Job

This directory contains a Kubernetes Job that performs comprehensive cleanup of all operators and their resources.

## Table of Contents
- [Purpose](#purpose)
- [Usage](#usage)
- [What Gets Cleaned Up](#what-gets-cleaned-up)
- [Verification](#verification)
- [Automatic ArgoCD Cleanup](#automatic-argocd-cleanup)
- [Notes](#notes)

## Purpose

Run this Job **BEFORE** a fresh deployment to prevent conflicts from previous installations:
- Removes all operator CSVs from ALL namespaces (solves the 91-namespace CSV issue)
- Deletes operator subscriptions and install plans
- Removes operator namespaces
- Deletes CRDs and webhook configurations

## Usage
```bash
# Apply the cleanup Job directly
oc apply -k manifests/00-cleanup

# Watch the cleanup process
oc logs -n openshift-operators job/cleanup-operators -f

# Wait for completion
oc wait --for=condition=complete job/cleanup-operators -n openshift-operators --timeout=600s

# Delete the Job (auto-deletes after 5 minutes anyway)
oc delete -k manifests/00-cleanup
```

## What Gets Cleaned Up

**IMPORTANT**: ArgoCD/GitOps is now **automatically cleaned up** by a separate Job spawned after operator cleanup completes. See "Automatic ArgoCD Cleanup" section below.

1. **ArgoCD Applications**
   - Deletes all operator-related Applications
   - Removes finalizers to prevent stuck deletions
   - **Preserves**: gitops and root-app Applications

2. **GPU Operator**
   - Subscription, ClusterPolicy, CSVs
   - Removes finalizers from ArgoCD hook jobs and serviceaccounts
   - Prevents stuck namespaces due to hook finalizers

3. **Network Operator**
   - Subscription, NicClusterPolicy, CSVs

4. **NFD Operator**
   - Subscription, CSVs
   - Removes finalizers from NodeFeatureDiscovery instances
   - Prevents stuck namespaces due to NFD finalizers

5. **SR-IOV Operator**
   - Subscription, CSVs, custom resources
   - Removes finalizers from SriovNetworks and SriovOperatorConfigs
   - Prevents stuck namespaces due to SR-IOV finalizers

6. **Network Performance Testing**
   - Job, ServiceAccount, ConfigMap in default namespace
   - Dynamically created DaemonSet (network-perf-test-worker)
   - ClusterRole and ClusterRoleBinding

7. **Namespaces**
   - nvidia-gpu-operator
   - nvidia-network-operator
   - openshift-nfd
   - openshift-sriov-network-operator
   - helm-charts
   - **NOTE**: `openshift-gitops` is cleaned by the automatic ArgoCD cleanup Job (see below)
   - Waits up to 2 minutes for namespaces to be fully deleted
   - **Auto-recovery**: If namespaces remain stuck, force-finalizes them using:
     - Removes any remaining custom resources (NFD instances, helm jobs)
     - Force-deletes remaining serviceaccounts, jobs, and pods
     - Force-finalizes namespace using Kubernetes raw API
     - Ensures 100% cleanup even with stubborn resources

8. **CRDs and Webhooks**
   - Operator-related CRDs (GPU, Network, NFD, SR-IOV)
   - Webhook configurations
   - **NOTE**: ArgoCD CRDs are cleaned by the automatic ArgoCD cleanup Job

9. **ArgoCD/GitOps** (automatic cleanup via separate Job)
   - ArgoCD Applications
   - openshift-gitops namespace
   - ArgoCD CRDs
   - ArgoCD operator subscription
   - ArgoCD RBAC resources

## Verification

After cleanup completes, verify:

```bash
# No operator CSVs remain
oc get csv -A | grep -E "(gpu|nvidia|nfd|sriov)" || echo "Clean ✓"

# No operator namespaces remain (GitOps should still exist)
oc get ns | grep -E "(nvidia|nfd|sriov|helm-charts)" || echo "Clean ✓"

# No operator CRDs remain (ArgoCD CRDs should still exist)
oc get crd | grep -E "(nvidia|mellanox|nfd|sriov)" || echo "Clean ✓"
```

## Automatic ArgoCD Cleanup

**NEW**: The cleanup Job now automatically removes ArgoCD/GitOps after operator cleanup completes!

**How it works:**
1. The main cleanup Job runs and removes all operator resources
2. Before completing, it spawns a **separate non-ArgoCD Job** in the `default` namespace
3. This second Job waits for the main cleanup to complete
4. Then it deletes ArgoCD namespace, CRDs, and resources
5. Finally, it self-deletes after 5 minutes

**Monitor the automatic ArgoCD cleanup:**
```bash
# Watch the ArgoCD cleanup Job
oc logs -n default job/argocd-cleanup -f

# Verify ArgoCD was removed
oc get ns openshift-gitops 2>/dev/null && echo "Still exists" || echo "Removed ✓"
oc get crd | grep argoproj || echo "CRDs removed ✓"
```

**Why this approach?**
- The second Job runs **outside ArgoCD's control** (in `default` namespace, no ArgoCD labels)
- This allows safe deletion of ArgoCD after the main cleanup completes
- Fully automated - no manual intervention required
- The ArgoCD cleanup Job auto-deletes after 5 minutes via TTL

## Notes

- The Job runs in the `openshift-operators` namespace
- Auto-deletes after 10 minutes (`ttlSecondsAfterFinished: 600`)
- Safe to run multiple times (idempotent)
- Uses cluster-scoped RBAC with minimal required permissions
- Automatically removes finalizers to prevent stuck namespaces
- Waits for namespaces to be fully deleted before completing
- **Force-finalization**: If namespaces remain stuck after 2 minutes, automatically force-finalizes them
- Handles edge cases like ArgoCD hook finalizers and SR-IOV network finalizers
- No manual intervention required - fully automated stuck namespace recovery
