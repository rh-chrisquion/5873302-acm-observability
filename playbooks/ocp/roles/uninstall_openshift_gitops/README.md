# Uninstall OpenShift GitOps (with backup of Argo RBAC and repo connections)

`uninstall_openshift_gitops` uninstalls the OpenShift GitOps (Argo CD) operator from the cluster after saving manifests needed to restore Argo RBAC and repository/cluster connections.

## What is saved

1. **Argo CD RBAC**
   - **ConfigMap** `argocd-rbac-cm` from the GitOps namespace (Argo CD policy CSV and scopes, if present).
   - **ClusterRoleBindings** listed in `gitops_rbac_clusterrolebinding_names` (e.g. `openshift-gitops-acm-admin`, `openshift-gitops-odf-admin`) that grant the Argo CD application controller permissions.

2. **Repo and cluster connections**
   - **Secrets** with label `argocd.argoproj.io/secret-type=repository` (repository credentials).
   - **Secrets** with label `argocd.argoproj.io/secret-type=cluster` (optional, when `save_cluster_connections` is true).

All saved manifests are written under `gitops_backup_dir` (default: `gitops-backup/` relative to the playbook directory).

## Uninstall order

1. Create backup directory and export the manifests above.
2. Run the **wipe_argo** role (included automatically): deletes all Argo CD Applications, ApplicationSets, AppProjects, repository secrets (repos), and repo-creds secrets in the GitOps namespace.
3. Delete the GitOpsService and Argo CD custom resource (openshift-gitops instance).
4. **Before removing the Argo CD `ConsolePlugin`**: patch the **`openshift-gitops-operator` `Subscription`** in `openshift-operators` to set **`DISABLE_DEFAULT_ARGOCD_CONSOLELINK=true`** in `spec.config.env` (so the operator drops the default ConsoleLink / console integration before the plugin CR is deleted). Optional short pause for reconciliation.
5. Delete the Argo CD **ConsolePlugin** CR(s) (e.g. `gitops`).
6. Delete the OpenShift GitOps operator Subscription.
7. Optionally delete the operator CSV.
8. When **`delete_gitops_namespace: true`**: strip finalizers on **Applications**, **ArgoCD**, **GitopsService**, OLM objects, and workloads **inside** `openshift-gitops`, then **`oc delete`/`k8s` the namespace**, then **`oc patch`** / optional **`jq` + `/finalize`** on the Namespace, then a **retry loop** if the namespace stays **Terminating** (same idea as `uninstall_acm`).

## Requirements

- `kubernetes.core` Ansible collection.
- Cluster access via `KUBECONFIG` or in-cluster authentication.

## Usage

### Basic usage

```yaml
- hosts: localhost
  connection: local
  gather_facts: false
  roles:
    - role: uninstall_openshift_gitops
```

Manifests are written to `./gitops-backup/` by default.

### Custom backup directory

```yaml
- hosts: localhost
  connection: local
  gather_facts: false
  roles:
    - role: uninstall_openshift_gitops
      vars:
        gitops_backup_dir: "/tmp/my-gitops-backup"
```

### Include additional ClusterRoleBindings

```yaml
- hosts: localhost
  connection: local
  gather_facts: false
  roles:
    - role: uninstall_openshift_gitops
      vars:
        gitops_rbac_clusterrolebinding_names:
          - openshift-gitops-acm-admin
          - openshift-gitops-odf-admin
          - my-custom-argocd-admin
```

### Uninstall and remove the GitOps namespace

```yaml
- hosts: localhost
  connection: local
  gather_facts: false
  roles:
    - role: uninstall_openshift_gitops
      vars:
        delete_gitops_namespace: true
```

## Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `gitops_namespace` | `openshift-gitops` | Argo CD instance namespace. |
| `gitops_operator_namespace` | `openshift-operators` | Namespace of the GitOps operator subscription. |
| `gitops_operator_subscription_name` | `openshift-gitops-operator` | Name of the Subscription to remove. |
| `gitops_backup_dir` | `{{ playbook_dir }}/gitops-backup` | Directory where manifests are saved. |
| `save_argocd_rbac_cm` | `true` | Save the `argocd-rbac-cm` ConfigMap. |
| `gitops_rbac_clusterrolebinding_names` | See defaults | List of ClusterRoleBinding names to export. |
| `save_repo_connections` | `true` | Save repository connection secrets. |
| `save_cluster_connections` | `false` | Save cluster connection secrets. |
| `delete_gitops_namespace` | `false` | Delete the GitOps namespace after uninstall. |
| `delete_operator_csv` | `true` | Delete the operator CSV after removing the subscription. |
| `disable_argocd_consolelink_via_subscription` | `true` | Patch Subscription with `DISABLE_DEFAULT_ARGOCD_CONSOLELINK=true` before deleting the ConsolePlugin. |
| `gitops_consolelink_disable_pause_seconds` | `15` | Seconds to pause after the patch (set `0` to skip). |
| `gitops_console_plugin_names` | `["gitops"]` | ConsolePlugin resources to delete after the patch. |
| `strip_namespace_finalizers_on_delete` | `true` | After requesting **`openshift-gitops`** deletion, patch namespace finalizers + optional `jq` `/finalize`. |
| `strip_namespace_use_jq_finalize` | `true` | `oc get ns … \| jq '(.spec //= {}) \| .spec.finalizers = [] \| .metadata.finalizers = []' \| oc replace --raw …/finalize` (set `false` if `jq` is missing). |
| `strip_finalizers_on_namespace_contents` | `true` | Before namespace delete, clear finalizers on Argo/OLM/workload objects in the GitOps namespace. |
| `gitops_namespace_aggressive_terminating_recovery` | `true` | Retry loop to strip finalizers if the namespace stays **Terminating**. |
| `gitops_namespace_terminating_recovery_attempts` | `12` | Retry count. |
| `gitops_namespace_terminating_recovery_delay_seconds` | `15` | Sleep between retries. |

## Re-applying saved manifests

After reinstalling OpenShift GitOps:

1. Recreate the GitOps namespace if you deleted it: `oc create namespace openshift-gitops`.
2. Wait for the Argo CD instance to be ready.
3. Apply the saved manifests (ConfigMaps and Secrets in the GitOps namespace, ClusterRoleBindings cluster-wide).  
   Before applying, remove server-managed fields from the saved YAML if you see conflicts: `metadata.resourceVersion`, `metadata.uid`, `metadata.creationTimestamp`, `metadata.generation`, `metadata.managedFields`, and the `status` section.
4. Restore repo and cluster secrets so Argo CD can connect to Git and clusters again.

## Clean reinstall

With `delete_operator_csv: true` and `delete_gitops_namespace: true`, the role removes the Argo CD instance, Subscription, CSV, and GitOps namespace so that OpenShift GitOps can be reinstalled (e.g. via the `install_openshift_gitops` role or `install-argo.yaml`) without conflicts. The playbook `uninstall-openshift-gitops.yaml` runs the role with these options and then verifies that the ArgoCD CR, Subscription, and (when applicable) namespace are gone.

## Notes

- If the GitOps namespace does not exist, backup tasks are skipped and only operator Subscription/CSV removal runs if present.
- Saved secrets contain credentials; restrict access to `gitops_backup_dir` and do not commit it to version control.
- If **`openshift-gitops`** still sticks in **`Terminating`**, ensure **`jq`** is available (or set **`strip_namespace_use_jq_finalize: false`**) and inspect **`oc get all -n openshift-gitops`**. The role strips **Application / AppProject / ArgoCD / GitopsService** and OLM object finalizers before namespace delete, then retries namespace finalizer removal.
