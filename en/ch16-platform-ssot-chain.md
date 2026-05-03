---
title: "Chapter 16 — Platform SSOT Chain: From brand_faq to page_type to alerts as Single Source of Truth"
description: "A 10K-tenant SaaS must unify all cross-page, cross-table, cross-crawler data paths under a single source of truth. This chapter records the complete engineering design of brand_faq SSOT three-way consistency (human / Schema.org / AXP shadow), page_type SSOT 4-place sync mechanism, and alerts 4-table UNION."
chapter: 16
part: 5
word_count: 3600
lang: en
authors:

  - name: Vincent Lin
    affiliation: Baiyuan Technology
license: CC-BY-NC-4.0
keywords:

  - Single Source of Truth
  - SSOT
  - brand_faq
  - page_type Registry
  - Schema.org FAQPage
  - alerts UNION
  - PostgreSQL UUID bigint
  - Multi-Tenant Consistency
last_updated: 2026-05-03
canonical: https://baiyuan.io/whitepaper/en/ch16-platform-ssot-chain
last_modified_at: '2026-05-03T02:45:21Z'
---




# Chapter 16 — Platform SSOT Chain: From brand_faq to page_type to alerts as Single Source of Truth

> The engineering reality of 10K tenants: any single inconsistency gets amplified 100x into 100 customer support complaints. SSOT isn't an aesthetic preference — it's a survival condition for scale.

## Table of Contents

