# Feishu Inbound Dev Issue Workflow

## 元数据

- **类型**: Workflow（开发者自建 Issue 快通道）
- **适用场景**: 在个人/自研 repo（rootgrove、opencode、taproot 等）手动创建 GitHub Issue，跳过 Pipeline A/B，直接进入 Pipeline C 完成态，等待 Pipeline D 执行。
- **触发**: personal repo issue、自建 issue、skip triage、feishu inbound dev、analyzed、approved-to-execute、developer self-improvement
- **创建日期**: 2026-06-23

---

## 核心原则

**开发者自建 = 已理解需求 → 跳过 A/B，手动完成 C，等待 D。**

> 这个因为是开发者自己开发的，所以不用走 triage，直接算是 analyzed 后的内容（即已经走完 pipeline C，人工 review 计划后，打标后，就可以进行 pipeline D 的阶段）。

跳过 A = 非 Bitable 表单入站；**仍须加** `feishu-inbound`（看板/巡检/routing 成员标记）。跳过 B = **不加** `triaged`。

Pipeline 状态：
- A（飞书入站）— **跳过**（非 Bitable 来源，**仍须加** `feishu-inbound` label）
- B（Triage）— **跳过**（不加 `triaged` label）
- C（Analysis）— **人工完成**：Agent 写 `## Feishu Inbound Analysis` 作为 **第一条 comment**（不是 body，D 的 `issue_executor.py` 从 comment 读 Analysis）
- Human Gate — 审核 Analysis 后加 `approved-to-execute`（trivial 可略）
- D（Execute + Smart PR）— 自动或手动触发

---

## 步骤

### Step 1 — 创建 Issue（gh 命令模板）

```bash
gh issue create \
  --repo <owner>/<repo> \
  --title "<title>" \
  --body "$(cat <<'BODY'
## 背景

<背景描述：来源、架构裁决等>

---

## Pipeline 状态（人工已完成 C，跳过 B）

- [x] **A** 飞书入站 — **跳过**（开发者自建，非 Bitable；**仍打** `feishu-inbound` 作为看板/巡检成员标记）
- [x] **B** Triage — **跳过**（不加 `triaged`）
- [x] **C** Analysis — 见首条 comment `## Feishu Inbound Analysis`
- [ ] **Human Gate** — 审核 Analysis 后加 label `approved-to-execute`
- [ ] **D** Execute + PR — `issue_executor.py`（见 Analysis §6）
- [ ] **E** Gate review
- [ ] **F** Dev handback

---

## 验收标准（D/E 后）

- [ ] <验收项 1>
- [ ] <验收项 2>

## 非目标

- <非目标 1>
BODY
)"
```

### Step 2 — 打 Labels

```bash
ISSUE_N=<N>
REPO=<owner>/<repo>

# 必须
gh issue edit $ISSUE_N --repo $REPO \
  --add-label "feishu-inbound" \
  --add-label "analyzed" \
  --add-label "<type>"  # enhancement | bug | documentation | refactor | chore

# 按难度选一（必须）
gh issue edit $ISSUE_N --repo $REPO --add-label "difficulty-trivial"   # 配置/文案/极简改动
# 或
gh issue edit $ISSUE_N --repo $REPO --add-label "difficulty-standard"  # 默认；≤8 文件
# 或
gh issue edit $ISSUE_N --repo $REPO --add-label "difficulty-complex"   # 跨模块/架构决策

# 可选：Human 审核 Analysis 后再加（trivial 可直接加）
gh issue edit $ISSUE_N --repo $REPO --add-label "approved-to-execute"

# 指派（确保 D 能扫到）
gh issue edit $ISSUE_N --repo $REPO --add-assignee "${GITHUB_ASSIGNEE:-369795172}"
```

### Step 3 — 写 Analysis Comment（首条 comment，非 body）

```bash
gh issue comment $ISSUE_N --repo $REPO --body "$(cat <<'COMMENT'
## Feishu Inbound Analysis

### 1. 需求概述
（2-3 句复述 Issue 事实）

### 2. 去重判断
（完全重复 / 部分相关 / 无重复 + 理由；`grep` 或阅读 codebase 后判定）

### 3. 影响模块（Evidence）
- **平台/Surface**: <core | tools | config | ...>
- **环境**: local（操作者本机）
- `path/to/file.py:123` — 说明

