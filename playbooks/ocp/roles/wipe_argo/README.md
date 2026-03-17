# wipe_argo

Deletes all Argo CD (OpenShift GitOps) **Applications**, **ApplicationSets**, **AppProjects**, **repository secrets** (repos), and **repo-creds** secrets in a given namespace. The Argo CD instance and operator are not removed.

Use this role when you want to clear all apps and repo config from GitOps without uninstalling the operator, or run it as part of `uninstall_openshift_gitops` (which includes it before tearing down the instance).

## Requirements

- `kubernetes.core` Ansible collection.
- Cluster access via `KUBECONFIG` or in-cluster authentication.

## Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `wipe_argocd_namespace` | `openshift-gitops` | Namespace containing the Argo CD instance to wipe. |

## Usage

### Standalone (wipe apps and repos only)

```yaml
- hosts: localhost
  gather_facts: false
  roles:
    - role: wipe_argo
      vars:
        wipe_argocd_namespace: "openshift-gitops"
```

### As part of uninstall

The `uninstall_openshift_gitops` role includes `wipe_argo` automatically before deleting the GitOpsService and operator, so running the uninstall playbook wipes apps/repos first.

## Notes

- If the namespace does not exist, all tasks are skipped.
- List/delete tasks use `ignore_errors: true` so missing CRDs (e.g. ApplicationSet) do not fail the play.
