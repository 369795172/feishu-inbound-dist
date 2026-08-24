# RFC-001 — 引擎 / Instance 架构与迁移计划

- 状态：Accepted（2026-06-10 会话决策）
- 决策人：Marvin
- 关联：PRD.md；asp-infra `tools/feishu_inbound/`；rootgrove `adhoc/20260603_feishu_inbound_workflow_analysis.md`

## Amendment 2026-06-16 — A-light hybrid

**Why A-light hybrid, not「继续在 asp-infra 演化 B–F」**：asp-infra 是租户仓；在那里长核心逻辑会让第二个 instance 只能再复制一份，直接违背 PRD 零分叉。A-light 把 B–F 收到本仓，Pipeline A 仍留在 instance GHA（飞书→Issue 的凭证与 org 边界不同）。被否的「把 A 也收进引擎」会把公司 Bitable token 与个人 token 的生命周期耦进同一发布面。

**裁决**：采用 A-light hybrid，不再在 asp-infra 演化 B–F 核心逻辑。

| 层 | 归属 |
|----|------|
| 引擎 B–F | `369795172/feishu-inbound-skill`（唯一 SSOT） |
| ASP instance | `AI-MYG/asp`：config、secrets、launchd、薄 wrapper |
| 日常发现 | rootgrove HOT skill + vendored 契约（单跳，非指针层） |
| Pipeline A | 公司飞书 Bitable → GitHub Issue（不变） |

**禁止**：asp-infra `tools/feishu_inbound/` 新增业务逻辑（仅 wrapper + config）。

**迁移**：保留 RFC §6 的 tag pin + shadow diff + wrapper 切换安全门；跳过仪式化多阶段文档负担。

**launchd**：消灭 `com.rootgrove.feishu-inbound-*` 与 `com.asp.*` 双装；B 后置迁 Mac Mini。

## Amendment 2026-06-17 — Instance runtime layout (`var/feishu_inbound`)

**Why instance `var/feishu_inbound/`, not surface repo, not git**：checkpoint 是租户运行时（扫到哪、锁了哪）。放进 asp-backend 工作树会让 surface PR 带着引擎状态；放进本仓 git 会把 ASP 与 Personal 的游标写成同一份假 SSOT。相对 **config 文件**解析路径，是为了否决「相对进程 cwd」（launchd 与人工 CLI 的 cwd 不同）。

**裁决**：Pipeline B–F 的 checkpoint / engine log 属于 **instance repo**，不进 git；**不属于 surface repo**（如 asp-backend）。

| Instance | Runtime root | 说明 |
|----------|--------------|------|
| ASP (`AI-MYG/asp`) | `var/feishu_inbound/state/` + `var/feishu_inbound/logs/` | 相对 instance `config.yaml`；`.gitignore` 含 `var/` |
| Personal (rootgrove) | 同上，在 monorepo 根 `var/feishu_inbound/` | `config/feishu_inbound_personal.yaml` |

**引擎**：`state_dir`、`logs_dir`、`worktree_dir` 等与 `logs_dir` 一致，一律相对 **config 文件目录** 解析（`EngineConfig.resolve_path`），禁止依赖进程 cwd。

**落档**：asp-infra `docs/feishu_inbound_instance_runtime_layout.md`；rootgrove 本地副本 `docs/tasks/asp/20260617_feishu_inbound_instance_runtime_layout.md`（`docs/` gitignored）

## 1. 决策摘要

| 决策点 | 结论 | 备选与否决理由 |
|--------|------|---------------|
| 打包维度 | 独立通用 skill repo（本 repo） | rootgrove monorepo 维度：私人仓库无法分发给团队；且正是 06-03 竞态事故的肇因维度 |
| 归属 | `369795172/feishu-inbound-skill`（个人账号） | AI-MYG org：引擎是个人资产，ASP 只是租户 |
| 消费方式 | 立项：`pip install from git tag`；现行：[PACKAGING.md](../PACKAGING.md) GitHub Release wheel | git submodule：团队成员摩擦大；vendored sync：重新制造分叉；GitHub Packages：不托管 Python |
| 引擎语言形态 | `src/feishu_inbound/` Python 包 + console entrypoints | 保持与 asp-infra 现有脚本逻辑同构，降低迁移 diff |

### 1.1 Why X, not Y（Gate B）

#### Why 独立 skill repo, not rootgrove monorepo？

rootgrove 是私人工作区，ASP 团队无法作为一等公民消费。2026-06-03 的竞态来自「文档指针 vs 另一份脚本正本」同属 monorepo 分叉维度。独立 repo 让引擎有自己的 tag / Release，instance 只 pin 版本。

#### Why 个人账号仓, not AI-MYG org？

引擎是个人资产；ASP 是租户。放到公司 org 会把 Personal / 其他 side-project instance 的发布权绑进 ASP 治理。被否方案的前提（「引擎只服务 ASP」）已经在第二个消费者出现时失效。

