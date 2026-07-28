# Feishu Inbound Skills

Agent-loadable skill files for operating the pipeline without engine source.

| Skill | File | When to load |
|-------|------|--------------|
| **Main engine** | [skill_feishu_inbound.md](./skill_feishu_inbound.md) | Any `feishu-inbound` CLI operation, config, launchd, triage/scan/execute/review/handback/lead-tick |
| **Promote** | [skill_feishu_inbound_promote.md](./skill_feishu_inbound_promote.md) | After `dev-accepted`, creating scoped promote PR |

## Cursor / Claude setup

1. Copy or symlink skill files into your project's `.cursor/skills/` or Claude skills directory
2. Or reference by URL: `https://github.com/369795172/feishu-inbound-dist/blob/main/skills/skill_feishu_inbound.md`
3. Ensure `feishu-inbound` CLI is installed in the active venv (see [docs/PACKAGING.md](../docs/PACKAGING.md))

## Human-readable workflows

Skills define **CLI contract**. Stage semantics and role definitions:

- [docs/workflows/pipeline.md](../docs/workflows/pipeline.md)
- [docs/workflows/](../docs/workflows/) (per-stage)

## Version alignment

Match skill docs to your pinned wheel version. Check [docs/CHANGELOG.md](../docs/CHANGELOG.md) when upgrading.

Current dist release: see [GitHub Releases](https://github.com/369795172/feishu-inbound-dist/releases).
