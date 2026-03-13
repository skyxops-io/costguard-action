# CostGuard GitHub Action

**Shift-left cost governance for cloud infrastructure by [SKYXOPS](https://skyxops.com).**

Pre-built GitHub Action that adds cost review to every pull request. Works with **Terraform** and **CloudFormation** — auto-detected, zero extra config.

## Quick Start

### 1. Set Repository Secrets

**Settings > Secrets and variables > Actions:**

| Secret | Description |
|--------|-------------|
| `COSTGUARD_API_KEY` | Your CostGuard API key |
| `COSTGUARD_BUDGET_CODE` | Budget code from CostGuard dashboard (optional) |

> `GITHUB_TOKEN` is provided automatically — no setup needed.

### 2. Create Workflow

`.github/workflows/costguard.yml`:

```yaml
name: CostGuard
on:
  pull_request:
    branches: [main]

permissions:
  contents: read
  pull-requests: write

jobs:
  terraform-plan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
      - run: |
          terraform init
          terraform plan -out=tfplan
          terraform show -json tfplan > plan.json
      - uses: actions/upload-artifact@v4
        with:
          name: plan
          path: plan.json

  cost-review:
    needs: terraform-plan
    runs-on: ubuntu-latest
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: plan
      - uses: skyxops-io/costguard-action@v1
        with:
          plan-path: plan.json
          api-key: ${{ secrets.COSTGUARD_API_KEY }}
          budget-code: ${{ secrets.COSTGUARD_BUDGET_CODE }}
```

### 3. Protect Your Branch

**Settings > Branches > Branch protection rules > main:**
- Require status checks to pass: `cost-review`
- Require PR before merging: Yes

That's it. Every PR now gets a cost review comment with ALLOW, WARN, or BLOCK decision.

## What You Get

On every PR push, CostGuard:
- Posts a **cost breakdown** as a PR comment (per-resource, with regions)
- Returns a **decision** — ALLOW, WARN, or BLOCK — that gates your merge
- Validates against your **budget** and **guardrails**
- Provides **AI-powered** cost optimization recommendations
- Uploads **HTML report** and **result JSON** as workflow artifacts

The comment updates automatically on each push — no duplicates.

## Inputs

| Input | Required | Default | Description |
|-------|:--------:|---------|-------------|
| `plan-path` | No | `plan.json` | Path to plan or changeset file |
| `api-key` | **Yes** | | CostGuard API key |
| `budget-code` | No | `""` | Budget code for cost validation |
| `github-token` | No | `${{ github.token }}` | Token for PR comments (auto-provided) |
| `skip-budget` | No | `false` | Skip budget check (pricing-only mode) |
| `skip-narrative` | No | `false` | Skip AI analysis (faster runs) |
| `skip-guardrails` | No | `false` | Skip guardrail evaluation |
| `auto-approve` | No | `false` | Pass when no budget found |
| `format` | No | `markdown` | Comment format: `markdown` or `terminal` |
| `post-comment` | No | `true` | Post cost breakdown as PR comment |
| `api-url` | No | *(built-in)* | CostGuard API URL (override for custom deployments) |
| `extra-args` | No | `""` | Extra CLI flags (escape hatch) |

## Outputs

| Output | Description |
|--------|-------------|
| `decision` | `ALLOW`, `WARN`, or `BLOCK` |
| `monthly-cost` | Estimated monthly cost |
| `result-file` | Path to cached result JSON |
| `report-file` | Path to HTML report |

```yaml
- uses: skyxops-io/costguard-action@v1
  id: costguard
  with:
    plan-path: plan.json
    api-key: ${{ secrets.COSTGUARD_API_KEY }}

- run: echo "Decision: ${{ steps.costguard.outputs.decision }}"
- run: echo "Monthly cost: ${{ steps.costguard.outputs.monthly-cost }}"
```

## Examples

**CloudFormation:**
```yaml
- uses: skyxops-io/costguard-action@v1
  with:
    plan-path: changeset.json
    api-key: ${{ secrets.COSTGUARD_API_KEY }}
```

**Pricing only (skip budget):**
```yaml
- uses: skyxops-io/costguard-action@v1
  with:
    plan-path: plan.json
    api-key: ${{ secrets.COSTGUARD_API_KEY }}
    skip-budget: 'true'
```

**Fast mode (skip AI):**
```yaml
- uses: skyxops-io/costguard-action@v1
  with:
    plan-path: plan.json
    api-key: ${{ secrets.COSTGUARD_API_KEY }}
    skip-narrative: 'true'
```

## Exit Codes

| Code | Decision | Workflow | Merge |
|:----:|----------|----------|-------|
| 0 | ALLOW | Passes | Allowed |
| 1 | BLOCK | Fails | Blocked |
| 2 | WARN | Passes | Allowed (warning annotation) |

## Versioning

```yaml
# Recommended — floating tag, auto-receives fixes
- uses: skyxops-io/costguard-action@v1

# Pinned to exact commit
- uses: skyxops-io/costguard-action@<commit-sha>
```

## Troubleshooting

| Problem | Fix |
|---------|-----|
| PR comment not posted | Add `permissions: pull-requests: write` to workflow |
| BLOCK doesn't prevent merge | Add `cost-review` as required status check |
| Plan file not found | Check `plan-path` matches your artifact download |
| 403 on comment | Ensure `GITHUB_TOKEN` has write access to pull requests |

## Other Platforms

| Platform | Integration |
|----------|-------------|
| **GitLab CI** | [costguard-templates](https://gitlab.com/skyxops-io/costguard-templates) |
| **Azure DevOps** | [CostGuard extension](https://marketplace.visualstudio.com/items?itemName=costguard-skyxops.costguard) |
| **Any CI** | `pip install costguard-cli` — [PyPI](https://pypi.org/project/costguard-cli/) |

## License

[MIT](LICENSE)

---

*Built by [SKYXOPS](https://skyxops.com) — shift-left cost governance for cloud infrastructure.*
