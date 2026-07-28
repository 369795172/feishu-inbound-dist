---
name: feishu-inbound-pipeline
description: >-
  Generic Feishu Bitable demand pool → GitHub Issue inbound pipeline engine (triage, analysis,
  execution + Smart PR, gate review, dev handback). Use when operating feishu inbound instances
  (ASP or personal/QMT), 飞书需求池入站, feishu inbound engine, 接入需求池.
disable-model-invocation: true
---

# Feishu Inbound Engine

Pipeline **B–F** engine: Feishu Bitable demand pool → GitHub Issue → keyword triage → agent analysis → gated worktree execution + Smart PR → cross-platform gate review → dev CI handback. Stage **A** (Bitable → Issue) is a GitHub Action each instance deploys separately.

## When to use

- Operate or debug ASP / personal feishu inbound launchd jobs
- Add or change instance `config.yaml` (scan scope, routing, schedules)
- Run scan-only / dry-run before production execution
- Pin or upgrade engine version on an instance

## Prerequisites

- Python **3.10+**
- Install engine from **public dist** (no source access required):

```bash
VERSION=0.1.37
pip install "https://github.com/369795172/feishu-inbound-dist/releases/download/v${VERSION}/feishu_inbound-${VERSION}-py3-none-any.whl"
feishu-inbound --version
```

- Instance installs **pin version only** (`feishu-inbound==X.Y.Z` in requirements). See [docs/PACKAGING.md](../docs/PACKAGING.md).

```bash
# ASP instance helper
FEISHU_INBOUND_INSTALL=package bash scripts/install_feishu_inbound_engine.sh
# Personal (rootgrove) helper
FEISHU_INBOUND_INSTALL=package bash tools/feishu_inbound/install_engine.sh
```

- Full setup guide: [docs/GETTING_STARTED.md](../docs/GETTING_STARTED.md) · config: [docs/instance-config.md](../docs/instance-config.md)


## CLI entry points

Preferred wrapper (repo venv):

```bash
cd projects/feishu-inbound-skill
scripts/run_cli.sh --version
scripts/run_cli.sh triage --config /path/to/config.yaml --scan-only
```

Console script (after `pip install -e .`):

```bash
feishu-inbound --version
python -m feishu_inbound.cli <subcommand> --config /path/to/config.yaml [options]
```

**All pipeline subcommands require `--config <instance/config.yaml>`.**

### Subcommands

| Subcommand | Pipeline | Purpose |
|------------|----------|---------|
| `triage` | B | Central triage: surface / scope / difficulty / assignee |
| `scan` | C | Per-developer deep analysis → `## Feishu Inbound Analysis` |
| `execute` | D | Worktree execution + Smart PR |
| `review` | E | AI gate review (PR review only) |
| `accept` | Acceptance | Dev acceptance after F handback → `dev-accepted` + Promote handoff |
| `promote` | Promote | Gated scoped promote PR after accept pass |
| `handback` | F | After CI success → notify issue owner + post handback comment (dev & prod) |
| `lead-tick` | C→D→E→D'→E'→F→Accept→Promote→F | D' revision; E' = `review --re-review-queue`; `accept` + `promote --scan-only` between F dev and F prod |
| `install-launchd` | — | Generate/load plists from `launchd_schedules` |
| `shadow-compare` | B–F | Dry-run diff vs legacy instance scripts |

Common flags: `--scan-only`, `--dry-run`, `--issue N`, `--repo owner/repo`, `--force`, `--batch N`, `--parallel`.

Pipeline F handback also supports:
- `--stage dev|prod` — select dev (default) or prod CI/CD stage
- `--from-cicd` — fast path called from GitHub Actions dispatch workflow
- `--cicd-stage dev|prod` — stage in CI-triggered mode
- `--head-sha SHA` — commit SHA from the CI event (optional)
- `--skip-notify` — suppress Feishu DM

CI dispatch templates are in `templates/`:
- `github-workflow-pipeline-f-handback-trigger.yml` — monitors `workflow_run` events and dispatches
- `github-workflow-pipeline-f-handback-dispatch.yml` — receives dispatch and runs `feishu-inbound handback --from-cicd`
- `github-workflow-promote-pr.yml` — `repository_dispatch` `feishu-inbound-promote` → scoped promote PR (Promote Skill only; not accept pass)

### Acceptance Skill (`accept` / `acceptance/`)

Runs **after Pipeline F dev handback** — not part of Pipeline E. Requester (or assignee) invokes:

```bash
feishu-inbound accept pass --config config.yaml --issue N --repo owner/repo
feishu-inbound accept fail --config config.yaml --issue N --repo owner/repo --reason "..."
feishu-inbound accept --config config.yaml --scan-only   # lead_tick polls /accept comments
```

**Comment contract:**

