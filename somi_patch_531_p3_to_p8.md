# IBM Software Hub 5.3.1 Patch Runbook
# Author: Alex Kuan (alex.kuan@ibm.com)

## Patch Context
**Customer:** State of Michigan
**Environment:** Production
**Current Patch:** Patch 3
**Target Patch:** Patch 8

### Components to be Patched
**IBM Software Hub Components (6):** cpd_platform, analyticsengine, datastage_ent_plus, wkc, mantaflow, ws_pipelines

**Scheduling Service:** n/a

### Service Instances
**Analytics engine** instance(s) update required for this patch

### Air-Gapped Environment
This is an air-gapped environment, ensure image mirroring steps are completed


---

## Table of Contents
1. [Patch Execution](#patch-execution)
2. [Post-Patch Tasks](#post-patch-tasks)

---


## Prerequisites

**IMPORTANT:** This runbook assumes the following tasks have been completed by the cluster administrator

1. CASE files downloaded for all components
2. Images mirrored to registry (if air-gapped)
3. Cluster administrator has necessary permissions
4. Backup of the cluster completed

If these prerequisites are not met, please complete them before proceeding with patch execution

---


# Patch Execution

## Update Cluster-Scoped Resources for IBM Software Hub

**IBM Documentation:** [Updating cluster-scoped resources for the instance](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=patches-updating-cluster-scoped-resources-instance)

#### 1. Generate Cluster-Scoped Resources

Generate the `cluster_scoped_resources.yaml` file for the instance
```bash
cpd-cli manage case-download \
  --components=${COMPONENTS_TO_PATCH} \
  --release=5.3.1 \
  --patch_id=${PATCH_ID} \
  --operator_ns=${PROJECT_CPD_INST_OPERATORS} \
  --case_download=false \
  --cluster_resources=true
```

#### 2. Apply Cluster-Scoped Resources

Apply the cluster-scoped resources commands returned in the terminal (make sure the correct cpd-cli-workspace path is referenced)
```bash
oc apply -f /root/cpd-cli-workspace/olm-utils-workspace/work/cluster_scoped_resources.yaml --force-conflicts --server-side
```

## Apply Patch to Services and Components

**IBM Documentation:** [Applying a patch](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=patches-applying-patch)

**Note:** This step patches the IBM Software Hub platform and all installed services/components (cpd_platform, zen, watson_discovery, etc.)

Service instance updates are performed separately in a later step

#### 1. Verify Prerequisites

Before applying the patch, verify environment variables and component status

Verify environment variables
```bash
echo $PROJECT_CPD_INST_OPERATORS
echo $PROJECT_CPD_INST_OPERANDS
echo $PATCH_ID
echo $IMAGE_PULL_PREFIX
echo $IMAGE_PULL_SECRET
```

Check all components are ready
```bash
cpd-cli manage get-cr-status --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS}
```

Check for problematic pods
```
oc get po -A -owide | egrep -v '([0-9])/\1' | egrep -v 'Completed'
```

#### 2. Apply Patch to Services and Components

**Note:** This command applies patches to ALL installed services and components and runs for an extended period (typically 30-90 minutes). Using `nohup` ensures the command continues if the terminal session disconnects.

Applying specific patch
```bash
nohup cpd-cli manage apply-patch \
  --release=5.3.1 \
  --patch_id=${PATCH_ID} \
  --operator_ns=${PROJECT_CPD_INST_OPERATORS} \
  --instance_ns=${PROJECT_CPD_INST_OPERANDS} \
  --image_pull_prefix=${IMAGE_PULL_PREFIX} \
  --image_pull_secret=${IMAGE_PULL_SECRET} > patch_output.log 2>&1 &
```

#### 3. Monitor Patching Progress

Monitor the output log
```bash
tail -f -n 100 patch_output.log
```

Check for completion message: `[SUCCESS] ... The apply-patch command ran successfully.`

Monitor the overall patching progress
```bash
# Watch component status
watch -n 60 'cpd-cli manage get-cr-status --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS}'
```

#### 4. Confirm Operands Status

Confirm that the status of all operands is `Completed`
```bash
cpd-cli manage get-cr-status --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS}
```

Check for any pods not in Running state
```bash
oc get pods -n ${PROJECT_CPD_INST_OPERANDS} | grep -v Running | grep -v Completed
```

# Post-Patch Tasks

## Verify CPD Profile

#### 1. Verify Existing Profile

Confirm your CPD profile is set up and working
```bash
cpd-cli service-instance list --profile=${CPD_PROFILE_NAME}
```

#### 2. Verify Patched Instance Status

Check the status of the patched instance
```bash
cpd-cli manage get-cr-status --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS}
```

## Verify Patch Application

#### 1. Verify IBM Software Hub and Component Versions

Confirm the platform and all components are running the patched version
```bash
cpd-cli manage get-cr-status --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS}
```

#### 2. Verify Pod Health

Check for problematic pods
```bash
oc get pods -n ${PROJECT_CPD_INST_OPERANDS} | grep -v Running | grep -v Completed
```

#### 3. Patch Analytics Engine Instances

To upgrade Analytics Engine powered by Apache Spark service instances
```bash
cpd-cli service-instance upgrade \
--service-type=spark \
--profile=${CPD_PROFILE_NAME} \
--all
```

#### 3. Verify Service Instance Status

Check that all service instances are ready:

List all service instances again
```bash
cpd-cli service-instance list --profile=${CPD_PROFILE_NAME}
```

**End of patch activity**
