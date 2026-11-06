# dsh-superagent 可执行方案报告书（代码评估版）

> **评估对象：** [`521fengzhizi/dsh-superagent-prototype`](https://github.com/521fengzhizi/dsh-superagent-prototype)  
> **评估基线：** `main`，代码快照 `8c32a9dc5d72c033816c6a8110b37f40cf94fba4`  
> **报告日期：** 2026-11-06  
> **报告性质：** 基于仓库代码和文档的技术实施、部署与验收方案；不代表已在目标机器完成安装或运行测试。

## 1. 执行摘要

该项目是一个运行在 DeepSeek Harness（下称 DSH）中的单仓库主动式编程智能体原型：从 GitHub 收集 Issue、PR、评论、CI 等事件，也可选接入飞书；按显式授权和准入规则决定是否接单；为每个 Ticket 创建持久 Worker Session 与独立工作目录；通过管理面板、通知和 GitHub/飞书渠道交付结果。

**建议结论：可进入受控试点，不建议直接全员、全仓库或自动合并上线。** 首轮应使用专用智能体账号与独立 DSH profile，先只开放 GitHub、只允许少数指定账号发起任务，评估模式设为 `observe`，人工检查候选结果后再考虑逐步开启 `auto`。仓库 README 明确标注这是仅供参考的研究原型，并非 DeepSeek 官方产品；生产使用前须由使用团队承担账号、权限、主机和输出审查责任。

本报告将“可执行”定义为：给出部署前置条件、分阶段操作步骤、可复制的验证命令、验收标准、风险控制和回滚办法。由于当前没有目标主机、目标业务仓库、GitHub Agent 账号或凭据，本报告提供可执行清单，不声称这些操作已代替运维人员完成。

## 2. 方案目标与范围

### 目标

1. 将仓库中的原型部署为独立的 DSH `superagent` profile，避免和操作者日常 Session 混用。
2. 用可审计的 Policy 决定谁能发起工作、哪些规则允许主动处理、Mandate 是否仍有效。
3. 以持久化 Ticket、Assignment、Input、Outbox 和重试队列，支撑运行中断后的恢复和人工接管。
4. 通过逐级放量和量化验收，控制误接单、通知噪声、错误代码交付和凭据误用。

### 本次建议纳入

- **首期：** 一个 GitHub 仓库、专用 GitHub Agent 用户、GitHub Source、面板监控、独立 Ticket 工作区、人工审查 PR。
- **可选后续：** 经 bot-only 凭据检查的飞书机器人和指定会话。
- **暂不建议：** 多仓库、多团队规模化部署；不受限的全仓评论接入；以 Agent 身份自动合并；把本地 CLI 包装器误当成安全隔离边界。

## 3. 代码方案与运行机制

### 3.1 处理链路

```text
GitHub / 飞书事件
        ↓
Source 轮询、历史恢复、稳定 Trigger ID 与 Thread 归属
        ↓
Engine 入库、幂等去重、Thread 串行 Eval、订阅 Ticket 扇出
        ↓
Policy Gate（身份/范围/提及/准入）→ Main Session assessment（规则/范围/证据）
        ↓
Decision + Mandate → Ticket + Assignment
        ↓
专属 Worker Session + Ticket 工作目录 → 代码修改/测试/报告
        ↓
持久化 Outbox → GitHub 评论/PR 或飞书回复；面板呈现健康、历史和待人工事项
```

### 3.2 关键架构事实

- `src/index.ts` 注册 DSH 插件并挂载 `Runtime`；`src/app/runtime.ts` 是组合根，组装 Store、Setup、Engine、SourceRegistry、Effects、Outbox、Sessions、Assessor、Executor、Settings 和 HTTP 面板接口。状态目录加锁后才打开 SQLite，避免多个进程同时迁移或写入。
- `src/core/engine.ts` 是域状态的唯一写入口，命令以同步事务更新 Trigger、Thread、Decision、Ticket 和 Assignment；通过 Effects 延后处理外部副作用。`src/app/store.ts` 使用单个 SQLite 数据库，文档指定位置为 `<stateDir>/orchestrator.sqlite`。
- `src/policy.ts` 定义 Zod 设置校验、Policy 版本、准入 Gate、Mandate 重验证和 assessment 结果约束。默认 Eval 模式为 `observe`；推断型请求必须有已启用匹配规则、明确的实施方向和意图证据，才能形成接单结果；`observe` 仅记录“would accept”，不实际接单。显式授权请求可在各模式处理。
- `src/sources/github.ts` 实现 GitHub 通知、仓库事件流、订阅 Thread、PR 和 CI 轮询。默认轮询至少 60 秒；接收全部 Issue/PR/评论/CI 有各自开关。`src/sources/feishu.ts` 提供可选飞书 Source、群和私聊接收范围、成员同步及 bot 列表。
- `src/executor/dsh.ts` 为 Ticket 使用 `worker-<ticket id>` Session 和工作目录；通过 `WorkerSlots` 限流，支持启动、输入投递、恢复、后台工作检查和失败报告。默认最大并发 Worker 数为 3，可在面板调整。
- `src/app/effects.ts` 和 `src/app/outbox.ts` 分别承载持久副作用队列和消息回执；失败按退避策略重试。Engine/Ticket 状态和来源游标均有持久化设计，服务重启时会恢复待处理工作。
- `src/client/` 提供 Superagent 侧栏面板，按 Overview、Source、Trigger、Eval、Execution、Settings 展示工作与健康状态；`locale/` 和客户端文案支持中英双语。
- `scripts/dsh-superagent` 提供 `start`、`stop`、`restart`、`status`，通过 tmux 运行并做本机 HTTP 健康检查；退出码区分运行故障（1）、组件故障（2）和等待配置（3）。

### 3.3 授权与安全边界

- 允许名单和 admission table 在各 Source 的身份体系内配置；代码不自动把 GitHub 用户映射到飞书用户。
- 订阅消息可以作为既有 Ticket 上下文扇出；这不等于给新请求授权。运行中 Ticket 的 Mandate 会记录 Policy 版本与来源依据；恢复时依规则重新校验。
- GitHub 操作使用单独的 Agent CLI 包装器；飞书配置要求严格 bot 身份。`docs/006-deployment.md` 明确提示：命令包装器、Worker preset 和工作目录提供的是默认约束，不构成操作系统级安全隔离；Worker 以操作者 OS 用户执行，并具有较广泛工具权限。
- 因此必须使用专用运行账户或受控主机、缩小文件系统和网络权限、保护 stateDir 与凭据目录、定期核对 Worker 产生的 PR 和消息；不要将可信边界建立在 prompt 或 PATH wrapper 上。

## 4. 建议实施路径

### 阶段 0：负责人确认与上线门槛（半天）

确认并记录：目标 GitHub `owner/name`、GitHub Agent 登录名、操作者/管理员、允许发起者名单、运行主机和系统账户、DSH 版本、stateDir、工作区根目录、回滚责任人。飞书是否首期接入也在此阶段确定。

**放行条件：** 有专用 Agent GitHub 用户（而非操作者个人账号）、目标仓库写权限经审核、有人负责日常审查其输出；未满足时不安装到可写生产仓库。

### 阶段 1：隔离环境与软件检查（半天至一天）

在专用机器或受控账户准备 Node.js 22.19+ 或 24+、pnpm 11.7.0、git 2.32+、Python 3.8+、tmux、GitHub CLI 2.48+，以及满足 `package.json` 的 DSH `engines.dsh` 范围的版本。仓库部署文档示例固定安装 `@deepseek-ai/dsh@0.2.1-rc.3`，执行前应结合当前兼容范围和团队依赖策略核对版本；不要直接覆盖已经在使用的 DSH profile。

在克隆目录进行静态/构建检查：

```sh
pnpm install --frozen-lockfile
pnpm check
```

`pnpm check` 执行 `typecheck`、Vitest 测试和 build。记录命令、提交 SHA、耗时和失败项；本报告没有替你实际运行这些命令。

**放行条件：** `pnpm check` 全绿；工作树干净；运行版本与包 `engines` 相容；有 SQLite 备份路径和恢复演练安排。

### 阶段 2：安装独立 profile 和身份（一天）

按 `docs/006-deployment.md` 操作：为 Agent 用户建立专用 `gh` 配置目录及包装器；用 Agent CLI 核验 `auth` 和目标仓库写权限。创建基于 Web profile 的 `superagent` profile，安装构建产物并复制 `examples/profile.cordis.patch.yml`，将占位符替换为**绝对路径**。建议 stateDir 权限仅运行账户可读写，备份也限制相同访问范围。

安装文档中的核心命令形态如下（`<...>` 必须替换；若仓库已在本地则不要再次 clone）：

```sh
npm install -g @deepseek-ai/dsh@<经核准且满足 engines.dsh 的版本>
git clone https://github.com/521fengzhizi/dsh-superagent-prototype.git dsh-superagent
cd dsh-superagent
pnpm install --frozen-lockfile
pnpm build
dsh superagent --from-default-profile web --dump-config > /dev/null
dsh plugin --profile superagent add "link:$PWD"
```

复制并编辑 profile patch 后启动：

```sh
scripts/dsh-superagent start
scripts/dsh-superagent status
```

**注意：** `dsh superagent --from-default-profile web --dump-config` 仅用于新 profile；部署文档指出重复运行会替换现有 profile 配置。修改 GitHub CLI、仓库或 Feishu CLI 等 Setup 字段会触发重启。

### 阶段 3：最小授权配置与观察模式（至少 3–5 个工作日）

在 Superagent 面板完成 Setup，并按顺序设置：

1. GitHub Source：只配置一个目标仓库和 Agent CLI。首期 `observeIssues`、`observePulls`、`observeComments`、`observeCi` 均关闭或仅按试点目标逐个开启。
2. Policy：允许名单只放 1–3 名试点用户；确认 GitHub bot 列表仅包含实际需要的自动化账号。
3. Eval：保持 `observe`，核对规则、命中理由、来源 scope 和“would accept”案例；不因想提高产量而默认打开接收全量消息。
4. Execution：设定专用目录根路径、最大并发 1（通过面板设置），准备命令用 `git worktree` 建立 Ticket 独立分支；个人 CLI 列入 blocked commands，但仍按 OS 权限隔离。
5. Prompts：填写自动化披露文本、PR 交付要求、Commit 作者、评审机器人和自动合并条款。试点阶段建议禁止 Agent 启用 auto-merge。

**观察期验收：** 每条候选可回溯到 Trigger、Gate、规则、Decision；误判案例已修订规则/Prompt 或明确不接；无未解释的授权绕过、重复接单、跨 Ticket 工作区串用和身份混淆。观察不足时继续保持 `observe`。

### 阶段 4：受控执行和人工交付（1–2 周）

由允许名单内成员在目标仓库 Issue/PR 中以 `@<agent login> <明确请求>` 触发 3–5 个低风险、范围明确的任务。每项任务应能用现有标准验证（例如文案/小型样式修改、明确的局部修复），不应要求产品决策、权限提升或秘密访问。人工检查 Agent 回复、分支、diff、测试和 PR；以“PR 可审阅但未经人工批准不合并”为交付目标。

可考虑将推断型任务从 `observe` 升到 `auto`，**仅在**候选误接率达到团队门槛、规则边界经过实际样本验证且有随时暂停机制后进行；先在一个来源 scope/低风险规则内灰度，而非全局切换。

### 阶段 5：运维固化与扩面评审（持续）

在操作者自己的 DSH profile 或独立监控系统创建每小时检查，避免放在 Agent 自身 profile 中。按部署文档要求，退出码 0 静默；退出码 2 通知负责人但不盲目重启；退出码 3 提示 Setup 问题；退出码 1 才按受控流程判断是否重启。监控命令示例：

```sh
scripts/dsh-superagent --profile superagent status
```

通过阶段评审后，才考虑开启更多 GitHub 接收范围、增加允许名单、接入飞书或提升并发。飞书首期必须使用 bot-only `lark-cli` 配置，先从一个指定群或私聊范围开始，不要直接向所有群接收。

## 5. 验收清单与度量

| 领域 | 验收项 | 首期通过标准 |
|---|---|---|
| 构建 | `pnpm check` | typecheck、全部测试和 build 均退出 0 |
| 服务 | `scripts/dsh-superagent status` | 正常运行退出 0；无重复进程；管理面板可访问 |
| 身份 | GitHub Setup | Agent CLI 返回预期 login；目标仓库 push 权限为 true；操作者账号不被 Worker 默认使用 |
| 授权 | Gate / Policy | 未授权人、未接收 scope、未列入 bot 按预期拒绝；`observe` 不启动推断型 Ticket |
| 幂等 | Trigger/副作用 | 重复轮询不会重复创建 Ticket 或重复发送已登记消息 |
| 恢复 | 重启/失败 | SQLite 备份可读；重启后状态恢复；失败/阻塞可在面板识别并人工 Retry |
| 工作区 | Ticket Execution | 每 Ticket 独立目录/分支；没有跨 Ticket 文件修改；准备失败可见且不被误认为成功 |
| 交付 | PR 与消息 | PR 有明确范围、测试证据、自动化披露；无未经批准的自动合并 |
| 运维 | health/status | Source、runtime、worker、reply 等组件错误可定位；退出码 1/2/3 处理责任明确 |
| 责任 | 人工审阅 | 指定 PR reviewer 和 on-call；明确 stop、pause、备份与回滚负责人 |

建议跟踪：候选总量、观察模式“would accept”量、人工确认的有效率、误接单数、Ticket 完成率、PR 首次审查通过率、重复/失败消息数、每 Ticket 人工介入次数、Worker 并发峰值、各 Source 错误持续时间。指标的分母和取数周期应在试点开始前固定，避免以任务量替代质量。

## 6. 主要风险及缓解措施

| 风险 | 仓库依据/影响 | 缓解措施 |
|---|---|---|
| 研究原型稳定性与兼容性 | README 明示仅供参考；包对 DSH 及多项 `0.2.1-alpha` 依赖有版本约束 | 固定提交和依赖锁文件；部署前 `pnpm check`；先 staging；升级前备份数据库并做兼容回归 |
| Worker 主机权限过宽 | `docs/006-deployment.md` 指明 Worker 以操作者 OS 用户运行、preset 为 `danger-full-access`；包装器不是安全边界 | 用专用 OS 账户/主机，限制仓库和文件权限、网络出口及可用凭据；敏感任务不交给 Agent；人工审阅全部输出 |
| 误接单或噪声 | 自动接收开关与 `auto` 可扩大触发面；推断依赖模型判断 | 先 `observe`、短 allowlist、关闭全量接收；按误判样本收紧规则和 Prompt；提供暂停开关和 on-call |
| 凭据混用/泄漏 | GitHub CLI 和可选 Feishu CLI 持有可操作身份；worker 环境含凭据选择包装 | 使用专用身份和专用配置目录；秘密不进入聊天、Prompt、仓库或 Ticket 输出；定期轮换并审计使用记录 |
| 并发和主机资源 | Worker Session、shell jobs 和 subagents 会占用资源 | 试点将 `maxWorkers` 设 1；观测 CPU、内存、磁盘、进程和后台作业后再逐步调高 |
| 数据增长和长期保留 | `docs/007-architecture.md` 表明 Trigger、Disposition、Decision、结束 Ticket 和 Outbox receipts 当前无清理机制 | 监测数据库和 stateDir 增长；定期离线备份并验证恢复；上线前确定保留/清理策略，清理前先评审迁移和审计需求 |
| 自动合并归责不清 | 文档提示 GitHub 将合并归到启用 auto-merge 的账号 | 试点不启用；如未来启用，需独立审批、校验审批/分支保护，并使用明确的专用身份和审计规则 |
| 本机管理 API 暴露 | API 是本地 HTTP 管理接口，代码实施 loopback/same-origin 检查 | 保持监听 `127.0.0.1`，不直接暴露公网；如需远程访问，走受控 VPN/SSH 隧道，不改成开放监听 |

## 7. 故障处置、暂停和回滚

1. **停止接单：** 面板暂停处理或关闭相应 Source；保留事件记录，核对当前运行中的 Ticket。`stop` 是有意停止并写入监控标记，不要将其与 `restart` 混用。
2. **检查状态：** `scripts/dsh-superagent status`；区分退出码 1（运行时/进程问题）、2（组件需要人工处理）、3（等待 Setup）。退出码 2、3 不应通过反复重启掩盖。
3. **失败 Ticket：** 在面板查看失败原因；修正 Setup、工作区或环境后由授权人员 Retry。不要删除工作目录或数据库来“清故障”。
4. **升级：** 确认 Worker 当前没有进行中的工作；备份 `<stateDir>/orchestrator.sqlite`；固定新提交，执行 `pnpm install --frozen-lockfile && pnpm check`，随后 build、restart、检查面板与 status。
5. **回滚：** 先停止服务并保护运行日志；恢复与旧代码版本匹配的数据库备份，再切回原先代码/依赖版本。部署文档指出新版本迁移的数据库不能由旧版本直接打开，禁止只回退代码而不回退数据库。
6. **凭据事件：** 立即禁用/轮换专用 GitHub 或 Feishu 凭据，暂停 Source，审查 Agent 最近的 API、提交、评论和主机进程；按团队安全事件流程处置。

## 8. 立即可执行的下一步

| 序号 | 动作 | 产出 | 责任角色 |
|---|---|---|---|
| 1 | 确认目标仓库、试点用户、GitHub Agent 用户、主机与负责人 | 已批准的最小权限配置单 | 仓库管理员/试点负责人 |
| 2 | 在隔离环境固定提交并运行 `pnpm check` | 构建与测试记录 | 工程负责人 |
| 3 | 配置专用身份、DSH profile、stateDir 和受限工作区 | 可重复的部署记录和备份 | 平台/运维 |
| 4 | 按部署文档配置 GitHub Source、最小 allowlist、`observe` 与并发 1 | 可观测但不自动接单的试点 | 试点负责人 |
| 5 | 收集至少 3–5 个工作日候选样本，人工审核并记录误判 | 规则校准与风险评审 | 试点用户/安全负责人 |
| 6 | 通过评审后执行 3–5 个低风险真实任务，逐个审查 PR | 试运行验收记录 | 工程团队 |
| 7 | 评审是否开启有限 `auto`、扩大接收范围或增加 Feishu | 有签字的扩面决策 | 项目负责人/安全负责人 |

## 9. 结论

代码具备较完整的原型级闭环：Source 接入、策略 Gate、双层 Eval、Mandate、Ticket/Assignment 生命周期、持久副作用、会话 Worker、面板、健康检查和安装文档。最适合的落地方式不是“一键全自动”，而是先把它作为**单仓库、专用身份、低并发、观察优先、人工审查**的团队试点。进入更大范围或生产职责前，必须先解决主机权限隔离、数据库保留/清理、依赖稳定性、凭据管理和持续运维责任等问题。

## 10. 依据文件

- `README.md`、`SETUP.md`
- `docs/000-philosophy.md` 至 `docs/008-cookbook.md`，重点为 `docs/001-domain.md`、`docs/002-authorization.md`、`docs/003-execution.md`、`docs/006-deployment.md`、`docs/007-architecture.md`
- `package.json`、`examples/profile.cordis.patch.yml`、`scripts/dsh-superagent`
- 核心实现：`src/index.ts`、`src/app/runtime.ts`、`src/core/engine.ts`、`src/policy.ts`、`src/sources/github.ts`、`src/sources/feishu.ts`、`src/executor/dsh.ts`
- 测试入口：`pnpm check`（本报告仅引用仓库配置，未在目标环境执行）
