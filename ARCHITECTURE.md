# SEOhub 技术架构

> 版本：v1.0 · 与 [MVP.md](MVP.md) v1.0 对齐
> 最后同步：2026-04-23

---

## 1. 系统全景图

```
┌──────────────────────────────────────────────────────────────────┐
│                      运营人（User，守门人）                       │
│        浏览器 → Admin(M1) / Dashboard(M3) → CF Pages              │
└───────────────────────────┬──────────────────────────────────────┘
                            │
┌───────────────────────────▼──────────────────────────────────────┐
│                    Control Plane (Cloudflare Workers)            │
│                                                                  │
│   ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐ │
│   │ Orchestrator│  │  Dashboard  │  │  Webhooks / Cron        │ │
│   │    (Hono)   │  │ (Next.js M3)│  │  (IndexNow, GSC poll)   │ │
│   └──────┬──────┘  └──────┬──────┘  └────────────┬────────────┘ │
│          │                │                      │              │
│          └────────┬───────┴──────────────────────┘              │
│                   │                                              │
│   ┌───────────────▼──────────────────────────────────────────┐  │
│   │                      Core Packages                        │  │
│   │   budget  │  content-gen  │  seo  │  db  │  adapters     │  │
│   └───────┬──────────────────────────────────────────────────┘  │
└───────────┼──────────────────────────────────────────────────────┘
            │
┌───────────▼──────────────────────────────────────────────────────┐
│                  Async Layer (CF Queues + Workflows)             │
│   content-generation / qa / publish / rank-check / indexnow      │
└───────────┬──────────────────────────────────────────────────────┘
            │
┌───────────▼──────────────────────────────────────────────────────┐
│                          Data Plane                              │
│   D1 (业务) · Analytics Engine (点击) · Vectorize (去重) ·       │
│   R2 (图片/fixture/备份) · KV (运行时配置/缓存)                  │
└───────────┬──────────────────────────────────────────────────────┘
            │
┌───────────▼──────────────────────────────────────────────────────┐
│                        Content Delivery                          │
│   子站 1 (Astro SSR on Workers) · 子站 N ·  主站 /blog           │
│   Pages Project × N · 独立域名 · rel="sponsored" 推荐位           │
└──────────────────────────────────────────────────────────────────┘
            │
┌───────────▼──────────────────────────────────────────────────────┐
│                      外部系统（出站依赖）                         │
│   Anthropic API  ·  5118 API  ·  GSC API  ·  Bing Webmaster     │
│   IndexNow · Unsplash · Replicate · Resend · Sentry · CF Registrar│
└──────────────────────────────────────────────────────────────────┘
```

---

## 2. 技术栈选型与理由

| 层 | 选型 | 理由 | 替代评估 |
|---|---|---|---|
| Runtime | **Cloudflare Workers** | 边缘分发，零运维，免费层够起步 | VPS：需自运维，成本高 |
| 子站渲染 | **Astro + SSR on Workers** | 静态友好 + ISR 按需，SEO 最佳 | Next.js：客户端 JS 过重 |
| 仪表盘（M3+） | **Next.js 14 App Router** | React 生态 + shadcn/ui 快速产出 | Astro+HTMX：M1 Admin 够用，M3 复杂交互吃力 |
| 数据库 | **D1 (SQLite)** + **Vectorize** + **Analytics Engine** | D1 存业务，Vectorize 存 embedding，Analytics Engine 存事件日志。三者分工规避 D1 写入瓶颈 | 单 D1：触顶后迁移昂贵 |
| ORM | **Drizzle** | TS 原生、对 D1 支持好 | Prisma：CF Workers 支持差 |
| AI | **Anthropic Claude**（Haiku 4.5 / Sonnet 4.6 / Opus 4.7） | 中文能力、价格梯度、prompt caching、batch API | OpenAI：中文次之；国产：合规复杂 |
| 异步 | **CF Queues + Workflows** | Durable Execution，长任务拆片 | 同步 Worker：30s CPU 上限必爆 |
| 包管理 | **pnpm workspaces** | Monorepo 轻量、磁盘节省 | Turborepo：M1 无需 |
| 语言 | **TypeScript** strict | 全栈一致 + IDE 支持 | - |
| 测试 | **Vitest** + **VCR fixtures** | Workers 兼容、AI mock 友好 | Jest：Workers 支持差 |

---

