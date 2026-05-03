---
title: "Chapter 14 — F12 Three-Layer Structural Optimizer: From Rule-based V1 to Dual-Engine v3.1"
description: "Architectural evolution of F12 (TLSO, Three-Layer Structural Optimizer): V1 rule-based scoring + LLM rewrite, V3.1 introducing AutoGEO + E-GEO dual engines + Hybrid integration, plus Plan Gate / Multi-Layer Cache / Tier Priority / Scale Trigger as 10K-tenant infrastructure."
chapter: 14
part: 5
word_count: 7400
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
last_modified_at: '2026-05-03T05:32:38Z'
---



# Chapter 14 — F12 Three-Layer Structural Optimizer: From Rule-based V1 to Dual-Engine v3.1

> Turn "AI citation rate optimization" from prompt-tuning alchemy into a measurable, reproducible, scalable engineering system.

## Table of Contents

- [14.1 Why F12 is Needed](#141-why-f12-is-needed)
- [14.2 V1: Three-Layer Analysis + LLM Optimizer](#142-v1-three-layer-analysis--llm-optimizer)
- [14.3 V3.1: Dual-Engine Parallel + Integration Strategy](#143-v31-dual-engine-parallel--integration-strategy)
- [14.4 10K-Tenant Infrastructure](#144-10k-tenant-infrastructure)
- [14.5 Deployment, Quota, and Billing](#145-deployment-quota-and-billing)
- [14.6 Observed Limitations and Open Problems](#146-observed-limitations-and-open-problems)
- [14.7 Engineering Lessons](#147-engineering-lessons)

---

## 14.1 Why F12 is Needed

### 14.1.1 The Problem Ch 3 Scoring Leaves Behind

Ch 3's seven-dimensional scoring answers "what is the brand citation rate?" but **does NOT tell you "how should the next article be written to improve citation rate?"** Commercial customers, holding the score, ask:

> "I know my AI citation rate is 32. How do I rewrite my next blog to push it to 60?"
>
> "Why does ChatGPT cite my fact-check page but Gemini doesn't?"
>
> "How much does citation rate differ if I rewrite the same content in different styles?"

A scoring system only "measures" — it does not "prescribe." Outsourcing the prescription to marketing copywriters or freelancers is equivalent to leaking the SaaS's core value to human experience — and human experience cannot scale to 10K tenants one-by-one.

### 14.1.2 Three Failure Modes of Early Hand-Tuning

Before V1 (Jan-Feb 2026), the platform had an internal "LLM content rewrite" tool — engineers ad-hoc ran customer content through OpenAI playground. It died within 3 months from three failures:

1. **Cost out of control**: Single customer content averages 4-6 reruns to satisfy. Unit cost ≈ $0.50/article × 30 customers × 22 page_types = 660 calls = $330. Twice monthly for full regeneration = $660, not counting retries. Customer monthly fee starts at $99 — gross margin negative.
2. **Non-reproducible**: Same prompt, different result the next day (GPT-4's default temperature is 1). Customer asks "Why did you write it this way last time and that way this time?" Engineer can't answer.
3. **Quality regression**: After LLM "optimization," customers don't recognize their own content. One anonymized case: an insurance agent's "savings insurance product introduction" was rewritten by GPT-4 into "investment advice" (because the prompt emphasized informativeness), violating Taiwan FSC regulations — the customer was nearly reported.

The name F12 is borrowed from the Web DevTools F12 key — pressing it opens the browser's "structural inspector." We bring this metaphor to LLM citation rate optimization: **any content should be "F12-able" to see its three-layer structural score, and rewriteable on demand.**

### 14.1.3 F12 vs Ch 3 Scoring

| | Ch 3 Scoring | F12 |
|---|---|---|
| Input | brand dimension + AI platform dimension | a piece of content (URL / text / Markdown) |
| Output | 0-100 score + signal items | three-layer scores + optimized content |
| Use | "Is my brand healthy?" | "How to rewrite this article to be cited?" |
| Subject | brand-level | content-level |
| Frequency | daily / weekly scan | triggered on every article write |
| Cost | scan platform call (LLM 1× per query) | analyzer + optimizer (rule-based + LLM as needed) |

**F12's core promise: given the same piece of content, the system produces multiple variants and selects the one the LLM is most likely to cite.**

### 14.1.4 Differentiation from SEO Tools

The core of SEO tools like SurferSEO / Clearscope is still "keyword density / TF-IDF," targeting Google search ranking. F12 targets LLM citation, and the signals do not overlap:

- SEO tools count keyword frequency; F12 measures "atomic fact density" (numbers, dates, source links)
- SEO tools assess backlink quality; F12 checks "entity markup" (Schema.org Person/Organization/Service completeness)
- SEO tools look at content length; F12 examines "whether paragraphs can be directly excerpted by LLM" (TL;DR, lead-in, clear H1)

The two are not opposed — good SEO content is often good GEO content, but the reverse is not true. F12 adds "LLM-specific" signals that traditional SEO tools cannot see.

---

## 14.2 V1: Three-Layer Analysis + LLM Optimizer

### 14.2.1 Three-Layer Structural Design

| Layer | Granularity | Inspection |
|---|---|---|
| **Macro** | document-level | Has ≥1 fact-check / FAQ / comparison? Clear H1? Entity declaration? |
| **Meso** | paragraph-level | Paragraph length distribution, TL;DR upfront, lead-in sentences for LLM excerpting |
| **Micro** | sentence-level | Number / date / source-link density, entity tagging, citable atomic facts |

Each layer scores independently (0-100). Final `overall = 0.4 × macro + 0.35 × meso + 0.25 × micro` (weights stored in `scoring_configs.f12_thresholds_<brand_type>` SSOT, admin-tunable without code change).

**Why three layers**: When LLMs decide what to cite, they empirically look at three scales:

- **Document-level**: Is this article "trustworthy" content (with fact-check, clear entity)?
- **Paragraph-level**: Is there an "answer paragraph" that can be directly extracted (LLMs love TL;DR mode)?
- **Sentence-level**: Is each sentence a "citable fact" (with numbers, dates, clear subject)?

Missing any layer drops LLM citation probability by 30-50%. We ran 3-month internal experiments in Q1 2026 confirming each layer is independently effective — removing macro, citation rate dropped 31%; removing meso, dropped 47%; removing micro, dropped 38%.

### 14.2.2 Why V1 is Rule-based, Not LLM-driven

V1's analyzer is **purely rule-based**: regex + AST parsing + statistical density, no LLM. Capable of 30+ articles/sec, suitable for batch full-platform scans.

Design rationale:

1. **Cost**: 30 brands × 22 pages = 660 articles/week, LLM analysis $0.02 per article = $13.2/week. Rule-based is essentially $0.
2. **Reproducible**: Rule-based always scores the same content the same; LLM has ±5 point fluctuation.
3. **Auditable**: Customer asks "Why is macro score 60?" — system lists "missing fact-check / H1 too long / no entity" specific issues. LLM gives "feels not authoritative enough" and can't answer.
4. **Speed**: Rule-based 30+ articles/sec vs LLM 0.5 articles/sec — 60x difference.

Concrete rules (excerpt):

```yaml
macro:
  - has_fact_check: 1 if brand has any fact-check page
  - has_clear_h1: 1 if first <h1> within 100 chars and ≥10 chars
  - has_entity_declaration: 1 if Schema.org Person/Org/Service in JSON-LD

meso:
  - avg_paragraph_chars: weighted by closeness to ideal range
  - has_tldr_in_first_300: 1 if "TL;DR" / "簡言之" / "重點" within first 300 chars
  - lead_in_score: count of paragraphs starting with question/statement hook

micro:
  - number_density: count(<digit>+) / total_chars
  - date_density: count(YYYY-MM-DD | YYYY 年 X 月 | etc) / total_chars
  - source_link_density: count(<a href>) / paragraph_count
  - entity_mention_count: count of Schema.org-marked entities in text
```

Each rule's threshold (`paragraph_ideal_min`, `number_density_target`, etc.) lives in `scoring_configs` SSOT — admin tunable without code change. Painful lesson from before: thresholds hardcoded in code, change required deploy + restart, customer waits 30 minutes for new score; SSOT made it instant.

### 14.2.3 LLM Optimizer Design

If score < 70 (`scoring_configs.f12_score_thresholds.low_score`), **automatically queue** for LLM Optimizer:

```text
[Low-score page] → fetch axp_pages.content_md → feed to LLM
              → input: original + 3-layer issues + reference templates
              → constraint: rewrite raises score by ≥20, but cosine similarity ≥0.90 to stay on-topic
              → write to axp_page_history (bidirectional rollback)
              → re-run analyzer to confirm new score ≥ original + 15
              → write to axp_pages and mark needs_recompile
```

LLM prompt structure (simplified):

```text
You are a GEO content optimization expert. Below is a piece of content scored X with issues: [list]
Please rewrite to:
1. Fix listed issues
2. Preserve original intent (no off-topic, no fabricated facts)
3. Prefer TL;DR + bullet + source link structure

5 reference template styles: [list]

Original:
[content_md]

Output JSON: { rewritten: "...", reasoning: "..." }
```

`min_similarity ≥ 0.90` is the spec anchor (raised from 0.85 to 0.90, migration 186). The original 0.85 seemed reasonable, but we hit two real bugs:

1. **Insurance → investment advice drift bug**: An insurance agent's axp_pages scored low. Optimizer, aiming to "improve informativeness," rewrote "savings insurance, $5000 monthly, 20-year term, guaranteed 1.8% interest" into "investment advice: switch your savings insurance to ETF investments." Cosine 0.86, passed 0.85, but content is now illegal under Taiwan FSC.
2. **Personal IP → corporate intro drift bug**: A personal_ip brand's "personal expertise" was rewritten by Optimizer into "the company provides the following services," tone completely off. Cosine 0.88, but brand_type effectively flipped from personal to corporate.

After raising to 0.90, the former dropped to 0.83 (blocked), the latter to 0.89 (blocked). The cost is some legitimate rewrites are also blocked (~12% false negative), but compared to the risk of "illegal content live in production," the conservative threshold is acceptable.

### 14.2.4 Bidirectional Rollback Design

`axp_page_history` table stores a snapshot before each Optimizer rewrite — admin UI can rollback with one click. Design motivation:

V1 early days had no history table. One cron rerun bug caused 30 brands' fact-check pages to all be rewritten. Investigation showed Optimizer received empty ground_truth due to RAG failure, producing vacuous content. No rollback at the time — only option was force-refresh RAG + rerun pipeline, 30 brands × 22 pages = 660 LLM calls, fully recovered after 5 hours.

After adding `axp_page_history`, similar incidents resolve in 30 seconds. Cost is +40 GB/year DB storage (`content_md` avg 8 KB × 30 brands × 22 pages × 365 days × 1.5 rerun rate), but compared to a 5-hour outage, it's worth it.

### 14.2.5 axpPageWriter Hook Full Coverage

Every path writing to `axp_pages` (9 generators + 4 legacy + admin manual edit) **all hook F12 analyze**, ensuring new content is immediately scored, low scores immediately queued for optimizer. Coverage verification:

- ✅ `axpPageWriter.service.js#upsertAxpPage` (main path)
- ✅ `hybridCoordinator.service.js` (6 core types)
- ✅ `factCheckGenerator.js#refreshFactCheckPage` (raw SQL, hook added)
- ✅ `axp.controller.js#updatePageContent` (admin edit)

Missing coverage is caught by weekly backfill cron (`0 2 * * 0`) as safety net.

**Why hook full coverage is hard**: The 9 generators were not written by the same team in the same sprint — `overviewGenerator` is V1, `comparisonGenerator` is V2, `factCheckGenerator` is V2.5, `featuresGenerator` is V3. Each generator's "write to axp_pages" logic differs slightly (some use service, some use raw SQL, some bypass service to INSERT directly).

We spent a week grepping the codebase to list all "write to axp_pages" paths:

```bash
grep -rn "INSERT INTO axp_pages\|UPDATE axp_pages\|axp_pages.*INSERT\|axp_pages.*UPDATE" backend/src/
```

Found 13 ingress points, added F12 hook to each. Then added weekly backfill as safety net — if a future generator is added without hook, backfill runs Sunday 02:00 to re-analyze all platform, catching the missed ones. Measured: weekly backfill takes ~12 minutes (30 brands × 22 pages = 660 articles, ~1 article/sec).

### 14.2.6 Five Cron Time-Slot Design

| Cron | Schedule | Purpose |
|------|----------|---------|
| `f12-weekly-backfill` | `0 2 * * 0` | Full brand × 22 page F12 reanalysis + brand_faq health check |
| `f12-trends-refresh` | `30 2 * * *` | REFRESH MATERIALIZED VIEW `structural_score_trends` |
| `f12-low-score-optimizer` | `0 4 * * *` | LLM rewrite low-score pages, bidirectional rollback |
| `f12-immediate-optimize` | priority queue | axpPageWriter hook trigger, score < 70 immediate enqueue |
| `f12-retention-cleanup` | `50 3 * * *` | 30-day cleanup of old optimization runs (later absorbed by v3.24 retention) |

Time-slot rationale:

- **02:00 UTC = Taiwan 10:00**: lowest platform traffic (Taiwan customers just arrived at work but not actively operating). Weekly backfill takes 12 min, impact negligible.
- **02:30 UTC**: trends MV refresh ~30 sec, must run after backfill (latter writes data, former reads).
- **03:50 UTC**: retention cleanup after trends (latter depends on history).
- **04:00 UTC = Taiwan 12:00**: LLM Optimizer runs platform-wide. This time slot was chosen because Anthropic / OpenAI's RPM peak has typically passed by UTC 04:00 (US west coast pre-dawn, no one prompting). Measured: same prompt at UTC 04:00 vs UTC 14:00, the latter is 1.8x slower (rate limit queueing).

priority queue is real-time triggered, not in cron table — axpPageWriter hook detects score < 70 in new content, immediately enqueues, worker processes within 1-2 minutes.

---

## 14.3 V3.1: Dual-Engine Parallel + Integration Strategy

V1 ran for 4 months before we found a clue to "next-level uplift" in two arxiv papers published in 2026.

### 14.3.1 Two Academic Sources

**AutoGEO** (arXiv:2510.11438, github.com/cxcscmu/AutoGEO):

Published by a CMU team. Core method: **automatically mine generalizable rewrite rules by comparing thousands of "cited by LLM vs not cited" content pairs**. Their experimental scope:

- 25 rules across two datasets:
  - Researchy-GEO (academic content) + Gemini: 15 rules
  - E-commerce + Gemini: 10 rules
- Each rule is an LLM-readable instruction (e.g., "add an explicit topic statement at the start")
- Experimental LLMs: primarily Gemini-2.0-flash + GPT-4o + Claude-3.5

Pros: **Derived from real comparative experiments**, not intuition
Cons: relatively narrow rule set (only 25), most targeting Gemini

**E-GEO** (arXiv:2511.20867, github.com/psbagga17/E-GEO):

Published by an independent researcher. Core method: **generate from different prompt styles (authoritative / technical / unique / fluent / clickable / diverse / quality / competitive / trick / format / FAQ / advertisement / language / minimalist / storytelling)**, measure citation-rate differences across LLMs.

- 15 templates (= 15 prompt styles)
- Each template is a "rewrite instruction" (e.g., authoritative = "rewrite in authoritative tone, add sources, data, citations")
- Experimental scope: GPT-4o / Claude-3.5 / Gemini-2.0 / DeepSeek-V3

Pros: **diverse templates** suitable for use cases needing specific tone
Cons: **no per-template uplift numbers given** (only overall macro experimental conclusions)

### 14.3.2 Why Port Both, Not Just One

The two methodologies are complementary:

- AutoGEO's rules are **"derived from objective comparative experiments,"** suitable for batch automation (most F12 cases)
- E-GEO's templates are **"manually defined prompt styles,"** suitable for "specific tone needed" cases (e.g., customer wants authoritative tone for whitepaper)

We **literally line-by-line port** both into the system, **without fabricating any paper_table_ref / expected_uplift** (the original papers don't give per-row mappings, fabricating would violate Engineering Constitution #1 "no simulated data").

> ⚠️ Important: V3.1's `autogeo_rules` table initial seed contained fabricated paper_table_ref ('Table 3', etc.) + fabricated expected_uplift (`1.32-1.92`), caught by Constitution #1 trigger and re-ported from real arxiv source. DB-level placeholder guard trigger added permanently. See 14.3.5.

### 14.3.3 Three-Engine Architecture

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
                    │ ME custom   │ ← personal_ip exclusive, gated by personal_profiles
                    │ engine      │
                    └─────────────┘
```

All four engines emit **standardized EngineResult** schema:

```typescript
interface EngineResult {
  engine: 'autogeo' | 'egeo' | 'hybrid' | 'me_custom';
  rule_or_template_id: string;         // arxiv source or internal id
  source: string;                       // 'arXiv:2510.11438', etc.
  original_content: string;
  rewritten_content: string;
  diff: { additions, deletions, similarity };
  scores: { before, after, delta };
  llm_call: { provider, model, tokens, cost };
  accepted: boolean;                    // optimizer self-determined (score uplift ≥15 + similarity ≥0.90)
}
```

`executeRoute` unified audit + persist + cache. Hybrid engine's "parallel" implementation:

```javascript
const [autogeoResult, egeoResult] = await Promise.allSettled([
  Promise.race([autogeoEngine.optimize(input), timeout(30_000)]),
  Promise.race([egeoEngine.optimize(input), timeout(30_000)]),
]);

const candidates = [autogeoResult, egeoResult]
  .filter(r => r.status === 'fulfilled' && r.value.accepted)
  .map(r => r.value);

if (candidates.length === 0) {
  return { engine: 'hybrid', accepted: false, reason: 'all_engines_failed' };
}

return candidates.reduce((best, c) =>
  c.scores.delta > best.scores.delta ? c : best
);
```

`Promise.allSettled` ensures one engine's failure doesn't affect the other; 30-sec timeout prevents LLM hang (aligned with platform constitution "LLM call must have ≤60s timeout").

### 14.3.4 Routing Decision

`adaptiveRouter.decide({ brandId, contentType, ... })` returns `{ path, template, target_engine }` based on this priority:

1. **plan gate**: starter always goes E-GEO `authoritative` (lowest cost); Pro+ can use sync API; Enterprise+ can use Hybrid and AutoGEO
2. **page_type → template**: `f12_page_type_to_template` SSOT (22+1 page_types → 5 template categories)
3. **content_type specialization**: whitepaper / case_study / fact_* forced to E-GEO `authoritative`; FAQ to `FAQ`; product to `clickable`
4. **brand_type fallback**: personal_ip to `me_custom` (only this baiyuan internal template)

Full decision tree:

```text
adaptiveRouter.decide(input)
├── if brand.tier === 'starter':
│     return { path: 'egeo', template: 'authoritative' }
├── if input.contentType in ['whitepaper', 'case_study', 'fact_check']:
│     return { path: 'egeo', template: 'authoritative' }   ← forced authoritative
├── if input.contentType === 'faq':
│     return { path: 'egeo', template: 'FAQ' }
├── if brand.brand_type === 'personal_ip':
│     return { path: 'me_custom' }                         ← ME gating
├── if brand.tier in ['enterprise', 'group']:
│     return { path: 'hybrid' }                            ← dual-engine parallel
└── default (Pro tier general content):
      template = mapPageTypeToTemplate(input.page_type);
      return { path: 'egeo', template };
```

**Why plan gate is dual-layered (router + executeRoute)**: The router blocks at decision time, but executeRoute is a defensive second layer — if router config changes incorrectly, executeRoute re-checks brand.tier vs path; if mismatch, reject directly. We hit this once: a router config change missed updating starter's path; a starter brand triggered 3 hybrid calls (3× LLM cost) before executeRoute caught it. After adding defensive plan_gate, similar incidents are no longer possible.

### 14.3.5 The Placeholder Guard Trigger Story

V3.1's `autogeo_rules` table initial seed (migration 189) hit a major incident — **fabricated paper_table_ref**.

The spec at the time said "each rule maps to paper Table 3 row N." Engineers intuitively filled in `'Table 3'` / `'Table 4'`, but actually checking arXiv:2510.11438, the paper does NOT give "rule X corresponds to Table Y row Z" mappings. Similarly, `expected_uplift` was filled with `1.32-1.92` ranges, but the paper's experiments only give overall uplift, not per-rule.

Engineering Constitution #1 "no simulated data" trigger blocks:

```sql
CREATE OR REPLACE FUNCTION autogeo_rules_placeholder_guard()
RETURNS trigger AS $$
BEGIN
  IF NEW.paper_table_ref IS NOT NULL
     AND NEW.source NOT LIKE 'arXiv_%'
     AND NEW.source NOT LIKE 'arXiv:%' THEN
    RAISE EXCEPTION 'paper_table_ref must be NULL or source must be arXiv (Constitution #1)';
  END IF;

  -- 6 patterns to detect placeholder
  IF NEW.rule_description ~ '(待填入|To be filled|Actual Data Pending|TODO:|FIXME:|XXX)' THEN
    RAISE EXCEPTION 'placeholder pattern detected in rule_description (Constitution #1)';
  END IF;

  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_autogeo_rules_placeholder_guard
BEFORE INSERT OR UPDATE ON autogeo_rules
FOR EACH ROW EXECUTE FUNCTION autogeo_rules_placeholder_guard();
```

migration 191 used `TRUNCATE ... RESTART IDENTITY CASCADE` to clear fabricated seed; migration 192 added the trigger and re-ported real arxiv source. The `autogeo_rules` table's 25 rows are all `source = 'arXiv:2510.11438'`, with real:

- `target_engine ∈ {gemini, gpt, claude}`
- `target_domain ∈ {researchy_geo, geo_bench, ecommerce}`
- `paper_table_ref` all NULL (arxiv has no per-row mapping; Constitution #1 forbids fabrication)
- `expected_uplift` all NULL (overall experimental uplift, not per-row — V2's real ML measurement will fill)

`egeo_templates` 15 rows all `source = 'arXiv:2511.20867'`, template names exactly aligned with paper Table 1 (authoritative / technical / unique / fluent / clickable / diverse / quality / competitive / trick / format / FAQ / advertisement / language / minimalist / storytelling).

Baiyuan internal 1 row `me_personal_ip` explicitly `source = 'baiyuan_internal_v3_1'`, not mixed with arxiv naming.

**This trigger is a "post-incident retrofit"** — installed only after fabrication occurred once. But once installed, no future admin edit / new generator / cron failure can violate it. At 10K-tenant scale, this hardware-layer protection is necessary.

---

## 14.4 10K-Tenant Infrastructure

V3.1 deployed scaling infrastructure across 9 batches (v3.21 → v3.27 over 8 days). Each batch addressed one bottleneck; all 9 together scaled the system from "can run 30 brands" to "designed to run 10K brands."

### 14.4.1 Quota and Billing (spec §4.3 + §4.4)

```text
TenantQuotaService(monthly_optimizations / max_content_size_kb)
  starter:  100 ops, 50 KB
  pro:      1000, 200
  enterprise: 10000, 1000
  group:    ∞, 5000

BillingTracker(per LLM call cost recording)
  9 model × per-token pricing (claude_haiku $0.0003 / opus $0.015 / 4o-mini $0.0002 ...)
  in/out multiplier 4×
```

`f12_quota_usage` table monthly counter UPSERT (`tenant_id+brand_id+action+period` unique); `f12_billing_records` 12M rows/year sufficient for single table (10K × 100 ops × 12 months). `quota_enforcement_enabled` flag defaults `false` (observation period only logs usage, doesn't block); `billing_recording_enabled` defaults `true` (always logged).

"Observation period" design: at launch, customers see "used X / limit Y this month" but aren't blocked. Observe ~2 weeks to confirm distributions are reasonable (no starter customer with 5000 ops/month outliers) before flipping to enforcement. Measured observation data:

- starter avg 18 ops/month (median 5), 95th percentile 47 — far below 100 limit
- pro avg 234 ops/month, 95th = 612 — below 1000
- enterprise avg 1820 ops/month, 95th = 4500 — below 10000

After flipping to enforcement, 0 customers were blocked, indicating quota design is reasonable.

### 14.4.2 Plan Feature Gate (spec §9)

`f12_plan_features` SSOT 4 tiers × 6 flags (`scoring_configs` table JSONB):

```yaml
egeo_template_count:    starter=0    pro=5     enterprise=15  group=15
autogeo_rules:          'none'       'none'    'full'         'full'
engine_specialization:  false        false     true           true
hybrid_engine:          false        false     true           true
sync_api:               false        true      true           true
```

Non-Enterprise running `whitepaper` / `case_study` / `fact-*` is forced to E-GEO `authoritative` (no longer AutoGEO); only Enterprise+ can use Hybrid + three-engine specialization. `adaptiveRouter` plan gate + `executeRoute` defensive plan_gate provide dual-layer enforcement.

Design rationale: starter / pro customers have low monthly fees ($99 / $499), can't afford the most expensive engine (Hybrid calls two LLMs at once), or gross margin goes negative. enterprise+ at $2000+/mo can absorb Hybrid cost.

`egeo_template_count` design: starter 0 (only default authoritative), pro 5 (authoritative / FAQ / clickable / fluent / diverse), enterprise+ all 15. Customers see "Upgrade to Enterprise to enable more templates" upsell.

### 14.4.3 Multi-Layer Cache (spec §5.2)

```text
L1 Redis (5 min TTL)        ← repeat query of same content + decision
L2 PG f12_result_cache (7d) ← cross-process / cross-instance
L3 S3 hook (TBD Phase 3)    ← needed when cross-region replication required
```

cache key = `sha256(content + decision.path + template + target_engine)`, **not including tenant_id, shared cross-tenant** (spec explicitly allows: same content + same rule → same result, sharing cuts LLM cost N-fold).

Only cache `accepted=true` results (avoid cache poisoning). `f12_cache_metrics` hourly aggregates L1/L2/miss → admin monitors 70% hit rate target.

**Cross-tenant sharing privacy boundary**: At first glance sketchy (could B's tenant see A's tenant cache?), analysis:

- cache key contains the **entire content** sha256, hits only if content is identical
- Customer A's content is 99.9% never identical to customer B's (content is KB-scale, sha256 collision near-zero)
- Real hit scenario: a generic FAQ template ("What is GEO?") used by multiple customers — sharing makes sense (no private info)

Measured 1-month cache hit rate is 38% (L1+L2), below 70% target. Reason: content carries brand-specific information (brand name, entities), so even if logic is identical, content has been uniquified. Phase 3 plans **"normalized cache key"** — replace brand-specific tokens with `<<BRAND_NAME>>` placeholders before hashing, enabling similar brands to share cache. This is V2 work.

### 14.4.4 LLM Gateway (spec §5.3)

per-model platform-layer RPM limits (in-memory sliding window 60s):

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

5 F12 LLM call sites all go through `gatewayAiCall`: autogeo / egeo / meCustom / V1 optimizer / hybrid (via the prior two). `opts.billing` if provided auto-records (backward compatible).

**Why platform-layer RPM limit is necessary**: Anthropic / OpenAI API keys are per-organization. 10K tenants share one key — single-customer burst can drain platform-wide quota. Gateway pre-checks current RPM before LLM call, queues or rejects if over.

Real scenario: an admin UI bug caused one brand to trigger 100 hybrid optimizes. Each hybrid is 2 LLM calls — 200 calls hit Claude Sonnet simultaneously. Without Gateway, this immediately exceeded Anthropic's 30 RPM, blocking all platform brands; with Gateway, blocked at our side, only affecting that brand, others continued normally.

**Limitation**: in-memory bucket is per-process — single backend container OK, multi-instance needs Redis sliding-window (hook reserved, not implemented). 10K-tenant scale expects 5-10+ backend instances; Phase 3 must switch to Redis-based.

### 14.4.5 Other 10K-Tenant Hooks

| Mechanism | Purpose |
|-----------|---------|
| **Tier-based Job Priority** (`group=1` highest → `starter=4`) | BullMQ priority, large customers first |
| **Sync/Async Dual API** | `/optimize` async returns jobId / `/optimize/sync` Pro+ only, content ≤5KB |
| **Concurrent Jobs Enforcement** | Prevent starter unlimited stacking that drags down workers (`starter=1`, `group=100`) |
| **Per-Tenant API RPM** | Redis sliding-window per brand, fail-open (when Redis fails, doesn't block) |
| **Retention TTL SSOT** | 6 tables daily 05:00 cron cleans expired rows, `scoring_configs.f12_retention_config` admin tunable |
| **Budget Alerts** | tier threshold (starter $1 / group $5000 monthly) warn@80%, critical@block_pct |
| **Scale-up Trigger** | daily 04:30 snapshot 8-dimensional metrics (tenant_count / monthly_ops / p99 / cache_hit / cost), spec-aligned phase 1→2→3 |
| **Tenant Isolation Strategy** | spec/actual dual-column, enterprise/group customer isolation gap visible to admin |
| **L3 S3 Cache hook** (Phase 3 reserve) | When `F12_S3_CACHE_BUCKET` env set, dynamic import `@aws-sdk/client-s3` |

Full SSOT in `scoring_configs.f12_*` series 14 keys; admin UI `/dashboard/admin/f12-dashboard` integrates 4 stat cards + 4 detail sections.

**Interdependency**: The 9 hooks are not independent — Concurrent Jobs Enforcement prevents worker overload → Job Priority ensures high-tier customers aren't squeezed by low-tier → Per-Tenant RPM prevents bursts → LLM Gateway prevents platform-wide quota overflow → Cache reduces LLM cost → Quota / Billing manages money → Budget Alerts prevent cost runaway → Scale-up Trigger signals when to upgrade phase. Missing any causes 10K-tenant scale failure.

---

## 14.5 Deployment, Quota, and Billing

### 14.5.1 Deployment: Wave 0 / Wave 1 Alignment

V3.1 completed spec Wave 0 (microservice contract OpenAPI) + Wave 1 (in-process implementation + SDK + admin):

- **OpenAPI 3.0 spec**: `backend/src/openapi/f12-v3-1.json` 7 paths × 14 schemas, `/api/v1/f12/openapi.json` public (before router auth) for AI agents / 3rd party machine-discovery
- **F12Client SDK**: `services/f12/f12Client.js` 5 main entry points (`diagnose / optimize / optimizeSync / getJob / health`), goes HTTP when `F12_SERVICE_URL` env set, in-process otherwise. **Caller signature unchanged** — hook for future microservice extraction
- **/api/v1/f12/health**: public endpoint before OAuth, monitoring systems can directly ping

Deployment timeline:

1. **v3.21** (2026-04-23): TenantQuotaService + BillingTracker + ME custom engine real logic + LLM Gateway
2. **v3.22** (2026-04-25): Plan Feature Gate + Multi-Layer Cache + Tier Priority + Sync/Async Dual API
3. **v3.23** (2026-04-27): F12Client SDK + OpenAPI + Tenant Isolation hook + L3 S3 hook + Scale-up Trigger
4. **v3.24** (2026-04-29): Concurrent Jobs + Per-Tenant RPM + Retention + Budget Alerts
5. **v3.25** (2026-04-30): Retention dedup + Redis Billing cache + Admin Dashboard
6. **v3.26** (2026-05-01): AI bot whitelist SSOT (aligned with 7-layer audit fix)
7. **v3.27** (2026-05-02): Phase 2-3 scaling infrastructure hooks (schema/db provisioner, Redis cluster, shard router, multi-region)

Each batch full flow: migration SQL → business logic → spec test → admin UI → CLAUDE.md doc → push PROD.

### 14.5.2 Quota: Observation → Enforcement

After launch, `quota_enforcement_enabled=false` (only logs usage), customers see "used X / limit Y" but aren't blocked. Observe ~2 weeks to confirm reasonable distribution before admin one-clicks to `true` — overage now returns HTTP 429 + reject reason `'quota_exceeded'`.

Pre-switch preparation:

1. **Customer notification**: UI shows "used X / Y this month, next month overage will be blocked, please contact for upgrade"
2. **Customer support training**: Blocked customers will ticket; CS must master "upgrade plan / month-end reset / emergency quota adjustment" three responses
3. **Monitoring red-light**: Admin dashboard shows "blocked brands in past 24h," > 5 triggers alert (if it's a bug, not real overage, must roll back fast)
4. **Rollback plan**: One-click flip back to `false` takes 5 sec (SSOT in DB, no deploy needed)

### 14.5.3 Billing: Per-Call Recording, End-of-Month Aggregation

`BillingTracker.recordCall(model, tokens, runId)` logs every LLM call. End-of-month admin runs `/admin/f12-billing-summary?period=YYYY-MM` to see top 50 cost brands.

Dual-write strategy: Redis (`f12:billing:tenant:{id}:{period}` INCRBYFLOAT + 35-day TTL) + DB (`f12_billing_records`). Current month `getBrandMonthlyCost` prioritizes Redis O(1) GET; past months or Redis miss falls back to DB SUM. 10K-tenant query monthly cost doesn't hit DB, sub-ms response.

35-day TTL is intentional: covers current + previous month, customer querying last month's bill on the 7th gets it instantly. After 35 days auto-expire, DB is always source of truth.

Monthly batch runs "cost vs subscription" comparison; brands with negative gross margin auto-alert (could be pricing error / customer misuse / bug), admin reviews.

---

## 14.6 Observed Limitations and Open Problems

### 14.6.1 No Correct Answer for "Merging" Dual-Engine Results

Hybrid engine runs AutoGEO + E-GEO in parallel, picks the higher `improvement`. But in real business contexts, improvement isn't a single metric:

- AutoGEO's rule rewrite is structurally stable but less brand personality
- E-GEO's templates are diverse but sometimes rewrite to where the customer doesn't recognize their content

V3.1 uses "engine improvement score higher" as selection criteria — actually insufficient. Next version wants to add **"accepted by client" signal**, but admin-side editing UI isn't online yet (Phase 3).

### 14.6.2 paper_table_ref / expected_uplift are NULL

"Literally porting arxiv" is a side effect of Engineering Constitution #1 — paper has no per-row table mapping, so the system **forbids fabrication**. But when admin UI shows "why this rule was selected," missing data displays blank.

V2 plan: **in-house measurement** — for each rule, run real ML eval (same prompt × 3 LLMs × 1000 brands, measure citation_rate uplift) and fill in real numbers. Requires time + Anthropic / OpenAI quota:

- 25 rules × 3 LLMs × 100 brands × 5 repeats = 37500 LLM calls
- Claude Sonnet $0.003/1k tokens, avg 5k tokens/call = $0.015/call
- Total cost ≈ $562
- Time: Anthropic RPM 2000/min, full run ≈ 18 minutes

Cost is low, but requires 1000-brand real baseline citation rate data. Currently only 30 brands; will run at Phase 3 scale (5000+ brands).

### 14.6.3 L3 S3 Cache Not Yet Deployed

Phase 3 only (estimated 5000+ tenant scale). Currently L1+L2 hit rate ~35-45%, far from 70% target. L3 expected to add 15-20% but requires S3 cost vs LLM recompute cost tradeoff analysis — not done.

Rough estimate: L3 S3 PUT $0.005/1000 obj, GET $0.0004/1000 obj, storage $0.023/GB/month. 10K brands × 100 ops × 5 KB / op = 5 GB, storage $0.115/month negligible. But each GET 0.4 ms latency vs L1 Redis 0.05 ms — L3 8x slower. Phase 3 must place L3 in cross-region (US + JP + EU) to amortize latency.

### 14.6.4 ME Custom Engine Gating Too Strict

`personal_profiles` table metadata all empty → reject `meta_incomplete`. Observed: ME platform 17 member brands, 5 have incomplete metadata (no full_name / known_for). These 5 can't go through ME custom engine, fallback to E-GEO `authoritative` but result is corporate-toned, unsuitable for personal IP. Customer service needs to chase customers to fill metadata, but this violates the "automation" promise.

Next version considers: **LLM auto-infers missing personal_profiles fields from brand_documents**, but needs high-confidence guard against hallucination. E.g., "What is Tina Chou's jobTitle?" — LLM infers from her publications / media coverage, only fills if confidence > 0.95.

### 14.6.5 Pre-Tested 10K-Tenant Coverage

V3.1 infrastructure all aligned with "10K-tenant capable" design, but real load test only goes to 17 ME brands + 21 GEO brands. Scale-up Trigger's phase_1_to_2 / phase_2_to_3 logic is literally aligned with spec line 1183-1193 but **never actually upgraded**. Theoretically cross-phase upgrade requires: tenant schema isolation switch (Phase 1 row_level → Phase 2 schema), Redis cluster sharding, DB cross-region replication. These hooks all reserved in spec (`schemaProvisioner` / `dbConnectionRouter` / `redisClusterAdapter` / `shardRouter`), but **untested in production**.

Next year's work items:

- **Synthetic load test**: write a generator simulating 1000 brands × 100 ops/month, run for 1 month to check Scale-up Trigger fires as expected
- **Phase 2 dry-run**: in staging actually switch schema isolation, test query paths and cross-schema JOIN still work
- **Phase 3 cross-region sketch**: test how Cloudflare Workers route to nearest region

These all can't be done in production directly; need staging environment + extra hardware budget.

### 14.6.6 Cross-Tenant Cache Privacy Boundary

14.4.3 mentioned cache key excludes tenant_id, sharing across tenants reduces LLM cost. Sketchy at first glance but design analysis:

- **Content-identical-only hits**: cache key is `sha256(content + decision)`, content is KB-scale, two customers writing identical content has near-zero probability (unless plagiarism)
- **Real hit scenarios**: generic FAQ template "What is GEO?" answer used by multiple customers — sharing makes sense (no private information)
- **Future risk**: if cache key normalization (replacing brand-name tokens with placeholders) launches, cross-brand hit rate rises, but normalization must strictly guarantee no information leakage
- **Regulatory alignment**: GDPR / Taiwan Personal Data Protection Act don't prohibit "shared LLM results for identical public content," because the content itself is brand voluntarily public information

V2 plans to add audit log: each cache hit records "from brand A → to brand B," customer can opt-out of cross-tenant sharing (in exchange for higher own LLM cost).

### 14.6.7 Cache Key Implicit Assumption — LLM Determinism

cache key = `sha256(content + decision.path + template + target_engine)`, implicit assumption: **LLM call is deterministic** (same input → same output). Reality:

- `temperature=0` mostly deterministic, but OpenAI / Anthropic don't guarantee 100% (infrastructure-layer micro-noise)
- Same prompt run 100 times: ~95% identical / ~5% wording-different but semantically same

Cache hit returns old result, customers can't tell difference, because 95% determinism is high consistency. But future, if LLM API becomes "explicitly non-deterministic" (e.g., OpenAI Sora / Gemini Veo video gen), cache must add versioning + invalidation.

Implementation reserves: `f12_result_cache` has `cache_version` column; Phase 3 LLM upgrade can one-click bump version to invalidate platform-wide cache.

---

## 14.7 Engineering Lessons

From V1→V3.1 evolution (7 months, 5 PROD incidents), 5 takeaway lessons:

### 14.7.1 Literal arxiv Port is Safer Than Self-Invention

We initially wanted to invent GEO rules ourselves (based on internal observation), but quickly discovered:

- Lacking comparative experiments (own 30 brands not enough to derive rules)
- Easy to overfit to own customer style
- No academic reference, customers don't buy ("Why this way?" "We feel it's good" ← weak)

After switching to "literal arxiv source port":

- ✅ 25 + 15 rules/templates immediately usable
- ✅ Customers see "arXiv:2510.11438" and instantly trust
- ✅ DB-level placeholder guard permanently prevents fabrication

But cost is paper_table_ref / expected_uplift are NULL, UI displays blank. We'd rather have blank than fabricate — that's the price of Constitution #1.

### 14.7.2 Platform-wide SSOT Beats Hardcoded Constants

V1 early days, all thresholds hardcoded (`const LOW_SCORE_THRESHOLD = 70`), changes required deploy + restart. Customer asks "Why is my page 65 not optimized?" — engineer greps code and finds `hard-coded 70`, changes to 65 and deploys. Whole process 30 minutes.

After switching to SSOT (`scoring_configs.f12_score_thresholds.low_score`):

- Admin UI changes a number, takes effect in 5 sec
- Different brand_types can have different thresholds (GEO 70, ME 75)
- One-click rollback on issues (SQL UPDATE prior value)

Cost: one extra layer of indirection — newcomers reading code need an extra grep. But compared to deploy speed, worth it.

### 14.7.3 DB-level Enforcement is More Reliable Than Application-level

placeholder guard was originally going to be done at application-level (each INSERT, application checks):

- ❌ Bypassable when admin UI directly executes SQL (psql to PROD does happen occasionally)
- ❌ Generators writing raw SQL `INSERT INTO axp_pages` bypass service layer
- ❌ Pipeline cron old code goes through service but service doesn't check

After switching to DB trigger:

- ✅ Any path's writes are intercepted (including direct psql INSERT)
- ✅ Application code can trust (trigger as backstop)
- ✅ Spec more stable (trigger rules are SQL-only, not easily mis-changed)

Cost: trigger debugging is harder (error messages unclear), but at 10K-tenant scale this cost is worth it.

### 14.7.4 Observation Period is SaaS Deployment Standard

quota_enforcement_enabled / billing_recording_enabled flags all default "observation period" (only log, don't block) — confirm distributions reasonable before flipping to "enforcement period." Observation period ≥ 2 weeks, during which:

- Observe real usage distribution
- Find outliers (could be bug, could be real usage)
- Notify customers "next month will be blocked"
- Train customer support

V1 early days lacked this mechanism, directly launched enforcement. Result:

- Week 1: 3 customers hit overage block, but actually admin had set quota wrong (100 written as 10)
- CS not trained, customer ticket couldn't be answered
- Eventually one-click rollback enforcement, customer experience already damaged

After adding observation period, similar incidents no longer happen.

### 14.7.5 Hook Coverage + Safety Net Both Required

axpPageWriter F12 hook must have 100% coverage, but 100% is hard to guarantee (13 ingress points, new generators added anytime):

- **Hook 100%**: first line of defense, real-time F12 analyze trigger
- **Weekly backfill cron**: second line of defense, Sunday 02:00 platform-wide reanalysis catches misses
- **Admin manual trigger**: third line of defense, admin can manually trigger single-brand reanalysis on anomaly

All three are required — hook alone misses → permanent miss; backfill alone is untimely (max 7 days reaction); manual alone doesn't scale.

V1 early days only had hook. Once factCheck generator missed hook, 4 brands' fact-check pages had abnormally low scores for 3 months before discovery. After adding backfill, similar incidents catch within 7 days max.

---

## Chapter Takeaways

- F12 fills in the "how to write next" question that Ch 3 scoring leaves behind, input content outputs three-layer scores + optimized version
- V1 rule-based three-layer analyzer + LLM Optimizer (`min_similarity ≥ 0.90` against drift), 5 cron pipeline with full platform coverage
- V3.1 introduces AutoGEO + E-GEO + Hybrid three engines, literal arxiv port, DB trigger prevents fabricated paper_table_ref
- 10K-tenant infrastructure 9-piece set: quota / billing / Plan Gate / Multi-Layer Cache / LLM Gateway / Job Priority / Sync-Async / Retention / Scale Trigger
- Limitations: engine merging no best answer / paper real uplift pending in-house measure / L3 cache not deployed / ME metadata gating too strict / phase upgrade untested / cross-tenant cache privacy boundary / cache key assumes LLM deterministic
- 5 engineering lessons: literal arxiv port > self-invention / SSOT > hardcoded / DB-level > application-level / observation period is standard / hook + safety net both required

## References

- [Ch 3 — Seven-Dimensional Scoring Algorithm](./ch03-scoring-algorithm.md)
- [Ch 6 — AXP Shadow Document Delivery](./ch06-axp-shadow-doc.md)
- [Ch 12 — Limitations, Open Problems, and Future Work](./ch12-limitations.md)
- AutoGEO paper: arXiv:2510.11438 — github.com/cxcscmu/AutoGEO
- E-GEO paper: arXiv:2511.20867 — github.com/psbagga17/E-GEO
- F12 v3.1 spec: `F12_TLSO_spec_v3_1.md` (repo root)

## Revision History

| Date | Version | Notes |
|------|---------|-------|
| 2026-05-03 | v1.1 | New chapter — covers V1 + V3.1 full architectural evolution |
| 2026-05-03 | v1.1.1 | Chapter expanded to ~7400 words — added 14.1.2 early hand-tuning failures, 14.2.4 bidirectional rollback, 14.2.5 hook coverage hardness, 14.2.6 cron timing, 14.3.5 placeholder guard story, 14.4 interdependency map, 14.5 deployment timeline, 14.6.6/7 cache privacy + assumption, 14.7 engineering lessons (5 takeaways) |

---

**Navigation**: [← Ch 13: Multimodal GEO](./ch13-multimodal-geo.md) · [📖 TOC](../README.md) · [Ch 15: rag-backend-v2 Hardening →](./ch15-rag-backend-v2-hardening.md)

<!-- AI-friendly structured metadata -->
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "TechArticle",
  "headline": "Chapter 14 — F12 Three-Layer Structural Optimizer: From Rule-based V1 to Dual-Engine v3.1",
  "description": "F12 (TLSO) architectural evolution: V1 rule-based + LLM Optimizer, V3.1 introduces AutoGEO + E-GEO + Hybrid three engines, plus 9-piece 10K-tenant infrastructure (quota/billing/Plan Gate/Multi-Layer Cache/LLM Gateway/Job Priority/Sync-Async/Retention/Scale Trigger). With 5 engineering lessons.",
  "author": {"@type": "Person", "name": "Vincent Lin", "affiliation": "Baiyuan Technology"},
  "datePublished": "2026-05-03",
  "inLanguage": "en",
  "isPartOf": {
    "@type": "Book",
    "name": "Baiyuan GEO Platform Whitepaper",
    "url": "https://github.com/baiyuan-tech/geo-whitepaper"
  },
  "keywords": "F12, TLSO, AutoGEO, E-GEO, Hybrid Engine, Plan Feature Gate, Multi-Layer Cache, LLM Gateway, Tier Priority, Scale Trigger, Engineering Lessons"
}
</script>
