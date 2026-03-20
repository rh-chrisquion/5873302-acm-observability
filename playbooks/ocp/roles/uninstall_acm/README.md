# Uninstall ACM

The role splits work into:

| Entry | File | What it does |
|-------|------|----------------|
| **Default / `tasks_from: main.yaml`** | `tasks/main.yaml` | When `cleanup_managed_clusters`: runs **`managed_clusters_detach.yaml`** first (KlusterletAddonConfig + **ManagedCluster** before **MultiClusterHub** in hub flows), then deletes **MultiClusterObservability** if present. |
| **Full hub + operator uninstall** | `tasks/full_uninstall.yaml` | **Imports `hub_uninstall.yaml` first** (MCO + detach when `full`, **MultiClusterHub** delete, optional CR-only `end_play`), then MCE/ACM operator teardown (Subscription, CSV, webhooks, namespaces, …). |
| **Hub phase only** | `tasks/hub_uninstall.yaml` | Same hub steps as above **without** importing operator teardown. Use for **`multiclusterhub_cr_only`** or when you only want MCO/MCH work without `full_uninstall`. |

### `acm_uninstall_mode` (hub phase — used by `hub_uninstall.yaml` and by `full_uninstall.yaml` via import)

| Mode | Behavior |
|------|----------|
| **`full`** (default) | Hub phase runs **main** (MCO + detach), then removes **MultiClusterHub**. If `tasks_from` is **`full_uninstall.yaml`**, operator/MCE/webhook/namespace teardown follows. |
| **`multiclusterhub_cr_only`** | Runs **`managed_clusters_detach.yaml`** first when `cleanup_managed_clusters`, then removes only the **MultiClusterHub** CR and **`end_play`s**. Does **not** run **main** (no MCO removal). |

## Usage

### Full uninstall (playbook)

`full_uninstall.yaml` removes **Multicluster Engine (MCE) before the ACM operator**: `MultiClusterEngine` CR (with a wait), MCE **Subscription**, **CSV** (including any leftover CSVs in the namespace), **InstallPlans**, **OperatorGroups**, then precleans and deletes the **`multicluster-engine`** namespace (when enabled). Only then does it remove the ACM **Subscription**/CSV and continue with webhooks and the **`open-cluster-management`** namespace.

```bash
ansible-playbook playbooks/ocp/uninstall-acm.yaml
```

Equivalent:

```yaml
- hosts: localhost
  gather_facts: false
  tasks:
    - ansible.builtin.include_role:
        name: uninstall_acm
        tasks_from: full_uninstall.yaml
```

### MCO + detach only (no hub/operator teardown)

```bash
ansible-playbook playbooks/ocp/uninstall-acm-mco-detach.yaml
```

Or:

```yaml
- hosts: localhost
  roles:
    - role: uninstall_acm   # runs tasks/main.yaml
```

### MultiClusterHub CR only (operator stays for reinstall)

```yaml
- hosts: localhost
  gather_facts: false
  tasks:
    - ansible.builtin.include_role:
        name: uninstall_acm
        tasks_from: hub_uninstall.yaml
      vars:
        acm_uninstall_mode: multiclusterhub_cr_only
```

### With custom variables (full teardown)

```yaml
- hosts: localhost
  tasks:
    - ansible.builtin.include_role:
        name: uninstall_acm
        tasks_from: full_uninstall.yaml
      vars:
        acm_multiclusterhub_name: "multiclusterhub"
        cleanup_managed_clusters: true
        delete_operator_namespace: false
```

## Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `acm_uninstall_mode` | `full` | Hub phase: `full` or `multiclusterhub_cr_only` (applies when using **`hub_uninstall.yaml`** or **`full_uninstall.yaml`**) |
| `cleanup_managed_clusters` | `true` | In **main**: remove KlusterletAddonConfigs and **ManagedCluster** CRs (detach). Set `false` to skip. |
| `acm_multiclusterhub_name` | `multiclusterhub` | MultiClusterHub CR name |
| `acm_multiclusterhub_namespace` | `open-cluster-management` | MultiClusterHub namespace |
| `mco_cr_name` | `observability` | MultiClusterObservability CR name |
| `mco_cr_namespace` | `open-cluster-management` | MultiClusterObservability namespace |
| `remove_argocd_application` | `false` | Set **`true`** to delete the Argo CD **Application** for MCO before removing the MCO CR (matches `apps_install_mco` / GitOps) |
| `mco_argocd_app_name` | `mco-observability` | Argo CD Application name |
| `mco_argocd_namespace` | `openshift-gitops` | Namespace of the Argo CD Application |
| `acm_operator_subscription_name` | `advanced-cluster-management` | ACM operator Subscription |
| `acm_operator_subscription_namespace` | `open-cluster-management` | Subscription namespace |
| `wait_for_multiclusterhub_deletion` | `true` | (reserved) wait for MCH deletion |
| `multiclusterhub_deletion_retries` | `60` | MCH wait retries |
| `multiclusterhub_deletion_delay` | `10` | MCH wait delay (seconds) |
| `delete_operator_csv` | `true` | Delete ACM operator CSV |
| `mce_operator_subscription_name` | `multicluster-engine` | MCE Subscription |
| `mce_operator_subscription_namespace` | `multicluster-engine` | MCE namespace |
| `delete_mce_operator` | `true` | Remove MCE Subscription/CSV |
| `delete_mce_operator_csv` | `true` | Delete MCE CSV |
| `delete_multiclusterengine_cr` | `true` | Delete **MultiClusterEngine** CR first |
| `mce_cr_deletion_retries` | `60` | After `MultiClusterEngine` delete, poll until CRs are gone |
| `mce_cr_deletion_delay` | `10` | Seconds between retries |
| `preclean_mce_namespace_before_delete` | `true` | Preclean MCE namespace |
| `mce_namespace_preclean_passes` | `3` | MCE preclean passes |
| `delete_mce_namespace` | `true` | Delete **multicluster-engine** namespace |
| `delete_acm_console_plugins` | `true` | Remove ACM/MCE console plugins |
| `acm_console_plugin_names` | `["acm","mce"]` | Plugin names to remove |
| `delete_operator_namespace` | `true` | Delete ACM operator namespace (`open-cluster-management`) |
| `strip_namespace_finalizers_on_delete` | `true` | After each targeted namespace delete, patch `metadata.finalizers` to `[]` (helps **Terminating** namespaces) |
| `strip_namespace_use_jq_finalize` | `true` | Run `oc get ns … -o json \| jq '(.spec //= {}) \| .spec.finalizers = [] \| .metadata.finalizers = []' \| oc replace --raw …/finalize` (set `false` if `jq` is missing) |
| `strip_finalizers_on_namespace_contents` | `true` | Before deleting **open-cluster-management**, clear finalizers on OLM/workload objects **inside** the namespace |
| `ocm_namespace_aggressive_terminating_recovery` | `true` | After the delete request, retry loop: strip in-namespace finalizers + namespace finalizers (handles re-added finalizers) |
| `ocm_namespace_terminating_recovery_attempts` | `12` | Retry count for the recovery loop |
| `ocm_namespace_terminating_recovery_delay_seconds` | `15` | Sleep between retries |
| `delete_observability_namespace` | `false` | Delete **`open-cluster-management-observability`** in `full_uninstall` (after MCO CR removal) |
| `observability_namespace_name` | `open-cluster-management-observability` | Observability namespace to delete when `delete_observability_namespace` is true |
| `preclean_ocm_namespace_before_delete` | `true` | Preclean OCM namespace before delete |
| `ocm_namespace_preclean_passes` | `3` | OCM preclean passes |
| `delete_target_apiservices` | *(list)* | APIServices to delete |
| `delete_target_clusterroles` | *(list)* | ClusterRoles to delete |
| `target_delete_mutating_webhook_configurations` | *(list)* | Mutating webhooks |
| `target_delete_validating_webhook_configurations` | *(list)* | Validating webhooks |
| `target_delete_crd_artifacts` | *(list)* | CRD-related cleanup |

## Notes

- **`multiclusterhub_cr_only`** does not remove MCO; with **`cleanup_managed_clusters: true`** it still **detaches ManagedClusters before MCH**. Use **full** or **main** if you need MCO removed.
- The role tolerates missing resources.
- **`delete_operator_namespace: true`** does not wait for namespace termination; with **`strip_namespace_finalizers_on_delete: true`** (default), the role patches **finalizers** on **multicluster-engine**, **observability** (if deleted), and **ACM** namespaces after delete requests.
- **Terminating** **`open-cluster-management`** often happens when **objects inside** the namespace still have finalizers, or the **ACM operator** reconciles and **re-adds** namespace `metadata.finalizers` until operands are gone. The role strips finalizers on **Subscriptions, CSVs, pods**, etc. **before** namespace delete, then runs **`oc patch`**, optional **`jq` + `/finalize`**, and a **retry loop** (`ocm_namespace_terminating_recovery_*`). If the namespace still sticks, install **`jq`** on the Ansible control host or set **`strip_namespace_use_jq_finalize: false`**, and inspect **`oc get all -n open-cluster-management -o wide`**.
