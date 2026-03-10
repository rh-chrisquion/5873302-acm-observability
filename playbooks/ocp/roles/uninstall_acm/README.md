# Uninstall ACM

`uninstall_acm` uninstalls Advanced Cluster Management (ACM) from an OpenShift cluster by:
1. Deleting the MultiClusterHub custom resource
2. Waiting for the MultiClusterHub to be fully removed
3. Deleting the ACM operator Subscription
4. Optionally deleting the operator CSV
5. Optionally cleaning up ManagedCluster and KlusterletAddonConfig resources
6. Optionally deleting the operator namespace

## Usage

### Basic Usage

```yaml
- hosts: localhost
  roles:
    - role: uninstall_acm
```

### With Custom Configuration

```yaml
- hosts: localhost
  roles:
    - role: uninstall_acm
      vars:
        acm_multiclusterhub_name: "my-multiclusterhub"
        acm_multiclusterhub_namespace: "open-cluster-management"
        acm_operator_subscription_name: "advanced-cluster-management"
        acm_operator_subscription_namespace: "open-cluster-management"
        cleanup_managed_clusters: true
        delete_operator_namespace: false
```

## Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `acm_multiclusterhub_name` | `multiclusterhub` | Name of the MultiClusterHub CR to delete |
| `acm_multiclusterhub_namespace` | `open-cluster-management` | Namespace where MultiClusterHub is installed |
| `acm_operator_subscription_name` | `advanced-cluster-management` | Name of the ACM operator Subscription |
| `acm_operator_subscription_namespace` | `open-cluster-management` | Namespace where the operator Subscription is installed |
| `wait_for_multiclusterhub_deletion` | `true` | Wait for MultiClusterHub to be fully deleted |
| `multiclusterhub_deletion_retries` | `60` | Maximum retries when waiting for deletion |
| `multiclusterhub_deletion_delay` | `10` | Delay between retries (seconds) |
| `delete_operator_csv` | `true` | Delete the operator CSV after uninstalling |
| `delete_operator_namespace` | `false` | Delete the operator namespace after uninstalling |
| `cleanup_managed_clusters` | `false` | Delete ManagedCluster and KlusterletAddonConfig resources |

## Notes

- The role will gracefully handle cases where resources don't exist
- If `cleanup_managed_clusters` is set to `true`, all ManagedCluster and KlusterletAddonConfig resources will be deleted
- Setting `delete_operator_namespace` to `true` will remove the entire namespace, which may affect other resources
- The uninstall process may take several minutes depending on cluster resources
