# IBM Software Hub 5.3.1.5 to 5.3.1.11 Patch Runbook

## Patch Context

**Customer:** Banco Bradesco
**Environment:** Dev
**Patch Date:** 2026-09-04
**Target Patch:** Patch 11

### Components to be Patched

**IBM Software Hub Components (6):** cpd_platform,wkc,datastage_ent,ws_pipelines,ws,ws_runtimes

---

## Table of Contents
1. [Patch Execution](#patch-execution)
2. [Post-Patch Tasks](#post-patch-tasks)

---


## Prerequisites

## Gathering information about installed components

Login to the cluster:
```bash
${CPDM_OC_LOGIN}
```
 
Generate a list of the components that are installed:
```bash
cpd-cli manage list-deployed-components \
--scheduler_ns=${PROJECT_SCHEDULING_SERVICE} \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--all=true
```

## Checking for new patches

Restart the olm-utils container
```bash
cpd-cli manage restart-container
```

Run the list-patch command to generate a list of patches are available for the instance.
```bash
cpd-cli manage list-patch
```

For example...
```bash
cpd-cli manage case-download --components=${COMPONENTS_TO_PATCH} --release=5.3.1 --patch_id=11
```
   
Mirror images to registry (if air-gapped):
```bash
cpd-cli manage mirror-images \
--components=${COMPONENTS_TO_PATCH} \
--release=5.3.1 \
--target_registry=${PRIVATE_REGISTRY_LOCATION} \
--arch=${IMAGE_ARCH} \
--case_download=false
```

4. Cluster administrator has necessary permissions

5. Backup of the cluster completed

If these prerequisites are not met, please complete them before proceeding with patch execution.

---

# Patch Execution

## Updating the cluster-scoped resources for the scheduling service

Generate the `cluster_scoped_resources.yaml` file for the scheduling service:
```bash
cpd-cli manage case-download \
  --components=${COMPONENTS_TO_PATCH} \
  --release=5.3.1 \
  --patch_id=${PATCH_ID} \
  --operator_ns=${PROJECT_CPD_INST_OPERATORS} \
  --case_download=false \
  --cluster_resources=true
```

Apply the cluster-scoped resources commands returned in the terminal (make sure the correct cpd-cli-workspace path is referenced):
```bash
oc apply -f /root/cpd-cli-workspace/olm-utils-workspace/work/cluster_scoped_resources.yaml --force-conflicts --server-side
```

---

## Applying the patch to the scheduling service

Login to the cluster:
```bash
${CPDM_OC_LOGIN}
```

Apply the patch for scheduling service:
```bash
cpd-cli manage apply-patch \
--release=5.3.1 \
--patch_id=${PATCH_ID} \
--scheduler_ns=${PROJECT_SCHEDULING_SERVICE} \
--image_pull_prefix=${IMAGE_PULL_PREFIX} \
--image_pull_secret=${IMAGE_PULL_SECRET}
```

Confirm the pods are running:
```bash
oc get pods --namespace=${PROJECT_SCHEDULING_SERVICE}
```

---

## Update Cluster-Scoped Resources for IBM Software Hub

**IBM Documentation:** [Updating cluster-scoped resources for the instance](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=patches-updating-cluster-scoped-resources-instance)

#### 1. Generate Cluster-Scoped Resources

Generate the `cluster_scoped_resources.yaml` file for the instance:
```bash
cpd-cli manage case-download \
  --components=${COMPONENTS_TO_PATCH} \
  --release=5.3.1 \
  --patch_id=${PATCH_ID} \
  --operator_ns=${PROJECT_CPD_INST_OPERATORS} \
  --case_download=false \
  --cluster_resources=true
```

---

Apply the cluster-scoped resources commands returned in the terminal (make sure the correct cpd-cli-workspace path is referenced):
```bash
oc apply -f /root/cpd-cli-workspace/olm-utils-workspace/work/cluster_scoped_resources.yaml --force-conflicts --server-side
```

## Apply Patch to Services and Components

**IBM Documentation:** [Applying a patch](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=patches-applying-patch)

**Note:** This step patches the IBM Software Hub platform and all installed services/components (cpd_platform, zen, watson_discovery, etc.). Service instance updates are performed separately in a later step.

#### 1. Verify Prerequisites

Before applying the patch, verify environment variables and component status:

```bash
# Set patch ID
export PATCH_ID=11

# Verify environment variables
echo $PROJECT_CPD_INST_OPERATORS
echo $PROJECT_CPD_INST_OPERANDS
echo $PATCH_ID
echo $IMAGE_PULL_PREFIX
echo $IMAGE_PULL_SECRET
```

Remove hot fix image_digests from CCS custom resource
```bash
oc patch ccs ccs-cr -n gov-operands --type=json -p='[
  {
    "op": "remove",
    "path": "/spec/image_digests"
  }
]'
```

Check wkc relevant custom resources for current ignoreForMaintenance status
```bash
oc get dataquality,enrichment,finley,glossary,knowledgegraph,metadataimports,policy,profiling,wkcgovui,wkc,workflow -n gov-operands -o custom-columns=KIND:.kind,NAME:.metadata.name,MAINTENANCE:.spec.ignoreForMaintenance
```

Unset ignoreForMaintenance flag in wkc relevant custom resources
```bash
for cr in dataquality/data-quality-cr enrichment/enrichment-cr finley/finley-cr glossary/glossary-cr knowledgegraph/knowledgegraph-cr metadataimports/metadataimports-cr policy/policy-cr profiling/profiling-cr wkcgovui/wkc-gov-ui-cr wkc/wkc-cr workflow/workflow-cr; do
  oc patch $cr -n gov-operands --type=merge -p '{"spec":{"ignoreForMaintenance":false}}'
done
```

Check all components are ready
```bash
cpd-cli manage get-cr-status --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS}
```

#### 2. Apply Patch to Services and Components

**Note:** This command applies patches to ALL installed services and components and runs for an extended period (typically 30-90 minutes). Using `nohup` ensures the command continues if the terminal session disconnects.

**Applying specific patch:**

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

Monitor the output log:

```bash
tail -f -n 100 patch_output.log
```

Check for completion message: `[SUCCESS] ... The apply-patch command ran successfully.`


#### 4. Confirm Operands Status

Confirm that the status of all operands is `Completed`:

```bash
cpd-cli manage get-cr-status --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS}
```

Check for any pods not in Running state:

```bash
oc get po -A -owide | egrep -v '([0-9])/\1' | egrep -v 'Completed'
```

# Post-Patch Tasks

## Verify CPD Profile

#### 1. Verify Existing Profile

Confirm your CPD profile is set up and working:

```bash
cpd-cli service-instance list --profile=${CPD_PROFILE_NAME}

#### 2. Patch Analytics Engine Instances

To upgrade Analytics Engine powered by Apache Spark service instances:

```bash
cpd-cli service-instance upgrade \
  --service-type=spark \
  --profile=${CPD_PROFILE_NAME} \
  --all
```

#### 4. Verify CR, Service Instance and Pod Statuses

Confirm the platform and all components are running the patched version:

```bash
cpd-cli manage get-cr-status --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS}
```

Check that all service instances are ready

List all service instances:
```bash
cpd-cli service-instance list --profile=${CPD_PROFILE_NAME}
```

Check for pods not in Running state:

```bash
oc get po -A -owide | egrep -v '([0-9])/\1' | egrep -v 'Completed'
```

**End of runbook**