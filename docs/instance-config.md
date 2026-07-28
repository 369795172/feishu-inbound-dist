# Instance config reference

`config.yaml` is owned by each **instance** (ASP, personal, GZUCM, etc.). The engine validates schema at startup and fails fast on missing keys.

Paths in config are resolved relative to the **config file directory** (`EngineConfig.resolve_path`), not the process cwd.

## Required top-level keys

| Key | Purpose |
|-----|---------|
| `github` | Default repo, labels, title prefix |
| `state_dir` | Checkpoint JSON (`issue_scanner_state.json`, locks, agent slots) |
| `worktree_root` / `worktree_dir` | Issue worktrees for Pipeline D |
| `logs_dir` | Engine subcommand logs |
| `prompts` | Domain context injected into agent prompts |
| `assignees` | `by_id` / `by_name` maps for routing |
| `pipeline_cd_scan` | Pipeline C/D scan scope (`org` / `repo` / `repos`) |

Multi-surface instances also need `surface_routing`, `assignee_routing`, and `surfaces_config_path`.

## Secrets

```yaml
secrets_prefix: "FI_ASP_"   # or FI_PERSONAL_, FI_MYAPP_, ...
```

Engine lookup order: `{prefix}{KEY}` → bare `{KEY}` → error with missing key name.

Common keys: `GITHUB_TOKEN`, `FEISHU_APP_ID`, `FEISHU_APP_SECRET`, `FEISHU_BITABLE_APP_TOKEN`, `OPENCODE_*`.

## Pipeline mode

### Multi-surface (ASP default)

```yaml
pipeline:
  mode: multi-surface   # default when omitted
```

Pipeline B routes surface keywords; C/D resolve surface from triage labels + `surfaces.yaml`.

### Single-repo (personal / QMT)

```yaml
pipeline:
  mode: single-repo
  repo: "owner/repo"
  worktree: projects/myapp   # optional hint
```

| Stage | Behavior |
|-------|----------|
| B | Optional; no surface routing. Personal default: no triage launchd |
| C | `primary_surface` = first key in `surfaces.yaml`; branch `issue-{N}` |
| D | Single worktree; `base_branch` from surfaces config |

See [rfc/RFC-001-engine-instance-architecture.md](./rfc/RFC-001-engine-instance-architecture.md) §4.

## Surfaces config (`surfaces.yaml`)

Per-surface metadata (instance file, referenced by `surfaces_config_path`):

```yaml
backend:
  repo: AI-MYG/asp-backend
  base_branch: dev
  worktree: projects/asp/backend
default:
  repo: owner/repo
  base_branch: main
```

SSOT for repo, base branch, and worktree path in Pipeline C/D.

## Agent client injection

Engine does not hardcode Cursor/OpenCode. Instance declares import chain:

```yaml
instance_tools_path: ../..          # repo root with tools/
agent_client:
  import_chain: tools.agent_client.AgentClient
```

Or:

```yaml
env_loader:
  import_chain: [tools.feishu_inbound.load_personal_env]
```

## launchd

```yaml
launchd_label_prefix: com.asp       # or com.personal
launchd_labels:
  lead_tick: com.asp.feishu-inbound-lead-tick
  feishu_inbound_triage: com.asp.feishu-inbound-triage
launchd_schedules:
  lead_tick:
    hours: all
    minutes: [20, 50]
  feishu_inbound_triage:
    hours: all
    minutes: [10, 40]
```

Install: `feishu-inbound install-launchd --config config.yaml --jobs lead_tick`

**Lead tick** chains C→D→E→D'→E'→F dev→accept→promote→F prod. C/D/E/F no longer have separate launchd jobs (since 2026-06-19).

### Personal umbrella (multiple single-repo configs)

One launchd tick runs `lead-tick` for each config with `umbrella_runner: true`. See [personal_bitable_setup.md](./personal_bitable_setup.md).

## Pipeline stage config

### `pipeline_c`

```yaml
pipeline_c:
  batch: 3                  # issues per scan batch
  max_attempts: 2
  cooldown_minutes: 60      # fair queue after failures
```

### `pipeline_d`

```yaml
pipeline_d:
  max_attempts: 3
  max_revision_rounds: 3
```

Gate: `difficulty-trivial` auto-executes; others need `approved-to-execute` label.

### `pipeline_e`

```yaml
pipeline_e:
  auto_merge: true          # merge PR on AI pass
  review_pool: [...]        # cross-platform reviewers
```

### `pipeline_f`

```yaml
pipeline_f:
  feishu_notify:
    bot: tts
    by_github_id:
      user: ou_xxx
  dev_cicd:
    owner/repo:
      workflow: "Backend Dev Test Container"
  prod_cicd:
    owner/repo:
      workflow: "Backend Prod Deploy"
  prod_eligibility_label: dev-accepted
  promote:
    dispatch_event: feishu-inbound-promote
    branch_mapping:
      backend: { source: dev, target: production }
```

### `acceptance`

```yaml
acceptance:
  release_owner: "github-user-id"
```

### `lead_tick`

```yaml
lead_tick:
  revision_execute_after_review: true   # D'/E' in same tick after E changes-requested
```

## Concurrency (v0.1.37+)

Global agent slot limit before Pipeline C/D/E spawn OpenCode/Cursor:

| Env / config | Default | Meaning |
|--------------|---------|---------|
| `FI_MAX_ACTIVE_AGENTS` | `2` | Max concurrent agent sessions |
| `FI_AGENT_SLOT_TIMEOUT` | `3600` | Wait for slot (seconds) |
| `FI_AGENT_SLOT_TTL` | `7200` | Lease TTL |
| `runtime.task_manager.max_active_agents` | `2` | YAML override |

State file: `{state_dir}/agent_slots.json`.

Stage `MAX_PARALLEL_*` env vars cap batch size within the global limit.

## Runtime layout (gitignored)

```
var/feishu_inbound/
  state/          # checkpoints, locks, agent_slots.json
  logs/           # engine CLI logs
worktrees/        # issue-{N}/{surface} checkouts
```

Not in surface repos (e.g. asp-backend). See RFC-001 Amendment 2026-06-17.

## Example instances

| Instance | Config location (reference) |
|----------|----------------------------|
| ASP | `AI-MYG/asp` → `tools/feishu_inbound/config.yaml` |
| Personal QMT | `config/feishu_inbound_personal.yaml` |
| rootgrove harness | `config/feishu_inbound_rootgrove.yaml` |

These live in consumer repos, not in this dist repo.

## Validation

```bash
feishu-inbound scan --config config.yaml --scan-only
```

Schema errors print at startup with the missing/invalid field name.
