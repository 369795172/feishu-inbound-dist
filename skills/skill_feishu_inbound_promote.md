---
name: feishu-inbound-promote
description: >-
  Promote Skill — after Acceptance pass (dev-accepted), validate surface/repo/head_sha
  gates and dispatch scoped promote PR to production. Use when creating production promote
  PR, promote handoff, dev-accepted promote, scoped promote.
disable-model-invocation: true
---

# Promote Skill

Runs **after Acceptance pass** (`dev-accepted`). Agent (or `feishu-inbound promote` CLI) validates production gates before dispatching `feishu-inbound-promote` to the surface repo.

**Breaking change (v0.1.32+)**: `accept pass` no longer auto-dispatches promote PR. It posts `## Promote — Handoff` and queues this skill.

## When to use

- Issue has `dev-accepted` + `## Promote — Handoff` + `## Dev Acceptance — Recorded`
- Agent or release owner needs to create scoped `promote/issue-{N}/{surface}` PR
- Recovery after `⚠️ promote blocked` gate failure

## Gates (fail-closed)

Before dispatch, validate:

1. **Surface SSOT**: merged `issue-{N}/{surface}` PR head > Analysis 执行路径 > surface label
2. **Repo match**: `dispatch_repo` from `surfaces.yaml` for resolved surface
3. **head_sha ∈ surface_repo** (not cross-repo central SHA)
4. **branch_mapping** from `pipeline_f.promote.branch_mapping` or surfaces.yaml
5. **Surface label** (if present) must match resolved surface

Any gate failure → `⚠️ promote blocked` comment + @release_owner; **no** dispatch.

## CLI

```bash
feishu-inbound promote --config <instance.yaml> --issue N --repo owner/repo
feishu-inbound promote --config <instance.yaml> --scan-only   # lead_tick
```

## Observability

- `issue_repo → dispatch_repo` logged on every dispatch attempt
- `promote PR: pending` — correctly dispatched, waiting for CI PR
- `⚠️ promote blocked` — gate validation failed (explicit error, not silent pending)

## Related

- [acceptance_gate.md](../docs/acceptance_gate.md)
- - [workflows/promote.md](../docs/workflows/promote.md)
- Mechanical PR creation: `templates/github-workflow-promote-pr.yml`, `tools/cicd/create_promote_pr.sh`

## Recovery (legacy issues)

Issues with `dev-accepted` and `promote_pr: pending` from pre-v0.1.32 auto-dispatch: run Promote Skill manually once; do not replay old cross-repo dispatch payloads.
