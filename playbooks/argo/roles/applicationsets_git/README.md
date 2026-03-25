# applicationsets_git

Creates a single Argo CD **`ApplicationSet`** using a **Git directory generator**. It scans **`gitops_repo_url`** at **`applicationset_repo_revision`** and creates one **`Application`** per matching directory (paths/globs from **`applicationset_git_directories`**).

## Prerequisites

1. **OpenShift GitOps** installed with the ApplicationSet controller (CRD `applicationsets.argoproj.io`).
2. Run after **[`configure_argo`](../configure_argo/)** (or equivalent): the target **`AppProject`** must allow **`gitops_repo_url`** in **`spec.sourceRepos`**, and a **repository credential** secret must exist if the repo is private.
3. **`spec.goTemplate: true`** is set; generated names/paths use **`{{ .path.basenameNormalized }}`**, **`{{ .path.path }}`**, etc. See [Git generator](https://argo-cd.readthedocs.io/en/stable/operator-manual/applicationset/Generators-Git/) and [GoTemplate](https://argo-cd.readthedocs.io/en/stable/operator-manual/applicationset/GoTemplate/). If your operator version differs, adjust the template or `goTemplate` flag to match.

## Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `applicationset_name` | `gitops-generated-apps` | `ApplicationSet` metadata name |
| `applicationset_namespace` | `openshift-gitops` | Namespace for the ApplicationSet and generated Applications |
| `applicationset_project` | `acm` | Argo CD `AppProject` for generated Applications |
| `applicationset_repo_revision` | `main` | Git ref for generator and `spec.source.targetRevision` |
| `applicationset_git_directories` | `[{ path: apps/* }]` | List of directory rules for the Git generator (see below) |
| `applicationset_destination_server` | `https://kubernetes.default.svc` | Destination cluster for all generated Applications |
| `applicationset_destination_namespace_mode` | `basename` | `basename` — target namespace = Git **`path.basenameNormalized`**; `fixed` — use **`applicationset_destination_namespace`** |
| `applicationset_destination_namespace` | `""` | Required when **`namespace_mode`** is **`fixed`** |
| `applicationset_directory_recurse` | `true` | `spec.source.directory.recurse` |
| `applicationset_sync_policy` | `Automatic` | `Automatic` or `Manual` |
| `applicationset_sync_options` | see `defaults/main.yaml` | `syncPolicy.syncOptions` |
| `applicationset_sync_retry_*` | see defaults | `syncPolicy.retry` backoff |

`gitops_repo_url` is provided by the **`arg_gitops_repo`** role dependency.

### `applicationset_git_directories`

Each item must include **`path`** (directory or glob under the repo). Optional **`exclude`**: set `true` to exclude matching paths (Argo Git generator semantics).

Examples:

```yaml
applicationset_git_directories:
  - path: apps/*
  - path: overlays/prod
```

```yaml
applicationset_git_directories:
  - path: '*'
    exclude: true
  - path: apps/*
```

### Namespace modes

- **`basename`**: each Application’s destination namespace (and metadata name) is derived from **`path.basenameNormalized`** (DNS-friendly). Ensure **`CreateNamespace=true`** remains in **`applicationset_sync_options`** if namespaces should be created automatically.
- **`fixed`**: set **`applicationset_destination_namespace`** to a single namespace for every generated Application.

## Playbook

[`applicationsets.yaml`](../../applicationsets.yaml) runs **`configure_argo`** then **`applicationsets_git`**. You must keep at least one Git directory rule in **`applicationset_git_directories`** (default `apps/*` is valid only if that path exists in your repo).

Example:

```bash
ansible-playbook playbooks/argo/applicationsets.yaml \
  -e '{"applicationset_git_directories":[{"path":"mco"}]}'
```
