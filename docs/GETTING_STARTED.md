# Getting Started

Install and operate a **feishu-inbound instance** using only this dist repo (wheel + docs). No access to the private engine source required.

## What you are setting up

```
飞书 Bitable 需求池
     │ Pipeline A (your GHA — not in wheel)
     ▼
GitHub Issue (labels + comments)
     │ Pipeline B–F (feishu-inbound CLI)
     ▼
Worktree + Smart PR → Gate review → Dev handback → Accept → Promote → Prod
```

**Engine** (`feishu_inbound` pip package): pipeline logic, CLI, launchd templates.  
**Instance** (your repo): `config.yaml`, secrets, `var/feishu_inbound/` runtime, thin wrappers, Pipeline A workflow.

See [rfc/RFC-001-engine-instance-architecture.md](./rfc/RFC-001-engine-instance-architecture.md).

## Prerequisites

| Requirement | Notes |
|-------------|-------|
| Python 3.10+ | macOS launchd is the reference scheduler |
| `gh` CLI | Issue/PR operations; optional for install |
| GitHub token | `repo` scope for your instance repos |
| Feishu app credentials | Bitable read (Pipeline A / board sync) |
| Agent platform | OpenCode / Cursor / Claude per `agent_client` config |

## Step 1 — Install the engine

Pin version in `requirements-feishu-inbound.txt`:

```text
feishu-inbound==0.1.37
```

Install from this public dist (no token):

```bash
VERSION=0.1.37
pip install "https://github.com/369795172/feishu-inbound-dist/releases/download/v${VERSION}/feishu_inbound-${VERSION}-py3-none-any.whl"
feishu-inbound --version
```

Details: [PACKAGING.md](./PACKAGING.md).

## Step 2 — Create instance config

Copy a template and edit for your org/repos. Minimal single-repo example:

```yaml
secrets_prefix: "FI_MYAPP_"

pipeline:
  mode: single-repo
  repo: "your-org/your-repo"

github:
  repo: "your-org/your-repo"
  default_labels: ["feishu-inbound"]

state_dir: ../var/feishu_inbound/state
logs_dir: ../var/feishu_inbound/logs
worktree_root: ..
worktree_dir: ../worktrees/feishu_inbound
surfaces_config_path: feishu_inbound_surfaces.yaml
instance_tools_path: ../tools

assignees:
  by_id:
    "12345678": "YourName"

pipeline_cd_scan:
  mode: repos
  repos: ["your-org/your-repo"]

launchd_label_prefix: com.myapp
launchd_labels:
  lead_tick: com.myapp.feishu-inbound-lead-tick
launchd_schedules:
  lead_tick:
    hours: all
    minutes: [20, 50]
```

Full schema: [instance-config.md](./instance-config.md).

## Step 3 — Secrets (Keychain)

Declare `secrets_prefix` in config. Store keys as `rootgrove/<PREFIX><NAME>` (or your loader's namespace):

| Key | Purpose |
|-----|---------|
| `FI_MYAPP_GITHUB_TOKEN` | GitHub API |
| `FI_MYAPP_FEISHU_APP_ID` | Feishu app |
| `FI_MYAPP_FEISHU_APP_SECRET` | Feishu app |
| `FI_MYAPP_FEISHU_BITABLE_APP_TOKEN` | Bitable (Pipeline A / board) |

Engine reads `{prefix}{KEY}` first, then bare `{KEY}` for backward compatibility.

## Step 4 — Smoke test (no agent)

```bash
export GITHUB_TOKEN=...
feishu-inbound triage --config path/to/config.yaml --scan-only
feishu-inbound scan    --config path/to/config.yaml --scan-only
feishu-inbound execute --config path/to/config.yaml --scan-only
feishu-inbound review  --config path/to/config.yaml --scan-only
feishu-inbound handback --config path/to/config.yaml --scan-only
```

Each command prints which issues would be processed. Fix config errors before enabling launchd.

## Step 5 — launchd (optional)

Generate and load plists from config schedules:

```bash
feishu-inbound install-launchd --config path/to/config.yaml --jobs lead_tick
```

**Personal single-repo** typically runs only `lead_tick` (chains C→D→E→D'→E'→F→accept→promote).  
**ASP multi-surface** also needs `feishu_inbound_triage` on a lead Mac.

## Step 6 — Pipeline A (Bitable → Issue)

Pipeline A is **not** in the pip package. Copy a GitHub Actions workflow into your target repo and set Feishu secrets. See your instance's `tools/feishu_inbound/pipeline_a/` or ASP `feishu-inbound.yml` template.

## Step 7 — Operate with Agent skills

Point your AI assistant at:

- [skills/skill_feishu_inbound.md](../skills/skill_feishu_inbound.md) — main CLI contract
- [workflows/pipeline.md](./workflows/pipeline.md) — stage semantics

Natural language examples:

- 「对 issue #42 跑 scan」→ `feishu-inbound scan --config ... --issue 42 --repo org/repo`
- 「按 Acceptance Skill，issue #42 验收通过」→ `feishu-inbound accept pass --config ... --issue 42 --repo org/repo`

## Next steps

| Goal | Read |
|------|------|
| Multi-surface ASP setup | [workflows/pipeline.md](./workflows/pipeline.md), ASP `asp-infra` config |
| Personal Bitable + umbrella | [personal_bitable_setup.md](./personal_bitable_setup.md) |
| Dev acceptance + prod promote | [acceptance_gate.md](./acceptance_gate.md), [workflows/promote.md](./workflows/promote.md) |
| CI handback / promote | [../templates/](../templates/) |
| Upgrade engine | [PACKAGING.md](./PACKAGING.md) |
| Debug failures | [troubleshooting.md](./troubleshooting.md) |
