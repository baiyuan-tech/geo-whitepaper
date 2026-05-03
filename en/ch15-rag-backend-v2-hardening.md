---
title: "Chapter 15 — rag-backend-v2 Hardening: Six Defense Layers Against LLM Hallucination"
description: "Engineering hardening journey of the central RAG microservice (rag-backend-v2): from Wiki Cascade 4-tier propagation, to six wikiCompiler hallucination patches (cutoff / partial UUID / no marker / DNS / LLM timeout / reasoning maxTokens), to Queue retention auto-cleanup. A complete picture of robustness design for LLM-driven systems."
chapter: 15
part: 5
word_count: 4000
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
last_updated: 2026-05-03
canonical: https://baiyuan.io/whitepaper/en/ch15-rag-backend-v2-hardening
last_modified_at: '2026-05-03T10:45:10+08:00'
---



# Chapter 15 — rag-backend-v2 Hardening: Six Defense Layers Against LLM Hallucination

> In a world where LLMs cannot be trusted, the engineer's job is not to force the LLM to be perfect, but to prepare an exit for every conceivable failure mode.

## Table of Contents

- [14.1 Why RAG Needs an Independent Microservice + Self-hosted Wiki](#141-why-rag-needs-an-independent-microservice--self-hosted-wiki)
- [14.2 Wiki Cascade: Four-Tier Propagating Deletion](#142-wiki-cascade-four-tier-propagating-deletion)
- [14.3 Six wikiCompiler Hallucination Failures + Corresponding Patches](#143-six-wikicompiler-hallucination-failures--corresponding-patches)
- [14.4 Worker Robustness: From Silent Hang to Auto-Recovery](#144-worker-robustness-from-silent-hang-to-auto-recovery)
- [14.5 10K-Tenant Deployment Pattern](#145-10k-tenant-deployment-pattern)
- [14.6 Observations and Open Problems](#146-observations-and-open-problems)

---

## 14.1 Why RAG Needs an Independent Microservice + Self-hosted Wiki

The GEO main backend and `rag-backend-v2` share the same PostgreSQL instance, but **two completely separated DBs**:

| | `geo_db` | `cs_rag_db` |
|---|---|---|
| Service | GEO Platform main backend | rag-backend-v2 microservice |
| Use | brands / users / axp_pages / ground_truths | tenant_documents / chunks / wiki_pages |
| Role | Business logic + multi-tenant data | Vector retrieval + LLM Wiki compilation |
| Cross-DB | `csRagQuery(text, params)` independent pool | — |

Three reasons for separation:

1. **Scale differences**: RAG `chunks` table (vector embeddings) explodes with customer growth; 100M-row single tables aren't unusual. Mixing with business tables hurts main DB performance
2. **Update cadence differences**: RAG documents are "upload once, never change"; main DB business data changes every second
3. **Clear failure boundary**: If RAG fails, main platform still runs (just AI quality degraded); main DB failure breaks everything — giving RAG an isolated surface enables fault isolation

### 14.1.1 Wiki's Role

V1 RAG only had chunks (L2 vector retrieval) + pgvector + BM25 + RRF. The L1 layer was added in April 2026 implementing the "**Karpathy LLM Wiki**" concept: tenant-uploaded source documents are first **LLM-compiled** into structured wiki pages (each with slug / page_type / TLDR / tags). Queries first try wiki (fast, cheap, explainable); fallback to L2 only on miss.

Real impact: same query "How does Baiyuan GEO help brands improve AI citation rate":

- L2 only: retrieve 5 chunks → LLM rewrite → 12 sec
- L1+L2: wiki direct answer → 1.5 sec + 80% LLM cost reduction

But L1 has a precondition: **Wiki compilation must be stable**. Over the month from early April to early May, we hit six LLM hallucination failure modes, each causing partial or total wiki compile failures. This chapter records all six layers of hardening.

---

## 14.2 Wiki Cascade: Four-Tier Propagating Deletion

When a customer clicks "delete document" in the frontend, we must cascade-clean:

```text
brand_documents (geo_db)
        │ via central_doc_id UUID
        ▼
tenant_documents (cs_rag_db)        ← LLM Wiki compile source
        │ via wiki_page_sources junction
        ▼
wiki_page_sources                   ← many-to-many link table
        │ via SSOT sync trigger
        ▼
wiki_pages.source_doc_ids           ← cited wiki pages
```

Four-tier auto-cascade via:

1. `brand_documents.central_doc_id` field (migration 170) — captures central-returned doc id at upload
2. DELETE handler: `ragDeleteDocument(tenantId, central_doc_id)` → central `trg_source_soft_delete` cascade trigger auto-cleans junction
3. wiki_pages losing their last source are auto soft-deleted
4. `migration 201: sync_wiki_page_sources trigger` — forces `wiki_pages.source_doc_ids` array to sync with `wiki_page_sources` junction (SSOT)

Design key: **junction is SSOT, array is redundant derivation**. Originally both were independently maintained, often desync'ing (observed 36 stale junction entries). After migration 201, trigger enforces consistency, so frontend's "N Wiki pages" UI is always correct.

### 14.2.1 UAT vs PROD Behavior Differences

UAT has no central RAG (`CENTRAL_RAG_URL` unset) → Phase 1 path auto-skips, local brand_documents purely uses pgvector adapter, **no main flow impact** (try/catch doesn't block). PROD has central RAG → full 4-tier cascade. Aligns with "clear failure boundary" design.

---

## 14.3 Six wikiCompiler Hallucination Failures + Corresponding Patches

Each failure observed in real PROD; recorded in chronological `Patch N` order:

### 14.3.1 Patch 1: 150K char cutoff bug

**Failure**: A KB had 84 docs but only 33 ended up in the wiki — **51 permanently orphaned**.

**Root cause**: `compileWiki()` had a cutoff borrowed from `compileIncremental` but **inappropriately applied**:

```js
// Buggy
let totalChars = 0;
const selectedDocs = [];
for (const doc of docs) {
  if (totalChars + (doc.content?.length || 0) > 150000) break;  // cuts off all after 150K
  selectedDocs.push(doc);
  totalChars += doc.content?.length || 0;
}
```

The main `compileWiki` runs **independent LLM call per doc**, so there's no aggregate token reason. But `compileIncremental` puts multiple docs in one prompt, which has prompt token limits. Borrowing without context check broke things.

`ORDER BY created_at ASC` puts **early docs first** in selectedDocs, so new docs always get cut. At 10K tenant scale: 84 docs × 4.5K avg = 378K → only 33 docs compiled → 51 docs orphaned forever.

**Fix**: Remove cutoff; include all ready wiki sources (per-doc LLM calls are sequential, no burst):

```js
// Patched
const selectedDocs = docs;
const totalChars = docs.reduce((s, d) => s + (d.content?.length || 0), 0);
```

### 14.3.2 Patch 2: parseCompiledArticles partial UUID

**Failure**: Entire KB compile aborted, subsequent docs unprocessed. PROD log:

```text
[WIKI] Compile error: invalid input syntax for type uuid: "b7bced27"
```

**Root cause**: LLM (DeepSeek) occasionally outputs truncated UUIDs in `SOURCE_IDS:` header (8-char prefix instead of full 36 chars). Original code didn't validate, passed directly to PG `INSERT INTO wiki_pages(source_doc_ids UUID[])` → PG RAISE → uncaught → entire batch transaction rollback.

**Fix**: Regex-validate LLM output:

```js
const UUID_RE = /^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$/i;
const sourceDocIds = sourceIdsRaw
  ? sourceIdsRaw.split(',').map(s => s.trim()).filter(s => UUID_RE.test(s))
  : [];
```

After filter, even if empty, downstream "safety" prepends real `doc.id`, so attribution remains correct regardless of LLM garbage.

### 14.3.3 Patch 3: fallbackWrapArticle — Defending Against LLM Marker Hallucination

**Failure**: LLM occasionally ignored prompt-mandated `---ARTICLE---/---CONTENT---` markers and emitted plain text. `parseCompiledArticles` couldn't find markers → returned 0 articles → that doc permanently orphaned. PROD observed `02_醫美業_FAQ_與_Query集.md` LLM emitted 5670 chars without markers.

**Root cause**: LLM output schemas are unreliable; any parser depending on specific tokens / markers / JSON shapes will sporadically fail.

**Fix**: Write a fallback wrap helper. When markers missing but ≥50 chars present, wrap into a single low-confidence article:

```js
function fallbackWrapArticle(text, doc) {
  const trimmed = (text || '').trim();
  if (trimmed.length < 50) return null;
  return {
    title: doc.title.replace(/\.md$/i, '').slice(0, 100),
    slug: deriveSlug(doc.title) || `fallback-${doc.id.slice(0, 8)}`,
    pageType: heuristicPageType(doc.title),  // faq/product/policy/concept
    confidence: 'low',                        // signals admin to review manually
    tags: ['wiki_fallback'],                  // admin filters by this tag
    content: trimmed,
    sourceDocIds: [doc.id],                   // bidirectional bind to original doc
    // ...
  };
}
```

Design key: **better to wrap with low confidence (tags=wiki_fallback) and alert admin than orphan a customer-uploaded source doc**. Both `compileWiki` per-doc loop and `compileIncremental` batch path have this hook.

### 14.3.4 Patch 4: Docker DNS internal+external dual fallback

**Failure**: rag container couldn't resolve `postgres` internal hostname → `pg.Pool.connect("postgres")` SERVFAIL **silent hang** (no throw) → wikiCompiler's first `await q(SELECT...)` waited forever for connection → entire wiki compile appeared normal but had 0 progress.

**Root cause**: Container `/etc/resolv.conf` pointed to host `systemd-resolved` (`127.0.0.53`) instead of Docker's internal DNS (`127.0.0.11`); a Lightsail Ubuntu daemon DNS edge case.

**Fix** (3 versions, first two failed):

| Version | Config | Result |
|---|---|---|
| Initial | `extra_hosts: ["postgres:172.18.0.10"]` hardcoded IP | Failed: postgres IP shifts on docker recreate |
| Second | `dns: ["127.0.0.11"]` only | Disaster: couldn't resolve `www.googleapis.com` → Google OAuth platform-wide broken |
| Final | `dns: ["127.0.0.11", "1.1.1.1", "8.8.8.8"]` | Success: internal + external hostnames both work |

Lesson: **setting only Docker embedded DNS isn't enough; external fallback must be preserved**. `docker-compose.prod.yml` must apply this to backend / worker / frontend / rag.

### 14.3.5 Patch 5: LLM call 60s timeout

**Failure**: wikiCompiler progress stuck at "AI compile (13/84): 官網-FAQ" for 10+ minutes, CPU 0.01%, DB 0 active queries. `aiCall` was waiting forever for a stream that never arrived from DeepSeek.

**Root cause**: OpenAI SDK default `timeout: 600_000ms` (10 min) + `maxRetries: 2`, theoretically 20 min stuck; Gemini SDK (`@google/generative-ai`) provides no native timeout / abort API. DeepSeek occasional `ECONNRESET` / half-open connections / slow responses can outlive default timeouts.

**Fix** (three timeout layers):

```js
// 1) OpenAI client constructor 60s + maxRetries=1
new OpenAI({ apiKey, timeout: 60_000, maxRetries: 1 });

// 2) per-request abort signal
client.chat.completions.create(params, { signal: AbortSignal.timeout(60_000) });

// 3) Gemini Promise.race + 60s timeout (no native abort)
withTimeout(genModel.generateContent(...), 60_000, 'gemini/${model}')
```

After fix: single doc failure max 60s, wikiCompiler per-doc try/catch absorbs → falls through to Patch 3 fallback or skips to next doc. **Entire 84-doc batch no longer dragged down by single LLM hang**. 10K-tenant scale safe.

### 14.3.6 Patch 6: wiki_query_route maxTokens 200 → 2000

**Failure**: Wiki L1 platform-wide miss; most queries fall back to L2 and return "no relevant data."

**Root cause**: PROD primary LLM switched to `deepseek-v4-flash` (reasoning model) on 2026-04-30. Reasoning models consume `reasoning_tokens` separately from `message.content`. `wikiCompiler.js` had two `wiki_query_route` calls using `maxTokens: 200`, sufficient for deepseek-v3 but **V4 reasoning consumed all 200 tokens for reasoning, leaving content always empty** → JSON parse fail → wiki query routing failed.

**Fix**: Both `maxTokens` 200 → 2000.

**10K-tenant impact**: Any tenant with wiki enabled + V4-style reasoning model (DeepSeek V4 / OpenAI o1-o4 / Anthropic with thinking) had Wiki L1 totally broken; fallback L2 could still answer but at higher cost and lower quality. Lesson generalized: **after switching to reasoning models, audit all maxTokens configs**.

### 14.3.7 Combined Effect of All 6 Patches

PROD observation after deploying all 6 patches:

| Metric | Before | After |
|---|---|---|
| Platform wiki orphan rate | 45% (156/348) | ≤25% (expected) |
| 5ce78a51 KB (84 docs) compile completion | 39% (33/84) | 100% (84/84) |
| Single doc LLM hang stalling whole batch | ~20% | <1% |
| Wiki L1 hit rate | 12% (post reasoning model) | 65% |

---

## 14.4 Worker Robustness: From Silent Hang to Auto-Recovery

WikiCompiler failures are just the tip. The full BullMQ worker system has the same trap.

### 14.4.1 Defensive Guard Pattern

Every cron worker checks entity existence before running business logic:

```js
async (job) => {
  const { brandId } = job.data;

  // Pre-check — if brand was deleted but cron still scheduled, throw → BullMQ accumulates failed → admin UI permanently red
  const { rows: [b] } = await query('SELECT id FROM brands WHERE id = $1', [brandId]);
  if (!b) {
    console.warn(`[Worker] Brand ${brandId} not found — removing orphan scheduler`);
    const q = new Queue(QUEUE_NAME, { connection });
    await q.removeJobScheduler(`monthly-${brandId}`);  // self-clean orphan
    await q.close();
    return { skipped: true, reason: 'brand_not_found', brandId };
  }

  // ... business logic
}
```

Key: **on check failure → return skipped, not throw**. Throw counts as failed; return counts as completed; UI permanently green. Also, opportunistically `removeJobScheduler` to prevent next-cycle re-scheduling.

PROD coverage: `monthly.worker.js` (brand deleted), `abTesting.worker.js` (experiment deleted → FK violation), `closedLoop sentinel` (table renamed `brand_prompts → multilingual_prompts`).

### 14.4.2 Queue Retention Worker

`backend/src/workers/queueRetention.worker.js` runs daily at 02:30:

```js
// 24 known queues, daily cleanup
for (const name of KNOWN_QUEUES) {
  const q = new Queue(name, { connection });
  await q.clean(7 * 24 * 3600 * 1000, 500, 'failed');     // 7-day failed
  await q.clean(30 * 24 * 3600 * 1000, 500, 'completed'); // 30-day completed
  await q.close();
}
```

Without cleanup: per-brand cron generates massive jobs daily; failed accumulation can OOM Redis, plus admin UI (`/admin/scan-frequencies`) red dots accumulate forever, masking new state.

### 14.4.3 Platform-wide LLM Call Timeout Alignment

Beyond wikiCompiler, geo-saas backend added matching LLM timeouts (preventing same silent hang):

- `services/modelRouter.service.js` Anthropic SDK `timeout: 60000, maxRetries: 0` (was default 600s)
- `services/modelRouter.service.js` Gemini call wrapped in `Promise.race` + 60s timeout
- `services/tier-a/visualEnrichment.service.js` Anthropic Vision API fetch with `signal: AbortSignal.timeout(60_000)`

Self-check command (run before commit):

```bash
grep -rn "fetch(.*api\.\(deepseek\|openai\|anthropic\|googleapis\|xiaomimimo\)" backend/src/services/ \
  | grep -v "AbortSignal\|signal:"
# Should be empty; non-empty = violates LLM timeout rule
```

---

## 14.5 10K-Tenant Deployment Pattern

`rag-backend-v2` is **not in the geo-saas git repo** (independent microservice); patch records are kept in `docs/rag-backend-v2-patches/`. Every RAG container rebuild must:

```bash
# 1. Pull latest from patches/
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

Every PROD container rebuild must run this verification block; otherwise patches silently disappear and we repeat history.

---

## 14.6 Observations and Open Problems

### 14.6.1 The Untrustability of LLMs

Six layers of patches seem to cover common failures, but LLM evolution outpaces patch velocity:

- Switching models (deepseek-v3 → v4-flash) introduces new hallucination forms (`maxTokens` insufficient for reasoning)
- Different prompt formats fail differently per model (GPT-4 doesn't drop markers but emits prefix "Here's the response:")
- Streaming vs non-streaming differ in behavior (streaming may break and not resume)

This means LLM-driven systems' robustness must **always assume LLMs will fail in new ways**; every new model addition requires another round of hallucination testing.

### 14.6.2 Quality of Fallback-Wrapped Articles

Patch 3's low-confidence fallback rescues customer docs, but `confidence='low'` + `tags=['wiki_fallback']` wiki page actual L1 retrieval quality has **no end-to-end evaluation yet**. Theoretically should:

- Not participate in wiki_query_route's "what category are you asking" LLM router (avoid misleading user)
- Display "⚠ pending review" badge in admin UI
- After N accumulated failures, system alerts admin

The third point isn't done.

### 14.6.3 Multi-LLM Routing vs Single-LLM Hardening

Theoretically, running the same query through two different LLMs (DeepSeek + Gemini) in parallel and cross-validating marker / UUID / content can solve most hallucination problems. But cost doubles and latency doubles — is it worth it? Insufficient data to decide; left for V2.

### 14.6.4 Reverse Operation of Wiki Cascade

Deletion is automatic via cascade, but **undo soft-delete** isn't done. Customer mistakenly deletes file → wiki_pages soft-deleted → 30 days later auto-hard-delete. If customer wants it back after 7 days, currently requires manual SQL UPDATE by support. Common need but not yet engineered.

### 14.6.5 Recoverability of Wiki Compile Failures

V1 compile failure → DB writes `last_compile_error`, UI shows "compile failed". But **no retry mechanism** — admin must manually click button. 10K-tenant readiness should:

- Auto-retry once (if LLM timeout-class transient error)
- N consecutive failures → admin alert, not silent death

Also V2 work.

---

## Key Takeaways

- rag-backend-v2 independent microservice + own cs_rag_db + Wiki Cascade 4-tier propagating delete ensures data consistency
- Six LLM hallucination failure modes correspond to six patches: cutoff / partial UUID / no marker / DNS / LLM timeout / reasoning maxTokens
- Worker robustness 3-piece kit: defensive guard (entity pre-check), per-item try/catch, daily retention auto-cleanup
- 10K-tenant deployment must use patches/ directory SSOT, every container rebuild must run verification block
- Open: LLM evolution > patch velocity; fallback quality unevaluated; multi-LLM cross-validation pending; wiki undo missing; failure retry missing

## References

- [Ch 5 — Multi-Provider Routing: Error Handling Layer (zh-TW)](../zh-TW/ch05-multi-provider-routing.md)
- [Ch 6 — AXP Shadow Document Delivery (zh-TW)](../zh-TW/ch06-axp-shadow-doc.md)
- [Ch 14 — F12 Three-Layer Structural Optimizer](./ch14-f12-structural-optimizer.md)
- BullMQ Job Cleaning API: <https://docs.bullmq.io/guide/queues/removing-jobs>
- Docker DNS configuration: <https://docs.docker.com/network/#dns-services>
- Karpathy on LLM Wiki concept (2024 public sharing)

## Revision History

| Date | Version | Notes |
|------|---------|-------|
| 2026-05-03 | v1.1 | New chapter — six LLM hallucination patches + Wiki Cascade + Worker robustness |

---

**Navigation**: [← Ch 14: F12 Structural Optimizer](./ch14-f12-structural-optimizer.md) · [📖 EN Index](./README.md) · [Ch 16: Platform SSOT Chain →](./ch16-platform-ssot-chain.md)

<!-- AI-friendly structured metadata -->
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "TechArticle",
  "headline": "Chapter 15 — rag-backend-v2 Hardening: Six Defense Layers Against LLM Hallucination",
  "description": "Engineering hardening journey of central RAG microservice (rag-backend-v2): Wiki Cascade 4-tier propagation, six wikiCompiler hallucination patches, Worker defensive guard + Queue retention. Complete robustness design picture for LLM-driven systems.",
  "author": {"@type": "Person", "name": "Vincent Lin", "affiliation": "Baiyuan Technology"},
  "datePublished": "2026-05-03",
  "inLanguage": "en",
  "isPartOf": {
    "@type": "Book",
    "name": "Baiyuan GEO Platform Whitepaper",
    "url": "https://github.com/baiyuan-tech/geo-whitepaper"
  },
  "keywords": "rag-backend-v2, LLM Wiki Compiler, Wiki Cascade, LLM Hallucination Hardening, Docker DNS, LLM Timeout, BullMQ Retention"
}
</script>
