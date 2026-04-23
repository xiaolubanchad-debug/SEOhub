# SEOhub API 文档

> 版本：v1.0 · 最后同步：2026-04-23
> 与 [ARCHITECTURE.md](ARCHITECTURE.md) v1.0 对齐
>
> 本文档分三部分：
> - **§2 内部 API** — monorepo 包之间的 TypeScript 契约
> - **§3 外部 API** — 消费的第三方 API（Anthropic、CF、5118、GSC、Bing、IndexNow 等）
> - **§4 控制面 API** — Admin / Dashboard / Webhook HTTP 端点

---

## 1. API 概览

| 类别 | 实现位置 | 对外？ | 鉴权 |
|---|---|---|---|
| 内部 TS 契约 | `packages/*` 导出 | 否（monorepo 内） | 不需要 |
| 控制面 HTTP | `apps/orchestrator` | 对 Admin/Dashboard 开放 | Basic Auth (M1) / CF Access (M3+) |
| Webhook 接收 | `apps/orchestrator/webhooks` | 对 CF / GSC / Bing 开放 | Signature 校验 |
| 外部 API 消费 | `packages/adapters` 封装 | - | API key / OAuth |

全局约定：
- Content-Type: `application/json` unless noted
- 时间戳：Unix epoch ms（`number`），不用 ISO 字符串
- 货币：USD，浮点，保留 4 位小数
- ID：UUIDv7（`type Id = string`）或带前缀短 ID（`cnt_*` / `site_*` / `dec_*`）
- 错误响应：`{ error: { code: string, message: string, details?: object } }`

---

## 2. 内部 API（packages 间契约）

### 2.1 `@seohub/budget`

**核心导出：**

```typescript
export type Model = 'haiku-4-5' | 'sonnet-4-6' | 'opus-4-7';

export type TaskType =
  | 'outline' | 'draft' | 'qa_score' | 'qa_factcheck'
  | 'keyword_research' | 'site_proposal' | 'veto_prefilter'
  | 'embedding' | 'internal_link_graph';

export interface CallClaudeOptions {
  taskType: TaskType;
  siteId?: string;
  contentId?: string;
  model: Model;
  messages: Array<{ role: 'user' | 'assistant'; content: string }>;
  system?: string;           // 自动 cache 若 ≥ 1024 token
  maxOutputTokens?: number;  // 预算预估必需
  metadata?: Record<string, string>;
}

export interface CallClaudeResult {
  text: string;
  usage: {
    input_tokens: number;
    output_tokens: number;
    cache_read_input_tokens: number;
    cache_creation_input_tokens: number;
  };
  costUsd: number;
  cacheHit: boolean;
  model: Model;
  latencyMs: number;
}

// 主入口：所有 Claude 调用的唯一出口
export function callClaude(opts: CallClaudeOptions): Promise<CallClaudeResult>;

// 预算状态查询
export function getBudgetStatus(): Promise<BudgetStatus>;

// 预算配置（Wrangler secret 注入）
export function getBudgetConfig(): BudgetConfig;

// 成本预估（不调用 API）
export function estimateCost(
  inputTokens: number,
  expectedOutputTokens: number,
  model: Model,
  cacheHitRatio?: number
): number;
```

**行为契约：**

- `callClaude()` 在调用前执行 pre-flight：
  1. 估算成本（基于 `messages` token + `maxOutputTokens` + 缓存命中历史均值）
  2. 对比 `budget.perTask` / `budget.remainingToday` / `budget.remainingMonth`
  3. 任一超限 → throw `BudgetExceededError`
- 成功响应写入 `ai_call_log`
- 失败 / 超时：指数退避重试 3 次，失败后 throw `ClaudeCallError`
- 自动降级：若当前模型被 soft cap 禁用，改为下一档便宜模型并在返回中标注 `degraded: true`

---

### 2.2 `@seohub/content-gen`

