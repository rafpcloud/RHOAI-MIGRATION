# Strategic Migration and Modernization of RHOAI (Version 2.25.x to 3.5)

> **Architectural & Engineering Guide** | A comprehensive technical roadmap for upgrading Red Hat OpenShift AI from version 2.25.x to 3.5.

---

## Table of Contents

- [1. Introduction & Journey Overview](#1-introduction--journey-overview)
- [2. Prerequisites & Critical Environment Preparation](#2-prerequisites--critical-environment-preparation)
  - [2.1 Technical Requirements Matrix](#21-technical-requirements-matrix)
  - [2.2 Baseline Stabilization Procedure (v2.25.10)](#22-baseline-stabilization-procedure-v22510)
  - [2.3 Pre-Upgrade Assessment Protocol](#23-pre-upgrade-assessment-protocol)
  - [2.4 Neutralizing DataScienceCluster (DSC) and DSCI](#24-neutralizing-datasciencecluster-dsc-and-dsci)
  - [2.5 Legacy Service Mesh v2 Removal](#25-legacy-service-mesh-v2-removal)
- [3. InferenceServices Conversion (Serverless to RawDeployment)](#3-inferenceservices-conversion-serverless-to-rawdeployment)
  - [3.1 Auditing Active Workloads](#31-auditing-active-workloads)
  - [3.2 Executing the Conversion Helper Script](#32-executing-the-conversion-helper-script)
  - [3.3 Post-Conversion Verification](#33-post-conversion-verification)
- [4. Orchestration Infrastructure: Kueue and JobSet Operators](#4-orchestration-infrastructure-kueue-and-jobset-operators)
  - [4.1 Kueue Configuration](#41-kueue-configuration)
  - [4.2 JobSet Configuration](#42-jobset-configuration)
- [5. Executing the Intermediate Upgrade (RHOAI 3.3)](#5-executing-the-intermediate-upgrade-rhoai-33)
  - [5.1 Channel Transition Protocol](#51-channel-transition-protocol)
  - [5.2 Health Validation Commands](#52-health-validation-commands)
- [6. Upgrading to RHOAI 3.5 & Transitioning to DSC v2 API](#6-upgrading-to-rhoai-35--transitioning-to-dsc-v2-api)
  - [6.1 Complete DSC v2 Configuration Manifest](#61-complete-dsc-v2-configuration-manifest)
- [7. Success Readiness Checklist & Conclusion](#7-success-readiness-checklist--conclusion)

---

## 1. Introduction & Journey Overview

Migrating **Red Hat OpenShift AI (RHOAI)** from version **2.25.x to 3.5** is not merely a routine software update; it is a strategic imperative for the long-term sustainability and scalability of your AI infrastructure. This transition marks a major architectural evolution toward `RawDeployment` and advanced resource orchestration, systematically eliminating legacy dependencies that previously constrained platform growth.

When planning this upgrade, organizations typically evaluate two core strategies:

* **In-Place Migration:** Migrating RHOAI directly within the existing cluster/server footprint.
* **Side-by-Side Environment:** Deploying a secondary environment running OpenShift AI 3.5 in parallel with the active 2.25.x cluster and migrating workloads incrementally.

> [!WARNING]
> **Important Operational Notice:** While an **In-Place** strategy reduces infrastructure overhead, the sheer magnitude of structural changes requires the engineering team to schedule a maintenance window. **RHOAI does not support Zero Downtime for this major version jump.**

---

## 2. Prerequisites & Critical Environment Preparation

Establishing cluster health and alignment with initial state requirements forms the foundation of this procedure. Before applying any structural changes, cluster administrators must ensure full compliance with the prerequisite matrix below.

### 2.1 Technical Requirements Matrix

| Component / Protocol | Requirement Standard | Notes & Mandates |
| :--- | :--- | :--- |
| **OpenShift Container Platform (OCP)** | v4.19.6+ *(Recommended: 4.19.9+)* | Required for full API compatibility with RHOAI 3.5. |
| **Base Operator State** | Version > 2.25.6 | Must be stabilized on the latest 2.25.x line (e.g., `2.25.10`) before attempting 3.x transitions. |
| **Mandatory Operators** | • `cert-manager for Red Hat OpenShift`<br>• `Red Hat Service Mesh v3` | Service Mesh v3 must be installed via the `stable` channel in `redhat-operators`. |
| **Support Protocol** | Proactive Case Mandatory | Opening a **Proactive Case** with Red Hat Support is required prior to execution. |

---

### 2.2 Baseline Stabilization Procedure (v2.25.10)

Before executing structural upgrades, bring the RHOAI operator to the latest release of the 2.x stream:

1. Navigate to **Installed Operators** → **Red Hat OpenShift AI** → **Subscriptions**.
2. Change the update channel to `stable-2.25.10` (or the latest version in the 2.25 stream).
3. Approve the generated `InstallPlan` and monitor execution until deployment stabilizes across the cluster. Depending on your current version, execute sequential updates until reaching the target baseline.

---

### 2.3 Pre-Upgrade Assessment Protocol

Running the pre-upgrade assessment is a mandatory gate for Proactive Case validation. Red Hat Support will not authorize moving to gated channels without analyzing the generated artifact (`rhai-cli.yaml`).

#### Step A: Create the Assessment Project
```bash
oc new-project rh-mig
```

#### Step B: Deploy the Assessment StatefulSet
```yaml
cat <<'EOF' | oc apply -n rh-mig -f -
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: rhai-cli
spec:
  serviceName: rhai-cli
  replicas: 1
  selector:
    matchLabels:
      app: rhai-cli
  template:
    metadata:
      labels:
        app: rhai-cli
    spec:
      containers:
        - name: rhai-cli
          image: registry.redhat.io/rhoai/rhai-cli-rhel9:v3.3.5
          command:
            - sleep
            - infinity
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
          volumeMounts:
            - name: backup
              mountPath: /tmp/rhoai-upgrade-backup
  volumeClaimTemplates:
    - metadata:
        name: backup
      spec:
        accessModes:
          - ReadWriteOnce
        # storageClassName: <your-storage-class> # Uncomment if PVC remains Pending
        resources:
          requests:
            storage: 1Gi
EOF
```

#### Step C: Grant RBAC Permissions and Run Assessment
```bash
# Provision Service Account Permissions
oc create clusterrolebinding rhoai-cli \
  --clusterrole=cluster-admin \
  --serviceaccount=rh-mig:default

# Execute Lint Assessment inside Pod
oc exec -it rhai-cli-0 -n rh-mig -- /opt/rhai-cli/bin/rhai-cli lint --target-version 3.3.5 > /tmp/rhoai-upgrade-backup/rhai-cli.yaml
```

> [!IMPORTANT]
> **Gate Criteria:** Inspect the output file (`rhai-cli.yaml`). If the summary contains `Failed` or `Prohibited` statuses, **halt the migration immediately** until remediation is complete. Attach `rhai-cli.yaml` to your Red Hat support ticket.

---

### 2.4 Neutralizing DataScienceCluster (DSC) and DSCI

To prevent reconciliation loop conflicts during operator binary updates, set all components to `Removed` state:

1. Edit the `DataScienceCluster` instance:
   ```bash
   oc edit DataScienceCluster default-dsc
   ```

2. Update component management states in the custom resource:
   ```yaml
   apiVersion: datasciencecluster.opendatahub.io/v1
   kind: DataScienceCluster
   metadata:
     name: default-dsc
   spec:
     components:
       codeflare:
         managementState: Removed
       dashboard:
         managementState: Removed
       datasciencepipelines:
         managementState: Removed
       kserve:
         managementState: Removed
       kueue:
         managementState: Removed
       llamastackoperator:
         managementState: Removed
       modelmeshserving:
         managementState: Removed
       ray:
         managementState: Removed
       trainingoperator:
         managementState: Removed
       trustyai:
         managementState: Removed
       workbenches:
         managementState: Removed
   ```

3. Neutralize Service Mesh initialization via patch on `DSCInitialization`:
   ```bash
   oc patch dsci default-dsci --type='merge' -p '{"spec":{"serviceMesh":{"managementState":"Removed"}}}'
   ```

---

### 2.5 Legacy Service Mesh v2 Removal

Because RHOAI 3.3+ standardizes on **Red Hat Service Mesh Operator v3**, legacy Service Mesh v2 (2.x) becomes obsolete and **must be uninstalled via the OpenShift Web Console** prior to selecting the 3.3 release channel.

---

## 3. InferenceServices Conversion (Serverless to RawDeployment)

RHOAI 3.3 introduces `RawDeployment` as the default deployment model for model serving. This shift streamlines the networking stack and eliminates Knative/Istio v2 overhead for standard serving instances.

### 3.1 Auditing Active Workloads
Audit active production namespaces to identify existing Serverless workloads:
```bash
oc get isvc -n <namespace> -o json | jq -r '["NAME","DEPLOYMENT_MODE","READY"], (.items[] | [.metadata.name, .status.deploymentMode, (.status.conditions[] | select(.type=="Ready") | .status)]) | @tsv' | column -t
```

### 3.2 Executing the Conversion Helper Script
Run the interactive conversion script (*Note: In In-Place mode, legacy resources are deleted and re-created under new specifications*):
```bash
/opt/rhai-upgrade-helpers/model-serving/before-upgrade/serverless-to-raw.sh -n <namespace> --delete-existing
```

#### Interactive Prompt Response Guide:

| Interactive Script Prompt | Recommended Technical Response |
| :--- | :--- |
| **InferenceServices Selection** | `all` |
| **Naming Strategy Option** | `1` *(Preserve original names / In-place replacement)* |
| **Confirm Replacement** | `yes` |
| **Preserve Transformed Manifests** | `y` |

### 3.3 Post-Conversion Verification
Verify that all `InferenceServices` reflect `RawDeployment` status with `Ready: True`.

---

## 4. Orchestration Infrastructure: Kueue and JobSet Operators

In RHOAI 3.3+, quota management and distributed workload scheduling are delegated to external dedicated operators: **Red Hat build of Kueue** and **JobSet Operator**. Both must be installed via OperatorHub prior to advancing the RHOAI channel.

### 4.1 Kueue Configuration
Apply the global cluster resource for framework integration:
```yaml
apiVersion: kueue.openshift.io/v1
kind: Kueue
metadata:
  name: cluster
spec:
  config:
    integrations:
      frameworks:
        - Deployment
        - Pod
        - PyTorchJob
        - RayCluster
        - RayJob
        - StatefulSet
        - TrainJob
    logLevel: Normal
    managementState: Managed
    operatorLogLevel: Normal
```

### 4.2 JobSet Configuration
Apply the JobSet operator configuration:
```yaml
apiVersion: operator.openshift.io/v1
kind: JobSetOperator
metadata:
  name: cluster
spec:
  logLevel: Normal
  managementState: Managed
  operatorLogLevel: Normal
```

> [!CAUTION]
> **Critical Troubleshooting Step:** If Kueue controller reconciliation fails due to leftover CRD schema conflicts from prior releases, manually remove stale CRDs:
> ```bash
> oc delete crd cohorts.kueue.x-k8s.io topologies.kueue.x-k8s.io
> ```

---

## 5. Executing the Intermediate Upgrade (RHOAI 3.3)

Access to version 3.3 is restricted behind the `support-required-upgrade` channel. This guardrail ensures that environment readiness has been verified through Red Hat Support.

### 5.1 Channel Transition Protocol
1. Open the OpenShift Web Console and go to **Installed Operators** → **Red Hat OpenShift AI Subscription**.
2. Switch the update channel to `support-required-upgrade`.
3. When the upgrade status changes to **Requires Approval**, click the notification.
4. Review the `InstallPlan` (which includes Service Mesh v3 dependencies) and click **Approve**.

### 5.2 Health Validation Commands
Monitor operator and system pod readiness using custom column outputs:

**Operator Namespace:**
```bash
oc get pods -n redhat-ods-operator -o custom-columns='NAME:.metadata.name,READY:.status.conditions[?(@.type=="Ready")].status,STATUS:.status.phase'
```

**Applications Namespace:**
```bash
oc get pods -n redhat-ods-applications -o custom-columns='NAME:.metadata.name,READY:.status.conditions[?(@.type=="Ready")].status,STATUS:.status.phase'
```

Once components stabilize, set the operator subscription channel to `stable-3.3` or `stable-3.x`.

---

## 6. Upgrading to RHOAI 3.5 & Transitioning to DSC v2 API

After stabilizing on 3.3, moving toward version 3.5 introduces the **DataScienceCluster `v2` API**, enabling features such as NVIDIA NIM serving and granular evaluation guardrails.

### 6.1 Complete DSC v2 Configuration Manifest
Re-enable active components under the modernized API specification:

```yaml
apiVersion: datasciencecluster.opendatahub.io/v2
kind: DataScienceCluster
metadata:
  name: default-dsc
spec:
  components:
    kserve:
      managementState: Managed
      modelsAsService:
        managementState: Removed
      nim:
        airGapped: false
        managementState: Managed
      rawDeploymentServiceConfig: Headless
    modelregistry:
      managementState: Removed
      registriesNamespace: rhoai-model-registries
    feastoperator:
      managementState: Managed
    trustyai:
      eval:
        lmeval:
          permitCodeExecution: deny
          permitOnline: deny
      managementState: Managed
      mcpGuardrailsMode: false
    aipipelines:
      argoWorkflowsControllers:
        managementState: Managed
      managementState: Managed
    ray:
      managementState: Managed
    kueue:
      defaultClusterQueueName: default
      defaultLocalQueueName: default
      managementState: Unmanaged
    workbenches:
      managementState: Managed
      workbenchNamespace: rhods-notebooks
    mlflowoperator:
      managementState: Managed
    dashboard:
      managementState: Managed
    trainingoperator:
      managementState: Managed
    llamastackoperator:
      managementState: Managed
```

> [!NOTE]
> The `kueue` component is marked as `Unmanaged` inside the DSC v2 spec because its lifecycle is independently managed by the standalone Kueue operator installed in Section 4.

---

## 7. Success Readiness Checklist & Conclusion

Before handing off the environment to data science engineering teams, confirm all operational validation items:

- [ ] **Final Version:** RHOAI Operator actively running and reporting version `3.5.x`.
- [ ] **Resource Status:** Both `DataScienceCluster` and `DSCInitialization` reporting `Ready` state.
- [ ] **Pod Readiness:** `Ready=True` across pods in `redhat-ods-operator` and `redhat-ods-applications`.
- [ ] **Gateway Configuration:** `oc get gatewayconfigs --all-namespaces` displays `default-gateway` as `READY: True`.
- [ ] **Inference Services:** All active `InferenceServices` fully operational in `RawDeployment` mode.
- [ ] **Hardware Profiles:** Successful automatic migration from `AcceleratorProfiles` to `HardwareProfiles` (`infrastructure.opendatahub.io`).
- [ ] **Cluster Hygiene:** Legacy Service Mesh v2 completely removed from the cluster.
- [ ] **Subscription Channel:** Operator subscription tracking `stable-3.x`.

### Conclusion

Upgrading to **RHOAI 3.5** delivers a modernized, resilient, and enterprise-grade AI foundation. The shift to `RawDeployment` and standalone queue management via Kueue gives the platform the stability needed to support large-scale AI workloads with reduced operational complexity and maximum throughput.
