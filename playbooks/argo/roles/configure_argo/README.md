# configure_argo

Creates the Argo CD AppProject, repository secret (GitLab HTTPS + personal access token), and optional RBAC for the GitOps namespace.

## Repository authentication (GitLab PAT)

Authentication uses **HTTPS and a GitLab personal access token** (no SSH keys).

1. In GitLab: **Settings → Access Tokens**. Create a token with scope `read_repository` (and `write_repository` if Argo CD will push).
2. In your vault-encrypted `secrets.yml` (e.g. `playbooks/argo/secrets.yml`), define:
   - `vault_repo_password`: your GitLab personal access token
   - `vault_repo_username`: (optional) username for auth; default is `git`
3. Ensure `repo_url` in role defaults (or group_vars) uses HTTPS, e.g. `https://gitlab.com/group/gitops-install-acm.git`.

Example minimal `secrets.yml` (encrypt with `ansible-vault encrypt secrets.yml`):

```yaml
vault_repo_password: "glpat-xxxxxxxxxxxxxxxxxxxx"
# vault_repo_username: "git"   # optional, default is git
```

## Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `repo_name` | `gitops-install-acm` | Name used for the repository secret. |
| `repo_url` | `https://gitlab.com/...` | Git repo URL (HTTPS for PAT auth). |
| `repo_username` | `git` | Default username for repo auth (override with `vault_repo_username`). |
| `argocd_project` | `acm` | AppProject to associate with the repo. |
| `app_project_name` | `acm` | AppProject name. |
| `argocd_rbac_viewer` | `[]` | List of users/groups to grant Argo CD admin. |
| `argocd_namespace` | `openshift-gitops` | Argo CD namespace. |

Vault (in `secrets.yml`): `vault_repo_password`, optional `vault_repo_username`.
