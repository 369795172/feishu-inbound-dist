# Changelog

## 0.1.39 (2026-08-13)

- Pipeline C validates and canonically renders `AnalysisContract v1` before publishing.
- Invalid agent output enters evidence-aware repair and route fallback before `analysis-failed`.
- Pipeline D rejects malformed v1 Analysis while retaining legacy comment compatibility.

## 0.1.38 (2026-08-07)

- Pipeline D recognizes Markdown-wrapped `无` as the no-product-ambiguity sentinel.
- Prefixes such as `无法确认` remain unresolved product ambiguity.

## 0.1.37 (2026-07-28)

- Global task manager (`runtime/task_manager.py`): file-lock + JSON lease limits concurrent AgentClient/OpenCode sessions across Pipeline C/D/E (default `max_active_agents=2`, env `FI_MAX_ACTIVE_AGENTS`).
- Stage `MAX_PARALLEL_*` batch caps are bounded by the global bucket (`effective_parallel_limit`).
- Docs: README + `skills/skill_feishu_inbound.md` — concurrency and Z.ai FUP alignment.

## 0.1.36 (2026-07-26)

- Pipeline C: `validate_analysis_consistency` gate (legacy `admin/backend` path, repo→worktree, branch surface, evidence/plan alignment).
- Pipeline C: `resolve_primary_surface` binds `primary_surface` to issue GitHub repo via `surfaces.yaml`.
- Triage: `detect_surface` drops `admin` when issue text is backend/API without frontend signals.
- Prompts: ASP repo topology hard rules (rules 9–10).

## 0.1.35

Prior release.