```markdown
## Dev Acceptance

/accept pass
验收环境: dev
验收人: @github-login
```

**Pass:** `dev-accepted` label → `## Promote — Handoff` → Promote Skill dispatches when gates pass → assign `acceptance.release_owner` for prod review/approve/merge (prod merge stays manual).

```bash
feishu-inbound promote --config config.yaml --issue N --repo owner/repo
```

Full contract: [docs/acceptance_gate.md](../docs/acceptance_gate.md) · Promote: [skill_feishu_inbound_promote.md](skill_feishu_inbound_promote.md) · [#63](https://github.com/369795172/feishu-inbound-skill/issues/63)

**Fail:** `review-changes-requested`; remove `executed` / `review-dev-pass` / `dev-accepted`; re-assign operator for Pipeline D.

Legacy `## Pipeline E Gate Review` + `/gate pass|fail` is handled by `human_gate` for in-flight issues only.

**Migration:** New dev acceptance uses `accept` only. Legacy `human_gate` (`## Pipeline E Gate Review`) remains for in-flight issues until a follow-up deprecates it.

### Pipeline F config keys

```yaml
pipeline_f:
  feishu_notify:
    bot: tts          # tts | ic
    by_github_id:     # GitHub login → Feishu open_id (fastest lookup)
      hujianfei: ou_xxx
  dev_cicd:
    owner/repo:
      workflow: "Backend Dev Test Container"
  prod_cicd:
    owner/repo:
      workflow: "Backend Prod Deploy"
      assume_cicd_success: false  # skip CI check for this repo
  prod_eligibility_label: dev-accepted   # label gate for prod handback
  promote:
    dispatch_event: feishu-inbound-promote
    branch_mapping:
      backend: { source: dev, target: production }
      admin:   { source: dev, target: main }
      app:     { source: dev, target: main }
  env_urls:
    dev: "https://dev.example.com"
    prod: "https://example.com"

acceptance:
  release_owner: "369795172"
```

`pipeline_f.promote` branch mapping is surface-repo metadata; promote PR creation is triggered by **Promote Skill** after Acceptance pass, not by accept pass or dev CI success alone.

### Examples

```bash
# Scan queue (no agent call)
scripts/run_cli.sh scan --config projects/asp-infra/tools/feishu_inbound/config.yaml --scan-only

# Lead tick chain (launchd uses this)
scripts/run_cli.sh lead-tick --config projects/asp-infra/tools/feishu_inbound/config.yaml

# Install personal launchd (com.personal.*) — lead_tick only; Pipeline B triage is optional
scripts/run_cli.sh install-launchd --config config/feishu_inbound_personal.yaml --jobs lead_tick
```

## Instance layout (engine vs instance)

| Layer | Location | Owns |
|-------|----------|------|
| **Engine (wheel)** | `369795172/feishu-inbound-dist` | pip package `feishu_inbound` + docs (this repo) |
| **Engine source (private)** | `369795172/feishu-inbound-skill` | Maintainers only |
| **ASP instance** | `AI-MYG/asp` → `projects/asp-infra/` | `config.yaml`, Keychain `FI_ASP_*`, launchd `com.asp.*`, thin wrappers |
| **Personal instance** | rootgrove | `config/feishu_inbound_personal.yaml`, `FI_PERSONAL_*`, `com.personal.*` |

Instance wrappers (paths unchanged for launchd):

```bash
cd projects/asp-infra
./venv/bin/python tools/feishu_inbound/issue_scanner.py --scan-only
./venv/bin/python tools/feishu_inbound/triage_agent.py --scan-only
bash launchd/install.sh   # bootstraps venv + engine if missing
```

## Config schema

Validated by `src/feishu_inbound/config.py`. Required top-level keys include `github`, `state_dir`, `worktree_root`, `prompts`, `assignees`, `pipeline_cd_scan`. Multi-surface instances also need `surface_routing`, `assignee_routing`. Single-repo mode: `pipeline.mode: single-repo` + `pipeline.repo` + `surfaces_config_path` (typically one surface key, e.g. `default`).

### Single-repo mode (Pipeline C/D)

When `pipeline.mode: single-repo` (RFC-001 §4):

| Stage | Contract |
|-------|----------|
| **B** | No surface routing; `surfaces=[]` is expected. **Optional for personal** (manual `triage` CLI); **required for ASP multi-surface** (keyword launchd). Personal default launchd = `lead_tick` only. |
| **C** | `primary_surface` = first key in instance `surfaces.yaml` (not ASP `"backend"` fallback) |
| **C prompt** | Branch in Analysis = `issue-{N}` (no `/surface` suffix) |
| **D** | `base_branch` from `surfaces.yaml` for that key (not `_SURFACE_BASE_BRANCH` map) |
| **D** | Ignore stale `issue-N/backend` in old Analysis comments; resolve surface from config |
| **sync** | Unknown surface names (e.g. `backend`) map to the config's sole surface key |

SSOT for surface key + base branch: instance `surfaces.yaml`, same as multi-surface.

### Documentation contract (Pipeline C / D / E) — mandatory

API-facing changes **must** stay in sync with human- and AI-readable docs. This is not optional polish.

| Stage | Agent must |
|-------|------------|
| **C (Analysis)** | If the fix touches HTTP routes, request/response shapes, error codes, or public API behavior: list **Swagger/OpenAPI or project-declared API doc paths** in「改动文件」and include explicit doc-sync steps in「推荐方案」. If no API surface changes, write「文档: 无 API 变更，无需更新 Swagger」. |
| **D (Execute)** | Implement code **and** update the listed API docs so paths, schemas, and examples match runtime. Missing doc updates when Analysis required them = incomplete execution. |
| **E (Gate review)** | Treat「文档是否已更新」as a **first-class blocking dimension**, not a nice-to-have. API/logic changes without matching Swagger/OpenAPI updates → **打回**. |

ASP backend SSOT examples (adjust per surface AGENTS.md): OpenAPI/Swagger files under the backend repo, admin API docs if admin-facing endpoints change.

Important instance keys:

- `state_dir` / `logs_dir` — runtime checkpoints + engine logs under instance `var/feishu_inbound/` (gitignored); paths relative to **config file**, not cwd
- `instance_tools_path` — repo root containing `tools/agent_client.py` (ASP: `../..` from config file)
- `agent_client.import_chain` — e.g. `tools.agent_client.AgentClient`
- `launchd_schedules` / `launchd_label_prefix` / `launchd_labels`
- `pipeline_e`, `pipeline_f`, `advisor.enabled`
- `runtime.task_manager.max_active_agents` — global OpenCode/Agent concurrency cap (default **2**; env `FI_MAX_ACTIVE_AGENTS` overrides). Stage env `MAX_PARALLEL_*` are per-pipeline batch limits bounded by this cap.

## Concurrency (Z.ai FUP)

Before Pipeline C/D/E spawn an AgentClient session, the engine acquires a **global agent slot** (`src/feishu_inbound/runtime/task_manager.py`). Default `max_active_agents=2` matches Z.ai Coding Max “~2 concurrent active projects”. When slots are full, `wait_acquire` queues (or times out per `FI_AGENT_SLOT_TIMEOUT`) instead of silently opening another OpenCode session. Slot leases live in instance `state_dir/agent_slots.json` (fcntl + JSON, shared across stages on one Mac).

## Invariants

1. GitHub Issue is the only hand-off (labels + comment markers).
2. Dual-gate idempotency: marker comment **and** label present → skip (`scan --force` or `request-reanalysis` label overrides).
3. Tag-driven re-analysis: add `request-reanalysis` on an open assigned issue; Pipeline C re-runs even when `analyzed` + Analysis comment exist, then clears downstream gate labels (`approved-to-execute`, `executed`, `review-*`, etc.).
4. No org/repo/Bitable hardcoding in engine `src/` (fixtures only).
5. One code SSOT; instances pin engine tag; no duplicate launchd labels (`com.asp.*` vs `com.rootgrove.*`).
6. **Docs follow code**: API changes ship with updated Swagger/OpenAPI (or declared API doc SSOT); Pipeline E rejects PRs that skip required doc updates.

## Troubleshooting

| Symptom | Check |
|---------|--------|
| F handback 后 issue 被打回 `review-changes-requested` | Engine `< v0.1.16`：`human_gate` 误读 F 指引；升级至 `v0.1.16+` 并重装 instance venv |
| launchd exit 1 | `projects/asp-infra/logs/*.err`; rerun with `--scan-only` |
| `AgentClient not importable` | `instance_tools_path` points at instance repo root; `load_asp_env.sh` sourced |
| `feishu-inbound: command not found` | Use `python -m feishu_inbound.cli` or instance venv |
| `execution-exhausted` 已移除但仍 skip / 需 `--force` | Engine `< v0.1.26`：本地 `failure_count` 与 label 不同步；升级至 `v0.1.26+` 后移除 label 即可 `execute`（自动 reconcile） |
| 最新 issue 分析反复失败，更早的 issue 永远不跑 | Engine `< v0.1.33` 或 `pipeline_c.batch` 仍为 1：升级并确认 lead-tick `scan --batch N`；冷却/公平队列见 `pipeline_c` |

## Further reading

- [docs/workflows/pipeline.md](../docs/workflows/pipeline.md) — stage overview
- [docs/troubleshooting.md](../docs/troubleshooting.md) — common failures
- [docs/INDEX.md](../docs/INDEX.md) — full doc index
