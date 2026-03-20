# Uninstall OpenShift GitOps

`uninstall_openshift_gitops` uninstalls the OpenShift GitOps (Argo CD) operator from the cluster.

## Uninstall order

1. Run the **wipe_argo** role (included automatically): deletes all Argo CD Applications, ApplicationSets, AppProjects, repository secrets (repos), and repo-creds secrets in the GitOps namespace.
2. Delete the GitOpsService and Argo CD custom resource (openshift-gitops instance).
3. **Before removing the Argo CD `ConsolePlugin`**: patch the **`openshift-gitops-operator` `Subscription`** in `openshift-operators` to set **`DISABLE_DEFAULT_ARGOCD_CONSOLELINK=true`** in `spec.config.env` (so the operator drops the default ConsoleLink / console integration before the plugin CR is deleted). Optional short pause for reconciliation.
4. Delete the Argo CD **ConsolePlugin** CR(s) (e.g. `gitops`).
5. Delete the OpenShift GitOps operator Subscription.
6. Optionally delete the operator CSV.
7. When **`delete_gitops_operator_dedicated_namespace: true`**: **`oc delete operatorgroup -n openshift-gitops-operator --all`**, delete **`Namespace/openshift-gitops-operator`**, then finalizer strip on that namespace if needed.
8. When **`delete_gitops_namespace: true`**: strip finalizers on **Applications**, **ArgoCD**, **GitopsService**, OLM objects, and workloads **inside** `openshift-gitops`, then delete that namespace, **`oc patch`** / **`jq` + `/finalize`**, and recovery retries (same idea as `uninstall_acm`).

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
| `gitops_operator_dedicated_namespace` | `openshift-gitops-operator` | Dedicated namespace used by some installs (OperatorGroup targets this). |
| `delete_gitops_operator_dedicated_namespace` | `true` | Run `oc delete operatorgroup -n <dedicated> --all` then delete that namespace (no-op if absent). |
| `gitops_operator_subscription_name` | `openshift-gitops-operator` | Name of the Subscription to remove. |
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

## Clean reinstall

With `delete_operator_csv: true` and `delete_gitops_namespace: true`, the role removes the Argo CD instance, Subscription, CSV, and GitOps namespace so that OpenShift GitOps can be reinstalled (e.g. via the `install_openshift_gitops` role or `install-argo.yaml`) without conflicts. The playbook `uninstall-openshift-gitops.yaml` runs the role with these options and then verifies that the ArgoCD CR, Subscription, and (when applicable) namespace are gone.

## Notes

- If the GitOps namespace does not exist, **wipe_argo** and instance deletes are skipped; operator Subscription/CSV removal runs if present.
- If **`openshift-gitops`** still sticks in **`Terminating`**, ensure **`jq`** is available (or set **`strip_namespace_use_jq_finalize: false`**) and inspect **`oc get all -n openshift-gitops`**. The role strips **Application / AppProject / ArgoCD / GitopsService** and OLM object finalizers before namespace delete, then retries namespace finalizer removal.
