# Strategic Migration and Modernization of RHOAI (Version 2.25.x to 3.5)

> **Architectural & Engineering Guide** | A comprehensive technical roadmap for upgrading Red Hat OpenShift AI from version 2.25.x to 3.5.

---

## Table of Contents

- [1. Introduction & Journey Overview](#1-introduction--journey-overview)
- [2. Prerequisites & Critical Environment Preparation](#2-prerequisites--critical-environment-preparation)
  - [2.1 Technical Requirements Matrix](#21-technical-requirements-matrix)
  - [2.2 Pre-Flight Dependency Validation](#22-pre-flight-dependency-validation)
  - [2.3 Installing Mandatory Operators](#23-installing-mandatory-operators)
    - [2.3.1 cert-manager for Red Hat OpenShift](#231-cert-manager-for-red-hat-openshift)
    - [2.3.2 Red Hat build of Kueue](#232-red-hat-build-of-kueue)
    - [2.3.3 JobSet Operator](#233-jobset-operator)
  - [2.4 Operator Configuration (Kueue & JobSet)](#24-operator-configuration-kueue--jobset)
    - [2.4.1 Kueue Configuration](#241-kueue-configuration)
    - [2.4.2 JobSet Configuration](#242-jobset-configuration)
  - [2.5 Baseline Stabilization Procedure (v2.25.10)](#25-baseline-stabilization-procedure-v22510)
  - [2.6 Pre-Upgrade Assessment Protocol](#26-pre-upgrade-assessment-protocol)
  - [2.7 Neutralizing DataScienceCluster (DSC) and DSCI](#27-neutralizing-datasciencecluster-dsc-and-dsci)
  - [2.8 Legacy Service Mesh v2 Removal](#28-legacy-service-mesh-v2-removal)
- [3. InferenceServices Conversion (Serverless to RawDeployment)](#3-inferenceservices-conversion-serverless-to-rawdeployment)
  - [3.1 Auditing Active Workloads](#31-auditing-active-workloads)
  - [3.2 Executing the Conversion Helper Script](#32-executing-the-conversion-helper-script)
  - [3.3 Post-Conversion Verification](#33-post-conversion-verification)
- [4. Executing the Intermediate Upgrade (RHOAI 3.3)](#4-executing-the-intermediate-upgrade-rhoai-33)
  - [4.1 Channel Transition Protocol](#41-channel-transition-protocol)
  - [4.2 Health Validation Commands](#42-health-validation-commands)
- [5. Upgrading to RHOAI 3.5 & Transitioning to DSC v2 API](#5-upgrading-to-rhoai-35--transitioning-to-dsc-v2-api)
  - [5.1 Complete DSC v2 Configuration Manifest](#51-complete-dsc-v2-configuration-manifest)
- [6. Troubleshooting & Common Issues](#6-troubleshooting--common-issues)
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
| **OpenShift Container Platform (OCP)** | v4.19.9+ *(Recommended: 4.19.9+)* | Required for full API compatibility with RHOAI 3.5. |
| **Base RHOAI Operator State** | Version ≥ 2.25.6 | Must be stabilized on the latest 2.25.x line (e.g., `2.25.10`) before attempting 3.x transitions. |
| **cert-manager for Red Hat OpenShift** | Latest stable (1.15.x or higher) | **MUST be installed before RHOAI upgrade.** Provides TLS certificate management for all RHOAI components. |
| **Red Hat Service Mesh v3** | Installed automatically by RHOAI 3.3+ | Service Mesh v3 is deployed automatically during RHOAI upgrade. If legacy v2 exists, it must be removed first. |
| **Red Hat build of Kueue** | Latest stable release | Quota management and workload scheduling framework. Installed via OperatorHub. |
| **JobSet Operator** | Latest stable release | Distributed job orchestration. Installed via OperatorHub. |
| **Support Protocol** | Proactive Case Mandatory | Opening a **Proactive Case** with Red Hat Support is required prior to execution. |

---

### 2.2 Pre-Flight Dependency Validation

Before beginning any operator installations, validate that your cluster meets foundational requirements:

#### Verify Cluster Version
```bash
oc version
# Expected output: OpenShift v4.19.9 or higher
```

#### Check Current RHOAI Version
```bash
oc get subscription -n redhat-ods-operator redhat-ods-operator -o jsonpath='{.spec.startingCSV}' && echo
# Example output: redhat-ods-operator.v2.25.10-...
```

#### Validate Available CPU and Memory
```bash
oc describe nodes | grep -A 5 "Allocated resources"
# Ensure sufficient capacity for upgrade workloads
```

#### Verify Storage Capacity
```bash
oc get pvc --all-namespaces | grep -E "redhat-ods|rh-mig"
oc get nodes -o json | jq -r '.items[] | "\(.metadata.name): \(.status.allocatable.storage)"'
```

---

### 2.3 Installing Mandatory Operators

In RHOAI 3.3+, three operators must be installed before upgrade: **cert-manager**, **Red Hat build of Kueue**, and **JobSet Operator**. Red Hat Service Mesh v3 will be installed automatically by RHOAI during the upgrade process.

#### 2.3.1 cert-manager for Red Hat OpenShift

**cert-manager** is a Kubernetes certificate management controller that automates TLS/SSL lifecycle management. RHOAI 3.3+ **requires** cert-manager for webhook security and certificate automation.

##### Why cert-manager is Mandatory

- **TLS Certificate Automation:** Automatically provisions and renews certificates for RHOAI services
- **Webhook Security:** Secures mutating/validating webhooks in Model Serving components
- **Integration:** Required by KServe `InferenceServices` for secure model traffic

##### Installation Steps

**Via OpenShift Web Console:**
1. Navigate to **Operators** → **OperatorHub**
2. Search for `cert-manager`
3. Click on **cert-manager for Red Hat OpenShift**
4. Click **Install**
5. Select:
   - **Update channel:** `stable` (or latest)
   - **Installation mode:** `All namespaces on the cluster`
   - **Approval strategy:** `Automatic` (recommended)
6. Click **Install**

**Via CLI:**
```bash
cat <<'EOF' | oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: cert-manager
  namespace: openshift-operators
spec:
  channel: stable
  installPlanApproval: Automatic
  name: cert-manager
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF

# Verify installation
oc get pods -n cert-manager --watch
# Wait for cert-manager-webhook, cert-manager-cainjector, and cert-manager controller pods to reach Running state
```

##### Verification

After cert-manager installation, validate full readiness:

```bash
# Check cert-manager namespace exists
oc get namespace cert-manager

# Verify all cert-manager pods are Running
oc get pods -n cert-manager -o custom-columns='NAME:.metadata.name,READY:.status.conditions[?(@.type=="Ready")].status,STATUS:.status.phase'

# Expected output:
# NAME                                      READY   STATUS
# cert-manager-...                          True    Running
# cert-manager-webhook-...                  True    Running
# cert-manager-cainjector-...               True    Running

# Verify cert-manager CRDs are present
oc get crd | grep cert-manager
# Expected: certificate.cert-manager.io, clusterissuer.cert-manager.io, issuer.cert-manager.io, etc.
```

---

#### 2.3.2 Red Hat build of Kueue

Kueue is the quota management and workload scheduling framework required for distributed training jobs and resource-constrained environments in RHOAI 3.3+.

**Via OpenShift Web Console:**
1. Navigate to **Operators** → **OperatorHub**
2. Search for `kueue`
3. Click on **Red Hat build of Kueue**
4. Click **Install**
5. Select:
   - **Update channel:** `stable` (latest release)
   - **Installation mode:** `All namespaces on the cluster`
   - **Approval strategy:** `Automatic`
6. Click **Install**

**Via CLI:**
```bash
cat <<'EOF' | oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: kueue
  namespace: openshift-operators
spec:
  channel: stable
  installPlanApproval: Automatic
  name: kueue
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF

# Verify Kueue controller is running
oc get pods -n kueue-system --watch
# Expected pods: kueue-controller-manager-*
```

---

#### 2.3.3 JobSet Operator

JobSet provides distributed job orchestration and is required for coordinated multi-pod workload management in RHOAI.

**Via OpenShift Web Console:**
1. Navigate to **Operators** → **OperatorHub**
2. Search for `jobset`
3. Click on **JobSet**
4. Click **Install**
5. Select:
   - **Update channel:** `stable`
   - **Installation mode:** `All namespaces on the cluster`
   - **Approval strategy:** `Automatic`
6. Click **Install**

**Via CLI:**
```bash
cat <<'EOF' | oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: jobset
  namespace: openshift-operators
spec:
  channel: stable
  installPlanApproval: Automatic
  name: jobset
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF

# Verify JobSet controller is running
oc get pods -n jobset-system --watch
# Expected pods: jobset-controller-manager-*
```

---

### 2.4 Operator Configuration (Kueue & JobSet)

After successful installation of Kueue and JobSet operators, configure them for RHOAI integration.

#### 2.4.1 Kueue Configuration

Apply the global cluster resource for framework integration:

```bash
cat <<'EOF' | oc apply -f -
apiVersion: kueue.x-k8s.io/v1beta1
kind: Kueue
metadata:
  name: cluster
  namespace: kueue-system
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
EOF

# Verify Kueue config applied
oc get kueue cluster -n kueue-system -o yaml | grep integrations -A 10
```

#### 2.4.2 JobSet Configuration

Apply the JobSet operator configuration:

```bash
cat <<'EOF' | oc apply -f -
apiVersion: operator.openshift.io/v1
kind: JobSetOperator
metadata:
  name: cluster
spec:
  logLevel: Normal
  managementState: Managed
  operatorLogLevel: Normal
EOF

# Verify JobSet config applied
oc get jobsetoperator cluster -o yaml | grep managementState
```

> [!CAUTION]
> **Critical Troubleshooting Step:** If Kueue controller reconciliation fails due to leftover CRD schema conflicts from prior releases, manually remove stale CRDs:
> ```bash
> # Check for conflicting CRDs
> oc get crd | grep kueue
>
> # Remove stale versions if necessary
> oc delete crd cohorts.kueue.x-k8s.io topologies.kueue.x-k8s.io 2>/dev/null || true
>
> # Re-apply Kueue operator via OperatorHub (will auto-recreate CRDs)
> ```

---

### 2.5 Baseline Stabilization Procedure (v2.25.10)

Before executing structural upgrades, bring the RHOAI operator to the latest release of the 2.x stream:

1. Navigate to **Installed Operators** → **Red Hat OpenShift AI** → **Subscriptions**.
2. Change the update channel to `stable-2.25.10` (or the latest version in the 2.25 stream).
3. Approve the generated `InstallPlan` and monitor execution until deployment stabilizes across the cluster. Depending on your current version, execute sequential updates until reaching the target baseline.

**Via CLI:**
```bash
# Get current subscription
oc get subscription -n redhat-ods-operator redhat-ods-operator -o yaml

# Patch to latest 2.25.x channel
oc patch subscription redhat-ods-operator -n redhat-ods-operator \
  -p '{"spec":{"channel":"stable-2.25.10"}}' --type merge

# Monitor installation plan
oc get installplan -n redhat-ods-operator --watch

# When InstallPlan appears, approve it
INSTALL_PLAN=$(oc get installplan -n redhat-ods-operator --sort-by=.metadata.creationTimestamp -o jsonpath='{.items[-1].metadata.name}')
oc patch installplan $INSTALL_PLAN -n redhat-ods-operator --type merge -p '{"spec":{"approved":true}}'

# Monitor operator rollout
oc rollout status deployment/rhods-operator -n redhat-ods-operator --timeout=5m
```

---

### 2.6 Pre-Upgrade Assessment Protocol

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
  namespace: rh-mig
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
      serviceAccountName: default
      containers:
        - name: rhai-cli
          image: registry.redhat.io/rhoai/rhai-cli-rhel9:v3.3.5
          imagePullPolicy: IfNotPresent
          command:
            - sleep
            - infinity
          resources:
            requests:
              cpu: 500m
              memory: 512Mi
            limits:
              cpu: 1000m
              memory: 1Gi
          volumeMounts:
            - name: backup
              mountPath: /tmp/rhoai-upgrade-backup
  volumeClaimTemplates:
    - metadata:
        name: backup
      spec:
        accessModes:
          - ReadWriteOnce
        # storageClassName: <your-storage-class> # Uncomment and set if needed (defaults to cluster default)
        resources:
          requests:
            storage: 5Gi
EOF
```

#### Step C: Grant RBAC Permissions and Run Assessment
```bash
# Provision Service Account Permissions
oc create clusterrolebinding rhoai-cli \
  --clusterrole=cluster-admin \
  --serviceaccount=rh-mig:default

# Wait for pod to reach Running state
oc wait --for=condition=Ready pod -l app=rhai-cli -n rh-mig --timeout=300s

# Execute Lint Assessment inside Pod
oc exec -it rhai-cli-0 -n rh-mig -- \
  /opt/rhai-cli/bin/rhai-cli lint --target-version 3.3.5 > /tmp/rhoai-assessment-report.yaml

# Copy result from pod
oc cp rh-mig/rhai-cli-0:/tmp/rhoai-upgrade-backup/rhai-cli.yaml \
  ./rhai-cli-report.yaml

# Review the report
cat ./rhai-cli-report.yaml | grep -E "^status:|failed:|prohibited:"
```

> [!IMPORTANT]
> **Gate Criteria:** Inspect the output file (`rhai-cli.yaml`). If the summary contains `Failed` or `Prohibited` statuses, **halt the migration immediately** until remediation is complete. Attach `rhai-cli-report.yaml` to your Red Hat support ticket and address all flagged issues before proceeding.

---

### 2.7 Neutralizing DataScienceCluster (DSC) and DSCI

To prevent reconciliation loop conflicts during operator binary updates, set all components to `Removed` state:

1. Edit the `DataScienceCluster` instance:
   ```bash
   oc edit DataScienceCluster default-dsc -n redhat-ods-applications
   ```

2. Update component management states in the custom resource:
   ```yaml
   apiVersion: datasciencecluster.opendatahub.io/v1
   kind: DataScienceCluster
   metadata:
     name: default-dsc
     namespace: redhat-ods-applications
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
   oc patch dsci default-dsci -n redhat-ods-applications --type='merge' \
     -p '{"spec":{"serviceMesh":{"managementState":"Removed"}}}'
   ```

4. Verify all components are `Removed`:
   ```bash
   oc get dsc default-dsc -n redhat-ods-applications -o jsonpath='{.status.conditions[?(@.type=="Ready")]}'
   ```

---

### 2.8 Legacy Service Mesh v2 Removal

Because RHOAI 3.3+ standardizes on **Red Hat Service Mesh Operator v3**, legacy Service Mesh v2 (2.x) becomes obsolete and **must be uninstalled prior to selecting the 3.3 release channel**.

#### Step A: Remove Service Mesh v2 via Web Console
1. Navigate to **Operators** → **Installed Operators**
2. Switch to **All Namespaces**
3. Find **Red Hat OpenShift Service Mesh** (v2.x)
4. Click the three-dot menu and select **Uninstall Operator**
5. Confirm deletion of all associated resources

#### Step B: Clean Up Service Mesh v2 Namespaces
```bash
# Remove istio-system namespace (contains Istio v2 control plane)
oc delete namespace istio-system --ignore-not-found

# Remove maistra-operator-ns if it exists
oc delete namespace maistra-operator-ns --ignore-not-found

# Verify removal
oc get ns | grep -E "istio|maistra" || echo "All Service Mesh v2 namespaces removed"
```

#### Step C: Verify v3 Installation Ready
```bash
oc get pods -n openshift-operators | grep -i "mesh\|istio"
# Should show only v3 components after cleanup
```

---

## 3. InferenceServices Conversion (Serverless to RawDeployment)

RHOAI 3.3 introduces `RawDeployment` as the default deployment model for model serving. This shift streamlines the networking stack and eliminates Knative/Istio v2 overhead for standard serving instances.

### 3.1 Auditing Active Workloads

Audit active production namespaces to identify existing Serverless workloads:

```bash
# List all InferenceServices and their deployment modes
for ns in $(oc get ns -o jsonpath='{.items[*].metadata.name}'); do
  ISVC=$(oc get isvc -n $ns -o json 2>/dev/null)
  if [ ! -z "$ISVC" ]; then
    echo "=== Namespace: $ns ==="
    oc get isvc -n $ns -o custom-columns='NAME:.metadata.name,MODE:.status.deploymentMode,READY:.status.conditions[?(@.type=="Ready")].status' 2>/dev/null
  fi
done
```

Alternatively, for a specific namespace:
```bash
oc get isvc -n <namespace> -o json | jq -r '["NAME","DEPLOYMENT_MODE","READY"], (.items[] | [.metadata.name, .status.deploymentMode, (.status.conditions[] | select(.type=="Ready") | .status)]) | @tsv' | column -t
```

### 3.2 Executing the Conversion Helper Script

Run the interactive conversion script (*Note: In In-Place mode, legacy resources are deleted and re-created under new specifications*):

```bash
# Locate the conversion script
oc exec -it rhai-cli-0 -n rh-mig -- find /opt -name "*serverless-to-raw*" -type f

# Execute conversion for a target namespace
oc exec -it rhai-cli-0 -n rh-mig -- \
  /opt/rhai-upgrade-helpers/model-serving/before-upgrade/serverless-to-raw.sh \
  -n <namespace> \
  --delete-existing
```

#### Interactive Prompt Response Guide:

| Interactive Script Prompt | Recommended Technical Response |
| :--- | :--- |
| **InferenceServices Selection** | `all` *(Converts all ISVC in the namespace)* |
| **Naming Strategy Option** | `1` *(Preserve original names / In-place replacement)* |
| **Confirm Replacement** | `yes` |
| **Preserve Transformed Manifests** | `y` *(Saves YAML for rollback/audit)* |

### 3.3 Post-Conversion Verification

Verify that all `InferenceServices` reflect `RawDeployment` status with `Ready: True`:

```bash
# Verify deployment mode transition
oc get isvc -n <namespace> -o custom-columns='NAME:.metadata.name,MODE:.status.deploymentMode,READY:.status.conditions[?(@.type=="Ready")].status'

# Expected output:
# NAME                        MODE            READY
# llama-model-isvc            RawDeployment   True
# granite-model-isvc          RawDeployment   True

# Verify pods are running under new deployment strategy
oc get pods -n <namespace> -l app.kubernetes.io/part-of=kserve
```

---

## 4. Executing the Intermediate Upgrade (RHOAI 3.3)

Access to version 3.3 is restricted behind the `support-required-upgrade` channel. This guardrail ensures that environment readiness has been verified through Red Hat Support.

### 4.1 Channel Transition Protocol

1. Open the OpenShift Web Console and go to **Installed Operators** → **Red Hat OpenShift AI Subscription**.
2. Switch the update channel to `support-required-upgrade`.
3. When the upgrade status changes to **Requires Approval**, click the notification.
4. Review the `InstallPlan` (which includes Service Mesh v3 dependencies) and click **Approve**.

**Via CLI:**
```bash
# Switch channel to support-required-upgrade
oc patch subscription redhat-ods-operator -n redhat-ods-operator \
  -p '{"spec":{"channel":"support-required-upgrade"}}' --type merge

# Monitor for InstallPlan creation
oc get installplan -n redhat-ods-operator --watch

# Get latest InstallPlan
INSTALL_PLAN=$(oc get installplan -n redhat-ods-operator --sort-by=.metadata.creationTimestamp -o jsonpath='{.items[-1].metadata.name}')

# Review the plan before approval
oc describe installplan $INSTALL_PLAN -n redhat-ods-operator

# Approve for execution
oc patch installplan $INSTALL_PLAN -n redhat-ods-operator --type merge -p '{"spec":{"approved":true}}'

# Monitor upgrade progress
oc rollout status deployment/rhods-operator -n redhat-ods-operator --timeout=10m
```

### 4.2 Health Validation Commands

Monitor operator and system pod readiness using custom column outputs:

**Operator Namespace:**
```bash
oc get pods -n redhat-ods-operator -o custom-columns='NAME:.metadata.name,READY:.status.conditions[?(@.type=="Ready")].status,STATUS:.status.phase,RESTARTS:.status.containerStatuses[0].restartCount'
```

**Applications Namespace:**
```bash
oc get pods -n redhat-ods-applications -o custom-columns='NAME:.metadata.name,READY:.status.conditions[?(@.type=="Ready")].status,STATUS:.status.phase,RESTARTS:.status.containerStatuses[0].restartCount'
```

**Alternative: Comprehensive Health Check**
```bash
echo "=== RHOAI Operator Status ==="
oc get deployment -n redhat-ods-operator rhods-operator -o wide

echo "=== Subscription Channel ==="
oc get subscription -n redhat-ods-operator redhat-ods-operator -o jsonpath='{.spec.channel}' && echo

echo "=== Installation Plan Status ==="
oc get installplan -n redhat-ods-operator --sort-by=.metadata.creationTimestamp | tail -1

echo "=== Cert-Manager Pods ==="
oc get pods -n cert-manager -o custom-columns='NAME:.metadata.name,READY:.status.conditions[?(@.type=="Ready")].status'

echo "=== Service Mesh v3 Status (Auto-Installed) ==="
oc get pods -n openshift-operators | grep -i mesh | head -5

echo "=== Kueue Pods ==="
oc get pods -n kueue-system 2>/dev/null || echo "Kueue not yet installed"

echo "=== JobSet Pods ==="
oc get pods -n jobset-system 2>/dev/null || echo "JobSet not yet installed"
```

Once components stabilize, set the operator subscription channel to `stable-3.3` or `stable-3.x`:

```bash
oc patch subscription redhat-ods-operator -n redhat-ods-operator \
  -p '{"spec":{"channel":"stable-3.3"}}' --type merge
```

---

## 5. Upgrading to RHOAI 3.5 & Transitioning to DSC v2 API

After stabilizing on 3.3, moving toward version 3.5 introduces the **DataScienceCluster `v2` API**, enabling features such as NVIDIA NIM serving and granular evaluation guardrails.

> [!NOTE]
> Ensure all RHOAI 3.3.x patches are stabilized before advancing to 3.5. Allow at least 1 week of production stability at 3.3 before proceeding.

### 5.1 Complete DSC v2 Configuration Manifest

Re-enable active components under the modernized API specification:

```bash
cat <<'EOF' | oc apply -f -
apiVersion: datasciencecluster.opendatahub.io/v2
kind: DataScienceCluster
metadata:
  name: default-dsc
  namespace: redhat-ods-applications
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
EOF

# Verify DSC v2 applied successfully
oc get dsc default-dsc -n redhat-ods-applications -o jsonpath='{.apiVersion}' && echo

# Monitor component reconciliation
oc describe dsc default-dsc -n redhat-ods-applications
```

> [!NOTE]
> The `kueue` component is marked as `Unmanaged` inside the DSC v2 spec because its lifecycle is independently managed by the standalone Kueue operator installed in Section 2.4. This prevents dual reconciliation conflicts.

---

## 6. Troubleshooting & Common Issues

### Issue: cert-manager pods not starting

**Symptoms:**
- cert-manager pods in `Pending` or `CrashLoopBackOff` state
- Error: `insufficient cpu` or `insufficient memory`

**Resolution:**
```bash
# Check resource requests
oc get deployment -n cert-manager cert-manager -o jsonpath='{.spec.template.spec.containers[0].resources}' | jq

# If insufficient resources, reduce requests
oc patch deployment cert-manager -n cert-manager --type='json' \
  -p='[{"op":"replace","path":"/spec/template/spec/containers/0/resources/requests/cpu","value":"100m"}]'

# Or add resources to cluster nodes
oc adm top nodes
```

### Issue: Kueue CRD conflicts from previous versions

**Symptoms:**
- Kueue controller logs: `error: resource mapping not found`
- InstallPlan stuck in `Installing` state

**Resolution:**
```bash
# List conflicting Kueue CRDs
oc get crd | grep kueue.x-k8s.io

# Delete stale CRDs (operator will recreate)
oc delete crd \
  cohorts.kueue.x-k8s.io \
  localqueues.kueue.x-k8s.io \
  resourceflavors.kueue.x-k8s.io \
  workloadpriorityclass.kueue.x-k8s.io \
  --ignore-not-found

# Restart Kueue operator pod
oc delete pod -n openshift-operators -l app.kubernetes.io/name=kueue
```

### Issue: DSC v2 transition fails with API version mismatch

**Symptoms:**
- Error: `no matches for kind "DataScienceCluster" in version "datasciencecluster.opendatahub.io/v2"`

**Resolution:**
```bash
# Verify RHOAI version is truly 3.5+
oc get csv -n redhat-ods-operator redhat-ods-operator.v* -o jsonpath='{.items[0].spec.version}'

# If still on 3.3, complete the 3.5 upgrade channel switch
oc patch subscription redhat-ods-operator -n redhat-ods-operator \
  -p '{"spec":{"channel":"stable-3.5"}}' --type merge

# Wait for InstallPlan and approve
sleep 30
INSTALL_PLAN=$(oc get installplan -n redhat-ods-operator --sort-by=.metadata.creationTimestamp -o jsonpath='{.items[-1].metadata.name}')
oc patch installplan $INSTALL_PLAN -n redhat-ods-operator --type merge -p '{"spec":{"approved":true}}'

# Monitor upgrade
oc rollout status deployment/rhods-operator -n redhat-ods-operator --timeout=10m
```

### Issue: InferenceServices stuck in `NotReady` after RawDeployment conversion

**Symptoms:**
- ISVC `Ready` condition remains `False`
- KServe predictor pods not starting

**Resolution:**
```bash
# Check ISVC events for errors
oc describe isvc <isvc-name> -n <namespace>

# Check predictor deployment logs
oc logs deployment/<isvc-name>-predictor -n <namespace> --tail=50

# Verify cert-manager certificates were generated
oc get certificate -n <namespace> | grep <isvc-name>

# If no certificates, verify cert-manager ClusterIssuer exists
oc get clusterissuer

# Manually trigger ISVC reconciliation
oc annotate isvc <isvc-name> -n <namespace> \
  --overwrite \
  kserve.io/force-reconcile="$(date +%s)"
```

### Issue: JobSet pods in `Pending` state

**Symptoms:**
- JobSet controller logs: `unable to schedule pod`
- Node capacity insufficient

**Resolution:**
```bash
# Check node resources
oc adm top nodes

# Check pod resource requests
oc describe pod <jobset-pod> -n <namespace> | grep -A 5 Requests

# Add more nodes to cluster or reduce pod resource requests
# Alternatively, set resource quotas properly
oc describe resourcequota -n <namespace>
```

---

## 7. Success Readiness Checklist & Conclusion

Before handing off the environment to data science engineering teams, confirm all operational validation items:

### Pre-Migration Validation
- [ ] **Cluster Version:** OCP running v4.19.6 or higher
- [ ] **RHOAI Baseline:** Stabilized on v2.25.10 or latest 2.25.x
- [ ] **Support Case:** Proactive Support case opened with Red Hat
- [ ] **Assessment Report:** Pre-upgrade `rhai-cli` assessment completed with no `Failed` or `Prohibited` flags

### Pre-Upgrade Operators (Manual Installation)
- [ ] **cert-manager:** All pods in `cert-manager` namespace reporting `Ready: True`
- [ ] **Kueue:** Controller manager pod active in `kueue-system`
- [ ] **JobSet:** Controller manager pod active in `jobset-system`

### Pre-Upgrade Cleanup
- [ ] **Service Mesh v2:** Completely removed from cluster (if exists from previous deployments)

### Post-Upgrade Validation
- [ ] **RHOAI Version:** Operator actively running version `3.5.x`
- [ ] **Service Mesh v3:** Auto-installed and healthy (validate Istio control plane pods in `openshift-operators`)
- [ ] **DSC Status:** Both `DataScienceCluster` and `DSCInitialization` reporting `Ready: True`
- [ ] **Pod Readiness:** All pods in `redhat-ods-operator` and `redhat-ods-applications` showing `Ready: True`
- [ ] **Gateway Configuration:** `oc get gatewayconfigs --all-namespaces` displays resources as `READY: True`
- [ ] **Inference Services:** All active `InferenceServices` fully operational in `RawDeployment` mode
- [ ] **Hardware Profiles:** Successful migration from `AcceleratorProfiles` to `HardwareProfiles` (infrastructure.opendatahub.io)
- [ ] **Subscription Channel:** Operator subscription tracking `stable-3.x` or `stable-3.5`

### Data Science Workload Validation
- [ ] **Model Serving:** Models accessible via KServe endpoints
- [ ] **Ray Clusters:** Ray Job resources successfully launching and completing
- [ ] **Training Jobs:** Training operator successfully running distributed training
- [ ] **Workbenches:** JupyterHub workbenches launching with proper resource allocation

### Conclusion

Upgrading to **RHOAI 3.5** delivers a modernized, resilient, and enterprise-grade AI foundation. The shift to `RawDeployment` and standalone queue management via Kueue gives the platform the stability needed to support large-scale AI workloads with reduced operational complexity and maximum throughput. With cert-manager handling TLS lifecycle and Service Mesh v3 providing advanced traffic management, your RHOAI infrastructure is positioned for sustained growth and operational excellence.

---

## Appendix: Useful Commands Reference

```bash
# Quick health check script
#!/bin/bash
echo "=== RHOAI Migration Health Check ==="
echo "OCP Version: $(oc version --short)"
echo "RHOAI Operator: $(oc get csv -n redhat-ods-operator redhat-ods-operator.* -o jsonpath='{.items[0].spec.version}' 2>/dev/null)"
echo "cert-manager: $(oc get deployment -n cert-manager cert-manager -o jsonpath='{.status.readyReplicas}/{.status.replicas}' 2>/dev/null || echo 'Not installed')"
echo "Service Mesh v3: $(oc get deployment -n openshift-operators -l app=istiod -o jsonpath='{.items[0].status.readyReplicas}/{.items[0].status.replicas}' 2>/dev/null || echo 'Auto-installing with RHOAI')"
echo "Kueue: $(oc get deployment -n kueue-system kueue-controller-manager -o jsonpath='{.status.readyReplicas}/{.status.replicas}' 2>/dev/null || echo 'Not installed')"
echo "JobSet: $(oc get deployment -n jobset-system jobset-controller-manager -o jsonpath='{.status.readyReplicas}/{.status.replicas}' 2>/dev/null || echo 'Not installed')"
```
