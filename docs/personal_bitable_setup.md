# Personal Bitable Instance Setup

Personal 飞书需求池（single-repo）运维指南。引擎 SSOT 见 [RFC-001 §4](./rfc/RFC-001-engine-instance-architecture.md) 与 [§4.2 umbrella 调度](./rfc/RFC-001-engine-instance-architecture.md#42-personal-umbrella-调度多-single-repo-config方案-c)。

## Umbrella 调度（方案 C，#37）

多个 personal single-repo 项目共用 **一个** launchd tick；runner 自动发现 config，失败隔离。

```
com.personal.feishu-inbound-lead-tick (:25/:55)
  → run_personal_lead_tick.sh
  → umbrella_configs.py
  → lead-tick --config personal.yaml
  → lead-tick --config rootgrove.yaml
  → lead-tick --config <new>.yaml   # umbrella_runner: true
```

| 类型 | 示例 | launchd |
|------|------|---------|
| Umbrella 成员 | QMT (`personal.yaml`)、rootgrove harness | 共用 umbrella tick |
| Standalone | GZUCM `:15/:45`、OpenCode `:05/:35` | 自有 tick，不被 umbrella 扫描 |

接入新项目（rootgrove）：

```bash
./venv/bin/python tools/feishu_inbound/new_instance.py \
    --slug ico --repo 369795172/ico --worktree projects/ico
# 无需 install_launchd — 下一轮 umbrella tick 自动纳入
```

## 推荐 launchd 组合

| Instance 类型 | 默认 launchd | Pipeline B (triage) |
|---------------|--------------|---------------------|
| Personal single-repo | `lead_tick` only（C→D→E→F） | **可选** — 手动 `triage` CLI 补 scope/difficulty |
| ASP multi-surface | `feishu_inbound_triage` + `lead_tick` | **必需** — keyword surface 分诊 |

Personal instance 安装示例（rootgrove）：

```bash
bash tools/feishu_inbound/install_personal_launchd.sh
# 等价于引擎 CLI：
# scripts/run_cli.sh install-launchd --config config/feishu_inbound_personal.yaml --jobs lead_tick
```

`config/feishu_inbound_*.yaml` 中可定义 `feishu_inbound_triage` schedule，但 personal 默认**不安装** triage plist。

## E2E 路径（#13 对齐）

```
飞书 Bitable (待处理)
     │ Pipeline A (GitHub Actions)
     ▼
Target repo Issue (feishu-inbound)
     │ lead_tick (:20/:50)
     ├─ C 深度分析 → analyzed
     ├─ D 执行 + Smart PR → executed
     ├─ E Gate Review → review-dev-pass
     └─ F Dev Handback → 验收指派
```

Pipeline B **非阻塞**：C 扫描 `open + assignee`，不要求 `triaged` label；缺 `difficulty-*` 时降级为 `standard`。

可选：手动跑 B 补 scope/difficulty 标注：

```bash
./venv/bin/python tools/feishu_inbound/triage_agent.py \
  --config config/feishu_inbound_personal.yaml --issue <N>
```

## Phase 3 checklist（#16 对齐）

- [ ] Pipeline A：Bitable → target repo issue
- [ ] `lead_tick` launchd 已安装且日志正常
- [ ] C 分析后 Routing 段显示 `N/A（single-repo…）` 而非「未检测到」
- [ ] D Smart PR + E gate + F handback 全链路可走通
- [ ] （可选）手动 triage 可补 `scope-*` / `difficulty-*` label

## Personal 看板同步（方案 A，已落地）

决策记录：[issue #16 comment](https://github.com/369795172/feishu-inbound-skill/issues/16#issuecomment-4921711444)。

实现（rootgrove instance，非引擎包）：

- `tools/feishu_inbound/board_stage.py` — label → 流水线阶段
- `tools/feishu_inbound/board_sync.py` — 本机批量回写 + A′（`需求状态=已完成` → accept comment + `dev-accepted` + close）
- `tools/feishu_inbound/run_board_sync.sh` — 加载 Keychain 后执行
- lead-tick 挂载：`run_personal_lead_tick.sh`、`run_gzucm_lead_tick.sh`

Bitable：需求池表字段「流水线阶段」（SingleSelect）。首次：

```bash
bash tools/feishu_inbound/run_board_sync.sh --ensure-field
```

飞书侧按该字段建看板视图。ASP 保持现有 GHA sync，不改。

## 相关

- [feishu-inbound-skill#37](https://github.com/369795172/feishu-inbound-skill/issues/37) — Personal umbrella 调度（方案 C）
- [feishu-inbound-skill#16](https://github.com/369795172/feishu-inbound-skill/issues/16) — Personal Bitable Epic
- [feishu-inbound-skill#25](https://github.com/369795172/feishu-inbound-skill/issues/25) — single-repo Pipeline B 策略
- [feishu-inbound-skill#13](https://github.com/369795172/feishu-inbound-skill/issues/13) — QMT E2E 验收
