---
title: "Chapter 14 — F12 Three-Layer Structural Optimizer: From Rule-based V1 to Dual-Engine v3.1"
description: "Architectural evolution of F12 (TLSO, Three-Layer Structural Optimizer): V1 rule-based scoring + LLM rewrite, V3.1 introducing AutoGEO + E-GEO dual engines + Hybrid integration, plus Plan Gate / Multi-Layer Cache / Tier Priority / Scale Trigger as 10K-tenant infrastructure."
chapter: 14
part: 5
word_count: 4200
lang: en
authors:

  - name: Vincent Lin
    affiliation: Baiyuan Technology
license: CC-BY-NC-4.0
keywords:

  - F12
  - TLSO
  - Three-Layer Structural Optimizer
  - AutoGEO
  - E-GEO
  - Hybrid Engine
  - Adaptive Router
  - LLM Gateway
  - Plan Feature Gate
  - Multi-Layer Cache
last_updated: 2026-05-03
canonical: https://baiyuan.io/whitepaper/en/ch14-f12-structural-optimizer
last_modified_at: '2026-05-03T02:45:21Z'
---




# Chapter 14 — F12 Three-Layer Structural Optimizer: From Rule-based V1 to Dual-Engine v3.1

> Turn "AI citation rate optimization" from prompt-tuning alchemy into a measurable, reproducible, scalable engineering system.

## Table of Contents

