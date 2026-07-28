# Documentation Index

Consumer documentation for `feishu-inbound` without access to the private engine source repo.

## Onboarding

| Document | Audience | Summary |
|----------|----------|---------|
| [GETTING_STARTED.md](./GETTING_STARTED.md) | New instance operator | Install wheel, minimal config, first `scan-only`, launchd |
| [PACKAGING.md](./PACKAGING.md) | DevOps / CI | Pin version, install helpers, GHA, upgrade |
| [instance-config.md](./instance-config.md) | Instance maintainer | `config.yaml` schema, paths, secrets, launchd |
| [troubleshooting.md](./troubleshooting.md) | On-call | Symptom → cause → fix |

## Agent skills (CLI contract)

Load these into Cursor / Claude / OpenCode as skills when operating the pipeline:

| Skill | File |
|-------|------|
| Main engine | [../skills/skill_feishu_inbound.md](../skills/skill_feishu_inbound.md) |
| Promote | [../skills/skill_feishu_inbound_promote.md](../skills/skill_feishu_inbound_promote.md) |
| Index | [../skills/INDEX.md](../skills/INDEX.md) |

## Pipeline workflows

| Stage | CLI | Workflow doc |
|-------|-----|--------------|
| Overview A→F | — | [workflows/pipeline.md](./workflows/pipeline.md) |
| B Triage | `triage` | [workflows/triage.md](./workflows/triage.md) |
| C Analysis | `scan` | [workflows/agent.md](./workflows/agent.md) |
| D Execute | `execute` | [workflows/executor.md](./workflows/executor.md) |
| E Gate review | `review` | [workflows/gate_review.md](./workflows/gate_review.md) |
| F Dev handback | `handback` | [workflows/dev_handback.md](./workflows/dev_handback.md) |
| Acceptance | `accept` | [workflows/acceptance.md](./workflows/acceptance.md) + [acceptance_gate.md](./acceptance_gate.md) |
| Promote | `promote` | [workflows/promote.md](./workflows/promote.md) |
| Dev self-issue | — | [workflows/dev_issue.md](./workflows/dev_issue.md) |
| Lead tick chain | `lead-tick` | [workflows/pipeline.md](./workflows/pipeline.md#lead-tick) |

## Architecture

| Document | Content |
|----------|---------|
| [rfc/RFC-001-engine-instance-architecture.md](./rfc/RFC-001-engine-instance-architecture.md) | Engine vs instance, secrets prefix, `var/feishu_inbound/`, single-repo |
| [PRD.md](./PRD.md) | Product goals, instances (ASP / personal), acceptance criteria |
| [personal_bitable_setup.md](./personal_bitable_setup.md) | Personal Bitable, umbrella tick, board sync |
| [CHANGELOG.md](./CHANGELOG.md) | Version history |

## CI templates

| File | Deploy to |
|------|-----------|
| [../templates/github-workflow-pipeline-f-handback-trigger.yml](../templates/github-workflow-pipeline-f-handback-trigger.yml) | Surface repo `.github/workflows/` |
| [../templates/github-workflow-pipeline-f-handback-dispatch.yml](../templates/github-workflow-pipeline-f-handback-dispatch.yml) | Instance or infra repo |
| [../templates/github-workflow-promote-pr.yml](../templates/github-workflow-promote-pr.yml) | Surface repo |

## External references

- Private engine (maintainers only): `369795172/feishu-inbound-skill`
- ASP instance example: `AI-MYG/asp` → `tools/feishu_inbound/config.yaml`
- Latest wheel: [Releases](https://github.com/369795172/feishu-inbound-dist/releases)