## 3. 目录结构（pnpm monorepo）

```
SEOhub/
├── apps/
│   ├── admin/              M1 轻量管理后台（Astro + HTMX + Basic Auth）
│   ├── dashboard/          M3 引入（Next.js 14）
│   ├── orchestrator/       Hono on Workers，AI 任务调度与 HTTP 入口
│   ├── site-template/      子站 Astro 模板（3-5 个 variant）
│   └── main-blog/          主站 /blog 内容生产（M2 引入）
├── packages/
│   ├── budget/             AI 预算三层封顶 + Ledger
│   ├── content-gen/        主题→大纲→草稿→多层 QA
│   ├── seo/                schema.org / sitemap / robots / internal-link graph
│   ├── db/                 Drizzle schema + migration
│   ├── adapters/           外部 API 封装（Claude / CF / 5118 / GSC / Bing / IndexNow）
│   ├── slots/              推荐位引擎（位置计算 + UTM + CTR 追踪）
│   ├── testing/            VCR fixture 录制与回放
│   └── config/             站点垂类锁、模型路由、常量
├── infra/
│   ├── workflows/          Queues consumer + Workflow definitions
│   ├── cron/               定时任务（rank check, indexnow, budget reset）
│   └── migrations/         D1 迁移脚本
├── fixtures/               VCR 录制的 Claude 响应
├── docs/                   项目文档（本架构/MVP/API/DEV）
└── scripts/                开发脚本
```

---

## 4. 核心子系统

### 4.1 Budget 模块（packages/budget）

**职责：** 所有 AI 调用的必经守门人，三层封顶 + 精确记账。

#### 4.1.1 三层设计

```typescript
// Layer 1: Hard Cap — 超限立即阻断
interface HardCap {
  monthly_usd: number;      // 例 $260
  daily_usd: number;        // 例 $12
  per_task_usd: number;     // 例 $0.80
}

// Layer 2: Soft Cap + 自动降级
interface SoftCap {
  warn_at_percent: 80;      // 80% 告警（Resend 邮件）
  degrade_at_percent: 90;   // 90% 自动 Sonnet → Haiku
}

// Layer 3: Pre-flight 单任务估算
function estimate(input_tokens: number, expected_output: number, model: Model): USD {
  return input_tokens * model.input_price + expected_output * model.output_price;
}
// 每次调用前先算，超 per_task_usd 或日 budget 剩余 → 拒绝
```

#### 4.1.2 Ledger 维度

所有 AI 调用记录到 `ai_call_log` 表，字段：

```
id, ts, task_type, site_id, model, prompt_hash, cache_hit,
input_tokens, output_tokens, cost_usd, latency_ms, status, error
```

月度/周度汇总进 KV 缓存，面板查询不打 D1。

#### 4.1.3 Prompt Caching 强制约定

- 所有系统提示词 ≥ 1024 token 必须缓存
- 调用前计算 `prompt_hash`，同 hash 命中率记入 ledger
- 目标命中率：M2 末 ≥ 35%、M4 末 ≥ 50%

### 4.2 Orchestrator（apps/orchestrator）

**职责：** HTTP 入口 + 任务调度 + 状态机。

- Hono 框架（Workers 原生）
- 接收 Admin / Dashboard / Cron / Webhook 请求
- 派发到 Queues，**不同步等 AI 响应**
- 长任务走 Workflows（Durable Execution），单步 ≤ 25s

#### 关键路由

```
POST  /api/v1/sites              创建子站（触发 Workflow）
POST  /api/v1/contents           排队一篇新内容
POST  /api/v1/contents/:id/veto  运营 veto 批准/打回/删除
POST  /api/v1/decisions/:id      批准新站提案
GET   /api/v1/budget/status      当前预算状态
POST  /webhooks/cf-pages         Pages 部署回调
POST  /webhooks/gsc-alert        GSC 告警接收
```

### 4.3 Content Pipeline（packages/content-gen）

**职责：** 主题 → 大纲 → 草稿 → 多层 QA → 发布队列。