#### Why Release wheel, not GitHub Packages / git+https / submodule / vendored copy？

GitHub Packages 不托管 Python（roadmap 已关，`pypi.pkg.github.com` 404）。`git+https` 与跟踪 `main` 让 instance 随引擎 HEAD 漂移。submodule 增加团队 clone 摩擦。vendored sync 再造分叉。立项写过 git tag 安装；现行 SSOT 是 dual GitHub Release wheel（private 源仓 + public `feishu-inbound-dist`），见 PACKAGING.md。

#### Why Python 包 + 现有脚本同构, not 换语言 / 重写流水线？

迁移成本由 diff 大小决定。换语言或重写 label 语义会让 M2 影子对比无法「决策输出 diff 为零」。被否的是「趁抽取一起现代化」。

#### Why AgentClient 协议注入, not 引擎绑死 OpenCode/Cursor？

ASP 与 Personal 的 executor 链不同。引擎只定义 `run(prompt, workdir, model?)`。绑死单一平台会把第二个 instance 的工具偏好做成引擎版本问题。

**决策考古缺口**：AgentClient 链在「迁移期 import instance 实现」vs「通用链收进引擎」之间，Phase 2 后期未形成带日期的否案；从 #111 日起新决策按 Why X not Y 记。

## 2. 切面划分

**引擎（本 repo）**：

- `scanner/`：issue 扫描（org/repo/repos 三模式 + single-repo）、双门卫幂等决策
- `triage/`：关键词路由、difficulty/scope 分级、assignee 路由（multi-surface）；single-repo 退化路径
- `executor/`：worktree 生命周期（含 stale 分支清理）、AgentClient 调用、Smart PR 接缝
- `reviewer/`：Pipeline E 跨平台 review 路由池
- `config.py`：schema 定义 + fail-fast 校验 + secrets 前缀解析
- `launchd.py`：plist 模板生成与 install/uninstall
- `contracts/`：label 契约、comment marker、prompt 模板（instance 可在 config 中 override prompt 的领域上下文段）

**Instance（asp-infra / rootgrove）**：

- `config.yaml`（业务路由、Bitable 字段映射、调度表、executor 链）
- secrets（Keychain，带前缀）
- `var/feishu_inbound/state/`（Pipeline B–F checkpoint JSON；gitignored）
- `var/feishu_inbound/logs/`（引擎子命令日志；gitignored）
- Pipeline A 的 GitHub Action workflow（从引擎模板复制）
- 领域 prompt 上下文（如 ASP 的代码库导读段）

## 3. Secrets 命名

```
rootgrove/FI_ASP_FEISHU_APP_ID          # ASP instance
rootgrove/FI_ASP_FEISHU_BITABLE_APP_TOKEN
rootgrove/FI_PERSONAL_FEISHU_APP_ID     # Personal instance
rootgrove/FI_PERSONAL_FEISHU_BITABLE_APP_TOKEN
```

引擎读取顺序：`{prefix}{KEY}` → 无前缀 `{KEY}`（存量兼容）→ 报错指明缺失项。
团队成员机器沿用 asp-infra 现约定（Keychain namespace `rootgrove/`，由 `load_secrets.sh` 加载）。

## 4. Single-repo mode

```yaml
pipeline:
  mode: single-repo        # 默认 multi-surface
  repo: 369795172/qmt
```

- Stage A：Bitable 行 → 直接在 `repo` 建 issue（label `feishu-inbound`）
- Stage B：只做 difficulty/scope 标注；不路由 surface、不 re-assign（assignee = config 默认人）。**Personal single-repo instance 默认不安装 Pipeline B launchd**（`lead_tick` 链 C→F 已足够）；可通过 CLI `triage` 手动跑 scope/difficulty，非阻塞。Multi-surface ASP instance 仍须 B launchd + keyword routing。
- **Launchd（personal）**：推荐仅 `lead_tick`（A→C→D→E→F）；`feishu_inbound_triage` schedule 可定义但不默认安装。

### 4.2 Personal umbrella 调度（多 single-repo config，方案 C）

同一组 `FI_PERSONAL_*` 凭证下可有**多个** single-repo config（QMT、rootgrove harness、未来新项目）。**不**为每个 config 再装 launchd tick。

| 概念 | 说明 |
|------|------|
| **Umbrella tick** | 唯一 launchd：`com.personal.feishu-inbound-lead-tick`（如 `:25/:55`） |
| **Umbrella runner** | instance 脚本（rootgrove `run_personal_lead_tick.sh`）调用 `umbrella_configs.py` 发现 config 列表，**逐 config** 执行 `lead-tick` |
| **失败隔离** | 某一 config 的 lead-tick 非零退出**不**中断其余 config；runner 对 launchd **exit 0**（失败写日志） |
| **Standalone instance** | GZUCM / OpenCode Android 等自有 `launchd_labels.lead_tick`（非 umbrella label）→ **排除**于 umbrella 发现 |

**纳入规则**（instance `umbrella_configs.py`，引擎仅文档化）：

