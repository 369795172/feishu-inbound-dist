# Changelog

## v0.1.51

- prod-ship confirmation hardening (#116 bot follow-up, PR #121): explicit `timeout_sec=0` is preserved end-to-end (`dict.get` default instead of `or` at both settings and runbook call sites); the "latest" `prod-ship-confirmed` label event is chosen by parsed timestamp instead of lexicographic string order (mixed `Z`/offset formats no longer flip stale decisions; fully unparseable events fail closed as BLOCKED); REST-shaped comment `user.login` is extracted properly instead of `str(dict)`.
- Removed `auto-tag-on-version-bump` workflow (#119 方案 A): GITHUB_TOKEN-pushed tags never trigger the publish workflow (GitHub anti-recursion), and its unconditional success echo was a standing false green. Release automation converges to `release_engine.sh` single track.

---

## v0.1.50

- Pipeline E infra circuit breaker (#112): consecutive `gate-infra-failure` (default 3) stops auto-retry (`needs_review` → `skip_infra_exhausted`); one summary comment + one DM; recover by clearing the label or `--force`.
- L2 bot review fetch (#112): `BotInlineComment.id` is `str` (GitHub GraphQL `PRR_` node ids); reviews vs inline comments fail independently instead of dropping the whole packet.
- L3 follow-up (#112): scan unadjudicated bot comments regardless of arrival time; E records adjudicated ids in reviewer state.
- prod-ship Option B (#115): promote PR confirm-then-merge — DM notify + confirm label, then merge/monitor/bypass/replay GHA record for production handback.
- OpenAPI gate hardening (#88 follow-up): `git add` failure and `git fetch`/export timeouts return `GateResult(outcome="infrastructure_unavailable")` instead of crashing Pipeline D/E.
- Scanner label filtering (#93 follow-up): `fetch_repo_labeled_issues` over-fetches when `exclude_labels` is set so excluded issues no longer silently shrink the result below `limit`; fatal CLI paths (acceptance / dev handback) log `Error:` instead of `WARN:`.
- Overlap marking robustness (#97 follow-up): `_post_pr_comment` failures are best-effort (label add/remove and later PRs continue); executor overlap wiring now covered by behavioral tests instead of source-string assertions.

---

## v0.1.49

- Fix L3 live-path crash: `get_agent_client` returns an instance, but `bot_review_followup` classify/implement double-instantiated it (`TypeError: 'AgentClient' object is not callable`). CI never caught it because tests inject `classify_override`; added a regression test running the live client path with an instance fake. Found during first production enablement of the follow-up job (2026-08-20).

---

## v0.1.48

- Pipeline E bot review gate (#106): request Copilot as a PR reviewer on create (L1), wait up to `pipeline_e.bot_review_wait_seconds` (default 600s) and adjudicate inline bot comments before merge (L2), then scan post-merge bot comments and open a fix PR with no GitHub issue (L3).
- L3 follow-up hardening (#106 revision): `gh pr create` failures no longer crash the scan job; unclassified LLM results retry with backoff instead of being consumed; follow-up branches are timestamped and pushed with `--force-with-lease`.

---

## v0.1.47

Auto-drafted from commits since previous release. Issues: [#104](https://github.com/369795172/feishu-inbound-skill/issues/104).

### Changes

- [#105](https://github.com/369795172/feishu-inbound-skill/pull/105): [FEIS-104] Analysis validation-failure diagnostics: persist violations + raw output for routing decisions

---

## 0.1.46 (2026-08-19)

- Accept-fail classification (#102): `accept fail --class {defect,spec-change}` (default `defect`) renders a machine-readable `分类:` line on the fail comment; free-form `/accept fail` comments may declare `分类: spec-change` / `fail-class: spec_change` / `分类: 规格变更`. Spec-change fails additionally add `request-reanalysis` and route to Pipeline C re-analysis; unclassified fails keep the existing D fast path.
- Pipeline D staleness gate (#102): after a spec-change accept-fail, D refuses to execute while the canonical Analysis predates the fail (`skip_stale_analysis` + idempotent skip note). Defect fails and round-1 issues are not gated.
- Pipeline D revision prompt precedence (#102): in revision rounds with acceptance feedback, the fail feedback + re-analyzed Analysis are declared the authoritative spec, overriding the legacy "strictly follow 推荐方案" hard rule.
- Pipeline E analysis version binding (#102): review comments carry an engine-appended `analysis-reviewed: round-N-canonical` line; the review prompt requires it and forbids conformance verdicts against a pre-fail Analysis in spec-change rounds. Engine mechanical post-check downgrades an approval issued against a stale canonical Analysis to `analysis-stale` rejection (routes to re-analysis, does not enqueue D revision).

## 0.1.45 (2026-08-19)

- Worktree baseline/sync dirty check ignores untracked files (`git status --porcelain --untracked-files=no`). Agent state dirs (e.g. `.omc/`) left in shared surface worktrees no longer make Pipeline E fail closed with `gate-infra-failure`; a clean-tracked stale tree now auto-fetches and resets to `origin/{base}` as designed. Untracked files are never touched by the reset; a path collision with a tracked file in the target revision still fails closed via git's own checkout guard.
- Applies to both `ensure_origin_baseline` (Pipeline E review worktree) and `sync_one` (Pipeline C/D surface sync).

## 0.1.44 (2026-08-18)

- Pipeline E: `gh pr view` failure is `infrastructure_unavailable` (infra_hold), not `not_applicable` (#99).
- Pipeline E: review prompt keeps literal `{}` in diff / tests / issue body (no double-escape on a single `str.format`) (#99).
- Pipeline D: product-ambiguity sentinel accepts `无【备注】` / `无[注]` (#99).
- Promote: labeled-issue queue uses per-repo enumeration; query failure exits nonzero (#99).
- State machine gains a revision round (`1 + accept-fail count`). Pipeline F handback dedup is `(issue, round)`; a previous F marker does not block the next round.
- Accept-fail replay requires the fail comment timestamp to be later than the latest dev PR merge / F handback; older fails do not roll state back.
- Pipeline E fetches/resets the review worktree to `origin/{base}` before delegate, or fails closed if the tree is dirty/stale.
- E and D prompts inject round context; lead 轮次附言 is mandatory.
- Dev handback sole-assigns the acceptance owner (see #82 contract below); accept-fail still assigns the operator.
- Pipeline F: resolve an acceptance owner (human author / bot-author registry lookup / fallback / executor self) and assign that login instead of always handing back to the issue author (#82).
- Pipeline F: handback comments and DMs fork on owner identity (`本人验收` vs `负责人代确认`); drop Gen1 "由提需人执行 Acceptance Skill" copy (#82).
- Pipeline F: prod assignment idempotency anchors the same acceptance owner as dev handback (#82).

## 0.1.43 (2026-08-18)

- Pipeline E: review prompt treats PR diff + test/OpenAPI mechanical results as first-class evidence; C Analysis is unauthorized background (not the implementation contract).
- Pipeline E: AI gate comments include a GateResult four-class table (check, interpreter, command).
- Pipeline E: missing worktree fails closed as `infrastructure_unavailable` / `gate-infra-failure` (no silent skip).
- Pipeline F / Acceptance: labeled-issue discovery uses per-repo `gh issue list`; query failure is observable (exit ≠ 0) (#91).


## 0.1.42 (2026-08-18)

- Pipeline F: dev handback markers are scoped to the latest merged PR round, so accept-fail reruns can hand back again.
- Pipeline F: `--force` can recover a dev handback skipped by an existing marker.
- Pipeline F: dev CI lookup filters push runs and paginates results to avoid missing the target workflow.
- Pipeline D: `无（注释）` / `无(注)` treated as the no-product-ambiguity sentinel (#89).

## 0.1.41 (2026-08-18)

- Pipeline D/E: `enforce_openapi_gate` returns typed `GateResult` (`pass` / `not_applicable` / `implementation_fail` / `infrastructure_unavailable`).
- Pipeline D: infrastructure gate failures do not increment business `failure_count`, do not strip `executed`, and do not enter the revision queue.
- Pipeline E: `gate-infra-failure` tickets re-run the mechanical OpenAPI gate only until the environment recovers; spec mismatch still bounces to D.

---

## 0.1.40 (2026-08-14)

- Pipeline D: revision prompts ingest the full latest `## Dev Acceptance` comment containing `/accept fail` (not just the `原因:` line).
- Pipeline D: Pipeline E feedback is injected only when the comment carries rejection semantics (`结论: 打回` / `Pipeline E 打回`); when both exist, accept-fail feedback precedes E feedback.
- Docs: acceptance fail decision tree — implementation drift → D revision; Analysis / product scope wrong → human `request-reanalysis`; free-form comments are ignored by the engine.

---

## 0.1.39 (2026-08-13)

- Pipeline C: add versioned `AnalysisContract v1` validation and canonical Markdown rendering.
- Pipeline C: repair prompts receive the failed artifact plus machine-readable violations before executor fallback.
- Pipeline C: add `analysis-failed` only after the configured cross-tick retry budget is exhausted.
- Pipeline D: fail closed for malformed v1 Analysis comments while retaining legacy comment compatibility.
- Routing: support a dedicated instance-configured `feishu_inbound_analysis` intent.

## 0.1.38 (2026-08-07)

- Pipeline D recognizes Markdown-wrapped `无` as the no-product-ambiguity sentinel.
- Prefixes such as `无法确认` remain unresolved product ambiguity.

## 0.1.37 (2026-08-03)

- Pipeline F prod handback resolves merged `promote/issue-N/<surface>` PRs instead of dev PRs.
- Successful production tips may prove earlier promote merges by GitHub commit ancestry.
- Existing prod markers with incomplete ownership enter repair-only mode without duplicate comments or notifications.
- Handback errors now suppress completion markers and notifications.

## 0.1.36 (2026-07-26)

- Pipeline C: `validate_analysis_consistency` gate (legacy `admin/backend` path, repo→worktree, branch surface, evidence/plan alignment).
- Pipeline C: `resolve_primary_surface` binds `primary_surface` to issue GitHub repo via `surfaces.yaml`.
- Triage: `detect_surface` drops `admin` when issue text is backend/API without frontend signals.
- Prompts: ASP repo topology hard rules (rules 9–10).

## 0.1.35

Prior release.
