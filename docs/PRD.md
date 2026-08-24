# PRD — 通用飞书入站流水线引擎

> 2026-06-10 立项。决策来源：rootgrove 会话讨论（repo 维度 vs monorepo 维度），结论为「独立通用 skill repo + 两个 instance」。
> 2026-08-21 scaffold v2 retrofit（#111）：补用户问题、可勾选成功标准、约束三形式。非目标为硬边界。Gate A 数值来自本文与 RFC-001 / PACKAGING 既有阈值，不新造产品数字。

## 用户问题

在飞书需求池已经能驱动 GitHub Issue 的前提下，**第二个真实消费者出现时**，如何让 ASP 团队与个人项目共用同一套入站流水线，而不再各持一份会竞态的引擎副本？

## 1. 背景与动机

- asp-infra（`AI-MYG/asp`）的 `tools/feishu_inbound/` 已实现完整的飞书需求池入站流水线（Pipeline A–E），但与 ASP 强耦合（`AI-MYG` org、surface 路由、公司 Bitable 字段映射散布在代码与 config 中）。
- 第二个消费者出现：**QMT 等个人项目**需要接入 Marvin 自己的飞书多维表格需求池——通用抽取的时机成立（N=2，符合「出现第二个消费场景才升级」触发器）。
- 历史教训（2026-06-03 根因分析）：rootgrove 旧分叉副本与 asp-infra 正本并存导致竞态、「文档说的 ≠ 跑的」。**单一 SSOT 是本项目的第一需求。**

## 2. 目标

1. 引擎与 instance 彻底分离：流水线逻辑（扫描、双门卫幂等、label 契约、worktree+Smart PR、review 路由池）收敛进 pip 包 `feishu_inbound`，零硬编码。
2. ASP instance 平滑迁移：asp-infra 消费 **GitHub Release wheel**（`feishu-inbound==X.Y.Z`；立项时写过 `pip install from git tag`，已被 [PACKAGING.md](PACKAGING.md) 取代），保留自己的 config/launchd/state；**生产流水线（`com.asp.*` 每 30 分钟）迁移期间不中断**。
3. Personal instance 落地：QMT 作为首消费者，接入个人 Bitable 需求池。

## 3. 非目标（硬边界，不是 future work）

砍错任一条都会回到 2026-06-03 的分叉竞态，或把本仓变成第二个 asp-infra。

- **不重写 Pipeline 语义**：label 契约、双门卫幂等、宽进严出原样继承。要改契约先写 RFC 否案，不在实现 PR 里默默改。
- **不做 Web UI / 多人 SaaS**。instance = 文件配置 + 本机 launchd。失败时停在本机日志与 GitHub comment，不「升级」成托管服务。
- **不在本 repo 存任何 instance 的 secrets 或业务配置**（org/repo、Bitable 字段、assignee、launchd label）。
- **不把引擎 checkpoint / 引擎日志写入 surface repo**（如 asp-backend）。运行时根是 instance 的 `var/feishu_inbound/`（RFC-001 Amendment 2026-06-17）。
- **不用 GitHub Packages 发 Python，不用 `git+https` pin，不让 instance 跟踪引擎 `main`**。安装面只有 Release wheel。
- **不在引擎仓做 GUI**。无 `docs/design.md`。

## 4. 用户与场景

| Instance | 用户 | 需求池 | Issue 落点 | 分诊形态 |
|----------|------|--------|-----------|---------|
| ASP | ASP 团队（Marvin、胡剑飞等，clone asp-infra） | 公司 Bitable | 中央 repo `AI-MYG/asp` → surface repos | 多 surface 关键词路由 + assignee 路由 |
| Personal | Marvin 单人 | 个人 Bitable | 目标项目 repo 直建（首个：qmt） | **single-repo mode**：跳过 surface 分诊，保留 difficulty/scope 标注；assignee 恒为 Marvin |

## 5. 关键需求

### 5.1 配置切面（instance 持有）

- `config.yaml`：org/repo、扫描范围（org/repo/repos 三模式，继承现有 `pipeline_cd_scan`）、Bitable 字段映射、surface/assignee 路由（single-repo mode 下可省略）、launchd 调度表、Pipeline E 路由池。
- config schema 由引擎定义并校验（启动时 fail-fast，错误信息指明缺失字段）。

### 5.2 Secrets 前缀隔离

- 现有 Keychain key（如 `rootgrove/FEISHU_BITABLE_APP_TOKEN`）为全局命名，两 instance 会撞名。
- 引擎支持 env 前缀：config 声明 `secrets_prefix`（如 `FI_ASP_` / `FI_PERSONAL_`），引擎按前缀读取；无前缀 key 作为回退以兼容存量。

### 5.3 Single-repo mode

- `pipeline: mode: single-repo` 时：Pipeline A 直接在目标 repo 建 issue；Pipeline B 退化为 difficulty + scope 标注（无 surface 路由、无 re-assign）；Pipeline D 在目标 repo 单 worktree 执行。

### 5.4 AgentClient 注入

