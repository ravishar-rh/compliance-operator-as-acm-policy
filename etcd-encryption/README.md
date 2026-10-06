# Standalone `etcd` Encryption Policy

This directory contains a standalone ACM governance policy to enforce AES-CBC encryption at rest on OpenShift's `etcd` database (`APIServer/cluster`).

## Why This Is a Separate Application

Enabling `etcd` encryption is an infrastructure-level control plane change that:
1. Generates an AES-CBC encryption key.
2. Triggers sequential rolling restarts of `kube-apiserver` and `openshift-apiserver` static pods across control plane nodes.
3. Performs a background migration rewriting all existing `Secrets`, `ConfigMaps`, `Routes`, and OAuth tokens into encrypted format (typically takes 15–30 minutes).

To prevent accidental control plane restarts during regular compliance policy syncs, this change is decoupled into its own Argo CD `Application` (`etcd-encryption/argocd-application.yaml`) intended to be triggered during a maintenance window.

## Cluster Targeting via Placement

This application includes a dedicated ACM `Placement` (`placement-enable-etcd-encryption`) configured with an explicit opt-in label:
* `vendor: OpenShift`
* `enable-etcd-encryption: "true"`

Syncing this application to the Hub will **not** affect any cluster until that cluster is explicitly opted in during a scheduled maintenance window.

### Opting In a Cluster
On the ACM Hub cluster:
```bash
# Label the target cluster to opt it into encryption
oc label managedcluster <cluster-name> enable-etcd-encryption=true

# Verify ACM Placement matched the cluster
oc get placementdecisions -n compliance-operator-policies -l cluster.open-cluster-management.io/placement=placement-enable-etcd-encryption
```

To remove a cluster from future enforcement (note: does not unencrypt existing data):
```bash
oc label managedcluster <cluster-name> enable-etcd-encryption-
```

## Deployment

### Option A: Via Argo CD / OpenShift GitOps (Recommended)
Apply the standalone child application in OpenShift GitOps:
```bash
oc apply -f etcd-encryption/argocd-application.yaml
```
Sync the application manually when ready:
```bash
oc get apps -n openshift-gitops etcd-encryption-policy
```

### Option B: Direct Apply via Kustomize
```bash
oc apply -k etcd-encryption/
```

## Monitoring Progress

Check the encryption status condition on the API server operators:
```bash
# Check kube-apiserver encryption progress
oc get kubeapiserver -o=jsonpath='{range .items[0].status.conditions[?(@.type=="Encrypted")]}{.reason}{": "}{.message}{"\n"}{end}'

# Check openshift-apiserver encryption progress
oc get openshiftapiserver -o=jsonpath='{range .items[0].status.conditions[?(@.type=="Encrypted")]}{.reason}{": "}{.message}{"\n"}{end}'
```

Once migration completes, the condition reports `Encrypted: True`. OpenShift will automatically rotate encryption keys on a weekly basis thereafter.

## Disaster Recovery Note

When `etcd` encryption is enabled, any subsequent `etcd` snapshot taken via `cluster-backup.sh` requires the corresponding encryption keys to be restored. The backup script automatically archives these keys in the static pod resources tarball (`static_kuberesources_<timestamp>.tar.gz`).
