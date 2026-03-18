# Uninstall ACM

The role splits work into:

| Entry | File | What it does |
|-------|------|----------------|
| **Default / `tasks_from: main.yaml`** | `tasks/main.yaml` | Deletes **MultiClusterObservability** (if present), then **detaches managed clusters** (KlusterletAddonConfig + ManagedCluster) when `cleanup_managed_clusters` is true. |
| **Full hub + operator uninstall** | `tasks/hub_uninstall.yaml` | For **`full`**: runs `main.yaml` first, then deletes **MultiClusterHub**, then operator/MCE/webhooks/namespace teardown. For **`multiclusterhub_cr_only`**: deletes **MultiClusterHub** only and ends the play (no MCO/MCO step, no operator removal). |

### `acm_uninstall_mode` (hub_uninstall only)

| Mode | Behavior |
|------|----------|
| **`full`** (default) | Runs **main** (MCO + detach), then removes **MultiClusterHub**, then **full_uninstall** (Subscription, MCE, console plugins, webhooks, optional namespace, …). |
| **`multiclusterhub_cr_only`** | Removes only the **MultiClusterHub** CR. Operator Subscription/CSV unchanged. Play ends. Does **not** run **main** (MCO and managed clusters are left as-is). |

## Usage

### Full uninstall (playbook)

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
        tasks_from: hub_uninstall.yaml
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

### With custom variables

```yaml
- hosts: localhost
  tasks:
    - ansible.builtin.include_role:
        name: uninstall_acm
        tasks_from: hub_uninstall.yaml
      vars:
        acm_multiclusterhub_name: "multiclusterhub"
        cleanup_managed_clusters: true
        delete_operator_namespace: false
```

## Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `acm_uninstall_mode` | `full` | Only for **hub_uninstall**: `full` or `multiclusterhub_cr_only` |
| `cleanup_managed_clusters` | `true` | In **main**: remove KlusterletAddonConfigs and **ManagedCluster** CRs (detach). Set `false` to skip. |
| `acm_multiclusterhub_name` | `multiclusterhub` | MultiClusterHub CR name |
| `acm_multiclusterhub_namespace` | `open-cluster-management` | MultiClusterHub namespace |
| `mco_cr_name` | `observability` | MultiClusterObservability CR name |
| `mco_cr_namespace` | `open-cluster-management` | MultiClusterObservability namespace |
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
| `preclean_mce_namespace_before_delete` | `true` | Preclean MCE namespace |
| `mce_namespace_preclean_passes` | `3` | MCE preclean passes |
| `delete_mce_namespace` | `true` | Delete **multicluster-engine** namespace |
| `delete_acm_console_plugins` | `true` | Remove ACM/MCE console plugins |
| `acm_console_plugin_names` | `["acm","mce"]` | Plugin names to remove |
| `delete_operator_namespace` | `false` | Delete operator namespace |
| `preclean_ocm_namespace_before_delete` | `true` | Preclean OCM namespace before delete |
| `ocm_namespace_preclean_passes` | `3` | OCM preclean passes |
| `delete_target_apiservices` | *(list)* | APIServices to delete |
| `delete_target_clusterroles` | *(list)* | ClusterRoles to delete |
| `target_delete_mutating_webhook_configurations` | *(list)* | Mutating webhooks |
| `target_delete_validating_webhook_configurations` | *(list)* | Validating webhooks |
| `target_delete_crd_artifacts` | *(list)* | CRD-related cleanup |

## Notes

- **`multiclusterhub_cr_only`** does not remove MCO or detach clusters; use **full** or run **main** separately if needed.
- The role tolerates missing resources.
- **`delete_operator_namespace: true`** does not wait for namespace termination; clear finalizers if stuck **Terminating**.
