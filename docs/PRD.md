# PRD — 通用飞书入站流水线引擎

> 2026-06-10 立项。决策来源：rootgrove 会话讨论（repo 维度 vs monorepo 维度），结论为「独立通用 skill repo + 两个 instance」。

## 1. 背景与动机

- asp-infra（`AI-MYG/asp`）的 `tools/feishu_inbound/` 已实现完整的飞书需求池入站流水线（Pipeline A–E），但与 ASP 强耦合（`AI-MYG` org、surface 路由、公司 Bitable 字段映射散布在代码与 config 中）。
- 第二个消费者出现：**QMT 等个人项目**需要接入 Marvin 自己的飞书多维表格需求池——通用抽取的时机成立（N=2，符合「出现第二个消费场景才升级」触发器）。
- 历史教训（2026-06-03 根因分析）：rootgrove 旧分叉副本与 asp-infra 正本并存导致竞态、「文档说的 ≠ 跑的」。**单一 SSOT 是本项目的第一需求。**

## 2. 目标

1. 引擎与 instance 彻底分离：流水线逻辑（扫描、双门卫幂等、label 契约、worktree+Smart PR、review 路由池）收敛进 pip 包 `feishu_inbound`，零硬编码。
2. ASP instance 平滑迁移：asp-infra 改为 `pip install from git tag` 消费引擎，保留自己的 config/launchd/state；**生产流水线（com.asp.* 每 30 分钟）迁移期间不中断**。
3. Personal instance 落地：QMT 作为首消费者，接入个人 Bitable 需求池。

## 3. 非目标

- 不重写 Pipeline 语义：label 契约、双门卫幂等、宽进严出原则原样继承。
- 不做 Web UI / 多人 SaaS。instance = 文件配置 + 本机 launchd。
- 不在本 repo 存任何 instance 的 secrets 或业务配置。

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

## 6. 验收标准

1. **ASP 回归**：迁移后 B/C/D/E 在 ASP instance 上行为与迁移前一致（同一批 issue 的 label/comment 输出 diff 为零或仅时间戳差异）；`com.asp.*` 调度无中断窗口。
2. **Personal 端到端**：个人 Bitable 建一条真实需求 → qmt repo 自动建 issue → triage 标注 → analysis comment → （审批后）执行 + PR，全链路走通一单。
3. **零分叉**：rootgrove 与 asp-infra 均不再持有引擎逻辑副本（仅 config + 薄 wrapper/symlink）。
4. 引擎 tests 覆盖：config schema 校验、双门卫幂等决策表、single-repo mode 路由、secrets 前缀解析。

## 7. 里程碑

| Phase | 交付物 | 追踪 |
|-------|--------|------|
| 1 Scaffold | 本 repo + PRD + RFC + Issues 立项 | 本文档 |
| 2 引擎抽取 | pip 包 + ASP instance 灰度切换 | Issues `phase-2` |
| 3 Personal instance | QMT 接入 + 端到端验证 | Issues `phase-3` |