```
┌──────────────────┐
│ 1. Topic Intake  │ ← AI 提案或人工指定
├──────────────────┤
│ 2. Keyword Check │ ← 5118 API 验证搜索量 + 竞争度
├──────────────────┤
│ 3. Outline       │ ← Sonnet 生成结构化大纲
├──────────────────┤
│ 4. Draft         │ ← Haiku 批量生成正文（带 cache）
├──────────────────┤
│ 5. QA Layer A    │ ← 确定性规则（长度 / 关键词密度 / Schema）
├──────────────────┤
│ 6. QA Layer B    │ ← Sonnet 自评分（originality / EEAT）
├──────────────────┤
│ 7. QA Layer C    │ ← Vectorize embedding 去重（阈值 0.88）
├──────────────────┤
│ 8. Route         │ ← 分 ≥ 阈值 → 自动发布；< 阈值 → Veto 队列
├──────────────────┤
│ 9. Opus Pre-filter│ ← Veto 队列每周 Top 10 进运营面板
└──────────────────┘
```

**QA 评分加权（初版）：**
- 原创度 40%（embedding 距离 + 重复短语检测）
- EEAT 信号 30%（作者页、参考来源、结构化数据完整度）
- 可读性 20%（Sonnet 打分）
- Schema 完整度 10%

< 70 分进 Veto 队列，< 40 分直接丢弃。

### 4.4 推荐位引擎（packages/slots）

**职责：** 在子站页面渲染时计算应激活哪些推荐位、注入 UTM、追踪 CTR。

#### 10 个候选槽位定义

| Slot ID | 位置 | 默认激活 | 条件 |
|---|---|---|---|
| `nav.recommend` | 顶导 "推荐" 入口 | 否 | 仅聚合页 |
| `sidebar.partner` | 侧栏合作伙伴卡片 | 是 | 所有页 |
| `footer.friend` | 页脚友情链接 | 是 | 所有页 |
| `article.inline` | 正文中 AI 判断相关位置 | 条件激活 | Sonnet 判相关性 > 0.7 |
| `article.end` | 文末"相关推荐"区 | 是 | 所有文章页 |
| `article.sticky` | 滚动 50% sticky 侧栏 | 否 | 长文（> 1800 字） |
| `list.mixin` | 列表页混入主站条目 | 是 | 聚合/标签页 |
| `page.partner-block` | 专区"合作伙伴" | 否 | /partners 专页 |
| `popup.welcome` | 首次访问 toast | 否 | 全局开关，M4 实验 |
| `popup.exit-intent` | 离站意图弹窗 | **禁用** | 百度降权风险 |

#### 激活规则（硬约束）

```typescript
interface SlotActivationRules {
  maxSponsoredPerPage: 2;
  sponsoredOnlyPositions: ['article.inline', 'article.end'];
  // 其他位置用 rel="nofollow" 非 sponsored（主站非赞助关系时）

  articleCoverageRatio: 0.7;    // 70% 文章挂位，30% 纯净
  siteOutboundToMainRatio: 0.18; // 站级指向主站占全出站 ≤ 18%

  anchorTextVariants: string[];  // ≥ 10 个变体
  maxBrandedAnchorRatio: 0.4;    // 纯品牌锚文本 ≤ 40%

  autoRetire: {
    ctrThreshold: 0.005;          // CTR < 0.5%
    windowDays: 14;               // 连续 14 天
  };
}
```

#### UTM 模板

```
?utm_source={site_slug}
 &utm_medium=seohub_slot
 &utm_campaign={slot_id}
 &utm_content={article_slug}
```

点击落在 CF Workers 的 `/go` 路径，301 到主站 + 写 Analytics Engine 事件。

### 4.5 SEO 基础设施（packages/seo）

- **schema.org 生成器：** Article / BreadcrumbList / FAQPage / HowTo / Person（作者） / RecommendationList
- **sitemap：** 按站分片（> 5 万 URL 必拆），每站 `/sitemap.xml` + 子分片
- **robots.txt：** 按站独立，明确允许主流 + 百度 spider
- **canonical / hreflang：** 每页强制注入
- **IndexNow：** 发布/更新时自动推送 Bing + Yandex
- **GSC / Bing Webmaster API：** 每日同步索引状态、手动处罚告警到 `index_status` 表
- **内链图谱：** 每次发布重新计算站内链接矩阵，保证入链分布均匀（无孤儿页）

### 4.6 Admin（M1）与 Dashboard（M3）

**M1 Admin（apps/admin）：**
- Astro 静态 + HTMX，CF Pages 部署，Basic Auth 保护
- 页面：Veto 队列 / Budget 状态 / Site 列表（只读）
- 目标：省掉 M1 前端复杂度

