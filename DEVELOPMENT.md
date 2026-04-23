# SEOhub 开发规范

> 版本：v1.0 · 最后同步：2026-04-23
> 适用对象：Claude Code / 人工贡献者

---

## 1. 总原则

[SKILL.md](SKILL.md) 的 **karpathy-guidelines** 是本项目的最高行为准则，任何冲突以其为准。核心四条：

1. **先思考再写** — 陈述假设，不够清楚就问
2. **最简实现** — 最少代码解决问题，无投机抽象
3. **外科手术式改动** — 只改与任务直接相关的行
4. **目标驱动** — 把任务变成可验证的成功标准

---

## 2. 代码规范

### 2.1 TypeScript

- `strict: true`，禁用 `any`（必要时 `unknown` + 类型收窄）
- `noUncheckedIndexedAccess: true`
- 全项目 ESM，顶部 `.ts` 扩展名必须（Workers 要求）
- 导出用 named export，避免 default export（Tree-shaking + IDE 跳转）

### 2.2 注释与命名

- **默认不写注释**；仅在"为什么非显然"时写单行注释
- **禁止写** "这段代码做什么"的解释性注释（好的命名已经够了）
- **禁止引用当前任务、PR 编号、临时 issue** 作为注释
- 函数名动词开头（`generateOutline` / `validateSlotBudget`），非动词（`outlineGenerator`）
- 变量名禁缩写（`req` / `res` / `cfg` 除外，常见惯例保留）
- 类型名 PascalCase，常量全大写 + 下划线

### 2.3 Lint / Format

- ESLint + `@typescript-eslint`（根 `.eslintrc.json`）
- Prettier 格式化（根 `.prettierrc`）
- 提交前 git hook（`lint-staged`）自动修
- CI 门禁：`pnpm lint && pnpm typecheck`

### 2.4 错误处理

- 仅在**系统边界**（HTTP 入口、外部 API 调用）处理错误
- 内部函数不做防御性 null check，信任类型系统
- 错误必须带**上下文**（不是 `throw new Error('failed')`）
- 不 catch 然后 swallow；要么处理，要么让它往上冒

---

## 3. Monorepo 约定

### 3.1 工具

- **pnpm 9+**，`pnpm-workspace.yaml` 管理
- 根 `package.json` 统一 scripts：`pnpm dev / build / test / lint / typecheck`
- 跨包依赖用 `workspace:*` 语法
- **禁止** 在子包 `package.json` 里固定版本，统一走根版本

### 3.2 包命名

- 所有包 scoped 为 `@seohub/*`
- apps 不发布 npm，packages 也不（私有 monorepo）

### 3.3 包分层规则

```
apps/ 依赖 packages/，不跨 apps/ 互相依赖
packages/ 之间有严格分层（低 → 高）：
  config → db → adapters → seo / budget / slots → content-gen → orchestrator
```

违反层级 = PR 必 reject。

---

## 4. Git 工作流

### 4.1 分支

- `main` — 始终可部署，受保护
- `feat/*` — 新功能
- `fix/*` — Bug 修复
- `docs/*` — 纯文档更新
- `refactor/*` — 重构（无行为变更）
- 小改动直接在 `main` 提交（项目早期，单人 or 少人团队）

### 4.2 Commit 规范

遵循 **Conventional Commits**：

```
<type>(<scope>): <subject>

<body 可选>

<footer 可选，Co-Authored-By 等>
```

**type：** feat / fix / docs / refactor / test / chore / perf / build

**scope：** 包名或 app 名（`budget` / `content-gen` / `dashboard` ...）

**例：**
```
feat(budget): add pre-flight cost estimation with cache-hit awareness
fix(slots): respect site-level sponsored ratio cap
docs(mvp): mark M1 acceptance criteria complete
```

禁止：
- 在 commit message 里引用临时任务 ID（除非团队约定）
- 跨包的混合 commit（拆分为多次）

### 4.3 PR / 合入