1. `umbrella_runner: true` → 纳入
2. `umbrella_runner: false` → 排除
3. `launchd_labels.lead_tick == com.personal.feishu-inbound-lead-tick` → 纳入（QMT / `feishu_inbound_personal.yaml`）
4. 另有独立 `launchd_labels.lead_tick` → 排除（standalone）
5. 无 `launchd_labels.lead_tick` → 纳入（如 rootgrove harness config）

**接入新项目**：instance `new_instance.py` 默认写 `umbrella_runner: true`，**无需**新 launchd；仅需 `config/feishu_inbound_<slug>.yaml` + surfaces。

**Why 方案 C（umbrella tick）, not A/B**：

- **A**（每项目一只 launchd）：plist 与错点时刻随项目数线性膨胀；失败面是「漏装一只 tick」。
- **B**（runner 硬编码路径 + `set -e`）：扩项目必须改 shell；一个 config 非零退出会拖死整组。
- **C**：发现规则在 instance `umbrella_configs.py`，失败隔离且 runner 对 launchd exit 0。引擎只文档化规则，不把 personal 路径写进 `src/`。

Instance SSOT 实现：rootgrove `tools/feishu_inbound/umbrella_configs.py`（见 [feishu-inbound-skill#37](https://github.com/369795172/feishu-inbound-skill/issues/37)）。

- Stage C：`primary_surface` 取 instance `surfaces.yaml` 的首个（或唯一）key（如 `default`），**禁止** fallback 到 ASP `"backend"`；Analysis 分支写 `issue-{N}`（无 surface 后缀）
- Stage D：单 worktree（`repo` 本身），分支 `issue-{N}`；`base_branch` 读 `surfaces.yaml` 中该 surface 的 `base_branch`（**禁止**全局 `backend→dev` 硬编码）；从旧 Analysis 解析 surface 时 single-repo 以 config key 为准
- Stage E：路由池照常；审查须包含「文档是否已更新」（见下 §4.1）

### 4.1 文档与 API 契约（C/D/E 共用）

凡改动 HTTP API（路由、请求/响应、错误码、鉴权行为）：

1. **Pipeline C** 在「推荐方案」与「改动文件」中**必须**列出 Swagger/OpenAPI（或 surface AGENTS.md 声明的 API 文档 SSOT）及同步步骤；无 API 变更则显式写「无需更新 Swagger」。
2. **Pipeline D** 执行 Agent **必须**在同一 PR 内完成文档更新，使对外 API 描述与实现一致。
3. **Pipeline E** Gate Review **必须**审查文档维度；代码符合需求但缺少必需的 API 文档更新 → **打回**（blocking）。

## 5. AgentClient 接缝

引擎定义 `AgentClientProtocol`（`run(prompt, workdir, model?) -> result`），instance config 声明实现链：

```yaml
agent_client:
  chain:
    - {executor: cursor_sdk, model: composer-2.5}
    - {executor: cursor_agent, model: composer-2.5}
    - {executor: opencode, model: glm-5.1}
```

迁移期允许直接 import 现有实现（asp-infra `tools/agent_client.py`、rootgrove `tools/agent_clients/`），Phase 2 后期再决定是否将通用链实现也收进引擎。

## 6. 迁移计划（灰度，com.asp.* 不中断）

1. **M1 平移**：把 asp-infra `tools/feishu_inbound/*.py` 平移进 `src/feishu_inbound/`，硬编码（`AI-MYG`、`ASP_WORKTREE_ROOT` 默认值、prompt 中 ASP 上下文）逐一上提到 config schema。tests 同步。
2. **M2 并行验证**：asp-infra venv 安装引擎包，新增 `--engine` 开关的影子运行（dry-run 对比 label/comment 决策输出 与存量脚本 diff）。连续 3 天 diff 干净为通过。
3. **M3 切换**：asp-infra 脚本改为薄 wrapper（import 引擎 + 本地 config），launchd 不动（runner 路径不变）。打 tag `v0.1.0`，asp-infra pin。
4. **M4 清理**：删除 asp-infra 内被引擎取代的逻辑代码；rootgrove `tools/feishu_inbound/` symlink 改为 Personal instance 的 config + wrapper。
5. **M5 Personal 接入**：Phase 3，见 PRD §7。

回滚：M3 的 wrapper 切换是单 commit，revert 即回到存量脚本；tag pin 保证引擎升级可回退。

## 7. 风险

| 风险 | 缓解 |
|------|------|
| 迁移期间生产流水线中断 | M2 影子运行 + M3 单 commit 切换 + launchd 路径不变 |
| prompt 上提 config 后行为漂移 | M2 用真实 issue dry-run diff 验证 prompt 渲染结果逐字节一致 |
| 个人/公司 secrets 误用 | 前缀强制 + 引擎启动时打印所用前缀（不打印值） |
| 引擎与 instance 版本错配 | tag pin + 引擎 `__version__` 写入每条 Analysis comment 尾部 |
