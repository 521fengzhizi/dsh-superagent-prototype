# dsh-superagent 可落地执行方案（傻瓜式版）

> 目标仓库：`521fengzhizi/dsh-superagent-prototype`
> 说明：本文档是按“傻瓜式”给出的一份落地执行方案，适合在当前仓库基础上做最小可控试点，不建议直接全量上线。

## 1. 先说结论

这个仓库已经不是一个空壳项目，而是一个相对完整的 DSH（DeepSeek Harness）智能体原型，核心能力包括：

- GitHub / Feishu 接入
- Trigger 收集与去重
- Policy Gate 与授权校验
- Ticket / Assignment / Worktree 执行
- 面板与管理 API
- 运行记录、状态、恢复、重试

因此，最稳妥的落地方式不是“重写”，而是：

1. 先确认基线稳定；
2. 先做最小授权、最小观察、最小执行；
3. 用测试和日志锁住行为；
4. 只在受控环境内逐步放量。

推荐第一版目标：

- 只接入一个 GitHub 仓库
- 一定用专用 agent 账号
- 默认保持 `observe` 模式
- Worker 并发固定为 1
- PR 由人审查，不允许自动合并
- 先观察 3–5 个工作日，收到足够样本后再决定是否开启有限 `auto`

---

## 2. 适合的落地路径

### 2.1 不建议做的事情

- 不要一开始就开全仓库、全事件接入
- 不要让所有 GitHub 评论、PR、CI 都被自动处理
- 不要默认启用自动合并
- 不要认为 CLI wrapper 就是安全沙箱
- 不要直接在开发者本机用户下运行 Worker

### 2.2 建议做的事情

- 用专用 GitHub Agent 用户
- 用独��� DSH profile（例如 `superagent`）
- 将 `stateDir` 单独放置
- 只允许一小撮人（1–3 人）发起试点请求
- 先观察，不执行
- 先看效果，再逐步扩展

---

## 3. 准备条件

本地或部署机器需要：

```bash
node -v
pnpm -v
git --version
tmux -V
python3 --version
gh --version | head -1
```

项目要求：

- Node.js 22.19+ 或 24+
- pnpm
- git 2.32+
- tmux
- Python 3.8+
- GitHub CLI 2.48+
- DSH 版本满足 `package.json` 中的 `engines.dsh`

---

## 4. 第一步：固定基线并跑通检查

在本地执行：

```bash
git checkout main
git pull --ff-only
git status --short
node --version
pnpm --version
pnpm install --frozen-lockfile
pnpm check
```

如果还没 clone：

```bash
git clone https://github.com/521fengzhizi/dsh-superagent-prototype.git
cd dsh-superagent-prototype
pnpm install --frozen-lockfile
pnpm check
```

### 结果要求

- 工作树干净
- `pnpm check` 全绿
- 记录 当前 HEAD SHA

```bash
git rev-parse HEAD
```

如果这里失败，说明你还没建立“可运行基线”，不能继续做试点升级。

---

## 5. 第二步：看懂现有关键代码和边界

仓库里已经有核心能力，最关键的文件如下：

- `src/policy.ts`：授权规则、Gate、评估结果、重校验
- `src/core/engine.ts`：事务式状态处理入口
- `src/executor/dsh.ts`：Worker / session / worktree / 失败恢复
- `src/app/store.ts`：SQLite 持久化
- `src/app/runtime.ts`：运行时组合根
- `tests/policy.spec.ts`：授权行为测试
- `docs/006-deployment.md`：部署和运维手册

### 这几个事实非常关键

1. 这个仓库原本就把处理链路串起来了：

```text
Source → Engine → Policy/Eval → Ticket/Assignment → Worker → Outbox
```

2. `policy.ts` 默认模式就是 `observe`，因此第一版强烈建议保持默认。

3. `dsh.ts` 里有工作区准备逻辑和失败恢复逻辑；我们要加固的是“不要把坏目录当作已准备好目录”。

4. 当前实现里，Worktree 和 CLI wrapper 不是安全边界；真正的隔离要靠专用 OS 账户、容器或虚拟机。

---

## 6. 第三步：先做最小授权试点（最重要）

### 6.1 配置原则

在最初试点中，至少满足：

- 只允许 1–3 个 GitHub 用户触发
- 仓库只开一个目标仓库
- `eval.mode` 设为 `observe`
- `maxWorkers` 设为 1
- 关闭自动合并

### 6.2 关键设置示例

这个项目用 `SettingsSchema` / `Source` / `Policy` 进行设置，用户可���在面板中配置，但代码层面要保证：