- [15.1 Why Platform-level SSOT Is Required](#151-why-platform-level-ssot-is-required)
- [15.2 brand_faq SSOT Three-way Consistency](#152-brand_faq-ssot-three-way-consistency)
- [15.3 page_type SSOT 4-place Sync Mechanism](#153-page_type-ssot-4-place-sync-mechanism)
- [15.4 Alerts 4-table UNION: Cross-table SSOT Type Trap](#154-alerts-4-table-union-cross-table-ssot-type-trap)
- [15.5 SSOT Design Pattern Summary](#155-ssot-design-pattern-summary)
- [15.6 Observations and Limitations](#156-observations-and-limitations)

---

## 15.1 Why Platform-level SSOT Is Required

The platform spans 5 hosts (geo / me / rag / pif / www) + 5 i18n locales + dark/light theme + AI bot vs human routing. A single piece of data (e.g., "Baiyuan Technology FAQ Q3 answer") can appear in:

1. Customer dashboard UI (human edits)
2. Main site home page Schema.org JSON-LD (human users see)
3. AXP shadow `/c/{slug}/schema.json` (AI bots scrape)
4. AXP shadow `/c/{slug}/faq` HTML page (AI bots scrape)
5. Cloudflare Worker injecting FAQPage Schema into origin HTML (idempotent fallback)
6. `llms.txt` / `llms-full.txt` public files (LLM training crawlers)
7. Customer site sitemap.xml URLs (Google indexing)
8. Central RAG KB (LLM Wiki compile source)

**Any single point out of sync** results in: Google indexes 21 FAQ Qs, but the customer dashboard shows 20; or ChatGPT cites Q3's old answer when the customer says "I already updated Q3."

Three months of observed SSOT violation cases:

| Violation | Consequence |
|---|---|
| Home page Schema.org sync 24h behind brand_faq table | Google rich result shows old Q&A |
| New page_type added to enum but sitemap.js missed | New page absent in sitemap, Google doesn't index |
| Alerts UI shows "unread 165" but list empty | Cross 4-table UNION type mismatch, query throws |

Each case stems from "two places independently maintaining the same concept." This chapter records the engineering implementation of three key SSOT chains.

---

## 15.2 brand_faq SSOT Three-way Consistency

### 15.2.1 Three-way Requirement

For any customer brand, FAQ content must be **completely consistent across three paths**:

| Path | Audience | Implementation |
|---|---|---|
| **A. Real human visitors** | dashboard UI + customer site embed | brand_faq table + Next.js page.tsx |
| **B. Google / Bing / general search engines** | Main site SSR Schema.org FAQPage | `frontend/src/app/HomepageJsonLd.tsx` |
| **C. AI bots (GPTBot / ClaudeBot / PerplexityBot)** | AXP shadow `/c/{slug}/schema.json` + `/faq` page | `backend/src/services/axp/shadowPublicFiles/generators/schemaJson.js` |

### 15.2.2 SSOT Table

`brand_faq` is the **only write point**, with 4 columns:

```sql
CREATE TABLE brand_faq (
  id UUID PRIMARY KEY,
  brand_id UUID NOT NULL REFERENCES brands(id),
  question TEXT NOT NULL,
  answer TEXT NOT NULL,
  is_published BOOLEAN DEFAULT TRUE,
  -- ...
);
```

**All read paths share this table**, with no cache layer or derived table in between.

### 15.2.3 Three Read Paths

#### Path A — Human visitors

`dashboard/brand-entity` page's FAQ editing area queries `brand_faq` directly; writes update the table. Customer site embed widgets read via `GET /api/v1/c/:slug/brand-faq.json`.

#### Path B — Schema.org main site SSR

`HomepageJsonLd.tsx` (Server Component) detects current host via `headers().get('host')` → `HOST_TO_SLUG` map → `fetch('/api/v1/c/' + slug + '/brand-faq.json')` → injects `<script type="application/ld+json">` FAQPage @graph.

#### Path C — AXP shadow

CF Worker intercepts AI bot UA → proxies to `/api/v1/axp/render` → backend calls `schemaJson.js#renderSchemaJson({ brandFaqs })`, where `brandFaqs` comes from `dataFetcher.fetchBrandPublicData()` reading `brand_faq` rows.

`/c/{slug}/faq` HTML page uses the same path.

### 15.2.4 Consistency Verification

`scripts/audit/crawler-7layer.sh` automatically runs 5-host × 4-UA cross-matrix:

```bash
for UA in "Mozilla/5.0" "Googlebot" "GPTBot" "PerplexityBot"; do
  for HOST in geo.baiyuan.io me.baiyuan.io rag.baiyuan.io; do
    Q=$(curl -sL -A "$UA" "https://$HOST/" | grep -oE '"@type":"Question"' | wc -l)
    echo "$HOST × $UA → $Q Questions"
  done
done
```

**Expectation**: same host across different UAs should yield same Q count (SSOT rule). One observed exception: `geo.baiyuan.io` Mozilla 21Q vs Bot 20Q (1 question diff), root cause being schemaJson generator's filter rule for one specific FAQ (answer too short or other criterion). Known minor inconsistency, doesn't affect Google rich result main flow.

### 15.2.5 Recurrence Prevention

Any new Schema.org FAQPage rendering entry point must read `brand_faq` table. Self-check:

```bash
# Find new injection points; they should trace back to brand_faq
grep -rn "'@type': *['\"]FAQPage" frontend/src/ backend/src/services/
# Each must trace to brand_faq source; no hardcoded FAQ
```

`CLAUDE.md §"Public files production rule"` explicitly forbids "hardcoded Question arrays in page.tsx / faq/page.tsx / any frontend file." Historical lesson: `pricing/layout.tsx` once hardcoded NT$1500, but DB had been adjusted to 2500; Schema deceived Google for two months before being caught.

### 15.2.6 ME Platform Special Case

`me.baiyuan.io/` middleware rewrites to `/personal`, so `<HomepageJsonLd />` injects Schema in `/personal/page.tsx`. `/personal/faq/page.tsx` has dual sources: brand_faq SSOT preferred + `_content/faq.ts` 5-locale hardcoded fallback (used when new member brand hasn't loaded data yet); **direct `_content/faq.ts` import is forbidden** (it's only fallback).

---

## 15.3 page_type SSOT 4-place Sync Mechanism

### 15.3.1 Platform-level page_type Concept

The AXP system has 23 page types (22 enterprise + 1 personal_ip exclusive `future_plans`): brand_overview / faq / pricing / comparison / fact_check / use_cases / updates / team / regions / slogan / trust / specs / plans / vs / what_is / how_to / best_for / testimonials / press / cases / integrations / alternatives / + 1 me-only.

Each page type involves:

- URL slug (sitemap.xml lists `/c/{brand}/{slug}`)
- Schema.org breadcrumb URL
- Chinese title (GEO uses "品牌總覽" / ME uses "個人簡介")
- AXP page meta (priority / changefreq for sitemap)
- Group classification (used for llms-full.txt section ordering)

### 15.3.2 4-place SSOT Must Sync

Adding / renaming / removing a page type requires **4 file changes**:

| # | File | Content |
|---|---|---|
| 1 | `backend/src/services/axp/pageTypeRegistry.js` | `PAGE_TYPE_TO_SLUG` + `PAGE_TYPE_GROUP` + `AXP_PAGE_META` + `PAGE_TYPE_TITLES_ZH` (GEO words) + `PAGE_TYPE_TITLES_PERSONAL_ZH` (ME words) |
| 2 | `frontend/src/lib/axpPageTypeRegistry.ts` | client-side mirror (slug + group + brand_type-aware label) |
| 3 | `frontend/src/lib/axpPageTypeLabels.ts` | 5 locales × ENTERPRISE/PERSONAL × {label, desc} |
| 4 | `backend/src/services/axp/shadowPublicFiles/generators/llmsFullTxt.js` | group + order + English section title |

Missing any file → sitemap missing URLs / Schema.org breadcrumb 404 / RSS feed wrong title / customers see unfamiliar labels.

### 15.3.3 Auto-derive vs Manual-sync Split

We auto-derive what we can; what we can't is enforced by PR template checklists:

- ✅ **Auto-derive**: `SLUG_TO_PAGE_TYPE` reverse-lookup via `Object.fromEntries(PAGE_TYPE_TO_SLUG)`; `hybridCoordinator.EXACT_SLUG_MAP` / `crawlerVisibility.GROUP_MAP` reference SSOT directly
- ⚠️ **Manual sync**: 5 locales × 23 entries × {label, desc} = 110+ translations; new page type requires all entries

Past pitfall: `Schema.org breadcrumb URL ≠ sitemap URL` (`brand_overview→brand` vs `→overview`), causing Google-indexed BreadcrumbList URLs to all 404 against CF Worker. Fix pulls both from `PAGE_TYPE_TO_SLUG` SSOT, so **Schema.org URL ↔ sitemap URL ↔ CF Worker route are all three identical**.

### 15.3.4 brand_type-aware Wording Auto-routing

`pageTypeTitleZh(pt, brandType)` helper switches 23 wordings:

| page_type | GEO (enterprise) | ME (personal_ip) |
|---|---|---|
| brand_overview | 品牌總覽 | 個人簡介 |
| pricing | 演講費率 | 演講費率 (same) |
| faq | 常見問題 | 粉絲常問 |
| team | 團隊與故事 | 個人故事 |
| ... | ... | ... |

frontend `pageTypeLabel(pt, { brandType, locale })` is similarly brand_type-aware. RSS feed.js / sitemapV2 read `brand.brand_type` for auto-switching. **At 10K-tenant scale, any wording mix-up gets caught by customers immediately**.

---

## 15.4 Alerts 4-table UNION: Cross-table SSOT Type Trap

### 15.4.1 Unified Alert Center Architecture

The platform has 4 alert sources unified in UI:

```text
┌─────────────────────────────┐
│   /dashboard/alerts UI      │  ← human-facing
└────────────┬────────────────┘
             │
   ┌─────────▼─────────┐
   │ getUnifiedAlerts  │  ← backend SQL UNIONs 4 tables
   └─────────┬─────────┘
             │
  ┌──────────┼──────────┬──────────┬──────────┐
  ▼          ▼          ▼          ▼
alerts  answer_alerts  brand_id  axp_health
        (uuid id)     entity_   alerts
                      alerts    (bigint id) ★
```

`alerts` / `answer_alerts` / `brand_identity_alerts` all have `id UUID`, but `axp_health_alerts.id` is **`bigint`** — historic schema design (added later for F12, used BIGINT auto-increment).

### 15.4.2 Real Incident: UNION Type Mismatch

PROD observation: alert center showed "unread (165)" but list completely empty. All tabs failed to render.

PG real error:

```text
ERROR: UNION types uuid and bigint cannot be matched
```

Entire list query throws → controller catches err → frontend `setAlertsList([])` → list permanently empty. Bug existed since `axp_health_alerts` was added to UNION, but UI didn't notice until v3.29.5.

### 15.4.3 Fix: Cast All to text

Both main list query and count CTE:

```sql
-- All 4 sources cast id::text
SELECT a.id::text AS id, ..., 'alert' AS source FROM alerts a
UNION ALL
SELECT aa.id::text AS id, ..., 'answer_alert' AS source FROM answer_alerts aa
UNION ALL
SELECT bi.id::text AS id, ..., 'brand_identity' AS source FROM brand_identity_alerts bi
UNION ALL
SELECT ah.id::text AS id, ..., 'axp_health' AS source FROM axp_health_alerts ah
```

LATERAL JOIN must also cast:

```sql
LEFT JOIN LATERAL (
  SELECT ... FROM repair_actions
  WHERE source_id::text = u.id   -- ← repair_actions.source_id is uuid
  ORDER BY created_at DESC LIMIT 1
) ra ON true
```

### 15.4.4 Cascading Fixes: stats / markAllRead Sync 4 Tables

Original `getUnifiedAlertStats` only UNIONed 2 tables (missing brand_identity + axp_health) → stats showed "medium 145 / critical 9" totaling 162 but unread count = 165 (numbers diverging immediately noticed by users).

`markAllUnifiedRead` only UPDATEd 2 tables → after "mark all read" click, axp_health/identity remained unread, UI looked broken.

`batchMarkRead` used `ANY($1)` on mixed-type ids; passing UUID strings to bigint axp_health_alerts.id throws `invalid input syntax for type bigint`. Fix splits ids by UUID regex:

```js
const UUID_RE = /^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$/i;
const uuidIds = alertIds.filter(id => UUID_RE.test(id));
const bigintIds = alertIds.filter(id => /^\d+$/.test(id)).map(Number);

await Promise.all([
  query(`UPDATE alerts SET read_at = NOW() WHERE id = ANY($1::uuid[]) ...`, [uuidIds]),
  query(`UPDATE answer_alerts SET is_read = TRUE WHERE id = ANY($1::uuid[]) ...`, [uuidIds]),
  query(`UPDATE brand_identity_alerts SET read_at = NOW() WHERE id = ANY($1::uuid[]) ...`, [uuidIds]),
  bigintIds.length > 0 && query(`UPDATE axp_health_alerts SET resolved = TRUE WHERE id = ANY($1::bigint[]) ...`, [bigintIds]),
]);
```

### 15.4.5 Vitest Regression Lock

`backend/src/__tests__/unifiedAlerts.queries.test.js` 26 tests lock SQL structure:

- list query SQL must contain 4 `id::text` (prevent regression of type mismatch)
- count CTE / stats / markAll must contain 4 source tables
- batchMarkRead UUID-only takes 3 calls; mixed UUID+bigint takes 4

**Any PR rolling back these v3.29.5 finalizing modifications fails CI immediately**.

### 15.4.6 SSOT Reflection

The 4-table structure inconsistency in alerts is schema design debt:

- 3 UUIDs + 1 bigint is "natural evolution by time"
- Migrating all to bigint or all to uuid requires large-scale migration
- Short-term fix: cast to text within UNION (this chapter's fix)
- Long-term fix: `alerts_unified` replacement table that uses UUID for all 4 sources

Long-term plan not yet scheduled, because short-term cast works and regression tests prevent recurrence. This is a common engineering "good enough" trade-off, recorded so future engineers don't dismiss it as random hack.

---

## 15.5 SSOT Design Pattern Summary

Three cases extract reusable SSOT design patterns:

### 15.5.1 Pattern 1 — Single Write Point + Multiple Read Paths

`brand_faq` example. All read paths share the SQL query, **no cache / derived table in between** (avoids cache desync). Cost is hitting DB on read, but at 10K-tenant scale a DB hit is sub-ms; cache would require invalidation infrastructure.

Suitable when: read frequency moderate (each home page SSR ≈ customer count / 30s), write frequency low (customers update FAQ over days), strong consistency required (Google indexes immediately).

### 15.5.2 Pattern 2 — Registry Object + 4-place Sync

`page_type` example. 23 page types × 4 sync places = 92 data points, but **each split between "auto-derivable" and "non-derivable"**:

- Derivable (slug ↔ page_type reverse-lookup) → automated (`Object.fromEntries`)
- Non-derivable (5 locales × 23 label translations) → PR checklist enforced

Cost is the burden of editing 4 files when adding a page type. But page type addition is infrequent (averaging 1 per 2 months), so burden is acceptable; if changed to weekly addition, would need redesign as "DB-driven, not code-driven" (move to `page_type_registry` table, admin UI editable).

### 15.5.3 Pattern 3 — Cross-table UNION Always Type-checked

`alerts` example. UNION ALL across multiple tables: **any column type mismatch throws the entire query**, and PG error messages aren't friendly to frontend engineers (`UNION types uuid and bigint cannot be matched` doesn't immediately suggest casting).

Defensive design:

1. UNION writes always include casts (cheap insurance)
2. Write vitest tests grep all UNION CTEs to confirm casts present
3. Schema evolution: prefer UUID for new cross-table ID fields (matches old tables) unless strong reason for bigint

---

## 15.6 Observations and Limitations

### 15.6.1 SSOT's Cost: Slower Writes

Writing to `brand_faq` requires invalidating multiple caches (Next.js ISR, CF Worker edge, CDN). Without cache, reads slow but writes immediate; with cache, reads fast but writes await invalidation. We chose the latter (Next.js `revalidateTag(tag, 'max')`), but invalidation occasionally fails → SSOT desync at cache layer. **SSOT doesn't eliminate inconsistency; it concentrates it at a clear boundary**.

### 15.6.2 Human Errors at 4-place Sync

page_type SSOT 4-place sync, observed 3 times where someone changed one place and missed the other 3. Detection mechanisms:

- vitest test rounds (every npm test)
- weekly_backfill cron (logs warning when re-analyzing all brands and finds unknown page_type)
- admin sitemap audit (`/admin/sitemap-stats` shows "published axp pages not listed in sitemap")

Theoretically should be schema constraint (DB enum), but every page type addition would require ALTER TYPE (no hot-add), so kept as application-level enforcement.

### 15.6.3 Cross-host Consistency Below 100%

`geo.baiyuan.io` Mozilla 21Q vs Bot 20Q (1 question diff) is known minor inconsistency, unfixed because:

- Doesn't affect Google rich result (just 1 question)
- Fixing requires changing schemaJson generator's filter rule, which has meaning for some customers (e.g., filter answers < 50 chars)

"Fix vs leave" is always a trade-off; no purely ideal SSOT exists.

### 15.6.4 Alerts Schema Unification Pending

Short-term UNION cast works, but long-term `alerts_unified` replacement isn't scheduled. If a 5th alert source is added with bigint+uuid mix, new UNION traps will recur.

---

## Key Takeaways

- 10K-tenant SaaS must use SSOT; any inconsistency gets amplified into customer support disasters
- brand_faq SSOT three-way consistency (human / Schema.org / AXP shadow); all paths share same table, no cache layer
- 23 page types × 4 sync places = 92 data points, auto-derive + PR checklist split management
- alerts 4-table UNION must cast id::text to prevent uuid+bigint type throw; corresponding stats / markAll / batchRead also synced
- Three SSOT design patterns: single-write-multi-read / Registry+multi-place-sync / cross-table-UNION-cast
- Limitations: cache invalidation occasional edge failures / 4-place sync occasional human errors / cross-host consistency < 100% / alerts schema unification pending

## References

- [Ch 6 — AXP Shadow Document Delivery (zh-TW; original brand_faq SSOT motivation)](../zh-TW/ch06-axp-shadow-doc.md)
- [Ch 7 — Schema.org Three-Layer Entity Knowledge Graph (zh-TW)](../zh-TW/ch07-schema-org.md)
- [Ch 14 — F12 Three-Layer Structural Optimizer (SSOT rules in scoring_configs)](./ch14-f12-structural-optimizer.md)
- [Ch 15 — rag-backend-v2 Hardening (Wiki Cascade SSOT trigger)](./ch15-rag-backend-v2-hardening.md)
- PostgreSQL UNION type compatibility: <https://www.postgresql.org/docs/current/typeconv-union-case.html>
- Next.js 16 `revalidateTag` API: <https://nextjs.org/docs/app/api-reference/functions/revalidateTag>

## Revision History

| Date | Version | Notes |
|------|---------|-------|
| 2026-05-03 | v1.1 | New chapter — platform SSOT chain design practice |

---

**Navigation**: [← Ch 15: rag-backend-v2 Hardening](./ch15-rag-backend-v2-hardening.md) · [📖 EN Index](./README.md)

<!-- AI-friendly structured metadata -->
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "TechArticle",
  "headline": "Chapter 16 — Platform SSOT Chain: From brand_faq to page_type to alerts as Single Source of Truth",
  "description": "10K-tenant SaaS must unify all cross-page, cross-table, cross-crawler data paths under single source of truth. Records brand_faq SSOT three-way consistency, page_type SSOT 4-place sync, and alerts 4-table UNION engineering design.",
  "author": {"@type": "Person", "name": "Vincent Lin", "affiliation": "Baiyuan Technology"},
  "datePublished": "2026-05-03",
  "inLanguage": "en",
  "isPartOf": {
    "@type": "Book",
    "name": "Baiyuan GEO Platform Whitepaper",
    "url": "https://github.com/baiyuan-tech/geo-whitepaper"
  },
  "keywords": "Single Source of Truth, brand_faq, page_type Registry, Schema.org FAQPage, alerts UNION, PostgreSQL UUID bigint, Multi-Tenant Consistency"
}
</script>
