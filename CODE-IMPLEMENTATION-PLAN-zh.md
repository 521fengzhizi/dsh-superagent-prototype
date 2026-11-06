# dsh-superagent 源码级改造与可执行实施方案（中文）

> **目标仓库：** [`521fengzhizi/dsh-superagent-prototype`](https://github.com/521fengzhizi/dsh-superagent-prototype)  
> **建议文件名：** `CODE-IMPLEMENTATION-PLAN-zh.md`  
> **方案基线：** 仓库 `main`，本方案参考 `EXECUTION-PLAN-zh.md`、`docs/006-deployment.md` 及当前 `src/` 结构。  
> **性质：** 这是供开发者按阶段实施的源码改造方案，不表示下述改动已经编码、测试或部署。

## 0. 先说结论

不建议重写现有 Engine、Policy 或 Executor。仓库已有 Source → Engine → Policy/Eval → Ticket/Assignment → Worker → Outbox 的完整原型闭环；应当沿现有边界做**纵向加固**：

1. 把“受控试点”的限制变成可验证的代码约束，而非只写在部署文档里。
2. 将一次候选从 Source 收取到 Ticket 决策的审计信息做成可查询、可测试的记录。
3. 把“工作区已存在就视为准备完成”改成可验证的工作区状态，减少错误目录复用。
4. 加强配置变更、暂停/恢复、重试、数据库升级的回归测试和运维可见性。
5. 明确代码内的命令包装器不是安全沙箱；权限隔离仍由专用 OS 账户、容器/虚拟机或 DSH 提供的真正沙箱承担。

**首轮代码目标：** 保持默认 `observe`，GitHub 单仓库、Worker 并发 1、PR 人工审查；先交付测试和可观测性，再评估有限 `auto`。不要把自动合并或扩大接收范围放进第一阶段。

---

## 1. 当前实现与改造边界

### 1.1 当前链路（已有能力，不重复造轮子）

```text
src/sources/*
  → SourceHost / Passes
  → src/core/engine.ts: receive / evaluateDue / decide
  → src/policy.ts: settingsPolicy / assessmentOutcome / policyEffects
  → src/app/store.ts + SQLite
  → src/executor/dsh.ts: Ticket workspace / Worker Session
  → src/app/effects.ts + src/app/outbox.ts
  → 管理面板 / GitHub 或 Feishu
```

现有关键事实：

- `src/core/engine.ts` 是域状态写入口，`receive()` 事务调用 Intake；`decide()` 事务调用 Eval；`policyChanged()` 会通知活动 Ticket 并重启相关 Eval。
- `src/policy.ts` 默认 `eval.mode` 为 `observe`；推断型候选只有在 `auto` 且匹配启用规则时才能接受。Policy 版本和 Mandate 恢复校验已存在。
- `src/app/runtime.ts` 负责组合 Store、Engine、Source、Effects、Outbox、Assessor 和 Executor；不要让 Source 插件绕过 Engine 直接写业务表。
- `src/executor/dsh.ts` 为 Ticket 建立 `worker-<ticketId>` Session，支持并发槽位、失败状态、有限瞬时错误重试和恢复。准备命令失败不会自动重试，这是正确的保守行为。
- `docs/006-deployment.md` 指出 Worker preset 为 `danger-full-access`，以操作者 OS 用户执行；CLI shim 仅选择凭据和命令，不构成 OS 级隔离。

### 1.2 本方案不做什么

- 不把 GitHub Source、Feishu Source 合并成一个跨身份授权表；各 Source 继续使用自己的身份。
- 不让 Source 直接创建 Ticket、绕过 `Engine.receive()` 或自己判定最终准入。
- 不把模型输出直接当作授权结论；模型 assessment 仍必须经过 `assessmentOutcome()` 和 Engine 校验。
- 不把“命令黑名单”“目录名检查”描述为安全沙箱。
- 不在本次计划中自动合并 PR、开放任意仓库或默认启用全部 Issue/PR/评论/CI 事件。

---

## 2. P0：先建立改动基线（半天）

### 2.1 固定代码基线并检查工作区

```sh
git checkout main
git pull --ff-only
git rev-parse HEAD
git status --short
node --version
pnpm --version
pnpm install --frozen-lockfile
pnpm check
```

**完成标准：** 记录 SHA；工作树干净；`pnpm check`（typecheck、Vitest、build）通过。失败先修复或登记基线失败，不要把既有失败误判为本轮回归。

### 2.2 梳理现有测试和 schema 迁移

```sh
rg "settingsPolicy|assessmentOutcome|policyEffects|Engine\(|DshExecutor|prepareDirectory" src
rg "schemaVersion|migration|migrate" src
rg "observeIssues|observePulls|observeComments|observeCi|maxWorkers|paused" src
```

记录每项改动对应的：源文件、既有测试文件、持久化 schema 版本、UI/API 设置入口。SQLite schema 如需变更，必须走当前迁移机制，并补新旧数据库测试；不得手工改生产 SQLite 表。

---

## 3. P1：把试点安全门槛下沉到配置和 Policy（1–2 天）

### 3.1 配置层：试点 profile 的显式安全基线

**建议落点：** `src/policy.ts` 的设置 schema / 归一化逻辑，`src/app/settings.ts` 的保存校验，相关管理 API 测试及设置 UI。

目标不是把所有部署锁死，而是让“低风险默认值”可以测试，并要求高风险配置有显式选择与可审计说明。例如新增一组受限试点配置（字段名需与现有 schema 风格统一）：

```ts
// 设计示例：实际提交时应按现有 SettingsSchema / Limits 风格落地。
const PilotConfigSchema = z.object({
  enabled: z.boolean().default(true),
  requireExplicitAutoApproval: z.boolean().default(true),
  maxWorkers: z.number().int().min(1).max(1).default(1),
}).strict()
```

建议的代码行为：

- 新安装继续 `observe`；从 `observe` 改为 `auto` 时，在保存边界强制走显式确认，并将操作者、时间、旧/新 Policy revision 和变更摘要写入审计记录。
- 高风险 Source 开关（如全量评论/CI 观察）变更时，显示影响范围预览；保存前校验仓库 scope 和 allowlist，不接受空仓库或含糊目标。
- 试点配置启用时限制 `maxWorkers = 1`；离开试点模式应是明确的配置变更，而不是悄悄把上限放开。
- 前端限制只改善操作体验；服务端 schema/API 必须重复校验，不能信任面板传入值。

**注意：** 现有 schema 的 `maxWorkers` 上限由 `LIMITS` 控制；如果采用试点开关，需避免与通用配置 schema、设置版本和 UI 保存流程产生互相矛盾的默认值。

### 3.2 Gate 不变量：补全单元与集成测试

不要重写 `settingsPolicy()`。优先针对现有分支补充表驱动测试，至少覆盖：

| 输入场景 | 期望结果 |
|---|---|
| allowlist 用户直接 @Agent，目标 Source/scope 已启用 | explicit，可进入 Eval |
| 不在 allowlist 的用户直接 @Agent | 按 admission 配置拒绝或推断，不得误升为 explicit |
| 未观察的仓库 scope 中出现允许用户消息 | 不因用户在 allowlist 中就越过 scope 限制 |
| 已授权 Ticket 的 follow-up | 按 Ticket 当前 Mandate 和 Policy 处理；不等于给新请求无条件授权 |
| 未列入 bot 列表的自动化账号 | 拒绝 |
| 列入 bot 列表的事件 | 最多作为 inferred candidate；不能绕过 `observe`/`auto` 约束 |
| `observe` 模式下满足规则的推断型候选 | 记录 `would_accept`，不得创建执行中的 Ticket/Assignment |
| `auto` 模式但没有启用规则匹配 | 拒绝接受 |
| 当前活动 Ticket 的规则被禁用或 requester 被移出 allowlist | 发出 Policy change，恢复/继续前重新校验 Mandate |

推荐测试形式（伪代码，仅展示断言意图）：

```ts
it.each(cases)('$name', ({ settings, trigger, expected }) => {
  const policy = settingsPolicy(settings, fakeSourceScope)
  expect(policy.gate(trigger, fakeThread, fakeTickets)).toMatchObject(expected)
})
```

**验收：** 每条拒绝都有稳定 reason/code；每条可执行请求都有可追溯的 source、scope、actor、admission basis、Policy version；测试不依赖真实 GitHub token、模型或网络。

---

## 4. P2：补齐候选决策审计和可回放测试（2–4 天）

### 4.1 将“为什么接/不接”变成稳定记录

当前 `assessmentOutcome()` 已产生 `audit`（规则评估、方向、证据，以及可能的 `would_accept`）；改造重点是验证这些字段端到端保存、展示、版本兼容，而非另造一套评估协议。

**建议落点：**

- `src/policy.ts`：约束 audit 的最大长度、字段和枚举；拒绝额外未知字段或对外展示前做脱敏。
- `src/core/` 的 Eval / Decision 模型与 `src/app/store.ts`：核实已有 Decision 是否保存完整必要信息；若要新增持久字段，增加 SQLite migration 和兼容测试。
- `src/app/views/`、`src/client/`：展示规则命中理由、scope、Policy revision、`would_accept`、拒绝原因；敏感证据不可原样暴露给无权用户。
- `src/app/api.ts`：管理 API 返回的审计信息应有长度上限，且遵从现有 loopback / same-origin 检查。

**不建议持久化模型的完整 chain-of-thought。** 保存结构化决策依据、规则 ID、简短理由、意图证据摘要和执行状态即可；不得把秘密、token、环境变量写入 audit。

示例记录（字段以当前 model/schema 最终定义为准）：

```json
{
  "trigger_id": "github:issue:42:comment:991",
  "actor": { "source": "github", "id": "pilot-user" },
  "scope": { "repository": "acme/widgets", "kind": "issue" },
  "policy_version": "<policy-hash>",
  "gate": { "admissible": true, "strength": "explicit", "via": "addressed" },
  "assessment": {
    "decision": "accept",
    "matched_rules": ["copy"],
    "intent_evidence": "明确要求修正 README 中的拼写错误"
  },
  "execution": { "ticket_id": "<ticket-id>", "state": "pending" }
}
```

上例仅用于说明审计结构，不应直接复制为数据库 schema。

### 4.2 增加不依赖外部服务的回放测试

为同一个固定 Trigger 构造确定性 Source/Policy/Assessor：

1. 第一次 `receive()` 后仅有一条 Trigger，重复相同稳定 ID 后无重复 Ticket/副作用。
2. `observe` 下评估得到候选记录，但没有 Worker launch。
3. 显式请求被接受后，一个 Ticket 和一个 Assignment 入库，Effect 只入队一次。
4. 同 Thread 并发候选按 Engine 的串行评估语义处理；每次 Decision 都能关联原 Trigger。
5. Policy revision 更新后，旧 Eval 作废/重启，Ticket Mandate 重新验证。
6. 重启后 effect/outbox 恢复，不重复发送已经确认的消息。

伪命令（结合仓库现有测试脚本和 fixture 调整）：

```sh
pnpm vitest run src/policy.test.ts
pnpm vitest run src/core
pnpm vitest run src/app
pnpm check
```

如果实际测试命名不同，使用 `pnpm exec vitest run <实际测试路径>`；不要为符合本方案而重复创建同一职责的测试文件。

---

## 5. P3：工作区准备状态与执行恢复加固（2–3 天）

### 5.1 问题与目标

`src/executor/dsh.ts` 当前在工作目录存在时按“已经准备”继续执行。这个策略兼容恢复，但目录可能是半成品、指向错误仓库，或在 prepare 失败后遗留了不完整状态。准备命令失败时不自动重跑是安全的；应补充**可验证状态**，而不是无条件重跑命令。

### 5.2 建议实现

**建议落点：** `src/executor/dsh.ts`、`src/app/execution-environment.ts`、`src/infra/environment.ts`，以及 Executor 单元测试。

新增一个版本化 workspace marker（例如 `.dsh-superagent-workspace.json`），只在 prepare 成功且目录验证完成后原子写入：

```json
{
  "version": 1,
  "ticket_id": "<ticket-id>",
  "assignment_id": "<assignment-id>",
  "repository": "acme/widgets",
  "created_at": "2026-11-06T00:00:00.000Z"
}
```

执行流程建议：

1. 生成预期绝对路径；确认目标位于配置的 execution root 之下，拒绝路径穿越和空路径。
2. 若目录不存在：运行现有 prepare 命令（保留 10 分钟超时）→ 验证目录/仓库身份 → 写临时 marker → `rename` 原子提交。
3. 若目录存在且 marker 完整匹配 Ticket、仓库、版本：恢复使用。
4. 若目录存在但 marker 缺失/损坏/不匹配：**失败并要求人工检查**；不得自动清空目录、覆盖 marker 或把它当作可信工作区。
5. 在 Worker launch 前再次检查当前 Assignment 仍有效、Ticket Mandate 仍有效、暂停状态未改变。
6. Retry 时继续沿用当前安全策略：保留目录、向人报告失败；只在人工核验/修正后重试。

测试案例：

- 新目录成功准备并写 marker；marker 不会在命令失败时生成。
- 命令超时、非零退出、目录验证失败时 Ticket 可见为 failed，Effect 不会启动 Worker。
- marker 指向别的 Ticket、仓库、版本或路径外目录时拒绝执行。
- 已有合法 marker 时重启恢复，不重复 fetch/worktree add。
- 同一 Ticket 的并发 launch 不会同时 prepare 两次（���用现有 Effect/slot/锁机制，按当前架构选合适的事务外锁）。

### 5.3 不要把 worktree 当隔离边界

Git worktree 解决文件工作区冲突，不限制 Worker 对主机其他文件、网络、进程或凭据的访问。部署文档应继续要求专用 OS 账户/主机；如果 DSH 增加可用的受限执行沙箱，再用明确的 host capability 接口集成并做拒绝式检查。不能仅靠检测到某字段就宣称已沙箱化。

---

## 6. P4：设置变更和恢复行为的端到端测试（1–2 天）

### 6.1 设置保存

对 `PUT settings/<block>` 和面板保存路径补测试：

- revision 正确时保存成功，旧 revision 返回冲突，不覆盖新设置。
- 无效 schema、未知字段、超限字符串、相对路径均被拒绝。
- 仓库或 CLI 变更先通过 Setup 检查；失败时不得将无效配置作为可运行配置提交。
- Policy 变化重新评估待处理 Eval，并为现有 Mandate 产生正确 effect。
- pause 打开时仍记录 Source 事件，但不得启动新 Eval 执行、Worker launch 或 Outbox 外发；恢复后只处理一次。

### 6.2 恢复矩阵

模拟进程重启和故障：

| 故障点 | 恢复后预期 |
|---|---|
| Trigger 已写库，Eval 尚未运行 | Eval 可继续且 Trigger 不重复 |
| Ticket 已创建，launch effect 未完成 | effect 重放、幂等启动 |
| Worker 正在执行时退出 | Assignment 被标记 interrupted，恢复说明进入 Worker；不会同时启动两个 Worker |
| Outbox 已发送但回执尚未写入 | 按现有 receipt/幂等协议恢复；测试外部端重复风险并记录限制 |
| prepare 失败留下非空目录 | 失败保持可见；不会把目录默认为准备成功 |
| Policy 降权或 Source scope 被关闭 | Mandate 校验失败后不继续未授权工作 |

对于外部消息，若远端 API 不支持幂等键，不能承诺“绝对恰好一次”；应把系统保证写为“本地队列幂等 + 远端结果可核对”，并用消息 ID/receipt 尽可能去重。

---

## 7. P5：安全边界和部署源头改造（持续）

### 7.1 代码能做的

- 在 Setup 中检查 GitHub CLI 身份、目标仓库、push 权限，以及 worker git 配置；失败时 fail closed，不启动接单循环。
- 环境变量传给 Worker 前继续剔除包含 `KEY`、`PASSWORD`、`SECRET`、`TOKEN` 的变量，并新增回归测试防止过滤被绕过。
- 限制 prompt、trigger、assessment、API body 的大小；所有用户内容按不可信数据处理，不拼接成 shell 命令。
- blocked commands、PATH shim、worker-gitconfig 只作为误操作降低措施；保留对应完整性校验和测试。
- 对危险设置变更记录操作者、前后 revision 和时间；审计内容不得包含 secret。

### 7.2 必须由运行环境承担的

- 专用 OS 账户（不与开发者日常用户共用）、专用机器或容器/VM。
- 文件系统、网络出口、进程和凭据访问控制；Agent 账户只获得目标仓库必要权限。
- `stateDir`、CLI 凭据目录和 SQLite 备份的权限控制与加密/保留策略。
- 管理面板只在 `127.0.0.1` 访问；远程管理走受控 VPN/SSH，不直接暴露公网。

**生产放行阻断项：** 如果无法提供真实执行隔离，不得把 `danger-full-access` preset、wrapper、blocked command 清单或 worktree 描述成安全边界；只能按组织风险接受流程在隔离主机上试点。

---

## 8. P6：观测、指标与故障定位（1–2 天）

沿用现有 HealthBoard、Overview、Trigger/Eval/Ticket/Outbox 视图，补齐可运维字段，不新建与现有状态重复的平行系统。

建议指标（避免标签中包含用户自由文本或 token）：

- Source：最近成功轮询时间、候选数、去重数、scope 拒绝数、Source 错误数。
- Eval：observe 的 `would_accept` 数、accept/ignore/unavailable 数、超时/重试数、Policy revision。
- Execution：pending/running/failed/blocked 数、prepare 时长、worker slot 使用数、手工 retry 数。
- Delivery：Outbox pending/retry/sent/failed 数，最近错误类别；区分 GitHub 与 Feishu。
- Storage：SQLite 大小、stateDir/workspace 占用、未清理 Ticket/Trigger 数。

示例告警规则（按实际监控系统改写）：

```text
若 status 退出码为 2：告警并附上 errors 中的 part/detail，不自动重启。
若 status 退出码为 3：通知负责人完成 Setup，不进行重启循环。
若 status 退出码为 1：告警运行时/进程故障；遵循故障流程，避免无界 restart。
若 worker/outbox 错误持续超过约定时间：暂停处理，人工检查后恢复。
```

继续使用仓库已有脚本：

```sh
scripts/dsh-superagent --profile superagent status
```

退出码及含义以 `docs/006-deployment.md` 为准：0 正常，1 运行时/进程问题，2 组件错误，3 等待 Setup。

---

## 9. P7：测试与交付命令（每个 PR 必须执行）

### 本地验证

```sh
pnpm install --frozen-lockfile
pnpm typecheck
pnpm test
pnpm build
pnpm check
git diff --check
git status --short
```

### PR 内容建议

每个 PR 控制在一个可审查主题，建议顺序：

1. **PR 1：** Policy/Gate/observe 行为测试，不改数据模型。
2. **PR 2：** 设置变更审计与面板说明，必要时新增 schema/migration。
3. **PR 3：** workspace marker、prepare/recovery 与失败测试。
4. **PR 4：** 指标、故障手册、部署检查和端到端回归。

每个 PR 描述至少列出：变更目的、受影响文件、迁移与回滚影响、测试命令及结果、是否改变默认授权/执行行为。涉及授权边界或持久化格式的 PR 必须有维护者审查。

---

## 10. staging 验收：按举例执行

以下是**示例流程**，所有占位符必须替换；仅在已授权的隔离 staging 环境执行。

### 10.1 只读观察

1. 配置一个 staging 仓库、专用 GitHub Agent 用户、1–3 个测试发起者。
2. 关闭不需要的 Issue/PR/comment/CI 观察开关，Eval 保持 `observe`，执行并发设为 1。
3. 发送一个明确但低风险的测试请求，例如：

```text
@<agent-login> 请只检查 README 中的一个拼写错误；报告拟修改文件和验证方式，不要创建 PR。
```

4. 面板中应看到 Trigger、Gate、规则 assessment 和 `would_accept`；不应看到 Worker Assignment、写入工作区或外发 PR。
5. 从不在 allowlist 的账号、未监听的仓库 scope、未列入 bot 列表的 bot 各发送一条测试事件���确认没有执行型 Ticket，并确认拒绝原因可追溯。

### 10.2 单任务人工交付

只有 observe 样本审查通过且负责人批准后，才在 staging 打开有限的显式请求执行。运行一个可回滚任务，例如修改一处文档错字：

```text
@<agent-login> 仅修复 docs/example.md 第 12 行的拼写错误。不要改其他文件。运行文档检查并提交 PR，PR 说明自动化参与及测试结果；禁止自动合并。
```

人工验收：Ticket 只有一个独立工作区；diff 只含批准文件；测试结果可复现；PR 由专用 Agent 身份创建；PR 没有自动合并；可从 PR 返回 Ticket、原始 Trigger、Policy revision 和执行报告。

### 10.3 暂停、重启、恢复

```sh
scripts/dsh-superagent --profile superagent status
# 在面板或受控管理 API 暂停处理
scripts/dsh-superagent --profile superagent restart
scripts/dsh-superagent --profile superagent status
```

确认暂停期间来源事件可按设计记录但无新执行/外发；恢复后没有重复 Ticket/PR/评论。**不要在生产环境用删除 SQLite 或工作目录的方式做恢复测试。**

### 放行标准

- `pnpm check` 全绿；CI 状态成功；无未解释的新增 warning。
- 授权矩阵测试通过；observe 不启动执行；policy 降权会阻止恢复执行。
- 重复轮询、重启和 effect 重试不产生重复 Ticket；外发幂等边界已有说明。
- 工作区错误、prepare 失败、marker 不匹配均 fail closed 且面板可见。
- 至少 3–5 个工作日 observe 记录经人工复核，误接单为 0 或已有明确处置并通过风险签字。
- 至少 3–5 个低风险 staging 任务经人工审查；没有未经批准的仓库外访问或自动合并。

---

## 11. 回滚与数据兼容

- 发布前记录应用 commit、Node/pnpm/DSH 版本、SQLite schema 版本和 profile patch。
- 升级前确认 Worker 空闲，备份 `<stateDir>/orchestrator.sqlite`；备份权限不得比原文件更宽。
- DB migration 必须有“空库 → 最新 schema”和“前一版本 DB → 最新 schema”测试。
- 回滚时停止进程，恢复**升级前**数据库备份，再切回匹配的旧代码/依赖；不要用新 schema 的 DB 配旧程序。
- 若只是 Policy 配置错误，先 pause/关闭 Source 并经面板修正；不要删除数据库来清理业务状态。
- 若凭据泄露，暂停 Source、撤销/轮换专用凭据，核对 Agent API 活动、提交、评论和主机进程。

---

## 12. 按周拆解建议

| 周期 | 交付 | 完成条件 |
|---|---|---|
| 第 0 天 | 基线、测试盘点、风险责任人 | SHA、环境、`pnpm check` 结果、staging/回滚负责人已记录 |
| 第 1 周 | P1 Policy/Gate 测试与试点配置校验 | 授权矩阵通过；默认 observe 不变；API 服务端校验通过 |
| 第 2 周 | P2 审计可见性与回放测试 | 候选能从 Trigger 追到 Decision/Policy revision；无 secret 泄漏 |
| 第 3 周 | P3 工作区 marker 与恢复加固 | 不可信或损坏工作区 fail closed；重启恢复测试通过 |
| 第 4 周 | P4–P6 运维测试、指标和文档 | 暂停、重启、失败、回滚演练均有记录 |
| 之后 | staging 小流量真实任务 | 逐项评审是否开启有限 auto；无自动合并默认值 |

## 13. 最终建议

项目已有不错的域模型、授权、持久化副作用、Ticket Worker 和运维面板。改造应首先把现有安全意图变成**可自动化验证的不变量**：observe 不执行、scope 不越权、Mandate 会重新校验、工作区准备失败不被伪装成成功、重复事件不重复产生副作用。随后再改善审计和运维体验。

**落地顺序：先测授权和恢复，再改执行工作区，再补观测；最后才考虑扩面或 `auto`。** 若无法提供真实 OS/容器级隔离，保持 staging 和人工审批，不应仅凭代码包装器批准生产自动执行。

## 14. 依据

- `EXECUTION-PLAN-zh.md`
- `docs/006-deployment.md`（安装、账号、权限边界、Source/Policy/Execution 配置、Management API、升级与故障处理）
- `package.json`（`pnpm check`：typecheck、Vitest、build；Node/DSH/pnpm 版本范围）
- `src/policy.ts`（Settings schema、observe/auto、Gate、assessment、Mandate revalidation）
- `src/core/engine.ts`（事务式域命令入口、接收、评估、Ticket 生命周期、Policy 变化）
- `src/app/runtime.ts`（组合根、状态目录锁、启动、恢复、shutdown）
- `src/executor/dsh.ts`（Worker Session、workspace prepare、并发、失败、重试、恢复）
- `src/app/store.ts`、`src/app/effects.ts`、`src/app/outbox.ts`（持久化、Effect 和外发回执；按实际 schema/测试核对后改动）
