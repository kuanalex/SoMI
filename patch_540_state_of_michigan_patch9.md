# IBM Software Hub 5.4.0.4 to 5.4.0.9 Patch Runbook

## Patch Context

**Customer:** State of Michigan

**Environment:** Production

**Patch Date:** 2026-11-04

**Target Patch:** Patch 9

---

### Components to be Patched

**IBM Software Hub Components:** cpd_platform, analyticsengine, datastage_ent_plus, wkc, mantaflow, ws_pipelines

**Scheduling Service:** ✓ Included

---

## Table of Contents
1. [Patch Execution](#patch-execution)
2. [Post-Patch Tasks](#post-patch-tasks)

---

## Prerequisites

**IMPORTANT:** This runbook assumes the following tasks have been completed by the cluster administrator:

1. CASE files downloaded for all components on the client workstation
```bash
cpd-cli manage case-download --components=${COMPONENTS_TO_PATCH} --release=5.4.0 --patch_id=9
```
   
4. Cluster administrator has necessary permissions
5. Backup of the cluster completed

If these prerequisites are not met, please complete them before proceeding with patch execution.

---

# Patch Execution

## Update Cluster-Scoped Resources for Scheduling Service

**IBM Documentation:** [Updating cluster-scoped resources for the scheduling service](https://www.ibm.com/docs/en/software-hub/5.4.x?topic=patches-updating-cluster-scoped-resources-scheduling-service)

**Note:** If the scheduling service is not installed in your environment, skip this section and proceed to applying the patch to the scheduling service section.

---

#### 1. Verify Scheduling Service Installation

Check if the scheduling service is installed
```bash
oc get scheduling -A
```

**If no resources are found:** Skip this section and the next section (Apply Patch to Scheduling Service), then proceed to updating cluster-scoped resources for IBM Software Hub.

**If scheduling resources are found:** Continue with the steps below.

---

#### 2. Generate Cluster-Scoped Resources for Scheduling Service

Generate the `cluster_scoped_resources.yaml` file for the scheduling service
```bash
cpd-cli manage case-download \
--components=scheduler \
--release=5.4.0 \
--patch_id=${PATCH_ID} \
--scheduler_ns=${PROJECT_SCHEDULING_SERVICE} \
--case_download=false \
--cluster_resources=true
```

---

#### 3. Apply Cluster-Scoped Resources

Apply the cluster-scoped resources commands returned in the terminal (make sure the correct cpd-cli-workspace path is referenced)
```bash
oc apply -f /root/cpd-cli-workspace/olm-utils-workspace/work/cluster_scoped_resources.yaml --force-conflicts --server-side
```

---

## Apply Patch to Scheduling Service

**IBM Documentation:** [Applying a patch to the scheduling service](https://www.ibm.com/docs/en/software-hub/5.4.x?topic=patches-applying-patch-scheduling-service)

**Note:** If the scheduling service is not installed, this section was skipped in the previous step.

#### 1. Verify Environment Variables

Ensure environment variables are set from previous steps
```bash
echo $PROJECT_SCHEDULING_SERVICE
echo $PATCH_ID
echo $IMAGE_PULL_PREFIX
echo $IMAGE_PULL_SECRET
```

#### 2. Apply Patch to Scheduling Service

**Applying specific patch:**

```bash
cpd-cli manage apply-patch \
--release=5.4.0 \
--patch_id=${PATCH_ID} \
--scheduler_ns=${PROJECT_SCHEDULING_SERVICE} \
--image_pull_prefix=${IMAGE_PULL_PREFIX} \
--image_pull_secret=${IMAGE_PULL_SECRET}
```

#### 3. Monitor Scheduling Service Pods

```bash
oc get pods --namespace=${PROJECT_SCHEDULING_SERVICE}
```

## Update Cluster-Scoped Resources for IBM Software Hub

**IBM Documentation:** [Updating cluster-scoped resources for the instance](https://www.ibm.com/docs/en/software-hub/5.4.x?topic=patches-updating-cluster-scoped-resources-instance)

#### 1. Generate Cluster-Scoped Resources

Generate the `cluster_scoped_resources.yaml` file for the instance
```bash
cpd-cli manage case-download \
--components=${COMPONENTS_TO_PATCH} \
--release=5.4.0 \
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

---

## Apply Patch to Services and Components

**IBM Documentation:** [Applying a patch](https://www.ibm.com/docs/en/software-hub/5.4.x?topic=patches-applying-patch)

**Note:** This step patches the IBM Software Hub platform and all installed services/components (cpd_platform, zen, watson_discovery, etc.). Service instance updates are performed separately in a later step.

#### 1. Verify Prerequisites

Before applying the patch, verify environment variables and component status
```bash
# Set patch ID
export PATCH_ID=9

# Verify environment variables
echo $PROJECT_CPD_INST_OPERATORS
echo $PROJECT_CPD_INST_OPERANDS
echo $PATCH_ID
echo $IMAGE_PULL_PREFIX
echo $IMAGE_PULL_SECRET

# Check all components are ready
cpd-cli manage get-cr-status --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS}
```

#### 2. Apply Patch to Services and Components

**Note:** This command applies patches to ALL installed services and components and runs for an extended period (typically 30-90 minutes). Using `nohup` ensures the command continues if the terminal session disconnects.

**Applying specific patch:**
```bash
nohup cpd-cli manage apply-patch \
--release=5.4.0 \
--patch_id=${PATCH_ID} \
--operator_ns=${PROJECT_CPD_INST_OPERATORS} \
--instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--image_pull_prefix=${IMAGE_PULL_PREFIX} \
--image_pull_secret=${IMAGE_PULL_SECRET} > patch_output.log 2>&1 &
```

#### 3. Monitor Patching Progress

Monitor the output log:

```bash
tail -f -n 100 patch_output.log
```

Check for completion message: `[SUCCESS] ... The apply-patch command ran successfully.`


#### 4. Confirm Operands Status

Confirm that the status of all operands is `Completed`
```bash
cpd-cli manage get-cr-status --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS}
```

Check for any pods not in Running state
```bash
oc get po -A -owide | egrep -v '([0-9])/\1' | egrep -v 'Completed'
```

---

# Post-Patch Tasks

## Verify CPD Profile

#### Verify Existing Profile

Confirm your CPD profile is set up and working
```bash
cpd-cli service-instance list --profile=${CPD_PROFILE_NAME}
```

---

## Service Instance Updates

#### Patch Analytics Engine Instances

To upgrade Analytics Engine powered by Apache Spark service instances
```bash
cpd-cli service-instance upgrade \
--service-type=spark \
--profile=${CPD_PROFILE_NAME} \
--all
```

#### Verify CR, Service Instance and Pod Statuses

Confirm the platform and all components are running the patched version
```bash
cpd-cli manage get-cr-status --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS}
```

Check that all service instances are ready

List all service instances
```bash
cpd-cli service-instance list --profile=${CPD_PROFILE_NAME}
```

Check for pods not in Running state
```bash
oc get po -A -owide | egrep -v '([0-9])/\1' | egrep -v 'Completed'
```

**End of runbook**
