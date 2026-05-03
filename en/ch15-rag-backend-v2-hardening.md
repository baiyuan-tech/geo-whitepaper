---
title: "Chapter 15 — rag-backend-v2 Hardening: Defensive Design for Six Layers of LLM Hallucination Failure Modes"
description: "Engineering hardening history of the central RAG microservice (rag-backend-v2): from Wiki Cascade four-tier cascade, six wikiCompiler hallucination failure mode patches (cutoff / partial UUID / no marker / DNS / LLM timeout / reasoning maxTokens), to Queue retention auto-cleanup — full robustness design practices for LLM-driven systems."
chapter: 15
part: 5
word_count: 6100
lang: en
authors:

  - name: Vincent Lin
    affiliation: Baiyuan Technology
license: CC-BY-NC-4.0
keywords:

  - rag-backend-v2
  - LLM Wiki Compiler
  - Wiki Cascade
  - LLM Hallucination Hardening
  - Docker DNS
  - LLM Timeout
  - AbortSignal
  - BullMQ Retention
  - PostgreSQL pgvector
last_updated: 2026-05-03
canonical: https://baiyuan.io/whitepaper/en/ch15-rag-backend-v2-hardening
last_modified_at: '2026-05-03T05:32:38Z'
---



# Chapter 15 — rag-backend-v2 Hardening: Defensive Design for Six Layers of LLM Hallucination Failure Modes

> In a world where LLMs can't be trusted, engineering's job isn't to force LLMs to be perfect — it's to prepare an exit for every possible failure mode.

## Table of Contents