- 引擎不绑定具体 agent 平台。instance config 声明 executor 链（如 ASP：OpenCode 为主；Personal：cursor_sdk → cursor_agent → opencode）。
- 接口对齐 rootgrove `tools/agent_clients` 与 asp-infra `tools/agent_client` 的最大公约数。

### 5.5 launchd 模板生成

- 引擎提供 `feishu-inbound install-launchd --config <path>` 从 config 的调度表生成并加载 plist；label 前缀来自 config（`com.asp.*` / `com.personal.*`），杜绝双装竞态。

## 6. 验收标准（可勾选）

阈值与验证方式取自立项原文与 RFC-001 M2 / 现有 pytest，不使用「足够多」「较好」。

- [ ] **ASP 回归（迁移门）**：同一批 open issue，引擎 dry-run 与对照 wrapper 的 label/comment **决策输出** diff 为零，或仅时间戳字段不同。测量点：`feishu-inbound shadow-compare`。适用域：ASP instance、迁移窗口。阈值：RFC-001 M2 规定 **连续 3 个日历日** diff 干净。验证：关对照面后 `com.asp.*` 仍按既有间隔触发（立项：每 30 分钟），迁移切换当天无「停掉 launchd 再装」的中断窗口。
- [ ] **Personal 端到端（一单）**：个人 Bitable **新建 1 行**真实需求 → 目标 repo（首消费者 qmt）出现带 `feishu-inbound` 的 Issue → Analysis comment `## Feishu Inbound Analysis` → 人工 `approved-to-execute` 后出现 `executed` + PR。验证：对该 issue 号检查 labels 与 comment markers，不靠口头确认。
- [ ] **零分叉**：`rg -n "def lead_tick|class EngineConfig" projects/asp-infra/tools/feishu_inbound rootgrove/tools/feishu_inbound` 不得命中引擎逻辑副本（仅 config + 薄 wrapper）。验证：在 instance 仓对 wrapper 目录做上述扫描为零逻辑文件。
- [ ] **引擎测试覆盖**：`.venv/bin/pytest -q` exit 0，且至少包含 config schema 校验、双门卫幂等、single-repo 路由、secrets 前缀解析（见 [test.md](test.md) 第 1 层）。适用域：无网络、无真实凭证。
- [ ] **config fail-fast**：缺必填字段或 secrets 前缀与无前缀均缺失时进程非零退出，错误信息指出缺失字段/候选 key，**不打印 secret 值**。验证：`tests/test_config_schema.py`。
- [ ] **版本钉死可安装**：instance requirements 为单一行 `feishu-inbound==X.Y.Z`（X.Y.Z 与引擎 tag 一致）；`FEISHU_INBOUND_INSTALL=package` 时从 Release wheel 安装成功。验证：[PACKAGING.md](PACKAGING.md) 的 public dist URL 或 private Release 回退。

## 6.1 约束（落地三形式）

约束词本身没有裁决力。本项目约束按三种形式之一书写。

1. **数字 + 测量点 + 适用域**
   - Python **3.10+**（`pyproject.toml` `requires-python`）。适用域：引擎包与 CI（`pr-test.yml` 用 3.11）。
   - ASP launchd 间隔 **30 分钟**（立项目标 §2.2）。适用域：`com.asp.*` 生产调度，不是 personal umbrella。
   - M2 影子对比 **3 个日历日**决策 diff 干净。适用域：ASP 迁移，不是日常 PR。
   - Pipeline E 连续 `gate-infra-failure` 上限默认 **3** 次后 `skip_infra_exhausted`。测量点：`needs_review` 返回值。适用域：AI gate，不是业务 accept-fail。
   - L2 bot review 等待默认 **600 秒**。适用域：PR 创建后、E 消费 Copilot 行内评论前。
2. **测试程序**
   - 离线正确性：`.venv/bin/pytest -q`（关网络仍须过；CI 无飞书/GitHub token）。
   - 迁移行为：`shadow-compare`（RFC-001 M2）。
   - 发版：`./scripts/release_engine.sh --dry-run` 后再发版。
3. **排除清单（含失败路径）**
   - 缺 config / 缺 secret → fail-fast 退出，不带着半套默认 org 继续跑。
   - 本地/工具链失败 → `infrastructure_unavailable` / `gate-infra-failure`，不静默 skip，不记入业务 `failure_count`。
   - OpenAPI 与实现不一致（asp-backend）→ D/E 打回；缺工具链按 infra 处理。
   - 见 §3 非目标：无 Web UI、无 Packages、无 `git+https`、无 surface-repo checkpoint。

冲突优先序（人已拍板、写在这里备查）：**生产流水线不中断 > 零分叉 SSOT > 新 instance 接入速度**。迁移期允许影子并行，不允许为了 Personal 接入而改 ASP label 语义。

## 7. 里程碑

| Phase | 交付物 | 追踪 |
|-------|--------|------|
| 1 Scaffold | 本 repo + PRD + RFC + Issues 立项 | 本文档 |
| 2 引擎抽取 | pip 包 + ASP instance 灰度切换 | Issues `phase-2` |
| 3 Personal instance | QMT 接入 + 端到端验证 | Issues `phase-3` |
