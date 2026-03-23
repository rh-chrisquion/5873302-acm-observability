# arg_gitops_repo

Provides **`gitops_repo_url`** — the single canonical Git repository URL (HTTPS) for:

- Argo CD `Application` `spec.source.repoURL` (via per-app `*_git_repo_url` aliases)
- `configure_argo` AppProject `sourceRepos` and repository `Secret`

Other Argo roles declare a **role dependency** on `arg_gitops_repo` so they all resolve the same URL.

**Override** (highest precedence wins): `group_vars`, playbook `vars_files`, or `-e gitops_repo_url=...`.

**Multiple Git repos:** Defaults assume one monorepo (different `path`/`targetRevision` per app). If an app must use another URL, set e.g. `mco_git_repo_url` in inventory and add that URL to `project_source_repos_extra` in `configure_argo` so the AppProject allows it.