**M3 Dashboard（apps/dashboard）：**
- Next.js 14 App Router + shadcn/ui + TanStack Query
- 5 页面：Overview / Site Matrix / Content Pipeline / Budget / Decision Center
- 支持批量审批、A/B 实验配置、告警静音

---

## 5. 数据模型

### 5.1 D1 表结构（Day-1 必建，不等 M2 再加）

```sql
-- 站点元数据
CREATE TABLE sites (
  id           TEXT PRIMARY KEY,
  slug         TEXT UNIQUE NOT NULL,
  domain       TEXT UNIQUE NOT NULL,
  vertical     TEXT NOT NULL,           -- 垂类锁
  template     TEXT NOT NULL,           -- 视觉模板 ID
  status       TEXT NOT NULL,           -- draft / live / archived
  created_at   INTEGER NOT NULL,
  updated_at   INTEGER NOT NULL
);

-- 垂类锁（防止跨主题漂移）
CREATE TABLE site_vertical_locks (
  site_id       TEXT PRIMARY KEY REFERENCES sites(id),
  vertical      TEXT NOT NULL,
  allowed_topics JSON NOT NULL,         -- 允许的语义图谱
  forbidden_patterns JSON NOT NULL
);

-- 内容
CREATE TABLE contents (
  id            TEXT PRIMARY KEY,
  site_id       TEXT NOT NULL REFERENCES sites(id),
  slug          TEXT NOT NULL,
  title         TEXT NOT NULL,
  body_md       TEXT NOT NULL,
  status        TEXT NOT NULL,         -- draft / veto_pending / approved / published / rejected / archived
  qa_score      REAL,
  qa_breakdown  JSON,
  published_at  INTEGER,
  created_at    INTEGER NOT NULL,
  updated_at    INTEGER NOT NULL,
  UNIQUE(site_id, slug)
);

-- 内容版本（审计 + 回滚）
CREATE TABLE content_revisions (
  id          TEXT PRIMARY KEY,
  content_id  TEXT NOT NULL REFERENCES contents(id),
  rev_num     INTEGER NOT NULL,
  body_md     TEXT NOT NULL,
  author      TEXT NOT NULL,           -- model id or 'operator'
  qa_score    REAL,
  created_at  INTEGER NOT NULL
);

-- URL 历史（slug 变更）
CREATE TABLE url_history (
  id         TEXT PRIMARY KEY,
  site_id    TEXT NOT NULL REFERENCES sites(id),
  old_path   TEXT NOT NULL,
  new_path   TEXT NOT NULL,
  changed_at INTEGER NOT NULL
);

-- 301/302 重定向规则
CREATE TABLE redirect_rules (
  id         TEXT PRIMARY KEY,
  site_id    TEXT NOT NULL REFERENCES sites(id),
  from_path  TEXT NOT NULL,
  to_path    TEXT NOT NULL,
  status     INTEGER NOT NULL,         -- 301 / 302
  enabled    INTEGER NOT NULL DEFAULT 1
);

-- 索引/排名状态
CREATE TABLE index_status (
  id              TEXT PRIMARY KEY,
  content_id      TEXT NOT NULL REFERENCES contents(id),
  engine          TEXT NOT NULL,       -- google / bing / baidu
  indexed         INTEGER NOT NULL,    -- 0/1
  rank_position   INTEGER,
  top_keyword     TEXT,
  checked_at      INTEGER NOT NULL
);

-- AI 调用明细（Budget Ledger）
CREATE TABLE ai_call_log (
  id              TEXT PRIMARY KEY,
  ts              INTEGER NOT NULL,
  task_type       TEXT NOT NULL,
  site_id         TEXT REFERENCES sites(id),
  content_id      TEXT REFERENCES contents(id),
  model           TEXT NOT NULL,
  prompt_hash     TEXT NOT NULL,
  cache_hit       INTEGER NOT NULL,
  input_tokens    INTEGER NOT NULL,
  output_tokens   INTEGER NOT NULL,
  cost_usd        REAL NOT NULL,
  latency_ms      INTEGER NOT NULL,
  status          TEXT NOT NULL,
  error           TEXT
);

-- Veto 队列（逻辑表：contents.status = 'veto_pending' 即属此队列）
-- Opus 预过滤结果
CREATE TABLE veto_priorities (
  content_id   TEXT PRIMARY KEY REFERENCES contents(id),
  priority     INTEGER NOT NULL,       -- 0-100, 越高越优先审
  reason       TEXT NOT NULL,
  ranked_at    INTEGER NOT NULL
);

-- 新站提案
CREATE TABLE site_proposals (
  id              TEXT PRIMARY KEY,
  vertical        TEXT NOT NULL,
  pitch           TEXT NOT NULL,
  keyword_data    JSON NOT NULL,
  estimated_cost  REAL,
  status          TEXT NOT NULL,      -- pending / approved / rejected
  created_at      INTEGER NOT NULL,
  decided_at      INTEGER,
  decided_by      TEXT
);

-- 推荐位 CTR 追踪（只存聚合，明细在 Analytics Engine）
CREATE TABLE slot_performance (
  id              TEXT PRIMARY KEY,
  site_id         TEXT NOT NULL REFERENCES sites(id),
  slot_id         TEXT NOT NULL,
  day             TEXT NOT NULL,        -- YYYY-MM-DD
  impressions     INTEGER NOT NULL,
  clicks          INTEGER NOT NULL,
  ctr             REAL NOT NULL,
  UNIQUE(site_id, slot_id, day)
);

-- 站级推荐位配额
CREATE TABLE sponsored_link_budget (
  site_id             TEXT PRIMARY KEY REFERENCES sites(id),
  total_outbound      INTEGER NOT NULL DEFAULT 0,
  sponsored_outbound  INTEGER NOT NULL DEFAULT 0,
  ratio               REAL GENERATED ALWAYS AS (
    CASE WHEN total_outbound = 0 THEN 0
         ELSE CAST(sponsored_outbound AS REAL) / total_outbound END
  ) VIRTUAL,
  updated_at          INTEGER NOT NULL
);

-- A/B 实验
CREATE TABLE experiments (
  id            TEXT PRIMARY KEY,
  name          TEXT NOT NULL,
  site_id       TEXT REFERENCES sites(id),
  variants      JSON NOT NULL,
  started_at    INTEGER NOT NULL,
  ended_at      INTEGER,
  winner        TEXT
);

-- 运营告警
CREATE TABLE alerts (
  id         TEXT PRIMARY KEY,
  type       TEXT NOT NULL,            -- budget / manual_action / dead_link / index_drop
  severity   TEXT NOT NULL,            -- info / warn / critical
  payload    JSON NOT NULL,
  ack_at     INTEGER,
  created_at INTEGER NOT NULL
);
```

