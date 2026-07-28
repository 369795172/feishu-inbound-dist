# Acceptance Gate

> **Status**: Implemented in engine `v0.1.17` ([#35](https://github.com/369795172/feishu-inbound-skill/issues/35)); promote handoff refactor `v0.1.32` ([#63](https://github.com/369795172/feishu-inbound-skill/issues/63)).
> **SSOT** for dev acceptance after Pipeline F dev handback. E (`review`) is **AI PR review only**.

## Purpose

After Pipeline F dev handback, the **issue owner** (GitHub 负责人) validates dev and records pass/fail via `accept pass`. On pass, the engine marks `dev-accepted`, posts a **Promote handoff**, and **assigns the issue to `acceptance.release_owner`**. Promote PR creation is deferred to **Promote Skill** (`feishu-inbound promote`), which validates surface/repo/head_sha before dispatch. Release owner merges prod manually; Pipeline F prod notifies after prod CI success.

**Principle**: issue owner validates dev; Promote Skill (agent-gated) creates promote PR; release owner ships prod.

## Flow

```text
E pass → operator merge dev → F dev handback
→ issue owner: feishu-inbound accept pass
   ├─ fail → comment + rollback to D
   └─ pass → dev-accepted + Promote handoff + assign release owner
→ Promote Skill: validate gates → dispatch promote PR (or ⚠️ promote blocked)
→ release owner review / approve / merge prod PR
→ F prod handback
```

## Naming (unique)

| Item | Value |
|------|-------|
| CLI subcommand | `accept` (`pass` / `fail` as first argument) |
| Promote CLI | `promote` (gated dispatch after accept) |
| Module | `acceptance/issue_acceptance.py`, `acceptance/issue_promote.py` |
| Pass comment marker | `## Dev Acceptance` + `/accept pass` |
| Fail comment marker | `## Dev Acceptance` + `/accept fail` |
| Recorded marker | `## Dev Acceptance — Recorded` |
| Promote handoff marker | `## Promote — Handoff` |
| Promote recorded marker | `## Promote — Recorded` |
| Promote blocked prefix | `⚠️ promote blocked` |
| Pass label | `dev-accepted` |
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

Prod merge is **not** automated. Release owner runs `gh pr review --approve` and merges promote PR.

## Fail behavior

1. Post `## Dev Acceptance` (`/accept fail`, reason required).
2. Add `review-changes-requested`; remove `executed`, `review-dev-pass`, `dev-accepted`.
3. Assign pipeline operator.
4. Pipeline D revises on next tick.

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

`lead-tick` runs `accept --scan-only` then `promote --scan-only` between F dev and F prod handback.

## Legacy recovery

Issues with `dev-accepted` and `promote_pr: pending` from pre-v0.1.32 auto-dispatch: run `feishu-inbound promote` manually once; do not replay old cross-repo dispatch.

## Legacy human_gate

`reviewer/human_gate.py` **unchanged in #35** for backward compat. Legacy `## Pipeline E Gate Review` + `/gate pass|fail` still parsed for in-flight issues. New dev acceptance uses `accept` only.

## Related

- [skill_feishu_inbound_promote.md](../../skills/skill_feishu_inbound_promote.md)
- [skill_feishu_inbound.md](../../skills/skill_feishu_inbound.md)