- 所有 PR 必须过 CI gate（lint + typecheck + unit + integration）
- 项目早期不强制 code review（单人开发）；引入第二位贡献者后强制
- 合入偏好 **rebase merge**（保持线性历史）

---

## 5. 测试要求

### 5.1 覆盖率目标

- `packages/budget`、`packages/slots`：**≥ 90%**（核心业务逻辑）
- `packages/seo`、`packages/content-gen`：**≥ 70%**
- `apps/*`：**≥ 50%**（集成测试为主）

### 5.2 AI 调用必须 mock

- **禁止** 在单元测试中直接调用 Anthropic / 5118 / GSC API
- 所有 AI 调用经 `packages/testing` 的 VCR 装置
- 录制新 fixture 必须提交到 `fixtures/` 目录（git-tracked）

### 5.3 VCR 用法

```typescript
import { withVCR } from '@seohub/testing';

it('generates outline', async () => {
  await withVCR('outline/astrology-basics', async () => {
    const outline = await generateOutline({ topic: '十二星座入门' });
    expect(outline.sections).toHaveLength(5);
  });
});
```

- `VCR_MODE=record`（本地手动跑）→ 真实打 API + 存 fixture
- `VCR_MODE=replay`（CI 默认）→ 仅回放；未命中 fixture 直接失败

### 5.4 CI 配置

```yaml
# .github/workflows/ci.yml（M2 补齐）
env:
  VCR_MODE: replay
  # 不注入任何真实 API key
```

---

## 6. 依赖与安全

### 6.1 依赖新增

- 加依赖前先评估：能否用 Web 标准 API 替代？
- 优先选 Cloudflare Workers 兼容的包（检查 `exports` 和 `browser` 字段）
- 大包（> 100KB gz）必须在 PR 说明中解释
- **禁止** 依赖 Node.js-only 原生模块（`fs` / `child_process` / `net`）

### 6.2 Secrets

- 本地：`.dev.vars`（gitignored）
- 线上：`wrangler secret put <KEY>`
- **禁止**：commit `.env`、硬编码 API key、把 key 写入注释
- 所有 Secrets 命名：`UPPER_SNAKE_CASE`，前缀表明属主（`ANTHROPIC_API_KEY` / `CF_API_TOKEN` / `WU118_API_KEY`）

### 6.3 漏洞扫描

- `pnpm audit` 每 PR 自动跑
- Dependabot / Renovate 每周 PR，优先合入 security patches

---

## 7. AI 调用强制约束

### 7.1 所有 AI 调用经 `packages/budget`

```typescript
// ❌ 禁止
import Anthropic from '@anthropic-ai/sdk';
const client = new Anthropic();
await client.messages.create({ ... });

// ✅ 必须
import { callClaude } from '@seohub/budget';
const response = await callClaude({
  taskType: 'outline',
  siteId: 'site_abc',
  model: 'haiku',
  messages: [...],
});
// callClaude 内部做 preflight + cache + ledger + retry
```

### 7.2 Prompt Caching 必须开

- 所有 system prompt ≥ 1024 token 强制加 `cache_control`
- Claude SDK 调用必须使用 `prompt-caching` beta header

### 7.3 Model 路由

| Task | Model | 理由 |
|---|---|---|
| 批量内容生成 | Haiku 4.5 | 便宜 |
| QA 评分 / Outline | Sonnet 4.6 | 平衡 |
| 新站决策 / Veto 预过滤 | Opus 4.7 | 关键判断 |
| 速率限流降级 | Haiku 4.5 | 兜底 |

路由表定义在 `packages/config/src/models.ts`，改动需 PR 评审。

---

## 8. 文档同步规范

**这是本项目最重要的约定之一。** 每完成一个开发阶段（里程碑 / 重要 PR），必须同步更新关联文档。

### 8.1 五份文档及其变更触发器

