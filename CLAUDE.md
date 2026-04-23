# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Status

SEOhub 是一个**白帽 AI SEO 站点矩阵项目**，目标是构建中文内容站矩阵，通过自然搜索流量 + 合规推荐位，把流量导到用户现有的广告变现主站。项目由 Claude AI 驱动执行，运营人每周 60–90 分钟仅做守门人审批。

**当前阶段：Pre-M1（文档定稿，待用户确认 [MVP.md §2](MVP.md#2-待用户确认的-7-项决策阻塞-m1-启动) 7 项决策后进入 M1 开发）。**

## 必读文档（按重要度）

Claude Code 开工前必须读完以下文档：

| 文档 | 内容 | 何时读 |
|---|---|---|
| [SKILL.md](SKILL.md) | karpathy-guidelines，最高行为准则 | **每次开工前** |
| [DEVELOPMENT.md](DEVELOPMENT.md) | 代码/测试/Git/文档同步规范 | **每次开工前** |
| [MVP.md](MVP.md) | 范围、预算、里程碑、KPI | 承接新任务时 |
| [ARCHITECTURE.md](ARCHITECTURE.md) | 技术栈、数据模型、部署拓扑 | 动代码时 |
| [API.md](API.md) | 内部/外部/控制面 API 契约 | 改或新增 API 时 |

## 项目硬约束（违反 = 工作作废）

1. **100% 白帽 SEO** — 无 cloaking / JS 自动跳转 / 门户页 / 链接农场 / scaled content abuse
2. **所有 AI 调用必须经 `packages/budget`** — 禁止直接 import `@anthropic-ai/sdk`
3. **垂类锁** — 每个子站单一主题，跨主题内容必须拦截
4. **推荐位上限** — 每页 `rel="sponsored"` ≤ 2，站级指向主站占全出站 ≤ 18%
5. **CI 无真实 API key** — 所有 AI 调用在测试中走 VCR fixture
6. **Day-1 完整数据模型** — 不事后迁移，见 [ARCHITECTURE.md §5.1](ARCHITECTURE.md#51-d1-表结构day-1-必建不等-m2-再加)
7. **Secrets 管理** — 本地 `.dev.vars`、线上 `wrangler secret put`，绝不入库

完整红线列表见 [DEVELOPMENT.md §11](DEVELOPMENT.md#11-约束红线违反--拒合入)。

## 技术栈（定稿）

- **基础设施：** Cloudflare Pages + Workers + D1 + Queues + Workflows + Vectorize + Analytics Engine + Registrar
- **子站：** Astro + SSR on Workers（ISR 按需渲染）
- **仪表盘：** Next.js 14 App Router（M3 引入，M1 先用 Astro+HTMX Admin）
- **编排：** Hono on Workers
- **数据库：** D1 (SQLite) + Drizzle ORM
- **AI：** Claude Haiku 4.5（批量） / Sonnet 4.6（QA/决策） / Opus 4.7（关键判断）
- **包管理：** pnpm workspaces monorepo
- **语言：** TypeScript strict
- **测试：** Vitest + VCR fixtures

## 文档同步约定（本项目最重要的约定之一）

**每完成一个里程碑（M1 / M2 / ...）或发生范围/架构/API 变更时，Claude Code 必须在同一个工作周期内同步更新所有关联文档。**

### 触发器矩阵

| 触发 | 必更新的文档 |
|---|---|
| 里程碑完成 | MVP.md §9 + 版本号 + KPI 快照到 `docs/snapshots/` |
| 新增模块 / 数据表 | ARCHITECTURE.md §4 / §5 |
| 新增 / 变更 / 弃用 API | API.md（对应章节 + §8 变更记录） |
| 新增开发约定 | DEVELOPMENT.md |
| 预算档位变更 | MVP.md §8 + ARCHITECTURE.md §10 |
| 删除功能 | MVP.md §10 删除清单 |
| 任一上述变更 | CLAUDE.md 索引（本文件）如受影响 |

### 里程碑完成 Checklist

见 [DEVELOPMENT.md §8.2](DEVELOPMENT.md#82-里程碑完成-checklist)。**未完成文档同步 = 里程碑未完成**，不允许"先开下阶段，文档等会儿补"。

### 文档版本号规则

- MINOR（v1.0 → v1.1）：章节新增 / 明显增改
- MAJOR（v1.x → v2.0）：结构重组、里程碑完成、范围大调整
- 每次同步更新"最后同步"日期为当前日期

## 行为约束（karpathy-guidelines 浓缩）

遵循 [SKILL.md](SKILL.md) 的四条：先思考再写、最简实现、外科手术式改动、目标驱动。具体到本项目：

- **开工前必问清楚**：当前在哪个里程碑、涉及哪些文档、会影响哪些约束
- **最简实现**：禁止提前抽象、禁止做 MVP 范围外的功能
- **文档优先**：改代码前先改文档（或同步改），反过来不行
- **成本意识**：任何引入新 API / 新服务的改动必须在 PR 说明中估算增量成本

## 约定但不在本文档展开的细节

- 代码风格、Git 规范、测试要求 → [DEVELOPMENT.md](DEVELOPMENT.md)
- 外部依赖清单与费用 → [API.md §3](API.md#3-外部-api消费契约) + [MVP.md §8.2](MVP.md#82-balanced-1600-支出明细)
- 里程碑与验收标准 → [MVP.md §9](MVP.md#9-里程碑-m1--m6)
- 数据模型 → [ARCHITECTURE.md §5](ARCHITECTURE.md#5-数据模型)

## 与用户协作方式

1. 用户每周花 60–90 min 看数据 + 审批，不写代码、不润色内容
2. 用户回复问题倾向简短，不要追问可以自己决定的事
3. 对不可逆或跨系统操作（部署、推送、删域名）必须先确认
4. 用中文与用户交流
