# Troubleshooting

Symptom → checks for `feishu-inbound` instances (wheel-only consumers).

## Install

| Symptom | Fix |
|---------|-----|
| `feishu-inbound: command not found` | Use `python -m feishu_inbound.cli` or ensure venv bin is on PATH |
| Public dist 404 | Check [Releases](https://github.com/369795172/feishu-inbound-dist/releases) for tag; bump pin or wait for mirror |
| Private fallback fails | Set `FEISHU_INBOUND_GH_TOKEN` with `repo` read on engine repo |
| Wrong version after install | `python -c "import feishu_inbound; print(feishu_inbound.__version__)"` vs requirements pin |

## launchd

| Symptom | Fix |
|---------|-----|
| Job exits 1 | Read `logs_dir/*.err` or `~/Library/LaunchAgents/*.plist` stderr path |
| No issues processed | Run same command manually with `--scan-only` |
| Duplicate processing | Check for duplicate launchd labels (`com.asp.*` vs `com.personal.*`) |

## Agent / import

| Symptom | Fix |
|---------|-----|
| `AgentClient not importable` | `instance_tools_path` must point at repo root containing `tools/agent_client.py` |
| OpenCode auth errors | Source instance env loader (`load_asp_env.sh` / `load_personal_env.sh`) |
| Slots never acquired | Check `state_dir/agent_slots.json`; stale leases expire per `FI_AGENT_SLOT_TTL` |

## Pipeline C (scan)

| Symptom | Fix |
|---------|-----|
| Issue never analyzed | Assignee must include `GITHUB_ASSIGNEE`; issue must be `open` |
| Skipped despite open issue | Dual-gate: both `analyzed` label **and** `## Feishu Inbound Analysis` → use `--force` or add `request-reanalysis` |
| Newest issue blocks queue | Upgrade ≥ v0.1.33; set `pipeline_c.batch` > 1; check cooldown/fair queue |
| Wrong surface in Analysis | single-repo: verify `surfaces.yaml` first key; upgrade ≥ v0.1.36 for repo binding |

## Pipeline D (execute)

| Symptom | Fix |
|---------|-----|
| Skipped with `analyzed` | Missing `approved-to-execute` (non-trivial); or `execution-in-progress` lock |
| `execution-exhausted` stuck | Upgrade ≥ v0.1.26; remove label to reconcile local failure count |
| Worktree dirty / sync fail | Run instance `sync_worktrees.py`; non-owned surfaces must be clean |
| Smart PR fails | Check instance `smart_pr.py` and `GITHUB_TOKEN` scopes |

## Pipeline E (review)

| Symptom | Fix |
|---------|-----|
| No review despite open PR | Needs `executed` label + open linked PR + no `review-dev-pass` |
| API change passed without docs | Expected fail on ≥ v0.1.15; D must update Swagger in same PR |
| App build without version bump | Blocking on ≥ v0.1.15 for app surfaces |

## Pipeline F (handback)

| Symptom | Fix |
|---------|-----|
| No handback after E pass | PR must be **merged**; dev CI workflow must **success** on merge commit |
| `skip_cicd_pending` | Wait for GHA; verify `pipeline_f.dev_cicd` workflow name matches repo |
| Issue reverted to `review-changes-requested` after F | Engine **< v0.1.16**: `human_gate` misread F comment; upgrade to v0.1.16+ |

## Acceptance / Promote

| Symptom | Fix |
|---------|-----|
| `accept pass` rejected | Requires `review-dev-pass` + `## Pipeline F Dev Handback`; no prior `dev-accepted` |
| Promote never dispatched (v0.1.32+) | **Expected**: `accept pass` only hands off; run `feishu-inbound promote` |
| `⚠️ promote blocked` | Read comment: surface/repo/head_sha/branch_mapping gate failed |
| Legacy `promote_pr: pending` | Manual `promote` once; do not replay old cross-repo dispatch |

## Idempotency rules

All stages use **dual-gate** idempotency:

1. Required **comment marker** exists (e.g. `## Feishu Inbound Analysis`)
2. Required **label** exists (e.g. `analyzed`)

Both present → skip (unless `--force` or `request-reanalysis`).

## Getting help

1. Note engine version: `feishu-inbound --version`
2. Reproduce with `--scan-only` or `--dry-run`
3. Collect issue number, labels, and latest pipeline comments
4. See [workflows/pipeline.md](./workflows/pipeline.md) for stage contracts
5. Maintainers: private engine repo issues (if you have access)