```ts
const settings = SettingsSchema.parse({
  paused: false,
  eval: {
    mode: 'observe',
    timeoutSeconds: 120,
  },
  executor: {
    maxWorkers: 1,
    directory: '',
    prepare: 'git worktree add ...',
    blockedCommands: [],
  },
  policy: {
    rules: [
      { id: 'copy', name: 'Copy fixes', enabled: true, instruction: 'Fix typos and wording.' },
      { id: 'visual', name: 'Visual fixes', enabled: true, instruction: 'Fix styling issues.' },
    ],
    sources: {
      github: {
        allowed: [{ id: 'alice' }],
        admission: {},
      },
    },
  },
})
```

第一阶段不要尝试把所有人都加入 allowlist。

---

## 7. 第四步：锁定授权矩阵测试

### 7.1 先跑核心现有测试

```bash
pnpm exec vitest run tests/policy.spec.ts
```

### 7.2 需要确保的行为

这些规则必须通过：

- 非允许用户发起请求不能执行
- 未监听仓库的消息不能被误接收
- `observe` 模式下不能启动执行型 Ticket
- `auto` 模式下，如果规则不匹配，不接受
- 旧 Mandate 失效后，必须重新校验

### 7.3 推荐新增的测试场景

```ts
it('does not execute inferred work in observe mode', () => {
  const settings = SettingsSchema.parse({ eval: { mode: 'observe' } })
  const outcome = assessmentOutcome(accept, settings, 'inferred')
  expect(outcome).toMatchObject({
    kind: 'ignore',
    reason: { code: 'observed' },
    audit: { would_accept: true },
  })
})
```

```ts
it('refuses work when sender is not on allowlist', () => {
  const gate = policy.gate(trigger, thread, [])
  expect(gate).toMatchObject({ admissible: false, reason: 'not_a_member' })
})
```

如果这些都能稳住，说明授权边界基本可控。

---

## 8. 第五步：加固工作区状态，不要误判“目录存在=已准备好”

这是最容易踩坑的地方之一。`src/executor/dsh.ts` 里有逻辑：

```ts
if (!await stat(workspace).then(() => true, () => false)) {
  await this.prepareDirectory(ticket)
}
```

这意味着：

- 工作区目录存在，直接视为可用
- 但目录可能是空的、坏的、错误的、半成品的

### 建议修正方向

增加一个 `workspace marker`，只在成功建立且验证完成之后才写入。

#### 示例代码

```ts
import { readFile, rename, writeFile } from 'node:fs/promises'
import { resolve, relative, isAbsolute, sep } from 'node:path'

const MARKER = '.dsh-superagent-workspace.json'

type WorkspaceIdentity = {
  version: 1
  ticketId: string
  assignmentId: string
  repository: string
}

export async function writeWorkspaceMarker(directory: string, identity: WorkspaceIdentity) {
  const markerPath = resolve(directory, MARKER)
  const tempPath = `${markerPath}.${process.pid}.tmp`

  await writeFile(tempPath, JSON.stringify(identity), { mode: 0o600 })
  await rename(tempPath, markerPath)
}

export async function verifyWorkspaceMarker(root: string, directory: string, expected: WorkspaceIdentity) {
  const rootResolved = resolve(root)
  const dirResolved = resolve(directory)
  const rel = relative(rootResolved, dirResolved)

  if (!rel || rel === '..' || rel.startsWith(`..${sep}`) || isAbsolute(rel)) {
    throw new Error('Workspace directory is outside the configured root')
  }

  const markerPath = resolve(directory, MARKER)
  const data = JSON.parse(await readFile(markerPath, 'utf8')) as WorkspaceIdentity

  if (
    data.version !== expected.version ||
    data.ticketId !== expected.ticketId ||
    data.assignmentId !== expected.assignmentId ||
    data.repository !== expected.repository
  ) {
    throw new Error('Workspace marker does not match the current assignment')
  }
}
```

### 这类逻辑要覆盖的测试

- 目录不存在时，prepare 成功才写 marker
- prepare 失败时，不写 marker
- 工作区目录越界时拒绝启动
- marker 缺失时拒绝恢复
- marker 不匹配时拒绝执行
- 已有合法 marker 时可恢复，不重复创建工作区

---

## 9. 第六步：设置保存、暂停、重启和恢复测试

### 9.1 需要验证的行为

- 旧 revision 更新时返回 409，不覆盖
- 不合法 schema 不能被保存
- `paused` 状态打开时停止启动新任务
- 重启后 pending 任务恢复，但不重复创建 Ticket
- Worker 退出后，状态能恢复，并在面板里清晰说明

### 9.2 关键文件

- `src/app/store.ts`
- `src/app/runtime.ts`
- `src/app/migration.ts`
- `tests/store.spec.ts`
- `tests/runtime.spec.ts`

### 9.3 典型命令

```bash
pnpm exec vitest run tests/store.spec.ts
grep -R "pause\|restart\|migration" tests
```

---

## 10. 第七步：部署到独立 profile（staging）

