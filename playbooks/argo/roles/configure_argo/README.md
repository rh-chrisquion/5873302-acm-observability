# configure_argo

Creates the Argo CD AppProject, repository secret (GitLab HTTPS + personal access token), and optional RBAC for the GitOps namespace.

## Repository authentication (GitLab PAT)

Authentication uses **HTTPS and a GitLab personal access token** (no SSH keys).

1. In GitLab: **Settings → Access Tokens**. Create a token with scope `read_repository` (and `write_repository` if Argo CD will push).
2. In your vault-encrypted `secrets.yml` (e.g. `playbooks/argo/secrets.yml`), define:
   - `vault_repo_password`: your GitLab personal access token
   - `vault_repo_username`: (optional) username for auth; default is `git`
3. Set **`gitops_repo_url`** (single canonical URL) in `roles/arg_gitops_repo/defaults/main.yaml`, or override via `group_vars`, `-e`/`--extra-vars`, or copy `vars/gitops_repo.yml.example` to `vars/gitops_repo.yml` and add it to your playbook `vars_files`. Use HTTPS, e.g. `https://gitlab.com/group/gitops-install-acm.git`.

Example minimal `secrets.yml` (encrypt with `ansible-vault encrypt secrets.yml`):

```yaml
vault_repo_password: "glpat-xxxxxxxxxxxxxxxxxxxx"
# vault_repo_username: "git"   # optional, default is git
```

## Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `gitops_repo_url` | (see `arg_gitops_repo` role) | **Single Git HTTPS URL** for Applications, AppProject `sourceRepos`, and repo Secret. |
| `repo_name` | `gitops-install-acm` | Name used for the repository secret. |
| `repo_url` | `{{ gitops_repo_url }}` | Alias for backwards compatibility. |
| `project_source_repos_extra` | `[]` | Additional URLs allowed in AppProject (rare). |
| `project_source_repos` | derived | `gitops_repo_url` + extras, for AppProject. |
| `repo_username` | `git` | Default username for repo auth (override with `vault_repo_username`). |
| `argocd_project` | `acm` | AppProject to associate with the repo. |
| `app_project_name` | `acm` | AppProject name. |
| `argocd_rbac_admins` | `[]` | List of users/groups to grant Argo CD admin. |
| `argocd_namespace` | `openshift-gitops` | Argo CD namespace. |

Vault (in `secrets.yml`): `vault_repo_password`, optional `vault_repo_username`.