```typescript
export interface Topic {
  siteId: string;
  keyword: string;
  intent: 'informational' | 'navigational' | 'commercial' | 'transactional';
  competitor_urls?: string[];
  forced?: boolean;  // 绕过垂类锁（仅管理员）
}

export interface Outline {
  title: string;
  h2_sections: Array<{ heading: string; h3?: string[]; notes: string }>;
  target_word_count: number;
  schema_types: string[];  // e.g. ['Article', 'FAQPage']
}

export interface DraftResult {
  contentId: string;
  body_md: string;
  wordCount: number;
  references: Array<{ url: string; title: string }>;
}

export interface QAResult {
  score: number;           // 0-100
  breakdown: {
    originality: number;
    eeat: number;
    readability: number;
    schema_completeness: number;
  };
  flags: string[];
  decision: 'auto_publish' | 'veto_queue' | 'discard';
}

// 管线入口
export function planTopic(siteId: string): Promise<Topic | null>;
export function generateOutline(topic: Topic): Promise<Outline>;
export function generateDraft(outline: Outline, siteId: string): Promise<DraftResult>;
export function runQa(contentId: string): Promise<QAResult>;
export function runFullPipeline(siteId: string): Promise<string /* contentId */>;
```

---

### 2.3 `@seohub/seo`

```typescript
export function generateSitemap(siteId: string): Promise<string /* XML */>;
export function generateRobots(siteId: string): string;
export function generateSchemaJsonLd(
  content: ContentRecord,
  site: SiteRecord
): object[];  // schema.org objects

export interface InternalLinkGraph {
  nodes: Array<{ contentId: string; slug: string }>;
  edges: Array<{ from: string; to: string; anchor: string }>;
  orphans: string[];       // 没入链的文章
  coverage: number;        // 0-1
}

export function computeInternalLinkGraph(siteId: string): Promise<InternalLinkGraph>;
export function suggestInternalLinks(
  contentId: string,
  graph: InternalLinkGraph
): Array<{ anchor: string; targetContentId: string }>;

// IndexNow
export function pushIndexNow(urls: string[]): Promise<void>;

// 外链预算
export function checkSponsoredBudget(
  siteId: string,
  willAdd: number
): Promise<{ allowed: boolean; reason?: string }>;
```

---

### 2.4 `@seohub/slots`

```typescript
export type SlotId =
  | 'nav.recommend' | 'sidebar.partner' | 'footer.friend'
  | 'article.inline' | 'article.end' | 'article.sticky'
  | 'list.mixin' | 'page.partner-block'
  | 'popup.welcome' | 'popup.exit-intent';

export interface SlotContext {
  siteId: string;
  pageType: 'home' | 'article' | 'list' | 'page';
  contentId?: string;
  wordCount?: number;
  semanticRelevance?: number;  // 0-1
}

export interface ActivatedSlot {
  slotId: SlotId;
  rel: 'sponsored' | 'nofollow';
  href: string;        // 主站 URL + UTM
  anchor: string;      // 轮换锚文本
  label: string;       // "推荐" / "合作"
}

// 核心决策：本次渲染激活哪些位
export function decideSlots(ctx: SlotContext): Promise<ActivatedSlot[]>;

// CTR 追踪（写 Analytics Engine）
export function trackImpression(slotId: SlotId, ctx: SlotContext): void;
export function trackClick(slotId: SlotId, ctx: SlotContext, utm: object): void;

// 自动下线低 CTR
export function retireLowPerformers(): Promise<number /* retired count */>;
```