| 文档 | 何时必须更新 |
|---|---|
| [CLAUDE.md](CLAUDE.md) | 任何面向 Claude Code 的约定变更 |
| [MVP.md](MVP.md) | 里程碑完成、范围变更、KPI 调整、预算档位变更、删除清单新增 |
| [ARCHITECTURE.md](ARCHITECTURE.md) | 新增模块、数据表 schema 变更、外部依赖新增、部署拓扑变更 |
| [API.md](API.md) | 内部 / 外部 API 端点新增 / 变更 / 弃用；数据契约变更 |
| [DEVELOPMENT.md](DEVELOPMENT.md) | 规范变更、新增约束、工作流调整 |

### 8.2 里程碑完成 Checklist

每个 M1 / M2 / ... 完成后，Claude Code 必须执行以下步骤才能声明"里程碑完成"：

```
□ 1. 所有该里程碑范围内的 PR 已合入 main
□ 2. CI 全绿
□ 3. 实际产出对照 MVP.md §9 验收标准，逐条勾选
□ 4. 更新 MVP.md §9：本里程碑状态改 "✅ Completed (YYYY-MM-DD)"
□ 5. 若本阶段引入新数据表 / 新模块 → 更新 ARCHITECTURE.md §5 / §4
□ 6. 若本阶段引入新端点或变更 → 更新 API.md
□ 7. 若本阶段改变开发约定 → 更新 DEVELOPMENT.md
□ 8. 顶部版本号 bump（v1.0 → v1.1 或 v2.0），"最后同步" 日期更新
□ 9. CLAUDE.md 索引同步（如指向了新文档 / 新章节）
□ 10. 产出 KPI 数据快照（存入 docs/snapshots/MX-YYYY-MM-DD.md）
□ 11. 提交一个专门的 `docs: sync for MX completion` commit
```

**未完成文档同步 = 里程碑未完成**。不允许"先开 M2，文档等会儿再补"。

### 8.3 增量更新 vs 重写

- **增量更新**（默认）：改动章节，保留历史结构
- **重写**（需说明）：版本 major bump，保留旧版本到 `docs/archive/`

### 8.4 版本号规则

- MINOR（v1.0 → v1.1）：章节新增 / 明显增改
- MAJOR（v1.x → v2.0）：结构重组、里程碑完成、范围大调整
- PATCH（v1.0 → v1.0.1）：错字 / 微调，可省略

---

## 9. 部署与发布

### 9.1 环境

见 [ARCHITECTURE.md §8.3](ARCHITECTURE.md#83-环境分层)。

### 9.2 发布流程（M2 建立）

1. `main` 分支合入 → GitHub Action 触发
2. 运行 CI gate
3. 部署到 staging（`*.pages.dev`）
4. 自动 smoke test
5. 手动批准 → 部署到 production
6. Sentry / Axiom 监控 15 分钟，异常回滚

### 9.3 回滚

- **子站内容**：`contents.status` 改回 `'archived'` + Workflow 触发单页重生
- **基础设施**：Wrangler `wrangler rollback`，回到上一版本

---

## 10. 运营工具

### 10.1 本地 CLI（`scripts/ops.ts`）

```bash
pnpm ops sites:list
pnpm ops sites:create --vertical=astrology --slug=zodiac-daily
pnpm ops content:generate --site=zodiac-daily --count=5
pnpm ops budget:status
pnpm ops veto:list
pnpm ops veto:approve <content_id>
```

### 10.2 运营操作 SOP

详见 [MVP.md §7 运营模型](MVP.md#7-运营模型)。日常不走 CLI，走 Admin / Dashboard UI。

---

## 11. 约束红线（违反 = 拒合入）

1. ❌ 任何形式的 JS/Meta 自动跳转子站 → 主站
2. ❌ 子站互链（任何指向其他子站的内链）
3. ❌ `rel="sponsored"` 缺失指向主站的链接
4. ❌ 绕过 `packages/budget` 直接调 Anthropic
5. ❌ 在 CI 中使用真实 API key
6. ❌ 在 commit 或代码中泄露 Secrets
7. ❌ 里程碑完成声明但文档未同步
8. ❌ 违反垂类锁（写跨主题内容未拦截）
9. ❌ 单页 `rel="sponsored"` 链接 > 2
10. ❌ 使用时效性新闻内容（易腐 + 沙盒）
