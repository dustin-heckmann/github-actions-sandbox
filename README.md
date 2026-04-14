# github-actions-sandbox — Deployment Tag Protection

Sandbox for testing `deployed/*` tag protection rulesets and the EE team reusable workflow pattern.

## What this demonstrates

- Regular users **cannot** create, update, or delete `deployed/*` tags (blocked by ruleset)
- The EE team's reusable workflow **can** bypass the ruleset via a privileged credential
- Consuming teams use the reusable workflow without needing to know about the bypass mechanism

## Workflows

| Workflow | Trigger | Owner |
|---|---|---|
| `ee-manage-deployment-tag.yaml` | `workflow_call` | EE team — manages tag operations using privileged credentials |
| `caller-tag.yaml` | `workflow_dispatch` | Product team — calls the EE reusable workflow |
| `manage-tags.yaml` | `workflow_dispatch` | Direct reference implementation (pre-reusable-workflow) |
| `test-read-tags.yaml` | `workflow_dispatch` | Smoke test — verifies read access works |

## Ruleset

**"Protect deployment tags"** — `refs/tags/deployed/**`

| Rule | Setting |
|---|---|
| Target | `refs/tags/deployed/**` |
| Enforcement | Active |
| Restrictions | creation, deletion, update |
| Bypass | Deploy key `tag-manager` (stand-in for GitHub App — see below) |

## Sandbox vs. org: how bypass credentials differ

| | Sandbox (personal repo) | Org (recommended) |
|---|---|---|
| **Bypass actor** | Deploy key | GitHub App installation |
| **Credential** | SSH private key (`DEPLOY_KEY` secret) | GitHub App private key (`EE_TAG_MANAGER_PRIVATE_KEY` org secret) |
| **Token minting** | `core.sshCommand` override | `actions/create-github-app-token@v1` |
| **Scope** | Repo-level secret | Org-level secret, accessible only from `ee-workflows` repo |

## Setting up the GitHub App (org migration)

### 1. Create the app

Go to **Org Settings → Developer Settings → GitHub Apps → New GitHub App**.

| Field | Value |
|---|---|
| Name | `ee-tag-manager` |
| Homepage URL | your org URL |
| Permissions | **Repository: Contents** → Read & write |
| Where can it be installed? | Only on this account |

Generate a private key and download it.

### 2. Install the app on target repos

Org Settings → GitHub Apps → `ee-tag-manager` → Install → select repos.

### 3. Store credentials as org secrets

```bash
# Scoped so only the ee-workflows repo can read them
gh secret set EE_TAG_MANAGER_APP_ID     --org YOUR_ORG --visibility selected --repos ee-workflows
gh secret set EE_TAG_MANAGER_PRIVATE_KEY --org YOUR_ORG --visibility selected --repos ee-workflows
```

### 4. Update the reusable workflow

In `ee-manage-deployment-tag.yaml`, replace the "Configure credentials" step with:

```yaml
- uses: actions/create-github-app-token@v1
  id: app-token
  with:
    app-id: ${{ secrets.EE_TAG_MANAGER_APP_ID }}
    private-key: ${{ secrets.EE_TAG_MANAGER_PRIVATE_KEY }}
    owner: ${{ github.repository_owner }}

- name: Checkout caller repo
  uses: actions/checkout@v4
  with:
    token: ${{ steps.app-token.outputs.token }}
    fetch-tags: true
```

And replace all `git push` steps with HTTPS (the app token is an HTTPS bearer token, no SSH needed).

### 5. Update the ruleset bypass actor

```bash
# Get the app installation ID
INSTALL_ID=$(gh api /orgs/YOUR_ORG/installations \
  --jq '.installations[] | select(.app_slug=="ee-tag-manager") | .id')

# Update the ruleset
gh api /repos/YOUR_ORG/YOUR_REPO/rulesets/RULESET_ID \
  --method PUT \
  --input - <<EOF
{
  "bypass_actors": [{
    "actor_id": $INSTALL_ID,
    "actor_type": "Integration",
    "bypass_mode": "always"
  }]
}
EOF
```
