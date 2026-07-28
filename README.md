# feishu-inbound-dist

**Artifacts + documentation** for the private engine [`369795172/feishu-inbound-skill`](https://github.com/369795172/feishu-inbound-skill).

This public repository hosts:

1. **GitHub Release wheels** for `feishu-inbound` (install without source access)
2. **Complete usage documentation** — CLI contract, instance setup, pipeline workflows, acceptance/promote skills, CI templates

Source code stays in the private engine repo. **You do not need engine source** to install, configure, and operate an instance.

## Quick install

Pin a version in your instance `requirements-feishu-inbound.txt`, then:

```bash
VERSION=0.1.37
pip install "https://github.com/369795172/feishu-inbound-dist/releases/download/v${VERSION}/feishu_inbound-${VERSION}-py3-none-any.whl"
feishu-inbound --version
```

Or use an instance helper (public dist first, private engine Release as fallback):

```bash
# ASP (asp-infra)
FEISHU_INBOUND_INSTALL=package bash scripts/install_feishu_inbound_engine.sh

# rootgrove personal
FEISHU_INBOUND_INSTALL=package bash tools/feishu_inbound/install_engine.sh
```

See [docs/PACKAGING.md](docs/PACKAGING.md) for CI, PAT fallback, and upgrade steps.

## Documentation map

| Start here | Purpose |
|------------|---------|
| [docs/GETTING_STARTED.md](docs/GETTING_STARTED.md) | First-time setup: install → config → launchd → smoke test |
| [docs/INDEX.md](docs/INDEX.md) | Full documentation index |
| [skills/skill_feishu_inbound.md](skills/skill_feishu_inbound.md) | **Agent CLI contract** (subcommands, flags, config keys) |
| [docs/workflows/pipeline.md](docs/workflows/pipeline.md) | End-to-end Pipeline A→F overview |
| [docs/instance-config.md](docs/instance-config.md) | `config.yaml` schema and examples |
| [docs/acceptance_gate.md](docs/acceptance_gate.md) | Dev acceptance (`accept`) + promote handoff |
| [skills/skill_feishu_inbound_promote.md](skills/skill_feishu_inbound_promote.md) | Promote Skill after `dev-accepted` |
| [docs/troubleshooting.md](docs/troubleshooting.md) | Common failures and fixes |

### Pipeline workflows (B–F)

| Stage | Doc |
|-------|-----|
| B Triage | [docs/workflows/triage.md](docs/workflows/triage.md) |
| C Analysis | [docs/workflows/agent.md](docs/workflows/agent.md) |
| D Execute | [docs/workflows/executor.md](docs/workflows/executor.md) |
| E Gate review | [docs/workflows/gate_review.md](docs/workflows/gate_review.md) |
| F Dev handback | [docs/workflows/dev_handback.md](docs/workflows/dev_handback.md) |
| Acceptance | [docs/workflows/acceptance.md](docs/workflows/acceptance.md) |
| Promote | [docs/workflows/promote.md](docs/workflows/promote.md) |
| Dev self-issue | [docs/workflows/dev_issue.md](docs/workflows/dev_issue.md) |

### Architecture & product

| Doc | Content |
|-----|---------|
| [docs/rfc/RFC-001-engine-instance-architecture.md](docs/rfc/RFC-001-engine-instance-architecture.md) | Engine vs instance split, secrets, single-repo mode |
| [docs/PRD.md](docs/PRD.md) | Product requirements and milestones |
| [docs/personal_bitable_setup.md](docs/personal_bitable_setup.md) | Personal Bitable / umbrella scheduling |
| [docs/CHANGELOG.md](docs/CHANGELOG.md) | Release notes (mirrored from engine) |

### CI templates

Copy into your surface repo `.github/workflows/`:

| Template | Purpose |
|----------|---------|
| [templates/github-workflow-pipeline-f-handback-trigger.yml](templates/github-workflow-pipeline-f-handback-trigger.yml) | Watch dev/prod CI `workflow_run` → dispatch handback |
| [templates/github-workflow-pipeline-f-handback-dispatch.yml](templates/github-workflow-pipeline-f-handback-dispatch.yml) | Run `feishu-inbound handback --from-cicd` |
| [templates/github-workflow-promote-pr.yml](templates/github-workflow-promote-pr.yml) | `repository_dispatch` → scoped promote PR |

## Publish model

| Surface | Repo | Contents |
|---------|------|----------|
| Source (private) | `369795172/feishu-inbound-skill` | Engine source, tests, RFCs, release tooling |
| Artifacts + docs (public) | `369795172/feishu-inbound-dist` (this repo) | Release wheels + documentation |

Every release is **dual-published**: the same wheel tag `vX.Y.Z` appears on both private engine Release and this public dist Release.

Maintainers: `./scripts/release_engine.sh` auto-runs `scripts/mirror_docs_to_dist.sh --push` after wheel publish. GHA `Publish Python Package` mirrors docs when `FEISHU_INBOUND_DIST_TOKEN` is set. Manual: `cd projects/feishu-inbound-skill && ./scripts/mirror_docs_to_dist.sh X.Y.Z --push`

## Notes

- Wheels are binary distributions; treat them as redistributable build products, not open source.
- Do not open PRs with engine **source code** here. Engine changes go to the private skill repo; documentation updates can go here directly or via the mirror script.
