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
2. Delete the Argo CD custom resource (openshift-gitops instance).
3. Wait for the Argo CD instance to be removed.
4. Delete the OpenShift GitOps operator Subscription.
5. Optionally delete the operator CSV and the GitOps namespace.

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
- If the GitOps namespace sticks in `Terminating` due to finalizers, remove blocking finalizers or wait for the operator to release them; the role does not modify finalizers.