**决策规则实现：** 见 [ARCHITECTURE.md §4.4](ARCHITECTURE.md#44-推荐位引擎packagesslots)。

---

### 2.5 `@seohub/db`

Drizzle schema 导出 + helpers：

```typescript
import { drizzle } from 'drizzle-orm/d1';
import { sites, contents, aiCallLog, /* ... */ } from '@seohub/db/schema';

export function getDb(env: Env): ReturnType<typeof drizzle>;

// 常用查询（避免手写 SQL 分散）
export async function getVetoQueue(db, siteId?: string): Promise<ContentRecord[]>;
export async function getBudgetLedger(db, range: DateRange): Promise<LedgerRow[]>;
export async function getSiteKpis(db, siteId: string): Promise<SiteKpis>;
```

**迁移：** `infra/migrations/*.sql` + `wrangler d1 migrations apply`。

---

### 2.6 `@seohub/adapters`

每个外部服务一个子模块，统一接口：

```typescript
// packages/adapters/src/anthropic.ts
export { anthropicClient };

// packages/adapters/src/cloudflare.ts
export interface CFClient {
  pages: { createProject, deploy, getProject };
  registrar: { searchDomain, register, listDomains };
  d1: { query, execute };
}

// packages/adapters/src/five118.ts（5118 API）
export interface Five118Client {
  keywordDifficulty(kw: string): Promise<{ kd: number; volume: number }>;
  longTailExpansion(seed: string, limit?: number): Promise<string[]>;
  competitorAnalysis(domain: string): Promise<CompetitorReport>;
}

// packages/adapters/src/gsc.ts
// packages/adapters/src/bing-webmaster.ts
// packages/adapters/src/indexnow.ts
// packages/adapters/src/unsplash.ts
// packages/adapters/src/replicate.ts
// packages/adapters/src/resend.ts
```

**约定：**
- 每个 adapter 支持 `mock` 模式（走 VCR fixture）
- 每个 adapter 有速率限制器（KV 存 token bucket）
- 错误统一包装为 `AdapterError`，带 `source` 字段

---

## 3. 外部 API（消费契约）

### 3.1 Anthropic Claude

- 端点：`https://api.anthropic.com/v1/messages`
- Header：
  ```
  x-api-key: {ANTHROPIC_API_KEY}
  anthropic-version: 2023-06-01
  anthropic-beta: prompt-caching-2024-07-31
  ```
- 模型 ID：
  - `claude-haiku-4-5-20251001`
  - `claude-sonnet-4-6`
  - `claude-opus-4-7`
- Batch API：非实时任务（QA / 关键词研究）使用 `/v1/messages/batches`
- 价格（2026-04 快照，仅参考，以官方为准）：

| Model | Input / 1M tok | Output / 1M tok | Cache Write | Cache Read |
|---|---|---|---|---|
| Haiku 4.5 | \$1.00 | \$5.00 | 1.25× input | 0.10× input |
| Sonnet 4.6 | \$3.00 | \$15.00 | 1.25× input | 0.10× input |
| Opus 4.7 | \$15.00 | \$75.00 | 1.25× input | 0.10× input |

（`packages/config/models.ts` 作为 source of truth，可热更新）

---

### 3.2 Cloudflare

- **Pages API：** `https://api.cloudflare.com/client/v4/accounts/{account_id}/pages/projects`
  - 创建 Pages project、触发部署、查询 deployment 状态
- **D1 API：** 通过 Wrangler binding（Workers 内直接查询）
- **Registrar API：** `https://api.cloudflare.com/client/v4/accounts/{account_id}/registrar/domains`
  - 搜索 / 注册 / 续费域名
- **Workers KV / R2 / Vectorize：** 通过 bindings
- **Analytics Engine：** `writeDataPoint()` via binding
- **Queues：** `env.MY_QUEUE.send()` via binding

Secrets：
- `CF_API_TOKEN`（权限：Pages Edit + Workers Edit + DNS Edit + Registrar Edit）
- `CF_ACCOUNT_ID`

---

### 3.3 5118

- 端点：`https://apis.5118.com/`
- 鉴权：Header `Authorization: APIKEY {WU118_API_KEY}`
- 主要方法：
  - `/keyword/word` — 关键词难度
  - `/keyword/longtail` — 长尾扩展
  - `/competitor/report` — 竞品分析
- 速率：根据套餐（通常 500-2000 次/日）
- 本项目在 `packages/adapters/five118.ts` 封装，KV token bucket 限流

---

### 3.4 Google Search Console API

- OAuth2 授权（user consent），refresh token 存 CF Secret
- 端点：`https://www.googleapis.com/webmasters/v3/sites/{siteUrl}/searchAnalytics/query`
- 主要调用：
  - `urlInspection.index.inspect` — 单页索引状态
  - `searchanalytics.query` — 查询排名数据
  - `sitemaps.submit` — 提交 sitemap
- 本项目每日 cron 同步每站核心页的索引状态到 `index_status` 表

---

### 3.5 Bing Webmaster API

- 端点：`https://ssl.bing.com/webmaster/api.svc/json/`
- 鉴权：API Key（header `apikey`）
- 主要调用：
  - `SubmitUrl` / `SubmitUrlBatch`
  - `GetUrlInfo`
  - `GetRankAndTrafficStats`
- IndexNow 推送统一走独立 adapter（见下）

---

### 3.6 IndexNow

- 端点：`https://api.indexnow.org/indexnow`
- 鉴权：URL 参数 `key={INDEXNOW_KEY}`（需在每站根目录放 `{key}.txt` 验证）
- 主要调用：`POST /indexnow` with JSON `{ host, key, keyLocation, urlList }`
- 覆盖 Bing + Yandex（Google 不接 IndexNow）
- 本项目发布 / 更新文章时自动推送

---

### 3.7 Unsplash

- 端点：`https://api.unsplash.com/`
- 鉴权：Header `Authorization: Client-ID {UNSPLASH_ACCESS_KEY}`
- 主要调用：`/search/photos`
- 速率：免费层 50 req/hr（超了用 Replicate Flux 兜底）

---

### 3.8 Replicate（Flux schnell）

- 端点：`https://api.replicate.com/v1/predictions`
- 鉴权：Header `Authorization: Token {REPLICATE_API_TOKEN}`
- 价格：Flux schnell ~\$0.003 / image
- 仅作为 Unsplash 匹配度低时的兜底

---

### 3.9 Resend

- 端点：`https://api.resend.com/emails`
- 鉴权：Header `Authorization: Bearer {RESEND_API_KEY}`
- 用途：运营告警邮件（预算超限、veto 队列积压、手动处罚、部署失败）
- 模板约定：`packages/adapters/resend/templates/*.tsx`（React Email）

---

### 3.10 Sentry（可选，Balanced 档位以上）

- DSN 通过 `SENTRY_DSN` 配置
- 只接错误 + 关键 transaction
- 生产环境 sample rate：errors 100% / transactions 10%

---

## 4. 控制面 HTTP API（apps/orchestrator）

基础路径：`https://orchestrator.seohub.workers.dev/api/v1`

鉴权（M1）：Basic Auth `Authorization: Basic base64(admin:pass)`
鉴权（M3+）：Cloudflare Access JWT

### 4.1 Health

```
GET  /api/v1/health
→ 200 { status: 'ok', version, uptime_s, budget: { status } }
```

### 4.2 Sites

```
GET    /api/v1/sites
GET    /api/v1/sites/:siteId
POST   /api/v1/sites                  { vertical, slug, template? }   → 202 触发 Workflow
PATCH  /api/v1/sites/:siteId          { status? template? }
DELETE /api/v1/sites/:siteId          → archive，不删除数据
GET    /api/v1/sites/:siteId/kpis     → 索引 / 排名 / UV / CTR
```

### 4.3 Contents

```
GET    /api/v1/contents               ?site=&status=&limit=&cursor=
GET    /api/v1/contents/:contentId
POST   /api/v1/contents               { siteId, topic? } → 202 排队
POST   /api/v1/contents/:contentId/veto    { action: 'approve'|'reject'|'regenerate' }
POST   /api/v1/contents/:contentId/publish    → 强制发布（跳过 veto）
GET    /api/v1/contents/:contentId/revisions
POST   /api/v1/contents/:contentId/rollback   { to_rev }
```

### 4.4 Decisions（新站提案）

```
GET    /api/v1/decisions                  ?status=pending
GET    /api/v1/decisions/:id
POST   /api/v1/decisions/:id/approve
POST   /api/v1/decisions/:id/reject       { reason }
POST   /api/v1/decisions/propose          (内部 cron 调用，运营不用)
```

### 4.5 Budget

```
GET    /api/v1/budget/status
  → { period, spent_usd, remaining_usd, projection, by_site[], by_task[], by_model[] }

GET    /api/v1/budget/ledger              ?from=&to=&site=&task=
POST   /api/v1/budget/config              { hardCap, softCap, perTask }   → 运营调整
```

### 4.6 Slots

```
GET    /api/v1/slots/performance          ?site=&slot=&days=30
POST   /api/v1/slots/anchor-variants      { siteId, variants: [] }
POST   /api/v1/slots/retire               { slotId, reason }
```

### 4.7 Alerts

```
GET    /api/v1/alerts                     ?severity=&ack=
POST   /api/v1/alerts/:id/ack
```

### 4.8 Experiments（M4+）

```
GET    /api/v1/experiments
POST   /api/v1/experiments                { name, site, variants }
POST   /api/v1/experiments/:id/stop       { winner? }
```

---

## 5. Webhook 接收（apps/orchestrator/webhooks）

基础路径：`https://orchestrator.seohub.workers.dev/webhooks`

### 5.1 CF Pages Deployment

```
POST /webhooks/cf-pages
Header: CF-Webhook-Signature: {hmac}
Body:  { project, deployment_id, status, url }

行为：更新 sites.status，触发 IndexNow 推送（若 status=success）
```

### 5.2 预算告警（内部）

```
POST /webhooks/budget-alert
触发：budget module 内部调用
行为：写 alerts 表，发 Resend 邮件
```

### 5.3 GSC 手动处罚 Polling

GSC 不主动推 webhook，我们用 cron polling 代替：

```
Cron: hourly
行为：拉 GSC 手动处罚列表 → diff → 新增处罚写 alerts + Resend
```

### 5.4 5118 / Bing Webmaster

均为 polling（cron），非 webhook。

---

## 6. 错误码约定

全局错误码前缀：`{SCOPE}_{REASON}`

```
BUDGET_EXCEEDED         硬封顶超限
BUDGET_DEGRADED         软封顶触发降级（info，非错误）
CLAUDE_RATE_LIMIT       Anthropic 限速
CLAUDE_INVALID_RESPONSE
CF_PAGES_DEPLOY_FAILED
GSC_AUTH_FAILED
SLOT_CAP_VIOLATED       请求激活的 slot 超过站级上限
VERTICAL_LOCK_VIOLATED  话题跨越垂类锁
QA_BELOW_DISCARD        QA 分数低于 40，丢弃
VETO_QUEUE_FULL         Veto 队列积压超限（运营未处理）
```

HTTP 状态码：
- `200 OK` — 成功同步响应
- `202 Accepted` — 异步任务已排队
- `400 Bad Request` — 客户端错误
- `401 / 403` — 鉴权失败
- `409 Conflict` — 状态冲突（如重复发布）
- `429 Too Many Requests` — 速率限制
- `5xx` — 服务端错误

---

## 7. 速率与限制

| 来源 | 限制 | 本项目实现 |
|---|---|---|
| Anthropic | TPM / RPM 按账户 | Budget 模块内部限流 |
| 5118 | 500-2000 次/日 | KV token bucket |
| CF Registrar | ~无 | 不限流 |
| GSC | 1200 QPM | 批量调用 |
| Bing Webmaster | 1000 /min | 批量调用 |
| IndexNow | 10000 URL / 请求 | 本项目每次 ≤ 100 URL |
| Unsplash | 50 req/hr (free) | KV token bucket + Flux 兜底 |

---

## 8. 变更记录

| 版本 | 日期 | 变更 | 关联里程碑 |
|---|---|---|---|
| v1.0 | 2026-04-23 | 首版，定义 M1-M6 全量 API 契约 | Pre-M1 |

> 每次 API 新增 / 变更必须在此表追加一行，详见 [DEVELOPMENT.md § 文档同步规范](DEVELOPMENT.md#文档同步规范)。
