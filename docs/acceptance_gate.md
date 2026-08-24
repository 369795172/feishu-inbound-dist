# Acceptance Gate

> **Status**: Implemented in engine `v0.1.17` ([#35](https://github.com/369795172/feishu-inbound-skill/issues/35)); promote handoff refactor `v0.1.32` ([#63](https://github.com/369795172/feishu-inbound-skill/issues/63)); prod-ship Option B ([#115](https://github.com/369795172/feishu-inbound-skill/issues/115)).
> **SSOT** for dev acceptance after Pipeline F dev handback. E (`review`) is **AI PR review only**.

## Purpose

After Pipeline F dev handback, the **issue owner** (GitHub 负责人) validates dev and records pass/fail via `accept pass`. On pass, the engine marks `dev-accepted`, posts a **Promote handoff**, and **assigns the issue to `acceptance.release_owner`**. Promote PR creation is deferred to **Promote Skill** (`feishu-inbound promote`), which validates surface/repo/head_sha before dispatch. When instance `prod_ship.enabled` is true, lead tick then runs **prod-ship** (Option B): mechanical checks + Feishu DM, wait for `prod-ship-confirmed`, then execute the instance runbook. Pipeline F prod handback still owns completion after prod CI success.

**Principle**: issue owner validates dev; Promote Skill (agent-gated) creates promote PR; release owner confirms prod-ship; engine runs the instance runbook; F prod records the green deploy.

## Flow

```text
E pass → operator merge dev → F dev handback
→ issue owner: feishu-inbound accept pass
   ├─ fail → comment + rollback to D
   └─ pass → dev-accepted + Promote handoff + assign release owner
→ Promote Skill: validate gates → dispatch promote PR (or ⚠️ promote blocked)
→ prod-ship (if `prod_ship.enabled`): mechanical checks → Feishu DM + Awaiting Confirmation
   → release_owner adds `prod-ship-confirmed`
   → instance runbook (merge / monitor / bypass / GHA record / CVM verify)
→ F prod handback
```

## Naming (unique)

| Item | Value |
|------|-------|
| CLI subcommand | `accept` (`pass` / `fail` as first argument) |
| Promote CLI | `promote` (gated dispatch after accept) |
| Prod-ship CLI | `prod-ship` (Option B confirm-then-merge after Promote Recorded) |
| Module | `acceptance/issue_acceptance.py`, `acceptance/issue_promote.py`, `acceptance/prod_ship.py` |
| Pass comment marker | `## Dev Acceptance` + `/accept pass` |
| Fail comment marker | `## Dev Acceptance` + `/accept fail` |
| Recorded marker | `## Dev Acceptance — Recorded` |
| Promote handoff marker | `## Promote — Handoff` |
| Promote recorded marker | `## Promote — Recorded` |
| Promote blocked prefix | `⚠️ promote blocked` |
| Prod-ship awaiting marker | `## Prod Ship — Awaiting Confirmation` |
| Prod-ship shipping marker | `## Prod Ship — Shipping` (mutex, not completion) |
| Prod-ship blocked prefix | `⚠️ prod-ship blocked` |
| Pass label | `dev-accepted` |
| Prod-ship confirm label | `prod-ship-confirmed` (release_owner only; must be after Awaiting Confirmation) |
| Fail label | `review-changes-requested` (remove `executed`, `review-dev-pass`, `dev-accepted`) |
| Promote dispatch event | `feishu-inbound-promote` |
| Promote workflow template | `templates/github-workflow-promote-pr.yml` |

No `dev-accept` alias. No `## Pipeline E Gate Review` for new dev acceptance.

## CLI

```bash
feishu-inbound accept pass --config config.yaml --issue N --repo owner/repo
feishu-inbound accept fail --config config.yaml --issue N --repo owner/repo --reason "..."
feishu-inbound accept --config config.yaml --scan-only

feishu-inbound promote --config config.yaml --issue N --repo owner/repo
feishu-inbound promote --config config.yaml --scan-only

feishu-inbound prod-ship --config config.yaml --scan-only
feishu-inbound prod-ship --config config.yaml --issue N --repo owner/repo
```

Natural language (Cursor skill): 「按 Acceptance Skill，issue #N 验收通过/不通过」→ runs the same `accept` command. Promote: 「按 Promote Skill，issue #N 创建 promote PR」.

## Pass behavior (v0.1.32+)

1. Gate: `review-dev-pass` + `## Pipeline F Dev Handback` present; skip if `dev-accepted` or `## Dev Acceptance — Recorded` exists.
2. Post `## Dev Acceptance` comment (`/accept pass`, env, requester).
3. Add `dev-accepted` label.
4. Resolve suggested surface/repo/head_sha/evidence (no dispatch).
5. Post `## Promote — Handoff` with structured suggestions.
6. Assign issue to **`acceptance.release_owner`** for prod gate.
7. Post `## Dev Acceptance — Recorded` with promote queued / pre-existing PR URL.

**Accept pass does not** call `repository_dispatch`. Use Promote Skill next.

## Promote Skill behavior

1. Gate: `dev-accepted` + handoff markers present.
2. Resolve surface SSOT: merged `issue-{N}/{surface}` > Analysis 执行路径 > surface label.
3. Validate `head_sha ∈ surface_repo`, branch_mapping, surface label consistency.
4. On failure: post `⚠️ promote blocked` + @release_owner; no dispatch.
5. On success: `repository_dispatch` → surface repo, event `feishu-inbound-promote`.
6. Poll open PR `promote/issue-{N}/{surface}` (timeout 5min; no duplicate dispatch).
7. Post `## Promote — Recorded` with promote PR URL.

Log line on dispatch: `issue_repo → dispatch_repo`.

## Prod-ship (Option B)

Scan: `dev-accepted` + `## Promote — Recorded` + open promote PR + instance `prod_ship.enabled: true` (default false).

1. Mechanical validation (fail-closed): recorded `head_sha` present; PR base matches `branch_mapping` target; PR tip equals recorded SHA **or** recorded SHA exists and PR files ⊆ `target...recorded_sha` (net-state).
2. Post `## Prod Ship — Awaiting Confirmation` (binds `pr_head_sha`) and Feishu DM to `acceptance.release_owner` with issue/PR links, evidence, and `gh issue edit <N> --add-label prod-ship-confirmed`.
3. Confirm gate: label actor is `release_owner`; labeled event is **after** the Awaiting Confirmation comment. Stale/wrong-actor/SHA-drift → `⚠️ prod-ship blocked` + DM, no execute.
4. `prod_ship.auto_confirm` is the Option A evolution slot (default false).
5. Execute via `AgentClient` + instance `prod_ship.runbook.path`. Engine does not hardcode CVM/skopeo/SSH.
6. Completion is existing `handback --stage prod`. No new done marker.

```yaml
prod_ship:
  enabled: false
  auto_confirm: false
  runbook:
    path: rules/skills/deploy_backend_promote_bypass_then_gha.md
    intent: execution
    timeout_sec: 3600
```

## Fail behavior

Default path: **fail → Pipeline D** (not C). Fail is scoped to the **current revision round**.

1. Post `## Dev Acceptance` (`/accept fail`, reason required).
2. Add `review-changes-requested`; remove `executed`, `review-dev-pass`, `dev-accepted`.
3. Assign pipeline operator (assignee 转回执行人).
4. Pipeline D revises on next tick. D injects the latest `## Dev Acceptance` + `/accept fail` comment **in full** into the execution prompt (not only the `原因:` field), plus revision-round context.
5. If Analysis or acceptance criteria are wrong, a human adds `request-reanalysis` so Pipeline C re-runs. Free-form comments do not reroute the pipeline.

### Stale fail replay

An `/accept fail` is applied only when its comment timestamp is **later than** the latest dev deploy / PR merge evidence (`mergedAt`, else latest `## Pipeline F Dev Handback` `created_at`). A fail that predates the current merge is ignored, so an old round-1 fail cannot roll the issue back after D′ / E / merge.

### Revision round (F 去重)

`revision round = 1 + count(/accept fail)`. Pipeline F handback 判重是 `(issue, round)`：上一轮 `## Pipeline F Dev Handback` 不阻止本轮再发。新 F 评论带 `revision-round: N`。dev handback 成功时 assignee 转为提需人。

## Promote PR (CI tool — agent-triggered only)

- Workflow template: `templates/github-workflow-promote-pr.yml` → surface `.github/workflows/promote-pr.yml`.
- Trigger: `repository_dispatch` `feishu-inbound-promote` **only** from Promote Skill (not accept pass).
- Uses `GITHUB_TOKEN` (bot author) + `create_promote_pr.sh --only-issue {N}`.
- Per-issue branch: `promote/issue-{N}/{surface}` → `production` or `main`.
- **SHA window**: workflow resolves `before` as `git merge-base(origin/{target}, head_sha)` (not `head_sha^`), so multi-PR issues collect all commits on the integration branch since the production fork point; `--only-issue` filters across that full range.
- **Scoped apply**: `--only-issue` uses net-state checkout (`git checkout head_sha -- <paths>`) instead of per-commit cherry-pick, avoiding conflicts from dev-only intermediate commits.
- **Fail-fast**: if promote creation fails, workflow comments on the issue with the run URL and manual recovery steps.

```yaml
pipeline_f:
  prod_eligibility_label: dev-accepted
  promote:
    dispatch_event: feishu-inbound-promote
    branch_mapping:
      backend: { source: dev, target: production }
      admin:   { source: dev, target: main }
      app:     { source: dev, target: main }

acceptance:
  release_owner: "369795172"
```

## lead_tick

`lead-tick` runs `accept --scan-only` then `promote --scan-only` then `prod-ship --scan-only` between F dev and F prod handback.

## Legacy recovery

Issues with `dev-accepted` and `promote_pr: pending` from pre-v0.1.32 auto-dispatch: run `feishu-inbound promote` manually once; do not replay old cross-repo dispatch.

## Legacy human_gate

`reviewer/human_gate.py` **unchanged in #35** for backward compat. Legacy `## Pipeline E Gate Review` + `/gate pass|fail` still parsed for in-flight issues. New dev acceptance uses `accept` only.

## Related

- [test.md](test.md) — 引擎分层测试与 fail-closed 命令（本文件不重复 pytest 门禁）
- [skill_feishu_inbound_promote.md](../skills/skill_feishu_inbound_promote.md)
- [skill_feishu_inbound.md](../skills/skill_feishu_inbound.md)
- rootgrove: [workflow_feishu_inbound_promote.md](https://github.com/369795172/feishu-inbound-skill/issues/63), [workflow_scoped_promote_pr.md](https://github.com/369795172/feishu-inbound-skill/issues/35)
