# Compliance Operator & File Integrity Operator as ACM Policy

Deploy and manage the OpenShift Compliance Operator and File Integrity Operator across your fleet using Red Hat Advanced Cluster Management (RHACM) Governance policies.

## What is the Compliance Operator?

The [Compliance Operator](https://docs.openshift.com/container-platform/latest/security/compliance_operator/co-overview.html) is an OpenShift operator that runs OpenSCAP-based compliance scans against your cluster. It leverages the [SCAP Security Guide (SSG)](https://www.open-scap.org/security-policies/scap-security-guide/) content to evaluate cluster and node-level compliance against industry benchmarks such as:

- **CIS Benchmarks** -- Center for Internet Security hardening guidelines for OpenShift
- **NIST SP 800-53 Moderate** -- Federal security controls (maps to SOC 2 Trust Service Criteria)
- **PCI-DSS** -- Payment Card Industry Data Security Standard
- **NERC-CIP** -- North American Electric Reliability Corporation Critical Infrastructure Protection
- **FedRAMP Moderate** -- Federal Risk and Authorization Management Program
- **Australian ACSC Essential Eight / ISM** -- Australian Cyber Security Centre guidelines

The operator produces `ComplianceCheckResult` custom resources for each rule evaluated, making compliance status queryable via the Kubernetes API and visible in the ACM Governance dashboard.

## What is the File Integrity Operator?

The [File Integrity Operator (FIO)](https://docs.openshift.com/container-platform/latest/security/file_integrity_operator/file-integrity-operator-understanding.html) continuously monitors OpenShift node filesystems for unauthorized modifications using [AIDE](https://aide.github.io/) (Advanced Intrusion Detection Environment). While the Compliance Operator performs point-in-time configuration assessments, FIO provides **runtime integrity monitoring** -- detecting changes to critical binaries, configuration files, and system libraries as they happen.

Key capabilities:
- **Continuous monitoring** -- AIDE runs as a DaemonSet on every targeted node, checking file hashes, permissions, ownership, and extended attributes
- **Kubernetes-native results** -- integrity status is reported via `FileIntegrityNodeStatus` custom resources
- **Customizable AIDE configuration** -- define exactly which paths and attributes to monitor
- **SOC 2 CC7.1/CC7.2 coverage** -- satisfies anomaly detection and system monitoring controls that the Compliance Operator alone cannot address

### How They Work Together

| | Compliance Operator | File Integrity Operator |
|---|---|---|
| **Detection model** | Scheduled scan (point-in-time) | Continuous daemon (runtime) |
| **What it checks** | Cluster/node configuration vs. SCAP benchmarks | File hashes, permissions, ownership on node filesystem |
| **SOC 2 coverage** | CC3, CC5, CC6, CC8 | CC7.1 (anomaly detection), CC7.2 (monitoring) |
| **Remediation** | Can auto-apply `ComplianceRemediation` CRs | Alert-only (no auto-remediation) |

The Compliance Operator's NIST Moderate profile includes the rule `ocp4-file-integrity-operator-exists`, which **checks whether FIO is installed**. Deploying both operators together ensures that rule passes and provides defense-in-depth.

## Repository Structure

```
.
├── 00-namespace.yaml                          # Hub namespace for policy resources
├── 01-policy-install-compliance-operator.yaml  # Install operator (NS, OperatorGroup, Subscription)
├── 02-policy-cis-scan.yaml                     # CIS benchmark scan configuration
├── 03-policy-check-compliance-results.yaml     # Inform-only policy to surface failures
├── 04-placement.yaml                           # Placement targeting OpenShift clusters
├── 05-placementbindings.yaml                   # Binds policies to placement
├── 06-policyset.yaml                           # Groups all policies for dashboard view
├── 07-policy-soc2-scan.yaml                    # SOC 2 / NIST Moderate scan configuration
├── 08-policy-install-file-integrity-operator.yaml  # Install FIO (NS, OperatorGroup, Subscription)
├── 09-policy-configure-file-integrity.yaml     # FileIntegrity CRs + custom AIDE config
├── 10-policy-check-file-integrity-results.yaml # Inform-only policy to surface integrity failures
├── kustomization.yaml                          # Kustomize overlay for deployment
└── README.md
```

### Policy Dependency Chain

Policies are ordered using `spec.dependencies` to ensure correct sequencing:

```
policy-install-compliance-operator
    ├── policy-cis-compliance-scan            (waits for operator install)
    ├── policy-soc2-compliance-scan           (waits for operator install)
    └── policy-check-compliance-results      (waits for CIS scan)

policy-install-file-integrity-operator
    └── policy-configure-file-integrity      (waits for FIO install)
        └── policy-check-file-integrity-results  (waits for FIO configuration)
```

## Deployment

### Quick Start (Direct Apply)

```bash
oc login --token=<token> --server=<acm-hub-api-url>
oc apply -k .
```

### GitOps App-of-Apps Pattern (Recommended)

This repository is designed to be consumed as a child application in an [Argo CD App-of-Apps](https://argo-cd.readthedocs.io/en/stable/operator-manual/cluster-bootstrapping/#app-of-apps-pattern) deployment model with OpenShift GitOps.

#### 1. Create the parent App-of-Apps

On your ACM hub cluster, create a root Argo CD `Application` that points to a Git repo containing child application manifests:

```yaml
# argocd/root-application.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: acm-policies
  namespace: openshift-gitops
spec:
  project: default
  source:
    repoURL: https://github.com/<org>/acm-hub-config.git
    targetRevision: main
    path: argocd/apps
  destination:
    server: https://kubernetes.default.svc
    namespace: openshift-gitops
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

#### 2. Add the Compliance Operator child application

In the `argocd/apps/` directory of your hub config repo, create a child `Application` that points to this repository:

```yaml
# argocd/apps/compliance-operator-policies.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: compliance-operator-policies
  namespace: openshift-gitops
  labels:
    app.kubernetes.io/part-of: acm-policies
  annotations:
    argocd.argoproj.io/sync-wave: "10"
spec:
  project: default
  source:
    repoURL: https://github.com/<org>/compliance-operator-as-acm-policy.git
    targetRevision: main
    path: .
  destination:
    server: https://kubernetes.default.svc
    namespace: compliance-operator-policies
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
```

#### 3. Full App-of-Apps directory structure

```
acm-hub-config/
├── argocd/
│   ├── root-application.yaml
│   └── apps/
│       ├── compliance-operator-policies.yaml    # <-- this repo
│       ├── certificate-policies.yaml
│       ├── network-policies.yaml
│       └── rbac-policies.yaml
```

#### 4. Kustomize overlays for environment-specific configuration

If you have different scan schedules or profiles per environment, use Kustomize overlays:

```
overlays/
├── production/
│   └── kustomization.yaml      # patches scan schedule to weekly
├── staging/
│   └── kustomization.yaml      # patches scan schedule to daily
└── development/
    └── kustomization.yaml      # disables SOC 2 scan
```

Example overlay to change the CIS scan schedule for production:

```yaml
# overlays/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../
patches:
  - target:
      kind: Policy
      name: policy-cis-compliance-scan
    patch: |
      - op: replace
        path: /spec/policy-templates/0/objectDefinition/spec/object-templates/0/objectDefinition/spec/schedule
        value: "0 3 * * 0"
```

Point the child Application's `spec.source.path` to the appropriate overlay:

```yaml
source:
  path: overlays/production
```

## Scan Configuration Reference

### ScanSetting

A `ScanSetting` defines how and when scans execute:

```yaml
apiVersion: compliance.openshift.io/v1alpha1
kind: ScanSetting
metadata:
  name: my-scan-setting
  namespace: openshift-compliance
spec:
  # Cron schedule for recurring scans
  schedule: "0 1 * * *"              # Daily at 1:00 AM

  # Number of scan result rotations to keep
  rotation: 3

  # Node roles to scan (node-level profiles only)
  roles:
    - worker

  # Storage for raw ARF results
  rawResultStorage:
    pvAccessModes:
      - ReadWriteOnce
    size: 2Gi
    rotation: 3

  # Auto-apply eligible remediations (use with caution)
  # autoApplyRemediations: false

  # Suspend scheduled scans without deleting the resource
  # suspend: false
```

**Built-in ScanSettings** (created automatically by the operator):

| Name | Schedule | Roles | Storage |
|------|----------|-------|---------|
| `default` | Daily 1 AM | worker, master | 1Gi |
| `default-auto-apply` | Daily 1 AM | worker, master | 1Gi, auto-remediate |

### ScanSettingBinding

A `ScanSettingBinding` connects profiles to a scan setting:

```yaml
apiVersion: compliance.openshift.io/v1alpha1
kind: ScanSettingBinding
metadata:
  name: my-scan
  namespace: openshift-compliance
spec:
  settingsRef:
    name: my-scan-setting             # References the ScanSetting above
    kind: ScanSetting
    apiGroup: compliance.openshift.io/v1alpha1
  profiles:
    - name: ocp4-cis                  # Platform-level CIS profile
      kind: Profile
      apiGroup: compliance.openshift.io/v1alpha1
    - name: ocp4-cis-node             # Node-level CIS profile
      kind: Profile
      apiGroup: compliance.openshift.io/v1alpha1
```

### Available Profiles

List profiles available on a managed cluster:

```bash
oc get profiles.compliance -n openshift-compliance
```

Common profiles for OpenShift 4.x:

| Profile | Description | SOC 2 Mapping |
|---------|-------------|---------------|
| `ocp4-cis` | CIS Benchmark (platform) | CC6.1, CC6.6, CC6.7 |
| `ocp4-cis-node` | CIS Benchmark (node) | CC6.1, CC6.6 |
| `ocp4-moderate` | NIST 800-53 Moderate (platform) | Full SOC 2 TSC coverage |
| `ocp4-moderate-node` | NIST 800-53 Moderate (node) | Full SOC 2 TSC coverage |
| `ocp4-pci-dss` | PCI-DSS v3.2.1 (platform) | CC6.1, CC6.5 |
| `ocp4-pci-dss-node` | PCI-DSS v3.2.1 (node) | CC6.1, CC6.5 |
| `ocp4-high` | NIST 800-53 High (platform) | Superset of Moderate |
| `ocp4-high-node` | NIST 800-53 High (node) | Superset of Moderate |
| `ocp4-nerc-cip` | NERC-CIP (platform) | CC6.1, CC7.1 |
| `ocp4-nerc-cip-node` | NERC-CIP (node) | CC6.1, CC7.1 |

### SOC 2 Compliance Scanning

There is no direct SOC 2 profile in the Compliance Operator. SOC 2 is an audit framework built around five Trust Service Criteria (TSC), not a technical benchmark. The recommended approach is to scan with **NIST SP 800-53 Moderate** (`ocp4-moderate` / `ocp4-moderate-node`), which provides the strongest coverage of SOC 2 controls:

| SOC 2 Trust Service Criteria | NIST 800-53 Control Family | Profile |
|------------------------------|---------------------------|---------|
| **CC6** -- Logical & Physical Access | AC (Access Control), IA (Identification & Authentication) | `ocp4-moderate`, `ocp4-moderate-node` |
| **CC7** -- System Operations & Monitoring | AU (Audit & Accountability), SI (System & Information Integrity) | `ocp4-moderate`, `ocp4-moderate-node` |
| **CC8** -- Change Management | CM (Configuration Management), SA (System & Services Acquisition) | `ocp4-moderate` |
| **CC3** -- Risk Assessment | RA (Risk Assessment) | `ocp4-moderate` |
| **CC5** -- Control Activities | CA (Security Assessment & Authorization) | `ocp4-moderate` |

The `07-policy-soc2-scan.yaml` in this repo configures exactly this -- daily NIST Moderate scans with 5 rotations retained for audit trail purposes.

For comprehensive SOC 2 coverage, combine with CIS benchmarks:

```yaml
profiles:
  - name: ocp4-moderate
    kind: Profile
    apiGroup: compliance.openshift.io/v1alpha1
  - name: ocp4-moderate-node
    kind: Profile
    apiGroup: compliance.openshift.io/v1alpha1
  - name: ocp4-cis
    kind: Profile
    apiGroup: compliance.openshift.io/v1alpha1
  - name: ocp4-cis-node
    kind: Profile
    apiGroup: compliance.openshift.io/v1alpha1
```

### TailoredProfile (Customizing Rules)

If you need to enable, disable, or adjust specific rules within a profile, use a `TailoredProfile`:

```yaml
apiVersion: compliance.openshift.io/v1alpha1
kind: TailoredProfile
metadata:
  name: ocp4-moderate-soc2-tailored
  namespace: openshift-compliance
spec:
  title: NIST Moderate Tailored for SOC 2
  description: NIST 800-53 Moderate with additional SOC 2 relevant rules enabled
  extends: ocp4-moderate
  disableRules:
    - name: ocp4-scc-limit-container-allowed-capabilities
      rationale: "Not applicable in this environment"
  enableRules:
    - name: ocp4-audit-log-forwarding-enabled
      rationale: "Required for SOC 2 CC7.2 monitoring"
  setValues:
    - name: ocp4-var-openshift-audit-profile
      value: "WriteRequestBodies"
      rationale: "SOC 2 CC7.2 requires detailed audit logging"
```

Reference the `TailoredProfile` in your `ScanSettingBinding` instead of the base profile:

```yaml
profiles:
  - name: ocp4-moderate-soc2-tailored
    kind: TailoredProfile
    apiGroup: compliance.openshift.io/v1alpha1
```

## Accessing Compliance Reports

### From the ACM Governance Dashboard

1. Navigate to **Governance** in the ACM console
2. Click the **Policy sets** tab to see the `compliance-operator-policyset`
3. Each policy shows per-cluster compliance status
4. Click into `policy-check-compliance-results` to see which clusters have failing checks

### From the CLI (on a managed cluster)

```bash
# Check overall suite status
oc get compliancesuites -n openshift-compliance
# NAME             PHASE   RESULT
# cis-compliance   DONE    NON-COMPLIANT
# soc2-compliance  DONE    COMPLIANT

# List all check results
oc get compliancecheckresults -n openshift-compliance

# Filter to only FAIL results
oc get compliancecheckresults -n openshift-compliance \
  -l compliance.openshift.io/check-status=FAIL

# Filter results by suite
oc get compliancecheckresults -n openshift-compliance \
  -l compliance.openshift.io/suite=soc2-compliance

# Get details on a specific failing check
oc describe compliancecheckresult/<result-name> -n openshift-compliance

# View remediations the operator can auto-apply
oc get complianceremediations -n openshift-compliance

# Export results in human-readable format
oc get compliancecheckresults -n openshift-compliance \
  -l compliance.openshift.io/suite=soc2-compliance \
  -o custom-columns=NAME:.metadata.name,STATUS:.status,SEVERITY:.severity,DESCRIPTION:.description
```

### Extracting Raw ARF/XCCDF Reports

For auditors who need the full SCAP report:

```bash
# List scan result PVCs
oc get pvc -n openshift-compliance

# Extract the raw ARF report from a scan result
SCAN_NAME="ocp4-moderate"
POD_NAME=$(oc get pods -n openshift-compliance \
  -l complianceoperator.openshift.io/scan-name=${SCAN_NAME} \
  -o jsonpath='{.items[0].metadata.name}')

oc cp openshift-compliance/${POD_NAME}:/results ./scan-results/

# The results directory will contain ARF XML files that can be
# opened with OpenSCAP Workbench or converted to HTML:
oscap xccdf generate report ./scan-results/report-arf.xml > report.html
```

### Compliance Reports via ACM Observability

If ACM Observability (Thanos) is enabled, compliance metrics are available for Grafana dashboards:

```promql
# Total compliance check results by status
compliance_operator_compliance_state{suite="soc2-compliance"}

# Count of FAIL results per cluster
count(compliance_operator_compliance_state{status="FAIL"}) by (cluster)
```

## File Integrity Operator Configuration

### FileIntegrity CR

A `FileIntegrity` resource deploys AIDE as a DaemonSet on nodes matching the given selector:

```yaml
apiVersion: fileintegrity.openshift.io/v1alpha1
kind: FileIntegrity
metadata:
  name: worker-fileintegrity
  namespace: openshift-file-integrity
spec:
  nodeSelector:
    node-role.kubernetes.io/worker: ""
  config:
    gracePeriod: 900        # Seconds to wait after node boot before first scan
    maxBackups: 5           # Number of AIDE database backups to retain
```

The policies in this repo create a `FileIntegrity` CR for worker nodes (`worker-fileintegrity`), which covers all nodes in the fleet since both the ROSA HCP hub and bare metal managed clusters have worker-only topologies.

### Custom AIDE Configuration

The `09-policy-configure-file-integrity.yaml` includes a custom `aide.conf` ConfigMap that monitors:

| Path | What's Protected |
|------|-----------------|
| `/hostroot/bin`, `/sbin`, `/usr/bin`, `/usr/sbin` | System binaries |
| `/hostroot/etc/kubernetes` | Kubernetes configuration (kubeconfig, manifests, PKI) |
| `/hostroot/etc/cni` | Container network configuration |
| `/hostroot/etc/ssh/sshd_config` | SSH daemon configuration |
| `/hostroot/etc/passwd`, `shadow`, `group`, `gshadow` | User and group databases |
| `/hostroot/etc/sudoers`, `sudoers.d` | Privilege escalation configuration |
| `/hostroot/etc/systemd` | Systemd unit files |

Each path is checked with the attribute set `p+i+n+u+g+s+b+acl+xattrs+sha512`:

| Attribute | Meaning |
|-----------|---------|
| `p` | Permissions |
| `i` | Inode number |
| `n` | Number of hard links |
| `u` | User ownership |
| `g` | Group ownership |
| `s` | File size |
| `b` | Block count |
| `acl` | POSIX ACLs |
| `xattrs` | Extended attributes (SELinux labels) |
| `sha512` | SHA-512 hash of file contents |

### Checking File Integrity Status

```bash
# Check FileIntegrity status on a managed cluster
oc get fileintegrities -n openshift-file-integrity
# NAME                   PHASE
# worker-fileintegrity   Active
# master-fileintegrity   Active

# Check per-node integrity status
oc get fileintegritynodestatuses -n openshift-file-integrity

# View details for a node with integrity failures
oc describe fileintegritynodestatus/<node-name> -n openshift-file-integrity

# Get the AIDE log from a failed node
NODE_NAME="worker-0"
AIDE_POD=$(oc get pods -n openshift-file-integrity \
  -l file-integrity.openshift.io/node=${NODE_NAME} \
  -o jsonpath='{.items[0].metadata.name}')
oc logs ${AIDE_POD} -n openshift-file-integrity

# Re-initialize the AIDE database after approved changes
# (annotate the FileIntegrity to reinitialize)
oc annotate fileintegrities/worker-fileintegrity \
  file-integrity.openshift.io/re-init= \
  -n openshift-file-integrity
```

### File Integrity in the ACM Governance Dashboard

The `policy-check-file-integrity-results` policy reports as **NonCompliant** in the ACM Governance dashboard when:
- The `FileIntegrity` CR is not in `Active` phase (operator not running correctly)
- Any `FileIntegrityNodeStatus` reports a `Failed` condition (unauthorized file changes detected)

This surfaces file integrity violations at the fleet level alongside compliance scan results.

## Architecture

This repo assumes the following topology:

```
┌──────────────────────────────────────┐
│  Hub Cluster (ROSA HCP on AWS)       │
│  ├── RHACM / Governance              │
│  ├── Policy resources (this repo)    │
│  ├── Compliance Operator  ◄── policy │
│  └── File Integrity Operator ◄─ policy│
│      (worker nodes only)             │
└──────────────┬───────────────────────┘
               │ ACM Governance
    ┌──────────┴──────────┐
    ▼                     ▼
┌──────────────┐  ┌──────────────┐
│ Managed       │  │ Managed       │
│ Cluster       │  │ Cluster       │
│ (Bare Metal)  │  │ (Bare Metal)  │
│ worker nodes  │  │ worker nodes  │
└──────────────┘  └──────────────┘
```

- **Hub cluster (ROSA HCP)**: Runs RHACM and is also a target for the policies via `local-cluster`. The Compliance Operator and FIO are deployed on the hub itself.
- **Managed clusters (bare metal)**: Worker-node-only clusters. Compliance scans and file integrity monitoring run on all worker nodes.
- **All clusters** have worker nodes only (no schedulable masters), so scan settings and FIO target the `worker` role exclusively.

The Placement targets all clusters labeled `vendor: OpenShift`, which includes `local-cluster` (the hub) automatically.

## Customization

### Target Specific Clusters

Edit `04-placement.yaml` to narrow cluster targeting:

```yaml
spec:
  predicates:
    - requiredClusterSelector:
        labelSelector:
          matchExpressions:
            - key: vendor
              operator: In
              values:
                - OpenShift
            - key: environment
              operator: In
              values:
                - production
                - staging
```

### Disable a Scan

Set `spec.disabled: true` on any policy, or remove it from the `PolicySet` and `PlacementBinding`.

### Auto-Remediation

To let the Compliance Operator automatically apply fixes, change the `ScanSetting`:

```yaml
spec:
  autoApplyRemediations: true
```

> **Warning**: Auto-remediation may trigger node reboots (e.g., for kernel parameters or MachineConfig changes). Test in non-production first.