### 4. 问题类别与根因（Evidence）
- **问题类别**: code defect（能力缺失）| enhancement | configuration/deployment | refactor
- **操作/使用排除**: 已排除 / N/A
- **数据/状态排查**: N/A（或具体说明）
- **代码结论**: 基于已读代码的技术结论

### 5. 推荐方案（唯一）
1. 步骤一 …
2. 步骤二 …
**改动文件**: `path/a.py`, `path/b.yaml`
**文档/API**: 无 API 变更，无需更新 Swagger（或具体路径）
**App/版本**: 无 App 构建，无需 bump 版本（或具体路径）
**为何不用其他方案**: 一句说明

### 6. 执行路径

**Config 对照表**（Pipeline D 必须匹配 issue 所在 repo，勿照抄单一 config）：

| Issue repo | Config | Surface surfaces 文件 |
|------------|--------|----------------------|
| `369795172/rootgrove` | `config/feishu_inbound_rootgrove.yaml` | `feishu_inbound_rootgrove_surfaces.yaml` |
| `369795172/feishu-inbound-skill` | `config/feishu_inbound_engine.yaml` | `feishu_inbound_engine_surfaces.yaml` |
| `369795172/qmt_longterm` | `config/feishu_inbound_personal.yaml` | `feishu_inbound_personal_surfaces.yaml` |
| `369795172/itias-coder` | `config/feishu_inbound_itias-coder.yaml` | `feishu_inbound_itias-coder_surfaces.yaml` |

- **Worktree**: 见上表 config 的 `pipeline.worktree`（single-repo）或 surfaces 条目
- **Surface**: Analysis §4 声明的 surface（`default` / `engine` / …）
- **分支**: `issue-{N}`（single-repo 模式）
- **提 PR**: `./venv/bin/python tools/feishu_inbound_instance/smart_pr.py --issue {N} --surface <surface> --issue-repo <owner>/<repo>`
- **Pipeline D 手动触发**（Human 加 `approved-to-execute` 后）:
  ```bash
  source tools/feishu_inbound/load_personal_env.sh
  export GITHUB_ASSIGNEE=369795172
  ./venv/bin/python -m feishu_inbound.cli execute \
    --config config/feishu_inbound_<instance>.yaml \
    --issue {N} --repo <owner>/<repo>
  ```
- **Base / Reviewer**: `main`；见对应 surfaces 文件的 `default_reviewers`

### 7. Scope
<S/M/L> — 一句依据（文件数、跨模块度）

### 8. 三角分工
| 角色 | 本 issue 产出 |
|------|---------------|
| Human | 审核本节推荐方案（一次 Gate）；加 `approved-to-execute`；验收 PR |
| Agent | worktree 实现 + Smart PR |
| Script | CI / smoke test（如有）|

### 待确认（产品）
无
COMMENT
)"
```

---

## Label 规则速查

| Label | 何时加 | 谁加 |
|-------|--------|------|
| `analyzed` | 写完 Analysis comment 后立即加 | Agent |
| `difficulty-trivial` | 配置/文案/极简 1 文件改动 | Agent |
| `difficulty-standard` | 默认；≤8 文件，有明确方案 | Agent |
| `difficulty-complex` | 跨模块 / 架构决策 / 迁移 | Agent |
| `approved-to-execute` | Human 审核 Analysis 后（trivial 可由 Agent 直接加） | Human（trivial 可 Agent） |
| `feishu-inbound` | **必须加**（看板/巡检/routing 成员标记；跳过的是 Bitable 入站，非 label）| Agent |
| `triaged` | **不加**（跳过 Pipeline B）| — |

**`approved-to-execute` 加的时机**：
- `difficulty-trivial`：Agent 可在写 Analysis 后直接加（无需等 Human review）
- `difficulty-standard`：Human 审核 Analysis，同意后加
- `difficulty-complex`：Human 审核且确认架构决策后加；建议 Agent 在 comment 里主动 @

---

## D 自动扫描条件

`issue_executor.py` 扫描：`state=open` + `assignee = GITHUB_ASSIGNEE` + `analyzed` label。
通过后检查 Gate：`difficulty-trivial` 直接执行；`standard/complex` 需 `approved-to-execute`。

---

## 相关

- [Feishu Inbound Pipeline（顶层编排）](./pipeline.md) — A–F 全流程
- [Stage C: Feishu Inbound Agent](./agent.md) — Analysis 8 章契约（ASP 版，本 workflow 适配自该契约）
- [Stage D: Feishu Inbound Executor](./executor.md) — 自动执行 + Smart PR
- [Smart PR Workflow](./workflow_smart_pr.md)