### 5.2 Analytics Engine 事件

所有高频事件（不进 D1）：

- `slot_impression` — 推荐位曝光
- `slot_click` — 推荐位点击
- `article_view` — 文章浏览（辅助，以 GA/Cloudflare Web Analytics 为主）
- `ai_cache_hit` — Prompt cache 命中情况（成本优化分析）

### 5.3 Vectorize 索引

```
index: content-embeddings
dimensions: 1024 (Cohere embed-v3) 或 1536 (OpenAI text-embedding-3-small)
metric: cosine
```

每篇新内容生成 embedding 后 upsert，去重阈值 0.88。

### 5.4 KV 配置

```
budget:current                   JSON，实时预算快照
budget:daily:YYYY-MM-DD          JSON，日汇总
rate_limit:claude:minute         INTEGER
rate_limit:5118:day              INTEGER
site:{slug}:config               JSON，站运行时配置
prompt_cache:{hash}:meta         JSON，缓存元数据
```

---

## 6. 数据流

### 6.1 内容生产链

```
Cron / Admin 触发
  → Orchestrator HTTP
  → Queue: content.create
  → Workflow: Topic → Keyword → Outline → Draft → QA
    - 每步走 Budget.preflight() 检查
    - 每次调用写 ai_call_log
  → 分数路由：auto-publish / veto-queue / discard
  → Publish Workflow: Pages API → IndexNow → 更新 index_status
```

### 6.2 决策链（新站提案）

```
Cron: 双周触发
  → Sonnet: 生成 3-5 个垂类候选
  → 5118 + Bing Webmaster: 验证搜索量/竞争度
  → 存入 site_proposals
  → Resend 告警：运营待审
  → 运营批准 → Workflow: 买域名 → 初始化仓库 → 部署首版
```

### 6.3 发布链

