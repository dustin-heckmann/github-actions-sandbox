# deployment-tag-sandbox

Sandbox repository for testing GitHub Actions tag protection rulesets.

## Purpose

Verifies that:
- GitHub Actions workflows (via `GITHUB_TOKEN`) **can** create, update, and delete `deployed/*` tags
- Regular users **cannot** create, update, or delete `deployed/*` tags (blocked by ruleset)

## Workflows

| Workflow | Trigger | Purpose |
|---|---|---|
| `manage-tags` | `workflow_dispatch` | Create/move/delete `deployed/*` tags |
| `test-read-tags` | `workflow_dispatch` | Read and print all `deployed/*` tags |

## Ruleset

A repository ruleset named **"Protect deployment tags"** targets `refs/tags/deployed/**` and:
- Restricts creations, deletions, and updates
- Grants bypass only to the **GitHub Actions** app
