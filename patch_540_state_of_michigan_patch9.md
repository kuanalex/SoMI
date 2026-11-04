# IBM Software Hub 5.4.0.4 to 5.4.0.9 Patch Runbook
# Author: Alex Kuan

## Environment Information
- Customer: State of Michigan
- Environment: Production
- Patch Date: 2026-11-09
- Target Patch: Patch 9
- IBM Software Hub Components: cpd_platform,analyticsengine,datastage_ent_plus,wkc,mantaflow,ws_pipelines

---

## Table of Contents
1. [Patch Execution](#patch-execution)
2. [Post-Patch Tasks](#post-patch-tasks)

---

## Prerequisites

**IMPORTANT:** This runbook assumes the following tasks have been completed by the cluster administrator:

1. CASE files downloaded for all components on the client workstation
2. Images have been mirrored to the private registry
3. Cluster administrator has necessary permissions
4. Backup of the cluster has been completed

If these prerequisites are not met, please complete them before proceeding with patch execution.

---

#### Gathering information about installed components

If your client workstation pulls images from the IBM Entitled Registry, ensure that the client workstation has the latest version of the olm-utils-v4 image

Note: If your client workstation pulls images from a private container registry, skip this step

You will run this command after you mirror images to the private container registry
```bash
cpd-cli manage restart-container
```

Login to the cluster
```bash
${CPDM_OC_LOGIN}
```

Run the following command to generate a list of the components that are installed
```bash
cpd-cli manage list-deployed-components --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} --br_operator_ns=${PROJECT_INST_BR_SVC} --all=true
```

The output of the command is saved to a file named scan.yaml, which is located in the work/scan/ directory

---

#### Checking for new patches

Run the list-patch command to generate a list of patches are available for the instance.
```bash
cpd-cli manage list-patch
```

The command downloads the patch-5.4.0.yaml file for the latest patch from the IBM/software-hub repository on GitHub

Then, the command compares the contents of patch-5.4.0.yaml file to the contents of the scan.yaml file

The output includes several sections, most importantly, a list of components that have available patches

To download CASE Packages for patches, run
```bash
cpd-cli manage case-download --components=cpd_platform,zen... --release=5.4.0 --patch_id=<patch-id>
```

Set the COMPONENTS_TO_PATCH environment variable to the list of components that you specified when you ran the case-download command

**Important**: The list of components must be the same as the list of components that were returned by the list-patch command
```bash
export COMPONENTS_TO_PATCH=cpd_platform,zen...
```

---

# Patch Execution

## Update Cluster-Scoped Resources for IBM Software Hub

**IBM Documentation:** [Updating cluster-scoped resources for the instance](https://www.ibm.com/docs/en/software-hub/5.4.x?topic=patches-updating-cluster-scoped-resources-instance)

#### Generate Cluster-Scoped Resources

Generate the `cluster_scoped_resources.yaml` file for the instance
```bash
cpd-cli manage case-download --components=${COMPONENTS_TO_PATCH} --release=5.4.0 --patch_id=${PATCH_ID} --operator_ns=${PROJECT_CPD_INST_OPERATORS} --case_download=false --cluster_resources=true
```

---

#### Apply Cluster-Scoped Resources

Apply the cluster-scoped resources commands returned in the terminal, for example
```bash
oc apply -f .../root/cpd-cli-workspace/olm-utils-workspace/work/cluster_scoped_resources.yaml --force-conflicts --server-side
```

---

## Apply Patch to Services and Components

**IBM Documentation:** [Applying a patch](https://www.ibm.com/docs/en/software-hub/5.4.x?topic=patches-applying-patch)

**Note:** This step patches the IBM Software Hub platform and all installed services/components (cpd_platform, zen, watson_discovery, etc.)

Service instance updates are performed separately in a later step

#### Verify Prerequisites

Before applying the patch, verify environment variables and component status
```bash
# Set VERSION and PATCH_ID
export VERSION=5.4.0
export PATCH_ID=9
```

Verify environment variables
```bash
echo $PROJECT_CPD_INST_OPERATORS
echo $PROJECT_CPD_INST_OPERANDS
echo $PATCH_ID
echo $IMAGE_PULL_PREFIX
echo $IMAGE_PULL_SECRET
```

Check all components are ready for upgrade
```bash
cpd-cli manage get-cr-status --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS}
```

---

#### Apply Patch to Services and Components

**Note:** This command applies patches to ALL installed services and components and runs for an extended period (typically 30-90 minutes)

Using `nohup` ensures the command continues if the terminal session disconnects

**Applying the patch:**
```bash
nohup cpd-cli manage apply-patch --release=5.4.0 --patch_id=${PATCH_ID} --operator_ns=${PROJECT_CPD_INST_OPERATORS} --instance_ns=${PROJECT_CPD_INST_OPERANDS} --image_pull_prefix=${IMAGE_PULL_PREFIX} --image_pull_secret=${IMAGE_PULL_SECRET} > patch_output.log 2>&1 &
```

---

#### Monitor Patching Progress

Monitor the output log:

```bash
tail -f -n 100 patch_output.log
```

Check for completion message: `[SUCCESS] ... The apply-patch command ran successfully.`

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
cpd-cli service-instance upgrade --service-type=spark --profile=${CPD_PROFILE_NAME} --all
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