```
内容 approved
  → Workflow: publish
    1. 写 contents.status = 'published'
    2. 调用 Pages deploy webhook（按需渲染，非全量 build）
    3. IndexNow 推送
    4. GSC / Bing Webmaster API 提交
    5. 更新 index_status
    6. 计算内链图谱差量
    7. 更新 sitemap 分片
```

---

## 7. 外部集成

详见 [API.md](API.md#外部-api)。关键点：

- **Claude**：全部调用过 `packages/budget`；使用 prompt caching（> 1024 token 前缀）；非实时用 batch API
- **Cloudflare**：Pages Deployments API、D1 bindings、Workers Secrets、Registrar API
- **5118**：关键词挖掘、竞争度查询，速率 KV 限流
- **GSC API**：每日拉取索引状态、查询性能数据
- **Bing Webmaster API**：每日拉取索引 + 提交 sitemap
- **IndexNow**：发布即推送，支持 Bing / Yandex
- **Unsplash / Replicate**：文章配图，先 Unsplash，不达标或不够再 Flux schnell
- **Resend**：运营告警邮件，免费层 3k/月
- **Sentry**：错误追踪（Lean 档位用自建日志替代）

---

## 8. 部署模型

### 8.1 CF Pages / Workers 拓扑

- **每个子站 = 独立 CF Pages project**，绑定独立域名
- **orchestrator / admin / dashboard** 各一个 Workers/Pages 部署
- **Queues / Workflows** 共用一套（account-level）
- 所有 Secrets 通过 Wrangler `secret put`，不入仓库

### 8.2 域名隔离

采纳 MVP.md D6 默认值：**每个子站独立顶级域名**，不使用主域子域，避免惩罚传导。CF Registrar 批量购买。

### 8.3 环境分层

| 环境 | D1 | Domain | 用途 |
|---|---|---|---|
| local | SQLite file | localhost:* | 开发 |
| staging | D1 `seohub-staging` | *.pages.dev | CI/预发 |
| production | D1 `seohub-prod` | 独立域名 | 线上 |

---

## 9. 测试策略

### 9.1 层级

- **单元测试（Vitest）：** 纯函数、schema 生成、slot 规则、budget 估算
- **集成测试（Vitest + Miniflare）：** Workers + D1 + KV 本地模拟
- **AI 管线测试（VCR）：** 所有 Claude 调用在 CI mock，使用 `packages/testing` 录制的 fixture 回放
- **E2E（Playwright）：** 仅覆盖关键用户路径（Veto 审批、站点创建）

### 9.2 VCR fixture 约定

- 录制：本地跑时设 `VCR_MODE=record`，真实打 API 并存 `fixtures/`
- 回放：CI 默认 `VCR_MODE=replay`，无 API key 也能跑
- 失败：fixture 未命中视为测试失败（不允许回放时新增调用）

### 9.3 CI gate

```
lint → typecheck → unit → integration (with mock) → build
```

**禁止** 在 CI 中使用真实 Anthropic / 5118 / GSC API key。

---

## 10. 扩展极限与退化策略

| 指标 | 当前栈上限 | 触发后动作 |
|---|---|---|
| 站点数 | ~20-30 | 拆 account 或迁多租户架构 |
| 文章数/站 | ~500 | sitemap 分片，更激进内链剪枝 |
| D1 写入 | 1000 万/月（Paid） | 点击/UTM 全进 Analytics Engine，D1 只留业务主记录 |
| Workers 请求 | 1000 万/月（Paid） | 加 KV 缓存层，边缘命中 > 80% |
| Pages Builds | 5000/月（Paid） | 全面 ISR，取消 full rebuild |
| Claude 成本 | 预算硬封顶 | Soft cap 自动降级 Sonnet→Haiku；Hard cap 停产出 |

---

## 11. 安全与合规

- 所有 Secrets 通过 Wrangler Secret 管理，不进代码/git
- Admin / Dashboard 使用 Basic Auth（M1）或 Cloudflare Access（M3+）
- 出站链接全部 `rel="sponsored noopener"`（推荐位）或 `rel="nofollow noopener"`（常规外链）
- 用户数据（邮箱订阅，如启用）加密存储，遵循 GDPR 基本规范
- 不收集 PII 用于二次销售

---

## 12. 与 MVP.md 的同步

本文档的任何架构变更必须同步更新 MVP.md 第 §9 里程碑表 + §10 删除清单。详见 [DEVELOPMENT.md § 文档同步规范](DEVELOPMENT.md#文档同步规范)。