- [15.1 Why RAG Needs an Independent Microservice + Self-Built Wiki](#151-why-rag-needs-an-independent-microservice--self-built-wiki)
- [15.2 Wiki Cascade: Four-Tier Cascade Deletion](#152-wiki-cascade-four-tier-cascade-deletion)
- [15.3 Six wikiCompiler Hallucination Failures + Corresponding Patches](#153-six-wikicompiler-hallucination-failures--corresponding-patches)
- [15.4 Worker Robustness: From Silent Hang to Auto-Recovery](#154-worker-robustness-from-silent-hang-to-auto-recovery)
- [15.5 10K-Tenant Deployment Pattern](#155-10k-tenant-deployment-pattern)
- [15.6 Observations and Open Problems](#156-observations-and-open-problems)
- [15.7 Engineering Lessons](#157-engineering-lessons)

---

## 15.1 Why RAG Needs an Independent Microservice + Self-Built Wiki

### 15.1.1 Karpathy's LLM Wiki Concept

Andrej Karpathy publicly shared in 2024 the concept: traditional RAG feeding chunks directly to LLMs is too "fragmented" without structure. Better to first compile all source documents into wiki using LLM, where each wiki has clear slug / page_type / TLDR / tags. On query, look at wiki first (fast, cheap, explainable); fallback to chunk-level RAG only if wiki misses.

We **actually implemented** this concept into cs-frontend (enterprise RAG knowledge base product) + GEO Platform shared rag-backend-v2 microservice — not just a marketing concept, but real production:

- Customer uploads source documents (PDF / Markdown / URL crawl) → enters `tenant_documents` table
- LLM compiler compiles all ready-to-compile docs into wiki pages → enters `wiki_pages` table
- User query → wiki_query_route LLM first decides "which page_type does this query belong to" → fetches corresponding wiki for direct answer
- Wiki miss → fallback to L2 chunk-level RAG (pgvector + BM25 + RRF)

This L1+L2 hybrid achieves real benefit. Same query "How does Baiyuan GEO help brands improve AI citation rate?":

- L2 only: retrieve 5 chunks → LLM rewrite → 12 sec, LLM cost ≈ $0.008
- L1+L2: wiki direct answer → 1.5 sec, LLM cost ≈ $0.0012 (cuts 85%)

But L1's establishment has one prerequisite: **wiki compilation must be stable**. In reality, we hit 6 LLM hallucination failure modes within April-May 2026 — each one caused partial or total wiki compile failure. This chapter records these six layers of hardening.

### 15.1.2 Why RAG is an Independent Microservice

GEO main backend and `rag-backend-v2` are on the same PostgreSQL instance, but **two completely separate DBs**:

| | `geo_db` | `cs_rag_db` |
|---|---|---|
| Service | GEO Platform main backend | rag-backend-v2 microservice |
| Purpose | brands / users / axp_pages / ground_truths | tenant_documents / chunks / wiki_pages |
| Role | business logic + multi-tenant data | vector retrieval + LLM Wiki compile |
| Cross-DB query | `csRagQuery(text, params)` independent pool | — |

Three reasons for splitting:

1. **Different data scale**: RAG's `chunks` table (vector embeddings) bursts with customer growth, single-table 100M rows not unusual; mixing with business tables hurts main DB performance
2. **Different update cadence**: RAG documents are "upload then untouched"; main DB business data is "second-by-second updates"
3. **Clear degradation boundary**: RAG fails, main platform still runs (just lower AI answer quality); main DB fails, everything breaks — giving RAG independent surface enables fault isolation

### 15.1.3 Cross-DB Query Cost

`backend/src/config/database.js` opens two independent PG pools:

```javascript
const geoPool = new Pool({ connectionString: DATABASE_URL });
const csRagPool = new Pool({
  connectionString: DATABASE_URL.replace(/\/geo_db$/, '/cs_rag_db'),
});

async function csRagQuery(text, params) {
  try {
    return await csRagPool.query(text, params);
  } catch (err) {
    console.error('[csRagQuery] failed:', err.message);
    return { rows: [] };  // fail-soft, doesn't block main flow
  }
}
```

UAT lacks cs_rag_db (`DATABASE_URL` for single-DB instance) → csRagQuery fails directly, but try/catch returns empty array, main flow not blocked. PROD has both DBs, csRagQuery works normally.

Cost: GEO main backend cross-DB fetch of `wiki_link_count` for "N Wiki pages" UI display must hit csRagPool, +30ms latency vs same-DB JOIN. But compared to maintenance cost of merging two DBs, 30ms is acceptable.

### 15.1.4 V1 to V2 Evolution

V1 RAG had only chunks (L2 vector retrieval) + pgvector + BM25 + RRF. L1 layer was added in April 2026 implementing "Karpathy LLM Wiki" concept. V1→V2 decision drivers:

- L2-only avg response 12 sec, customers complained slow (especially cs-frontend customer-service scenarios)
- LLM cost / month / brand ~$80, gross margin negative
- Customer asks "Why this answer?" — L2 chunks too fragmented to explain source

After adding wiki to V2, L1 hit returns in 1.5 sec + can point to wiki page_id as "source," customer satisfaction improved. But L1 must be stable — any compile failure shows customer "no relevant data" false impression.

---

## 15.2 Wiki Cascade: Four-Tier Cascade Deletion

### 15.2.1 Why Four-Tier Cascade is Needed

When a customer clicks "delete document" in frontend, we need to cascade-clean:

```text
brand_documents (geo_db)
        │ via central_doc_id UUID
        ▼
tenant_documents (cs_rag_db)        ← LLM Wiki compile source
        │ via wiki_page_sources junction
        ▼
wiki_page_sources                   ← many-to-many junction table
        │ via SSOT sync trigger
        ▼
wiki_pages.source_doc_ids           ← cited wiki page
```

Four-tier full auto cascade via:

1. `brand_documents.central_doc_id` column (migration 170) — capture central-returned doc id on upload
2. DELETE handler: `ragDeleteDocument(tenantId, central_doc_id)` → central `trg_source_soft_delete` cascade trigger auto-cleans junction
3. Junction loses last source's wiki_pages auto soft-delete
4. `migration 201:sync_wiki_page_sources trigger` — forces `wiki_pages.source_doc_ids` array to sync with `wiki_page_sources` junction (SSOT)

Design key: **junction is SSOT, array is redundant derivation**. Originally both maintained separately, often desync (observed 36 stale junction entries). Migration 201 trigger forces consistency, so customer-facing "N Wiki pages" UI display is always correct.

### 15.2.2 SSOT Trigger SQL Implementation

Migration 201's trigger forces sync:

```sql
CREATE OR REPLACE FUNCTION sync_wiki_page_sources()
RETURNS TRIGGER AS $$
BEGIN
  -- INSERT/UPDATE wiki_pages.source_doc_ids → sync junction
  IF TG_TABLE_NAME = 'wiki_pages' AND (TG_OP = 'INSERT' OR TG_OP = 'UPDATE') THEN
    DELETE FROM wiki_page_sources
     WHERE wiki_page_id = NEW.id
       AND NOT (source_doc_id = ANY(NEW.source_doc_ids));
    INSERT INTO wiki_page_sources(wiki_page_id, source_doc_id)
    SELECT NEW.id, doc_id
      FROM unnest(NEW.source_doc_ids) AS doc_id
     WHERE NOT EXISTS (
       SELECT 1 FROM wiki_page_sources
        WHERE wiki_page_id = NEW.id AND source_doc_id = doc_id
     );
  END IF;

  IF TG_TABLE_NAME = 'wiki_page_sources' AND TG_OP = 'DELETE' THEN
    UPDATE wiki_pages
       SET deleted_at = NOW(), source_doc_ids = '{}'
     WHERE id = OLD.wiki_page_id
       AND NOT EXISTS (
         SELECT 1 FROM wiki_page_sources WHERE wiki_page_id = OLD.wiki_page_id
       );
  END IF;

  RETURN COALESCE(NEW, OLD);
END;
$$ LANGUAGE plpgsql;
```

Before deploying this trigger, our cleanup logic was scattered across 7 places (4 in app code + 3 in migration scripts), often desync. After trigger, change 1 place doesn't need to change 7 — necessary for 10K-tenant scale.

### 15.2.3 36 Stale Junction Entries Tracedown

When trigger went online, did one-time reconciliation:

```sql
SELECT wps.*
  FROM wiki_page_sources wps
  LEFT JOIN wiki_pages wp ON wp.id = wps.wiki_page_id
  LEFT JOIN tenant_documents td ON td.id = wps.source_doc_id
 WHERE wp.id IS NULL OR td.id IS NULL;
-- → 36 rows
```

Single DELETE cleared, trigger ensures no new ones can be created. Where did these 36 come from? Tracking down:

- 12 from early app code versions that INSERTed junction but forgot updating array
- 14 from cron rewriting wiki race conditions (old wiki soft-deleted but junction not cleaned)
- 10 from migration script historical attempts at backfill leaving residue

After trigger, all three case classes are dead — guarantees: **junction always reflects array contents**.

### 15.2.4 UAT vs PROD Behavior Difference

UAT lacks central RAG (`CENTRAL_RAG_URL` unset) → Phase 1 path auto skip, local brand_documents purely uses pgvector adapter, **doesn't affect main flow** (try/catch doesn't block). PROD has central RAG → full four-tier cascade. Aligned with "clear degradation boundary" design.

---

## 15.3 Six wikiCompiler Hallucination Failures + Corresponding Patches

Every failure mode comes from real PROD observation; below numbered Patch N. **These six patches were chained together in just one week at end of April, beginning of May 2026**, each corresponding to a bloody incident moment.

### 15.3.1 Patch 1: 150K char cutoff bug

**Failure mode**: Some KB had only 33 of 84 docs compiled into wiki, **51 permanent orphans**.

**Discovery**: Customer reported "I uploaded 84 FAQ files but wiki shows only 33 wiki pages." First suspected LLM compile failure, log all green; then suspected docs not ready, DB checked all `status='ready'`. Finally grepped `wikiCompiler.js` to find cutoff logic.

**Root cause**: `compileWiki()` function had cutoff logic at top, borrowed from `compileIncremental` but **logic doesn't apply**:

```js
// Buggy original
let totalChars = 0;
const selectedDocs = [];
for (const doc of docs) {
  if (totalChars + (doc.content?.length || 0) > 150000) break;
  selectedDocs.push(doc);
  totalChars += doc.content?.length || 0;
}
```

The issue: main `compileWiki` runs **independent LLM calls per doc**, no aggregate token limit reason. But `compileIncremental` packs multiple docs into one prompt, has prompt token limit. Borrowing code without reading usage context.

Plus `ORDER BY created_at ASC` makes **early docs enter selectedDocs first**, new docs always cut. 10K-tenant scale: 84 docs × 4.5K avg = 378K → only 33 docs compiled → 51 docs permanent orphan.

**Fix**: Remove cutoff, all ready wiki sources included (per-doc LLM call is sequential, no burst):

```js
// Patched
const selectedDocs = docs;
const totalChars = docs.reduce((s, d) => s + (d.content?.length || 0), 0);
```

### 15.3.2 Patch 2: parseCompiledArticles partial UUID

**Failure mode**: Whole KB compile aborted, subsequent docs uncompiled. PROD log:

```text
[WIKI] Compile error: invalid input syntax for type uuid: "b7bced27"
```

**Discovery**: After Patch 1 deploy, rerun compile, aborted at doc 13. Log shows "invalid uuid" but full UUID is `b7bced27-3e4f-4abc-...` — looks truncated to first 8 chars.

**Root cause**: LLM (DeepSeek) occasionally outputs truncated UUID (8-char prefix instead of full 36 chars) on `SOURCE_IDS:` header line. Original code didn't validate, passed straight to PG `INSERT INTO wiki_pages(source_doc_ids UUID[])` → PG RAISE → uncaught → whole batch compile transaction rollback.

Key: **Single LLM tiny error blows up entire batch**. At 10K-tenant scale, just one occasional hallucination triggers full-KB rerun. Unacceptable.

**Fix**: Regex validation of LLM output:

```js
const UUID_RE = /^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$/i;
const sourceDocIds = sourceIdsRaw
  ? sourceIdsRaw.split(',').map(s => s.trim()).filter(s => UUID_RE.test(s))
  : [];
```

After filter, if empty array, fallback logic still prepends `doc.id` real UUID — LLM giving garbage doesn't affect attribution correctness.

### 15.3.3 Patch 3: fallbackWrapArticle — Defending Against LLM Marker Hallucination

**Failure mode**: LLM occasionally ignores prompt's `---ARTICLE---/---CONTENT---` markers and outputs plain text. `parseCompiledArticles` finds no marker → returns 0 articles → that doc is permanent orphan. PROD observed `02_Aesthetic_FAQ_and_Query_Set.md` LLM output 5670 chars with no marker.

**Discovery**: After Patch 2 deploy, rerun, 84 docs all compiled, but `wiki_pages` only added 51. Grep'd corresponding doc ids, confirmed some docs lacked corresponding wiki pages. Checking LLM output log, found some docs' LLM directly output plain text — no `---ARTICLE---` at all.

**Root cause**: LLM output schema is untrustworthy. Any parser depending on specific token / marker / JSON shape will occasionally fail. Anthropic / OpenAI / Gemini all occasionally "forget instructions"; DeepSeek's rate is higher (~5-8%).

**Fix**: Write fallback wrap helper. If marker missing but content ≥50 chars, wrap as single article:

```js
function fallbackWrapArticle(text, doc) {
  const trimmed = (text || '').trim();
  if (trimmed.length < 50) return null;
  return {
    title: doc.title.replace(/\.md$/i, '').slice(0, 100),
    slug: deriveSlug(doc.title) || `fallback-${doc.id.slice(0, 8)}`,
    pageType: heuristicPageType(doc.title),
    confidence: 'low',
    tags: ['wiki_fallback'],
    content: trimmed,
    sourceDocIds: [doc.id],
  };
}
```

Key design: **rather wrap with low confidence (tags=wiki_fallback) + admin warning, than let customer-uploaded sources orphan**. `compileWiki` per-doc loop and `compileIncremental` batch hook both apply.

Admin UI uses `WHERE 'wiki_fallback' = ANY(tags)` to filter for human review. Measured 84-docs KB had 14 docs in fallback bucket; humans see content mostly OK just unstructured — signals still adequate.

### 15.3.4 Patch 4: Docker DNS Internal/External Dual Fallback (Three-Version Evolution)

**Failure mode**: rag container can't resolve `postgres` internal hostname → `pg.Pool.connect("postgres")` SERVFAIL **silent hang** (no throw) → wikiCompiler's first `await q(SELECT...)` waits forever for connection → entire wiki compile appears normal but 0 progress.

**Discovery**: After Patch 3 deploy, PROD observed wiki compile "stuck" — admin UI shows `progress: 0/84`, container CPU 0.01%, DB 0 active query, no error log. 30 minutes no progress. SSH'd into container `nslookup postgres` → SERVFAIL.

**Root cause**: container `/etc/resolv.conf` points to host `systemd-resolved` (`127.0.0.53`) instead of Docker internal DNS (`127.0.0.11`) — Lightsail Ubuntu daemon DNS configuration edge case. **SERVFAIL is not NXDOMAIN** — pg client interprets it as "temporary failure, keep retrying" instead of "domain doesn't exist, give up," waiting indefinitely.

**Fix** (three-version evolution, first two failed):

| Version | Configuration | Result |
|---------|---------------|--------|
| Initial | `extra_hosts: ["postgres:172.18.0.10"]` hardcoded IP | Failed: postgres IP shifts on docker recreate |
| V2 | `dns: ["127.0.0.11"]` only | Disaster: can't resolve `www.googleapis.com` → Google OAuth platform-wide broken 8 hours |
| Final | `dns: ["127.0.0.11", "1.1.1.1", "8.8.8.8"]` | Pass: both internal and external hostnames work |

V2 was particularly tragic — after changing backend / worker / frontend / rag all to `dns: [127.0.0.11]`, in-container Node.js could not resolve `www.googleapis.com` (used by Google OAuth), Google login returned `getaddrinfo EAI_AGAIN`, customers couldn't log in for 8 full hours. Emergency rollback then traced root cause: Docker embedded DNS (127.0.0.11) only handles internal hostnames (`postgres` / `redis` / `backend`), external hostnames it forwards to host's resolv.conf, but Lightsail Ubuntu's systemd-resolved was misconfigured causing forward to fail.

Final version added `1.1.1.1` (Cloudflare) + `8.8.8.8` (Google) as external fallback. Internal hostnames go through 127.0.0.11, external through fallback — both work.

Lesson: **just setting Docker embedded DNS isn't enough, must keep external fallback**. `docker-compose.prod.yml` for backend / worker / frontend / rag all need this. Any new container **"dns: must contain internal + external fallback"** — written into platform constitution.

### 15.3.5 Patch 5: LLM Call 60s Timeout

**Failure mode**: wikiCompiler progress stuck on "`AI compile (13/84): official-FAQ`" 10+ min, CPU 0.01%, DB 0 active query. `aiCall` waiting on DeepSeek stream that never returns.

**Discovery**: After Patch 4 deploy, DNS works, but customer reported one compile "stuck on 13th." SSH'd into container, `top` shows node process 0% CPU, `netstat` shows `ESTABLISHED` connection to DeepSeek API but 0 byte traffic for 5 min.

**Root cause**: OpenAI SDK default `timeout: 600_000ms` (10 min) + `maxRetries: 2`, theoretical 20 min stuck; Gemini SDK (`@google/generative-ai`) has no native timeout / abort API. DeepSeek occasionally `ECONNRESET` / half-open / slow response, default timeout can't handle.

Worse: **streaming response mid-cut doesn't throw** — SDK thinks "stream not done, keep waiting" so it never gets `[DONE]` event.

**Fix** (three-layer timeout):

```js
// 1) OpenAI client constructor 60s + maxRetries=1
new OpenAI({ apiKey, timeout: 60_000, maxRetries: 1 });

// 2) per-request abort signal (defends streaming chunk stuck)
client.chat.completions.create(params, { signal: AbortSignal.timeout(60_000) });

// 3) Gemini Promise.race + 60s timeout (SDK has no abort)
function withTimeout(promise, ms, label) {
  return Promise.race([
    promise,
    new Promise((_, reject) =>
      setTimeout(() => reject(new Error(`${label} timeout ${ms}ms`)), ms)
    ),
  ]);
}
withTimeout(genModel.generateContent(...), 60_000, `gemini/${model}`);
```

After fix: single doc fails max 60s, wikiCompiler per-doc try/catch catches → goes Patch 3 fallback or skips next doc. **Whole batch 84 docs no longer dragged down by single LLM hang**. Safe for 10K-tenant scale.

This patch later became platform constitution **"LLM call must have ≤60s timeout"** — applies to rag-backend-v2 + geo-saas backend + frontend at all LLM call sites. Violation: a single half-open LLM connection can drag down entire workflow, 10K brands fail simultaneously.

### 15.3.6 Patch 6: wiki_query_route maxTokens 200 → 2000

**Failure mode**: Wiki L1 platform-wide miss, most queries fall back to L2, return "no relevant data."

**Discovery**: 2026-04-30 we changed PROD primary LLM from `deepseek-v3-direct` to `deepseek-v4-flash` (reasoning model is theoretically smarter, wiki query routing more accurate). Next day customer reported "asking anything gets no answer." Grep'd wiki query log all empty.

**Root cause**: reasoning model's `message.content` consumes additional `reasoning_tokens` (can take 70%+ of token budget). `wikiCompiler.js` two `wiki_query_route` calls used `maxTokens: 200` — enough for deepseek-v3, but V4 reasoning **eats all 200 tokens for reasoning, content always empty string** → JSON parse fail → wiki query routing fails → fallback L2 also returns "no hit."

Most baffling: **no error**, LLM API 200 OK returns `{ content: "", reasoning_tokens: 198 }`, just empty content. Original try/catch can't catch.

**Fix**: Two `maxTokens` 200 → 2000. Measured 2000 tokens enough for both reasoning + content; wiki query routing returns to normal.

**10K-tenant impact**: Any tenant using wiki + V4-style reasoning model (DeepSeek V4 / OpenAI o1-o4 / Anthropic with thinking) suffered Wiki L1 total failure, fallback L2 mostly works but expensive and lower quality. This lesson became platform constitution: **after switching reasoning model, must check all maxTokens configurations**. Any maxTokens < 1500 on reasoning model needs verification that content is not empty.

### 15.3.7 Combined Effect of All 6 Patches

After full 6 patches deployed, PROD observation:

| Metric | Before | After |
|--------|--------|-------|
| Platform wiki orphan rate | 45% (156/348) | ≤25% (expected) |
| 5ce78a51 KB (84 docs) compile completion rate | 39% (33/84) | 100% (84/84) |
| Single doc LLM hang dragging down batch probability | ~20% | <1% |
| Wiki L1 hit rate | 12% (post reasoning model) | 65% |

Orphan rate dropped from 45% to ≤25% but **didn't reach 0%** because: Patch 3 fallback is "save baseline" not "perfect compile." Docs entering fallback bucket are still marked `confidence='low'` and counted as partial orphan in some admin metrics. Eliminating orphan completely requires LLM to never miss markers (impossible) or all docs use structured schema (JSON output mode, partial LLM support) — V2 work.

### 15.3.8 The Causal Chain of Patch Order

In retrospect, the discovery order of the six patches was very serendipitous — each new patch was invisible until the prior one was fixed:

```text
Patch 1 (cutoff)        → fixed, then saw all docs entering compile
        ↓
Patch 2 (partial UUID)  → fixed, then saw all docs completing
        ↓
Patch 3 (no marker)     → fixed, then saw real wiki page generation
        ↓
Patch 4 (DNS)           → fixed, then saw generation actually running
        ↓
Patch 5 (LLM timeout)   → fixed, then saw single-doc failures not dragging batch
        ↓
Patch 6 (maxTokens)     → fixed, then saw wiki query actually hitting
```

Each layer is "**previous-layer silent failure was masking next-layer failure**." Typical LLM-driven system debug pattern: **errors are silent-swallowed across multiple layers, peel like an onion to see**.

---

## 15.4 Worker Robustness: From Silent Hang to Auto-Recovery

WikiCompiler failures are just the tip of the iceberg. The whole BullMQ worker system also hit the same kind of trap.

### 15.4.1 Defensive Guard Pattern

Each cron worker must check entity existence before business logic:

```js
async (job) => {
  const { brandId } = job.data;

  const { rows: [b] } = await query('SELECT id FROM brands WHERE id = $1', [brandId]);
  if (!b) {
    console.warn(`[Worker] Brand ${brandId} not found — removing orphan scheduler`);
    const q = new Queue(QUEUE_NAME, { connection });
    await q.removeJobScheduler(`monthly-${brandId}`);
    await q.close();
    return { skipped: true, reason: 'brand_not_found', brandId };
  }
}
```

Key point: **check fails → return skipped not throw**. throw counts as failed, return as completed, UI permanent green. Also opportunistically `removeJobScheduler` to clean orphan scheduler.

PROD observed application: `monthly.worker.js` (brand deleted), `abTesting.worker.js` (experiment deleted → FK violation), `closedLoop sentinel` (table renamed `brand_prompts → multilingual_prompts`).

### 15.4.2 Queue Retention Worker

`backend/src/workers/queueRetention.worker.js` runs daily at 02:30:

```js
// 24 known queues, daily cleanup
for (const name of KNOWN_QUEUES) {
  const q = new Queue(name, { connection });
  await q.clean(7 * 24 * 3600 * 1000, 500, 'failed');     // 7 days failed
  await q.clean(30 * 24 * 3600 * 1000, 500, 'completed'); // 30 days completed
  await q.close();
}
```

Without cleanup: per-brand cron generates many jobs daily, failed ones accumulate to Redis OOM, and admin UI (`/admin/scan-frequencies`) permanent red mask new state.

Measured one-time cleanup result:

```text
[QueueRetention] Cleaned 2026-05-02 02:30
  monthly-report-queue: 7 failed, 312 completed
  ab-testing-queue: 30 failed, 845 completed
  geo-closed-loop-queue: 4 failed, 198 completed
  geo-rlhf-queue: 0 failed, 67 completed
  geo-axp-queue: 2 failed, 630 completed
  ...
  Total: 43 failed + 2052 completed cleaned
```

After cleanup, admin UI returned to real-time state — any new failures immediately visible. 10K-tenant scale estimates daily cleanup ≥10K rows; retention worker is mandatory.

### 15.4.3 KNOWN_QUEUES Maintenance

The retention worker's `KNOWN_QUEUES` array must contain all platform queue names — otherwise new queues will never be cleaned:

```js
const KNOWN_QUEUES = [
  'geo-axp-queue',
  'geo-axp-immediate',
  'monthly-report-queue',
  'ab-testing-queue',
  'geo-closed-loop-queue',
  'geo-rlhf-queue',
  'geo-f12-optimize-queue',
  'queue-retention-queue',
  // ... others
];
```

Missing some queue → failures accumulate forever. To prevent missing, we added admin UI `/admin/scan-frequencies` page — lists all BullMQ queue names vs `KNOWN_QUEUES` array, set difference shows "⚠ not in retention" red. New queue added but forgot to update KNOWN_QUEUES → admin sees immediately.

### 15.4.4 LLM Call Site Timeout Platform-Wide Alignment

Beyond wikiCompiler, geo-saas backend also added LLM timeout (preventing same-class silent hang):

- `services/modelRouter.service.js` Anthropic SDK `timeout: 60000, maxRetries: 0` (default 600s)
- `services/modelRouter.service.js` Gemini call adds `Promise.race` + 60s timeout
- `services/tier-a/visualEnrichment.service.js` Anthropic Vision API fetch adds `signal: AbortSignal.timeout(60_000)`

Verification command (for self-check before commit):

```bash
grep -rn "fetch(.*api\.\(deepseek\|openai\|anthropic\|googleapis\|xiaomimimo\)" backend/src/services/ \
  | grep -v "AbortSignal\|signal:"
# Should be empty. Any output = violates LLM timeout constitution
```

We added this grep to pre-commit hook (planned), violation → block commit. 10K-tenant scale tolerates no slip-throughs.

---

## 15.5 10K-Tenant Deployment Pattern

### 15.5.1 Patches Directory Structure

`rag-backend-v2` is not in geo-saas git repo (independent microservice); patch records kept in `docs/rag-backend-v2-patches/`:

```text
docs/rag-backend-v2-patches/
├── README.md                          ← deployment order + verification block
├── wikiCompiler.js                    ← Patch 1, 2, 3, 6 all change this
├── modelRouter.js                     ← Patch 5 changes this
├── docker-compose.rag.yml             ← Patch 4 DNS settings
└── verification-block.sh              ← one-click verify 6 patches live
```

Each RAG container rebuild must scp three files together (missing one = patch regression).

### 15.5.2 Deployment + Verification Block

```bash
# 1. Get latest from patches/
scp wikiCompiler.js     ubuntu@PROD:/home/ubuntu/rag-backend-v2/src/services/
scp modelRouter.js      ubuntu@PROD:/home/ubuntu/rag-backend-v2/src/services/
scp docker-compose.rag.yml  ubuntu@PROD:/home/ubuntu/geo-saas/

# 2. Rebuild + recreate
ssh ubuntu@PROD
cd /home/ubuntu/rag-backend-v2
sudo docker build -t geo-saas-prod-rag:latest .

cd /home/ubuntu/geo-saas
sudo docker compose -f docker-compose.rag.yml --project-name geo-saas-prod up -d --force-recreate rag

# 3. Verify all 6 patches live
sudo docker exec geo-saas-prod-rag-1 sh -c '
  grep -c "selectedDocs = docs" /app/src/services/wikiCompiler.js     # patch 1: 1
  grep -c UUID_RE /app/src/services/wikiCompiler.js                   # patch 2: 2
  grep -c fallbackWrapArticle /app/src/services/wikiCompiler.js       # patch 3: 3
  grep -c postgres /etc/hosts                                         # patch 4 prep: 1+
  grep -c LLM_CALL_TIMEOUT_MS /app/src/services/modelRouter.js        # patch 5: 8
  grep -c "maxTokens: 2000" /app/src/services/wikiCompiler.js         # patch 6: ≥3
'
```

Expected output `1 / 2 / 3 / 1 / 8 / ≥3`. Any mismatch = patch regression, immediately re-scp + rebuild. Each PROD container rebuild must run this verification block, otherwise patches silently lost and we relive the failures.

### 15.5.3 10K-Tenant Readiness Verification

Beyond patch deployment, 10K-tenant readiness requires:

- **L1 hit rate monitoring**: `f12_cache_metrics` hourly aggregates, target ≥65%. Below 50% → admin alert
- **per-tenant LLM cost cap**: `f12_billing_records` monthly aggregate, starter > $1 / pro > $50 → email warn
- **fallback bucket size**: `SELECT COUNT(*) FROM wiki_pages WHERE 'wiki_fallback' = ANY(tags)` > 100 / brand → admin review
- **orphan tenant_documents**: `SELECT COUNT(*) FROM tenant_documents td WHERE NOT EXISTS (SELECT 1 FROM wiki_page_sources WHERE source_doc_id=td.id)` > 10% → re-trigger compile

These 4 metrics enter `/admin/rag-health` admin dashboard, daily cron computes and writes alert.

### 15.5.4 RAG Container Rebuild History

From 2026-04-23 to 2026-05-02 — short 9 days, rag container recreated 11 times:

| Date | Reason | Patch |
|------|--------|-------|
| 04-23 | Upgrade DeepSeek SDK | — |
| 04-25 | Add Wiki Cascade trigger | migration 170 |
| 04-27 | Add sync_wiki_page_sources trigger | migration 201 |
| 04-29 | Patch 1 (cutoff) | wikiCompiler.js |
| 04-30 | Patch 2 (UUID) | wikiCompiler.js |
| 04-30 | Switch V4 reasoning (pre-Patch 6) | platform_llm_config |
| 05-01 | Patch 3 (no marker) | wikiCompiler.js |
| 05-01 | Patch 4 v1 (extra_hosts IP) | docker-compose.rag.yml |
| 05-01 | Patch 4 v2 (dns 127.0.0.11 only) | docker-compose.rag.yml |
| 05-01 | Patch 4 v3 (dns + fallback) | docker-compose.rag.yml + .prod.yml |
| 05-02 | Patch 5 + 6 | modelRouter.js + wikiCompiler.js |

Each rebuild is ~3 min downtime (during build & recreate); 11 rebuilds cumulative ~33 min. Next major change recommend blue-green deployment (alternating dual containers) to reduce downtime — current quota insufficient for two simultaneous containers.

---

## 15.6 Observations and Open Problems

### 15.6.1 The Untrustworthy Nature of LLMs

Six layers of patches seem to cover common failure modes, but LLM evolution speed exceeds patch speed:

- Switching model (deepseek-v3 → v4-flash) brings new hallucination forms (`maxTokens` insufficient for reasoning)
- Different prompt formats fail differently across models (GPT-4 doesn't miss markers but adds prefix "Here's the response:")
- streaming vs non-streaming behavior differs (streaming may not retransmit after interruption)

This means LLM-driven system robustness design must **always assume LLM will fail in new ways** — every new model added requires running through hallucination test again.

### 15.6.2 Fallback Wrap Article Quality

Patch 3's low-confidence fallback saved customer docs, but `confidence='low'` + `tags=['wiki_fallback']` wiki page's actual quality in L1 retrieval is **untested end-to-end**. Theoretically should:

- Not participate in wiki_query_route's "what category is your query" LLM routing (avoid misleading user)
- Show "⚠ pending review" badge in admin UI
- Cumulative N failures → system alert admin

The third hasn't been done.

Further, fallback content's "source binding" exists (`sourceDocIds`), but "LLM rewrite quality" untested — possibly LLM mixes content from two unrelated docs. Requires human or LLM-as-judge sample review. V2 work.

### 15.6.3 Multi-LLM Routing vs Single-LLM Hardening

Theoretically, if same query runs on two different LLMs (DeepSeek + Gemini) in parallel, cross-validate marker / UUID / content, can solve most hallucination problems. But cost doubles and latency multiplies — is it worth it? Not enough data to decide. V2 evaluation.

Concrete quantification: single wiki compile call ~$0.0003 (DeepSeek V4 flash), dual LLM $0.0006 + max latency 1.5s → 3s. 84 docs KB compile $0.025 → $0.05. 10K-tenant monthly 100 KB recompile = $50/month, acceptable. But need A/B test first to verify dual LLM actually reduces hallucination (possibly two LLMs occasionally make same error).

### 15.6.4 Reverse of Wiki Cascade

Deletion is auto-handled by cascade, but **restore** (undo soft-delete) isn't done. Customer mistakenly deletes file → wiki_pages soft-deleted → 30 days later auto hard-delete. If customer wants it back 7 days later, currently customer support manually SQL UPDATE. Common need but engineering hasn't scheduled.

Full implementation requires:

- During soft delete period, give customer "30-day restorable" UI
- Restore reverses cascade: wiki_page restore → junction restore → tenant_document restore → brand_document restore
- All 4 layers need `deleted_at IS NOT NULL` check + restore function
- Admin force-hard-delete flow (GDPR compliance — customer demands complete deletion)

Engineering effort ~1-2 weeks, V2 schedule.

### 15.6.5 Recoverability of Wiki Compile Failure

V1 compile failure → DB writes `last_compile_error`, UI shows "compile failed." But **no retry mechanism** — admin must manually click button. 10K-tenant readiness should:

- Failure auto retry once (if LLM timeout type transient error)
- N consecutive failures → admin alert, not silently die
- Failure classification: transient (timeout / 5xx) → auto retry; permanent (invalid input / over quota) → no retry

Also V2 work.

### 15.6.6 Cross-Microservice Idempotency Guarantee

`ragDeleteDocument` failure: `brand_documents` already soft-deleted but `tenant_documents` not cleaned, creating orphan tenant_document (no brand_document corresponding). Currently relies on daily cron grabbing orphan and re-cleaning, but this cron may miss runs or fail.

Cleaner design is outbox pattern:

```sql
-- On brand_documents.delete, write outbox row
INSERT INTO outbox_events(event_type, payload, status)
VALUES ('rag_delete_document', '{"central_doc_id": "..."}', 'pending');

-- Independent worker reads outbox, calls ragDeleteDocument, marks 'done' on success
```

But would need to refactor entire cascade flow to apply outbox — V2 large work, current daily cron temporarily sufficient.

---

## 15.7 Engineering Lessons

From 6 patches + Worker robustness evolution (2 weeks, 5 PROD incidents), 5 takeaway lessons:

### 15.7.1 LLM Failures are Always Silent More Than Vocal

None of the six patches were LLM directly throwing errors. All were "appears successful but actually empty / truncated / stuck":

- Patch 1: cutoff silently cuts 60% of docs, no warning
- Patch 2: LLM output partial UUID, PG throws but transaction swallows
- Patch 3: LLM misses marker, parser returns 0 articles without throwing
- Patch 4: DNS SERVFAIL, pg-pool waits forever without throwing
- Patch 5: streaming interrupted, SDK waits for [DONE] forever without throwing
- Patch 6: reasoning maxTokens insufficient, API 200 OK returns empty content without throwing

**LLM-driven system debug rule #1: don't trust "no error seen," must verify "output is truly expected shape."** Every LLM call needs schema validation (UUID_RE / marker / minLength) — basic action.

### 15.7.2 Silent Hang is the Invisible Killer of Distributed Systems

DNS SERVFAIL / streaming interrupt / reasoning empty content — three scenarios all leave system "appearing to run normally" with 0 progress. Traditional "failure throws" assumption broken.

Defensive engineering patterns:

- **Timeouts mandatory**: Any external call (LLM / DB / internal service) must have timeout — rather manually retry than wait forever
- **Progress heartbeat**: Long-running tasks write heartbeat every 10 sec; no heartbeat = considered stuck
- **Silent more dangerous than throw**: When writing try/catch, remember "successful path also needs output validation," not just catch error

### 15.7.3 Three-Layer Fallback is More Reliable Than One-Layer Retry

Patch 3's fallback design is "**LLM fail → loose parse → still fail → wrap with doc title baseline**" — three layers, any layer success gives baseline. Compared to one-layer retry (fail retry 3 times, throw if still fail), three-layer fallback always has exit:

- First-layer ideal: LLM perfect output, direct parse
- Second-layer loose: LLM misses marker / adds prefix, loose regex catches
- Third-layer baseline: All fail, wrap doc title + content as single article

Cost: fallback bucket wiki page quality is lower, requires admin review. But compared to "LLM fail → doc orphan," worth it.

10K-tenant scale tolerates no "permanent orphans"; any customer-uploaded content must **have an exit**.

### 15.7.4 Patch Order Reveals "Downstream Illusion of Normal" Principle

Reviewing six-patch discovery order — each fix "exposes" the next:

```text
Saw 33/84 complete → fixed cutoff → saw all docs entering compile
Saw all docs entering compile → saw 13th breaks → fixed UUID → saw all docs completing
Saw all docs completing → saw wiki page count off → fixed marker → saw real wiki generation
Saw real wiki generation → saw compile stuck → fixed DNS → saw compile running
Saw compile running → saw stuck on 13th → fixed LLM timeout → saw compile stable
Saw compile stable → saw wiki query all miss → fixed maxTokens → saw wiki real hit
```

Each layer is "**previous-layer silent failure was masking the next layer**." Debugging must peel one layer at a time, **rushing to fix all at once misjudges root cause**.

### 15.7.5 patches/ Directory is Microservice SSOT

rag-backend-v2 not in geo-saas git repo, but easy for patches to be lost when going in. We designed `docs/rag-backend-v2-patches/` as SSOT:

- Each PROD scp pulls from this directory
- README writes deployment order + verification block
- Each container rebuild must run verification to check patches live

**Without verification block, next rebuild will forget patches and regress**. We hit this once: a RAG upgrade rebuild forgot scp wikiCompiler.js, Patch 6's maxTokens=2000 reverted to 200, customer reported "Wiki not hitting again." Grep `'maxTokens: 2000'` count=0 immediately revealed patch regression. Since then every RAG container recreate must run verification — won't happen again.

---

## Chapter Takeaways

- rag-backend-v2 independent microservice + own cs_rag_db + Wiki Cascade 4-tier cascade ensures data consistency
- Six LLM hallucination failure modes correspond to six patches: cutoff / partial UUID / no marker / DNS / LLM timeout / reasoning maxTokens
- Patch order reveals "downstream illusion of normal" principle: each fix reveals next-layer silent failure
- Worker robustness three-piece set: defensive guard (entity pre-check), per-item try/catch, daily retention auto-cleanup (43 failed + 2052 completed cleaned in one shot)
- 10K-tenant deployment must use patches/ directory SSOT, each container rebuild must run verification block
- Open: LLM evolution > patch speed / fallback quality untested / multi-LLM cross-validation pending evaluation / wiki restore mechanism missing / failure retry missing / outbox pattern pending refactor
- 5 engineering lessons: LLM failures silent > vocal / silent hang invisible killer / three-layer fallback > one-layer retry / Patch order principle / patches/ SSOT must run verification

## References

- [Ch 5 — Multi-Provider Routing: Error Handling Layer](./ch05-multi-provider-routing.md)
- [Ch 6 — AXP Shadow Document Delivery](./ch06-axp-shadow-doc.md)
- [Ch 14 — F12 Three-Layer Structural Optimizer](./ch14-f12-structural-optimizer.md)
- BullMQ Job Cleaning API: <https://docs.bullmq.io/guide/queues/removing-jobs>
- Docker DNS configuration: <https://docs.docker.com/network/#dns-services>
- Karpathy on LLM Wiki concept (2024 public sharing)

## Revision History

| Date | Version | Notes |
|------|---------|-------|
| 2026-05-03 | v1.1 | New chapter — six layers of LLM hallucination patches + Wiki Cascade + Worker robustness |
| 2026-05-03 | v1.1.1 | Chapter expanded to ~6100 words — added 15.1.1/3 Karpathy concept and V1→V2 evolution, 15.2.2/3 SSOT trigger SQL and 36-entry tracedown, 15.3.x discovery process per patch, 15.3.8 Patch order causal chain, 15.4.3 KNOWN_QUEUES maintenance, 15.5.4 RAG 11 rebuilds history, 15.6.6 outbox pattern, 15.7 engineering lessons (5 takeaways) |

---

**Navigation**: [← Ch 14: F12 Three-Layer Structural Optimizer](./ch14-f12-structural-optimizer.md) · [📖 TOC](../README.md) · [Ch 16: Platform SSOT Chain →](./ch16-platform-ssot-chain.md)

<!-- AI-friendly structured metadata -->
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "TechArticle",
  "headline": "Chapter 15 — rag-backend-v2 Hardening: Defensive Design for Six Layers of LLM Hallucination Failure Modes",
  "description": "Engineering hardening history of central RAG microservice (rag-backend-v2): Wiki Cascade 4-tier cascade, 6 wikiCompiler hallucination patches, Worker defensive guard and Queue retention. Full robustness design practices for LLM-driven systems. With 5 engineering lessons.",
  "author": {"@type": "Person", "name": "Vincent Lin", "affiliation": "Baiyuan Technology"},
  "datePublished": "2026-05-03",
  "inLanguage": "en",
  "isPartOf": {
    "@type": "Book",
    "name": "Baiyuan GEO Platform Whitepaper",
    "url": "https://github.com/baiyuan-tech/geo-whitepaper"
  },
  "keywords": "rag-backend-v2, LLM Wiki Compiler, Wiki Cascade, LLM Hallucination Hardening, Docker DNS, LLM Timeout, BullMQ Retention, Engineering Lessons"
}
</script>