- [13.1 Why F12 is Needed](#131-why-f12-is-needed)
- [13.2 V1: Three-Layer Analysis + LLM Optimizer](#132-v1-three-layer-analysis--llm-optimizer)
- [13.3 V3.1: Dual-Engine Parallel + Integration Strategy](#133-v31-dual-engine-parallel--integration-strategy)
- [13.4 10K-Tenant Infrastructure](#134-10k-tenant-infrastructure)
- [13.5 Deployment, Quota, and Billing](#135-deployment-quota-and-billing)
- [13.6 Observed Limitations and Open Problems](#136-observed-limitations-and-open-problems)

---

## 13.1 Why F12 is Needed

The seven-dimension scoring in Ch 3 answers "what is the brand citation rate," but **not "how should the next article be written to improve citation rate."** Commercial customers actually ask:

> "I already know the AI citation rate is 32. How do I rewrite my next blog to bring it to 60?"

F12 (named after the browser DevTools F12 "structural inspection" metaphor) is the engineering answer: input a piece of content, output **quantified structural scores** (three layers: macro / meso / micro) + **a directly applicable optimized version**. It differs from Ch 3 scoring as follows:

| | Ch 3 Scoring | F12 |
|---|---|---|
| Input | Brand × AI platform dimensions | Single content piece (URL / text / Markdown) |
| Output | 0–100 score + signal breakdown | Three-layer scores + optimized content |
| Use | "Is my brand healthy?" | "How should this article be rewritten?" |
| Target | brand-level | content-level |

**F12's core promise: given the same content, the system can produce multiple variants and select the one most likely to be cited by an LLM.**

---

## 13.2 V1: Three-Layer Analysis + LLM Optimizer

### 13.2.1 The Three Layers

| Layer | Granularity | Checks |
|---|---|---|
| **Macro** | Document level | Has ≥1 fact-check / FAQ / comparison? Clear H1? Entity declared? |
| **Meso** | Paragraph level | Paragraph length distribution, TL;DR up front, lead-in sentences attractive for LLM excerpting |
| **Micro** | Sentence level | Numbers / dates / source link density, entity markup, atomic facts citable |

Each layer scored 0–100 independently; final `overall = 0.4 × macro + 0.35 × meso + 0.25 × micro` (weights stored in `scoring_configs.f12_thresholds_<brand_type>` SSOT, admin-tunable without code change).

V1 analyzer is **purely rule-based**: regex + AST parsing + statistical density, no LLM call. 30+ articles/sec, suitable for full-platform batch scanning.

### 13.2.2 LLM Optimizer

Pages scoring < 70 are **automatically queued** to LLM Optimizer:

```text
[Low-score page] → fetch axp_pages.content_md → feed LLM
              → input: original + 3-layer issues + reference templates
              → require: rewrite to gain ≥ 20 score, but cosine similarity ≥ 0.90 (no topic drift)
              → write to axp_page_history (bidirectional rollback)
              → re-run analyzer to confirm new score ≥ original + 15
              → write to axp_pages, mark needs_recompile
```

`min_similarity ≥ 0.90` is a spec-anchored threshold (raised from 0.85), to prevent the LLM rewriting customer content into "pretty but off-topic" output. (Real case: LLM rewrote insurance-product content into investment advice, violating financial regulator rules.)

### 13.2.3 axpPageWriter Hook Coverage

All paths that write to `axp_pages` (9 generators + 4 legacy + admin manual edit) **all hook F12 analyze**, ensuring new content is scored immediately and queued for optimizer if below threshold.

Coverage verification:

- ✅ `axpPageWriter.service.js#upsertAxpPage` (main path)
- ✅ `hybridCoordinator.service.js` (6 core types)
- ✅ `factCheckGenerator.js#refreshFactCheckPage` (direct SQL, hook added)
- ✅ `axp.controller.js#updatePageContent` (admin edit)

The weekly backfill cron (`0 2 * * 0`) is the safety net for missed paths.

### 13.2.4 Five Cron Jobs

| Cron | Schedule | Purpose |
|------|------|------|
| `f12-weekly-backfill` | `0 2 * * 0` | All brands × 22 page F12 re-analysis + brand_faq health check |
| `f12-trends-refresh` | `30 2 * * *` | REFRESH MATERIALIZED VIEW `structural_score_trends` |
| `f12-low-score-optimizer` | `0 4 * * *` | LLM rewrite of low-score pages, bidirectional rollback |
| `f12-immediate-optimize` | priority queue | axpPageWriter hook trigger, score < 70 queues immediately |
| `f12-retention-cleanup` | `50 3 * * *` | 30-day cleanup of old optimization runs (later subsumed by v3.24 retention) |

---

## 13.3 V3.1: Dual-Engine Parallel + Integration Strategy

After V1 ran for 4 months, two arXiv papers in early 2026 pointed to the next order-of-magnitude improvement:

### 13.3.1 Two Academic Sources

- **AutoGEO** (arXiv:2510.11438, github.com/cxcscmu/AutoGEO): Mining transferable rewrite rules from thousands of "cited vs not-cited" content pairs (currently 25 rules: `Researchy-GEO` × `Gemini` 15 + `E-commerce` × `Gemini` 10)
- **E-GEO** (arXiv:2511.20867, github.com/psbagga17/E-GEO): Generating from different prompt styles (`authoritative` / `technical` / `unique` / `fluent` / `clickable` × 15 styles total), measuring per-style citation rate per LLM

We **literally** ported (line-by-line) both sources into the system, **without fabricating any paper_table_ref / expected_uplift** (papers don't provide per-rule mappings; fabrication would violate engineering constitution #1 "no simulated data").

> Important: V3.1 `autogeo_rules` table's initial seed contained fabricated paper_table_ref (`'Table 3'` etc.) + fabricated expected_uplift (`1.32-1.92`). Caught by engineering constitution #1 trigger and re-ported from real arxiv source. DB-level placeholder guard trigger added permanently to prevent recurrence.

### 13.3.2 Three-Engine Architecture

```text
                 ┌──────────────────┐
                 │ adaptiveRouter   │ ← routes by brand tier / page_type / use case
                 └─────────┬────────┘
                           │
        ┌──────────────────┼──────────────────┐
        ▼                  ▼                  ▼
  ┌──────────┐       ┌──────────┐       ┌──────────┐
  │ AutoGEO  │       │ E-GEO    │       │ Hybrid   │ ← parallel call, picks higher improvement
  │ engine   │       │ engine   │       │ engine   │
  └──────────┘       └──────────┘       └──────────┘
        │                  │                  │
        └──────────────────┼──────────────────┘
                           ▼
                    ┌─────────────┐
                    │ ME custom   │ ← personal_ip exclusive,
                    │ engine      │   gated by personal_profiles
                    └─────────────┘
```

All four engines emit a **standardized EngineResult** schema (content + score + diff + source rule/template id), which `executeRoute` audits + persists + caches uniformly.

### 13.3.3 Routing Decisions

`adaptiveRouter.decide({ brandId, contentType, ... })` returns `{ path, template, target_engine }` based on:

1. **Plan gate**: `starter` always uses E-GEO `authoritative` (lowest cost); `pro` only allows sync API; Enterprise+ allows Hybrid and AutoGEO
2. **page_type → template**: `f12_page_type_to_template` SSOT (22+1 page_type → 5 template categories)
3. **content_type specialization**: whitepaper / case_study / fact_* forced to E-GEO `authoritative`; FAQ → `FAQ`; product → `clickable`
4. **brand_type fallback**: personal_ip → `me_custom` (the single baiyuan-internal template)

### 13.3.4 Real-arxiv Alignment

`autogeo_rules` 25 entries all `source = 'arXiv:2510.11438'`, with real:

- `target_engine ∈ {gemini, gpt, claude}`
- `target_domain ∈ {researchy_geo, geo_bench, ecommerce}`
- `paper_table_ref` all NULL (paper has no per-rule mapping; constitution #1 forbids fabrication)
- `expected_uplift` all NULL (paper reports overall experiment uplift, not per-rule; will be filled by real ML measurement in V2)

`egeo_templates` 15 entries all `source = 'arXiv:2511.20867'`, with template names matching paper Table 1 exactly.

The single baiyuan-internal `me_personal_ip` template is explicitly tagged `source = 'baiyuan_internal_v3_1'` and is not mixed with arxiv naming.

---

## 13.4 10K-Tenant Infrastructure

V3.1 laid down 10K-tenant infrastructure in 9 batches (v3.21 → v3.27 spanning 8 days):

### 13.4.1 Quota and Billing (spec §4.3 + §4.4)

```text
TenantQuotaService (monthly_optimizations / max_content_size_kb)
  starter:  100 ops, 50 KB
  pro:      1000, 200
  enterprise: 10000, 1000
  group:    ∞, 5000

BillingTracker (per-LLM-call cost recording)
  9 models × per-token pricing (claude_haiku $0.0003 / opus $0.015 / 4o-mini $0.0002 ...)
  in/out multiplier 4×
```

`f12_quota_usage` UPSERTs monthly counters; `f12_billing_records` at 12M rows/year is a single-table sweet spot. `quota_enforcement_enabled` flag defaults `false` (observation phase records usage but doesn't block); `billing_recording_enabled` defaults `true` (always recorded).

### 13.4.2 Plan Feature Gate (spec §9)

`f12_plan_features` SSOT 4 tiers × 6 flags (JSONB inside `scoring_configs`):

```yaml
egeo_template_count:    starter=0    pro=5     enterprise=15  group=15
autogeo_rules:          'none'       'none'    'full'         'full'
engine_specialization:  false        false     true           true
hybrid_engine:          false        false     true           true
sync_api:               false        true      true           true
```

Non-Enterprise + `whitepaper` / `case_study` / `fact-*` content forced to E-GEO `authoritative` (no AutoGEO); only Enterprise+ can use Hybrid + tri-engine specialization. Both `adaptiveRouter` plan gate and `executeRoute` defensive plan_gate enforce.

### 13.4.3 Multi-Layer Cache (spec §5.2)

```text
L1 Redis (5 min TTL)        ← repeated query with same content + decision
L2 PG f12_result_cache (7d) ← cross-process / cross-instance
L3 S3 hook (TBD Phase 3)    ← cross-region replication if needed
```

cache key = `sha256(content + decision.path + template + target_engine)`, **excluding tenant_id for cross-tenant sharing** (spec-allowed: same content + same rules → same result, sharing reduces N× LLM cost).

Only cache `accepted=true` results (avoid cache poisoning). `f12_cache_metrics` aggregates L1/L2/miss hourly → admin monitors 70% hit rate target.

### 13.4.4 LLM Gateway (spec §5.3)

per-model platform-level RPM throttling (in-memory sliding window 60s):

```yaml
claude_haiku:  10000 / min
sonnet:         2000
opus:            500
gpt-4o:         5000
gpt-4o-mini:   10000
gemini-flash:  20000
qwen_direct:   30000
deepseek_v4:    8000
```

All 5 F12 LLM call sites go through `gatewayAiCall`: autogeo / egeo / meCustom / V1 optimizer / hybrid (which call the first two indirectly). `opts.billing` triggers automatic recording (backward-compatible).

### 13.4.5 Other 10K-Tenant Hooks

| Mechanism | Effect |
|------|------|
| **Tier-based Job Priority** (`group=1` highest → `starter=4`) | BullMQ priority, large customers processed first |
| **Sync/Async Dual API** | `/optimize` async returns jobId / `/optimize/sync` Pro+ only, content ≤5KB |
| **Concurrent Jobs Enforcement** | Prevents starter from infinitely queuing and overwhelming workers (`starter=1`, `group=100`) |
| **Per-Tenant API RPM** | Redis sliding-window per brand, fail-open (does not block when Redis down) |
| **Retention TTL SSOT** | 6 tables daily 05:00 cron clears expired rows, `scoring_configs.f12_retention_config` admin-tunable |
| **Budget Alerts** | tier thresholds (starter $1 / group $5000 monthly), warn@80%, critical@block_pct |
| **Scale-up Trigger** | daily 04:30 snapshot 8-dimension metrics (tenant_count / monthly_ops / p99 / cache_hit / cost), spec-aligned phase 1→2→3 escalation |
| **Tenant Isolation Strategy** | spec/actual dual-column, exposes enterprise/group customer isolation gap to admin |
| **L3 S3 Cache hook** (Phase 3 reserved) | dynamic `import @aws-sdk/client-s3` when `F12_S3_CACHE_BUCKET` env is set |

Complete SSOT lives in 14 `scoring_configs.f12_*` keys; admin UI at `/dashboard/admin/f12-dashboard` integrates 4 stat cards + 4 detail sections.

---

## 13.5 Deployment, Quota, and Billing

### 13.5.1 Deployment: Wave 0 / Wave 1 Alignment

V3.1 completed spec Wave 0 (microservice contract OpenAPI) + Wave 1 (in-process implementation + SDK + admin):

- **OpenAPI 3.0 spec**: `backend/src/openapi/f12-v3-1.json` 7 paths × 14 schemas, public at `/api/v1/f12/openapi.json` (mounted before router auth) for AI agents / 3rd-party machine-discovery
- **F12Client SDK**: `services/f12/f12Client.js` 5 entry points (`diagnose / optimize / optimizeSync / getJob / health`), HTTP mode when `F12_SERVICE_URL` env set, otherwise in-process. **Caller signature unchanged** to preserve future microservice-split flexibility
- **/api/v1/f12/health**: public endpoint mounted before OAuth, for monitoring systems to ping directly

### 13.5.2 Quota: Observation → Enforcement

Initial deploy `quota_enforcement_enabled=false` (records usage only); customers see "this month used / limit is" but are not blocked. After ~2 weeks of distribution analysis, admin one-click flips to `true`; subsequent overage immediately returns HTTP 429 + reject reason `'quota_exceeded'`.

### 13.5.3 Billing: Per-call Recording, Monthly Aggregation

`BillingTracker.recordCall(model, tokens, runId)` writes per-LLM-call. Month-end admin runs `/admin/f12-billing-summary?period=YYYY-MM` to see top 50 cost brands.

Dual-write strategy: Redis (`f12:billing:tenant:{id}:{period}` INCRBYFLOAT + 35-day TTL) + DB (`f12_billing_records`). Current-month `getBrandMonthlyCost` reads Redis O(1) GET first; past months or Redis miss falls back to DB SUM. 10K-tenant monthly-cost queries don't hit DB, sub-ms response.

---

## 13.6 Observed Limitations and Open Problems

### 13.6.1 No "Correct" Way to Merge Dual-Engine Results

Hybrid engine runs AutoGEO + E-GEO in parallel, picks the one with higher `improvement`. But in actual commercial scenarios, improvement isn't a single metric:

- AutoGEO rule rewrites yield stable structure but less brand personality
- E-GEO templates produce more variation but sometimes too divergent from customer's voice

V3.1 picks "higher engine improvement score" as criterion; this is insufficient. Next version wants to add **"accepted by client" signal**, but admin-side approval UI hasn't shipped (Phase 3).

### 13.6.2 paper_table_ref / expected_uplift Are NULL

"Literal arxiv port" is a side effect of engineering constitution #1 — papers don't provide per-rule table mappings, so fabrication is forbidden. But admin UI showing "why this rule was chosen" is blank. V2 plans **in-house measurement**: per-rule real ML eval (same prompt × 3 LLMs × 1000 brands, measure citation_rate uplift), filling in real numbers. Requires time + Anthropic / OpenAI quota.

### 13.6.3 L3 S3 Cache Not Yet Deployed

Phase 3 is needed (estimated 5000+ tenant scale). Current L1+L2 hit rate ~35-45%, still below 70% target. L3 expected +15-20% uplift but requires S3 cost vs LLM recompute trade-off analysis, not done.

### 13.6.4 ME Custom Engine Gating Too Strict

`personal_profiles` table metadata empty → reject `meta_incomplete`. Among 17 ME platform member brands, 5 have incomplete metadata (missing full name / known_for); these 5 fall back to E-GEO `authoritative` but the result reads enterprise-toned, unsuitable for personal IP. Customer service must follow up to complete metadata, but this violates the "automation" promise.

### 13.6.5 Pre-/Post- 10K Tenant Testing

V3.1 infrastructure aligns with "tolerate 10K tenants" design, but real load testing only reached 17 ME brands + 21 GEO brands. Scale-up Trigger's phase_1_to_2 / phase_2_to_3 logic literally aligns with spec line 1183-1193, but **has never been actually escalated**. Theoretically cross-phase escalation requires: tenant schema isolation switch (Phase 1 row_level → Phase 2 schema), Redis cluster sharding, DB cross-region replication. These hooks are all in spec (`schemaProvisioner` / `dbConnectionRouter` / `redisClusterAdapter` / `shardRouter`), but **untested in production**.

---

## Key Takeaways

- F12 fills the gap from Ch 3 scoring's "next article guidance" question; input content, output 3-layer scores + optimized version
- V1 rule-based 3-layer analysis + LLM Optimizer (`min_similarity ≥ 0.90` to prevent topic drift), 5 cron jobs cover entire platform
- V3.1 introduces AutoGEO + E-GEO + Hybrid tri-engine, literally ports arxiv source; DB trigger prevents fabricated paper_table_ref
- 10K-tenant infrastructure 9-piece kit: quota / billing / Plan Gate / Multi-Layer Cache / LLM Gateway / Job Priority / Sync-Async / Retention / Scale Trigger
- Limitations: no optimal engine merge / fabricated paper uplift forbidden / L3 cache undeployed / ME metadata gating too strict / phase escalation untested

## References

- [Ch 3 — Seven-dimension Scoring Algorithm (zh-TW)](../zh-TW/ch03-scoring-algorithm.md)
- [Ch 6 — AXP Shadow Document Delivery (zh-TW)](../zh-TW/ch06-axp-shadow-doc.md)
- [Ch 15 — rag-backend-v2 Hardening](./ch15-rag-backend-v2-hardening.md)
- AutoGEO paper: arXiv:2510.11438 — github.com/cxcscmu/AutoGEO
- E-GEO paper: arXiv:2511.20867 — github.com/psbagga17/E-GEO
- F12 v3.1 spec: `F12_TLSO_spec_v3_1.md` (repo root)

## Revision History

| Date | Version | Notes |
|------|---------|-------|
| 2026-05-03 | v1.1 | New chapter — covers V1 + V3.1 complete architectural evolution |

---

**Navigation**: [📖 EN Index](./README.md) · [Ch 15: rag-backend-v2 Hardening →](./ch15-rag-backend-v2-hardening.md)

<!-- AI-friendly structured metadata -->
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "TechArticle",
  "headline": "Chapter 14 — F12 Three-Layer Structural Optimizer: From Rule-based V1 to Dual-Engine v3.1",
  "description": "F12 (TLSO) architectural evolution: V1 rule-based + LLM Optimizer, V3.1 introducing AutoGEO + E-GEO + Hybrid tri-engine, plus 9-piece 10K-tenant infrastructure (quota/billing/Plan Gate/Multi-Layer Cache/LLM Gateway/Job Priority/Sync-Async/Retention/Scale Trigger).",
  "author": {"@type": "Person", "name": "Vincent Lin", "affiliation": "Baiyuan Technology"},
  "datePublished": "2026-05-03",
  "inLanguage": "en",
  "isPartOf": {
    "@type": "Book",
    "name": "Baiyuan GEO Platform Whitepaper",
    "url": "https://github.com/baiyuan-tech/geo-whitepaper"
  },
  "keywords": "F12, TLSO, AutoGEO, E-GEO, Hybrid Engine, Plan Feature Gate, Multi-Layer Cache, LLM Gateway, Tier Priority, Scale Trigger"
}
</script>