按 `docs/006-deployment.md` 说明，推荐用独立 profile：

```bash
npm install -g @deepseek-ai/dsh@0.2.1-rc.3
git clone <this repository> dsh-superagent
cd dsh-superagent
pnpm install --frozen-lockfile
pnpm build
dsh superagent --from-default-profile web --dump-config > /dev/null
dsh plugin --profile superagent add "link:$PWD"
```

然后复制 `examples/profile.cordis.patch.yml` 到 `~/.dsh/profiles/superagent/cordis.patch.yml`，替换 `stateDir` 等配置，然后启动：

```bash
scripts/dsh-superagent start
scripts/dsh-superagent status
```

### 生产/试点前必须确认

- 使用专用 GitHub Agent 账号，而不是操作者个人账号
- 目标仓库有写权限
- 运行时以专用 OS 账户或受控主机执行
- `stateDir` 单独保护
- `worker-bin` / `worker-gitconfig` 仅由程序写入

---

## 11. 第八步：严格执行 3–5 个工作日观察期

### 允许名单

开始时只允许 1–3 个用户发起任务：

```text
@agent-login 请检查 README 中一个小拼写问题
```

### 观察期要求

- 只有允许名单中的用户才能触发
- `observe` 模式下只记录 `would_accept`
- 面板中应能看到 Trigger、Gate、Rule assessment、Policy version
- 不能出现 Worker 自动启动、PR 自动提交、无审查代码写入

### 验收标准

- 误接单为 0 或非常低
- 每个候选都能追踪到触发来源与授权依据
- 没有跨仓库、不允许的操作
- 不存在未说明的无权限写入

---

## 12. 第九步：只有观测样本足够后，再考虑有限执行

只有在观察期充分后，才允许进入“受控执行”阶段：

- 稍微放大允许名单
- 仍保持 `maxWorkers = 1`
- 仍保持 `observe` 或受控 `auto`
- 每个任务范围明确，且允许人工审查

**禁止**：

- 默认开自动合并
- 允许所有 Issue/PR 评论被自动处理
- 让 Agent 直接在生产环境内做“大范围碎片化修复”

---

## 13. 第十步：对外交付、审查、回滚

### 13.1 交付前执行

```bash
pnpm install --frozen-lockfile
pnpm typecheck
pnpm test
pnpm build
pnpm check
git diff --check
git status --short
```

### 13.2 目标 PR 结构建议

推荐 PR 拆分：

1. PR 1：授权 / Gate / observe 测试
2. PR 2：设置变更和恢复行为
3. PR 3：worktree marker 和 prepare/retry 修正
4. PR 4：运维指标和文档完善

### 13.3 回滚方案

- 只要状态不稳定，就暂停 Source
- 不要直接删 SQLite 试图“清理故障”
- 先备份 `<stateDir>/orchestrator.sqlite`
- 先停服务，再恢复旧代码和旧数据库
- 若凭据泄漏，立刻轮换 GitHub / Feishu 凭据

---

## 14. 最终落地清单（傻瓜版）

### 先做（第 1 周）

- [ ] `pnpm check` 全绿
- [ ] 建立专用 agent 账号
- [ ] 独立 DSH profile
- [ ] 目标仓库配置单
- [ ] allowlist 只放 1–3 人
- [ ] `eval.mode = observe`
- [ ] `maxWorkers = 1`

### 再做（第 2 周）

- [ ] 追加授权测试
- [ ] 追加 worktree marker 验证
- [ ] 追加恢复重试测试
- [ ] 确认 `pause` / `restart` 不重复 Ticket

### 最后做（第 3–4 周）

- [ ] 小流量真实试点
- [ ] 人工审查 PR
- [ ] 决定是否增加有限 `auto`
- [ ] 如无充分证据，不扩大范围

---

## 15. 一句话总结

这个仓库的价值在于：它已经把“Source → Policy → Ticket → Worker → Outbox”链路做成了一个原型。真正的落地，不是重写，而是把它变成“最小授权 + 可观测 + 可恢复 + 可审查”的受控系统，并且先用观察期验证再放量执行。

如果你要真正落地，最关键的三个动作是：

1. 保持默认 `observe`；
2. 先锁住授权与恢复测试；
3. 先观察，再做有限执行。

---

## 16. 相关文件

- `README.md`
- `SETUP.md`
- `EXECUTION-PLAN-zh.md`
- `CODE-IMPLEMENTATION-PLAN-zh.md`
- `docs/006-deployment.md`
- `src/policy.ts`
- `src/executor/dsh.ts`
- `tests/policy.spec.ts`

如果需要，我下一步可以继续直接给你：

- “一键复制版”执行命令清单；或
- “适合 GitHub PR 提交”的详细实施计划；或
- “按此仓库结构直接写好的可执行文档文件内容”。
