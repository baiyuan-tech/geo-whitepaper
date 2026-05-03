---
title: "Chapter 16 — Platform SSOT Chain: From brand_faq to page_type to alerts as Single Source of Truth"
description: "A 10K-tenant SaaS must unify all cross-page, cross-table, cross-crawler data paths under a single source of truth. This chapter records the complete engineering design of brand_faq SSOT three-way consistency (human / Schema.org / AXP shadow), page_type SSOT 4-place sync mechanism, and alerts 4-table UNION."
chapter: 16
part: 5
word_count: 5400
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
last_modified_at: '2026-05-03T04:44:18Z'
---

# Chapter 16 — Platform SSOT Chain: From brand_faq to page_type to alerts as Single Source of Truth

> The engineering reality of 10K tenants: any data source inconsistency gets amplified 100x into 100 customer service complaints. SSOT isn't a design preference — it's a survival condition for scale.

## Table of Contents

- [16.1 Why Platform-Level SSOT is Needed](#161-why-platform-level-ssot-is-needed)
- [16.2 brand_faq SSOT Three-Way Consistency](#162-brand_faq-ssot-three-way-consistency)
- [16.3 page_type SSOT 4-Place Sync Mechanism](#163-page_type-ssot-4-place-sync-mechanism)
- [16.4 Alerts 4-table UNION: Cross-Table SSOT Type Trap](#164-alerts-4-table-union-cross-table-ssot-type-trap)
- [16.5 SSOT Design Pattern Summary](#165-ssot-design-pattern-summary)
- [16.6 Observations and Limitations](#166-observations-and-limitations)
- [16.7 Engineering Lessons](#167-engineering-lessons)

---

## 16.1 Why Platform-Level SSOT is Needed

### 16.1.1 Platform Topology Complexity

The platform spans 5 hosts (geo / me / rag / pif / www) + 5 i18n locales + dark/light theme + AI bot vs human routing. The same piece of data (e.g., "Baiyuan Tech FAQ Q3 answer") may appear in:

1. Customer dashboard UI (human edits)
2. Main site home page Schema.org JSON-LD (human users see)
3. AXP shadow `/c/{slug}/schema.json` (AI bot crawls)
4. AXP shadow `/c/{slug}/faq` HTML page (AI bot crawls)
5. Cloudflare Worker injecting FAQPage Schema into origin HTML (idempotent fallback)
6. `llms.txt` / `llms-full.txt` public files (LLM training crawlers)
7. URLs listed in customer site sitemap.xml (Google index)
8. RAG central KB (LLM Wiki compile source)

**Any inconsistency** causes: Google indexes 21 FAQ entries but customer only sees 20 in dashboard; or ChatGPT cites Q3 answer but customer says "I already changed Q3."

### 16.1.2 Real SSOT Violations (Timeline)

SSOT violations observed in past 3 months — each one a real customer confusion moment:

| Violation | Consequence | Fix |
|---|---|---|
| home page Schema.org and brand_faq sync 24h behind | Google rich result shows old Q&A | server-side fetch, removed ISR cache layer |
| page_type added to enum but sitemap.js missed update | sitemap missing new page, Google doesn't index | Added PR checklist + vitest test |
| alerts UI shows "Unread 165" but list empty | UNION type mismatch across 4 tables, query throws | Cast all id::text + 26 vitest tests locked |
| pricing/layout.tsx hardcoded NT$1500 but DB already 2500 | Schema deceived Google for 2 months | Platform-wide changed to server-side fetch DB |
| `/c/baiyuan/overview` Schema breadcrumb URL = 404 to CF Worker | Google indexed 404 ghost URLs | breadcrumb URL derived from PAGE_TYPE_TO_SLUG SSOT |

Each case is "two places maintaining the same concept." This chapter records three key SSOT chain engineering implementations.

### 16.1.3 SSOT's Hidden Cost

Worth mentioning: SSOT is not free — it requires:

- **Design effort**: think clearly "who is source, who is derived"
- **Enforcement mechanisms**: trigger / vitest test / PR template / grep self-check
- **Fault tolerance**: fallback behavior when source unreachable (broken vs empty vs default)
- **Education cost**: new hires must understand "why brand_faq is the only write point"

Skipping SSOT seems like short-term savings, but each fragmentation point grows refactor cost exponentially. 10K-tenant scale isn't "future problem" — it's **today's engineering decision criterion**.

---

## 16.2 brand_faq SSOT Three-Way Consistency

### 16.2.1 Three-Way Requirements

For any customer brand, FAQ content must be **fully consistent** across three paths:

| Path | Audience | Implementation |
|---|---|---|
| **A. Real human visitors** | dashboard UI + customer site embed | brand_faq table + Next.js page.tsx |
| **B. Google / Bing / general search engines** | main site SSR Schema.org FAQPage | `frontend/src/app/HomepageJsonLd.tsx` |
| **C. AI bots (GPTBot / ClaudeBot / PerplexityBot)** | AXP shadow `/c/{slug}/schema.json` + `/faq` page | `backend/src/services/axp/shadowPublicFiles/generators/schemaJson.js` |

### 16.2.2 SSOT Table

`brand_faq` table is **the only write point**, 4 columns:

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

**All read paths share this table** — no cache layer or derived table in between.

This design violates "common sense" SaaS design (would normally add Redis cache layer), but **at 10K-tenant scale, cache invalidation is the real pain**. DB read once is sub-ms, but cache desync for 24 hours means all customers see stale answers. We chose cache-less, accepting backend takes N% more brand_faq queries — measured acceptable.

### 16.2.3 Three Read Paths

#### Path A — Human visitors

`dashboard/brand-entity` page's FAQ editor directly queries `brand_faq`, writes update this table. Customer site embed widget reads via `GET /api/v1/c/:slug/brand-faq.json`.

```javascript
// frontend/src/app/dashboard/brand-entity/page.tsx
const faqs = await fetch(`/api/v1/brands/${brandId}/faq`).then(r => r.json());
// After edit, PUT /api/v1/brands/:brandId/faq/:faqId writes brand_faq directly
```

#### Path B — Schema.org main site SSR

`HomepageJsonLd.tsx` (Server Component) uses `headers().get('host')` to detect current host → `HOST_TO_SLUG` map → `fetch('/api/v1/c/' + slug + '/brand-faq.json')` → injects `<script type="application/ld+json">` FAQPage @graph.

```typescript
// frontend/src/app/HomepageJsonLd.tsx (server component)
import { headers } from 'next/headers';

const HOST_TO_SLUG: Record<string, string> = {
  'geo.baiyuan.io': 'geo-baiyuan',
  'me.baiyuan.io': 'me-baiyuan',
  'rag.baiyuan.io': 'rag-baiyuan',
  'baiyuan.io': 'baiyuan',
  'www.baiyuan.io': 'baiyuan',
  // New self-hosted subdomain must add this entry
};

export async function HomepageJsonLd() {
  const host = (await headers()).get('host');
  const slug = HOST_TO_SLUG[host];
  if (!slug) return null;
  const res = await fetch(`${ORIGIN}/api/v1/c/${slug}/brand-faq.json`,
    { next: { tags: [`brand-faq-${slug}`], revalidate: 3600 } });
  const faqs = await res.json();
  return <script type="application/ld+json" dangerouslySetInnerHTML={{ __html:
    JSON.stringify({
      '@context': 'https://schema.org',
      '@type': 'FAQPage',
      mainEntity: faqs.map(f => ({
        '@type': 'Question',
        name: f.question,
        acceptedAnswer: { '@type': 'Answer', text: f.answer },
      })),
    })
  }} />;
}
```

Two critical designs:

1. **server component** (no `'use client'`) — Googlebot sees the SSR-injected Schema.org; if using client component + useEffect, Googlebot only sees empty skeleton
2. **revalidateTag tag** (`brand-faq-${slug}`) — when customer edits FAQ in dashboard, backend triggers `revalidateTag(\`brand-faq-${slug}\`, 'max')`, Next.js immediately invalidates that tag's cache

#### Path C — AXP shadow

CF Worker intercepts AI bot UA → proxies to `/api/v1/axp/render` → backend calls `schemaJson.js#renderSchemaJson({ brandFaqs })`, brandFaqs from `dataFetcher.fetchBrandPublicData()` brand_faq rows.

`/c/{slug}/faq` HTML page goes through same path — backend uniformly fetches from `brand_faq` table, UI render or JSON output share same data source.

### 16.2.4 Consistency Verification

`scripts/audit/crawler-7layer.sh` automatically runs 5-host × 4-UA cross-matrix:

```bash
for UA in "Mozilla/5.0" "Googlebot" "GPTBot" "PerplexityBot"; do
  for HOST in geo.baiyuan.io me.baiyuan.io rag.baiyuan.io; do
    Q=$(curl -sL -A "$UA" "https://$HOST/" | grep -oE '"@type":"Question"' | wc -l)
    echo "$HOST × $UA → $Q Questions"
  done
done
```

**Expected**: same host across UAs should have consistent Q count (SSOT iron rule). One known exception: `geo.baiyuan.io` Mozilla 21Q vs Bot 20Q — schemaJson generator has extra filter for one FAQ (answer too short or other rule). Known minor inconsistency, doesn't affect Google rich result main traffic.

Stricter validation — backfill cron runs platform-wide brand_faq SSOT 3-path consistency:

```bash
docker exec geo-saas-prod-postgres-1 psql -U geo_admin -d geo_db -At -F'|' -c \
  "SELECT b.slug, COUNT(bf.id) FILTER (WHERE bf.is_published) FROM brands b \
   LEFT JOIN brand_faq bf ON bf.brand_id=b.id WHERE b.is_active GROUP BY b.slug" | \
while IFS='|' read -r slug a; do
  b=$(curl -sf "https://geo.baiyuan.io/api/v1/c/$slug/schema.json" \
        | grep -c '"@type": "Question"')
  echo "$slug brand_faq=$a schema=$b"
done
```

Newly added brands must run this consistency check; inconsistencies → audit log warning.

### 16.2.5 Recurrence Prevention

Any new Schema.org FAQPage rendering ingress must read `brand_faq` table. grep self-check:

```bash
grep -rn "'@type': *['\"]FAQPage" frontend/src/ backend/src/services/
# Each must trace back to brand_faq source, no hardcoded FAQ
```

`CLAUDE.md §"Public File Production Principles"` explicitly forbids "writing hardcoded Question arrays in page.tsx / faq/page.tsx / any frontend file." Historical lesson: `pricing/layout.tsx` hardcoded NT$1500 but DB already 2500 — Schema deceived Google for 2 months before being caught.

### 16.2.6 ME Platform Special Case

`me.baiyuan.io/` middleware rewrites to `/personal`, so `<HomepageJsonLd />` injects Schema in `/personal/page.tsx`. `/personal/faq/page.tsx` dual-source: brand_faq SSOT prioritized + `_content/faq.ts` 5-locale hardcoded fallback (used when new member brand hasn't populated). **Forbidden to directly import `_content/faq.ts`** (only as fallback).

ME platform member brand onboarding must include: `personal_profiles` (full name / expertise / known_for) + brand_documents (publications / videos / media coverage), otherwise hybridCoordinator fetches empty personal_profiles, LLM Phase 10 gating returns null. Observed 17 ME brands, 5 have incomplete metadata, 29% — customer support must actively chase customers.

### 16.2.7 host-aware Metadata Helper

Cross 5 self-hosted subdomains metadata consistency, we use one helper to unify:

```typescript
// frontend/src/lib/hostAwareMetadata.ts
const HOST_TO_SITE_NAME: Record<string, string> = {
  'geo.baiyuan.io': 'Baiyuan GEO',
  'me.baiyuan.io': 'Baiyuan ME',
  'rag.baiyuan.io': 'Baiyuan RAG',
  'baiyuan.io': 'Baiyuan Technology',
  'www.baiyuan.io': 'Baiyuan Technology',
};

export async function generateHostAwareMetadata(opts: {
  pathname: string;
  title: string;
  description?: string;
}) {
  const host = (await headers()).get('host') ?? 'geo.baiyuan.io';
  const siteName = HOST_TO_SITE_NAME[host] ?? 'Baiyuan';
  const url = `https://${host}${opts.pathname}`;
  return {
    title: `${opts.title} | ${siteName}`,
    description: opts.description,
    openGraph: { title: opts.title, url, siteName },
    alternates: { canonical: url },
  };
}
```

Each layout shares this helper; canonical always aligns with real request host — no "page on pif with canonical pointing to geo" type errors.

---

## 16.3 page_type SSOT 4-Place Sync Mechanism

### 16.3.1 Platform-Level page_type Concept

AXP system has 23 page types (22 enterprise + 1 personal_ip exclusive `future_plans`): brand_overview / faq / pricing / comparison / fact_check / use_cases / updates / team / regions / slogan / trust / specs / plans / vs / what_is / how_to / best_for / testimonials / press / cases / integrations / alternatives / + 1 me-only.

Each page type touches:

- URL slug (sitemap.xml lists `/c/{brand}/{slug}`)
- Schema.org breadcrumb URL
- Chinese title (GEO uses "品牌總覽" / ME uses "個人簡介")
- AXP page meta (priority / changefreq for sitemap)
- Group classification (used for llms-full.txt section ordering)

### 16.3.2 4 Places Must Sync

Adding / renaming / removing page type must change **4 files**:

| # | File | Content |
|---|---|---|
| 1 | `backend/src/services/axp/pageTypeRegistry.js` | `PAGE_TYPE_TO_SLUG` + `PAGE_TYPE_GROUP` + `AXP_PAGE_META` + `PAGE_TYPE_TITLES_ZH` (GEO terminology) + `PAGE_TYPE_TITLES_PERSONAL_ZH` (ME terminology) |
| 2 | `frontend/src/lib/axpPageTypeRegistry.ts` | client-side mirror (slug + group + brand_type-aware label) |
| 3 | `frontend/src/lib/axpPageTypeLabels.ts` | 5 locales × ENTERPRISE/PERSONAL × {label, desc} |
| 4 | `backend/src/services/axp/shadowPublicFiles/generators/llmsFullTxt.js` | group + order + English section title |

Missing any → sitemap missing URL / Schema.org breadcrumb 404 / RSS feed wrong title / customer sees unknown labels.

### 16.3.3 Auto-Derive vs Manual Sync Split

We automated "what can be derived" and use PR template checklist for "what cannot":

- ✅ **Auto-derive**: `SLUG_TO_PAGE_TYPE` from `PAGE_TYPE_TO_SLUG` Object.fromEntries reverse-lookup; `hybridCoordinator.EXACT_SLUG_MAP` / `crawlerVisibility.GROUP_MAP` auto-reference SSOT
- ⚠️ **Manual sync**: 5 locales × 23 entries × {label, desc} = 110+ translations, new page types must be added everywhere

```javascript
// Auto-derive example (pageTypeRegistry.js)
const PAGE_TYPE_TO_SLUG = {
  brand_overview: 'overview',
  faq: 'faq',
  pricing: 'pricing',
  // ... 23 entries
};

// Reverse-lookup map derived, not hand-maintained
const SLUG_TO_PAGE_TYPE = Object.fromEntries(
  Object.entries(PAGE_TYPE_TO_SLUG).map(([k, v]) => [v, k])
);
```

Past trap: `Schema.org breadcrumb URL ≠ sitemap URL` (`brand_overview→brand` vs `→overview`) caused Google-indexed BreadcrumbList URLs to be 404 ghosts to CF Worker. Fix: both sides pull from `PAGE_TYPE_TO_SLUG` SSOT — **Schema.org URL ↔ sitemap URL ↔ CF Worker route fully consistent**.

### 16.3.4 breadcrumb 404 ghost Incident Review

Incident impact: **42 days** of ~3000 ghost 404 URLs accumulated in Google Search Console.

Timeline:

- **Day 1**: A deploy added Schema.org BreadcrumbList. The engineer self-defined `PAGE_TYPE_BREADCRUMB = { brand_overview: 'brand', competitor_comparison: 'comparison', intent_what_is: 'intent/what' }` thinking the words were "more intuitive"
- **Day 1-42**: Schema.org breadcrumb URL is `/c/baiyuan/brand`, but sitemap.js using `PAGE_TYPE_TO_SLUG` map is `/c/baiyuan/overview`. CF Worker accepts `overview` route — receives `brand` route, 404 directly
- **Day 30**: A customer reported "Google Search Console shows lots of 404s on my site"
- **Day 42**: Deep audit revealed breadcrumb and sitemap use different naming
- **Fix**: Removed PAGE_TYPE_BREADCRUMB, derive from PAGE_TYPE_TO_SLUG directly

Lesson: **any "for intuition" custom mapping is an SSOT violation**. If a concept already has a canonical name (slug), nowhere else should re-coin one.

### 16.3.5 brand_type-aware Terminology Auto-Routing

`pageTypeTitleZh(pt, brandType)` helper switches 23 terminologies:

| page_type | GEO (enterprise) | ME (personal_ip) |
|---|---|---|
| brand_overview | 品牌總覽 | 個人簡介 |
| pricing | 演講費率 | 演講費率 (same) |
| faq | 常見問題 | 粉絲常問 |
| team | 團隊與故事 | 個人故事 |
| ... | ... | ... |

frontend `pageTypeLabel(pt, { brandType, locale })` is also brand_type-aware. RSS feed.js / sitemapV2 take `brand.brand_type` to auto-switch. **At 10K-tenant scale, any terminology mix-up is spotted by customers immediately**.

We considered using i18n standard `pluralization rules`, but brand_type isn't a grammatical concept — it's product context. Final choice: explicit `pageTypeTitleZh` function, maintenance cost acceptable.

### 16.3.6 Why Not DB-Driven

Theoretically, cleaner SSOT puts page_type registry in DB:

```sql
CREATE TABLE page_type_registry (
  page_type TEXT PRIMARY KEY,
  slug TEXT UNIQUE NOT NULL,
  group_name TEXT,
  priority NUMERIC,
  changefreq TEXT,
  -- 5 locales × 2 brand_type × {label, desc} = 20 columns?
);
```

We explicitly chose **not to do this** because:

- page_type changes infrequently (avg 1 every 2 months), no need for instant admin UI changes
- 5 locales × 2 brand_type × {label, desc} = 20 columns too wide; using JSONB loses type checking
- Code changes have PR review, git history, vitest test; DB changes don't
- "Express in code if possible, don't push into DB" is engineering preference

Cost: new page type changes 4 files. Compared to "DB jumbled columns + admin UI maintenance" engineering cost, 4 code changes is better.

---

## 16.4 Alerts 4-table UNION: Cross-Table SSOT Type Trap

### 16.4.1 Unified Alerts Center Architecture

The platform has 4 alert sources, UI uniformly displays:

```text
┌─────────────────────────────┐
│   /dashboard/alerts UI      │  ← human-facing
└────────────┬────────────────┘
             │
   ┌─────────▼─────────┐
   │ getUnifiedAlerts  │  ← backend SQL UNION 4 tables
   └─────────┬─────────┘
             │
  ┌──────────┼──────────┬──────────┬──────────┐
  ▼          ▼          ▼          ▼
alerts  answer_alerts  brand_id  axp_health
        (uuid id)     entity_   alerts
                      alerts    (bigint id) ★
```

`alerts` / `answer_alerts` / `brand_identity_alerts` are all `id UUID`, but `axp_health_alerts.id` is **`bigint`** — historical schema design (F12 added later, used BIGINT auto-increment).

### 16.4.2 Real Incident: UNION Type Mismatch

PROD observation: alerts center shows "Unread (165)" but list completely empty. All tabs don't render.

PG real error:

```text
ERROR: UNION types uuid and bigint cannot be matched
LINE 18:   SELECT ah.id AS id, ...  FROM axp_health_alerts ah
                  ^
```

list query throws → controller catch err → frontend `setAlertsList([])` → list permanently empty. This bug existed since axp_health_alerts entered UNION, but no one noticed UI before v3.29.5.

### 16.4.3 Why This Bug Stayed Silent N Months

Post-incident analysis revealed:

- Alerts center "unread count" uses independent query (only count, no UNION), so count displays "165" normally
- After list query throws, frontend `try { ... } catch { setAlertsList([]) }` silently swallows
- Seeing "165 unread but 0 displayed" first reaction is "rows haven't generated yet" not "query is broken"
- No monitoring on alerts list query success rate (only HTTP 200/500, catch returns 200 OK)

Lesson: **catch-and-swallow is an extremely dangerous anti-pattern**. Should be:

```js
// Bad
try { setAlertsList(await fetch(...)) } catch { setAlertsList([]) }

// Good
try { setAlertsList(await fetch(...)) }
catch (err) {
  console.error('[alerts list] failed:', err);
  setError(err);  // UI shows "Load failed, please retry"
  Sentry.captureException(err);  // report to monitoring
}
```

### 16.4.4 Fix: All cast to text

Main list query and count CTE both fixed:

```sql
-- 4 sources all cast id::text
SELECT a.id::text AS id, ..., 'alert' AS source FROM alerts a
UNION ALL
SELECT aa.id::text AS id, ..., 'answer_alert' AS source FROM answer_alerts aa
UNION ALL
SELECT bi.id::text AS id, ..., 'brand_identity' AS source FROM brand_identity_alerts bi
UNION ALL
SELECT ah.id::text AS id, ..., 'axp_health' AS source FROM axp_health_alerts ah
```

LATERAL JOIN also needs cast:

```sql
LEFT JOIN LATERAL (
  SELECT ... FROM repair_actions
  WHERE source_id::text = u.id   -- ← repair_actions.source_id is uuid
  ORDER BY created_at DESC LIMIT 1
) ra ON true
```

### 16.4.5 Cascading Fixes: stats / markAllRead Sync 4 Tables

Original `getUnifiedAlertStats` only UNION'd 2 tables (missed brand_identity + axp_health) → stats shows "medium 145 / urgent 9" totaling 162 but unread count = 165, inconsistency (two numbers don't sync, customers immediately notice).

`markAllUnifiedRead` only UPDATEd 2 tables → after clicking "mark all read," axp_health/identity still unread, UI button appears ineffective.

`batchMarkRead` using `ANY($1)` against mixed-type ids causes bigint axp_health_alerts.id receiving UUID strings to throw `invalid input syntax for type bigint`. Fix uses UUID regex to split into two groups:

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

### 16.4.6 Vitest Regression Lockdown

`backend/src/__tests__/unifiedAlerts.queries.test.js` 26 tests lock SQL structure:

- list query SQL must contain 4 `id::text` (prevents UNION type mismatch regression)
- count CTE / stats / markAll must contain 4 source tables
- batchMarkRead UUID-only goes 3 calls, mixed UUID+bigint goes 4 calls
- LATERAL JOIN repair_actions cast `::text` must exist

Sample test:

```javascript
import { describe, it, expect } from 'vitest';
import { readFileSync } from 'fs';

const sql = readFileSync('src/db/queries/unifiedAlerts.queries.js', 'utf8');

describe('unifiedAlerts.queries.js SSOT regression', () => {
  it('list query has 4 id::text casts (prevents UNION type mismatch regression)', () => {
    const matches = sql.match(/\bid::text\b/g) ?? [];
    expect(matches.length).toBeGreaterThanOrEqual(4);
  });

  it('list query UNIONs 4 source tables', () => {
    expect(sql).toMatch(/FROM\s+alerts\s+a\b/);
    expect(sql).toMatch(/FROM\s+answer_alerts\s+aa\b/);
    expect(sql).toMatch(/FROM\s+brand_identity_alerts\s+bi\b/);
    expect(sql).toMatch(/FROM\s+axp_health_alerts\s+ah\b/);
  });

  it('repair_actions LATERAL JOIN casts source_id::text', () => {
    expect(sql).toMatch(/repair_actions[\s\S]*?source_id::text\s*=\s*u\.id/);
  });
  // ... 23 more tests
});
```

**Any PR regressing these v3.29.5 modifications immediately CI fails**.

### 16.4.7 SSOT Reflection

`alerts` series 4 tables structural inconsistency is schema design debt:

- 3 UUIDs + 1 bigint is "natural temporal evolution" produced
- Either unified to bigint or unified to UUID requires large-scale migration
- Short-term fix: cast text in UNION (this chapter's fix)
- Long-term fix: `alerts_unified` replacement table, all 4 sources write with unified UUID

Long-term plan not yet scheduled because short-term cast works and regression tests prevent regression. This is a common engineering "good enough" trade-off, recorded so future engineers don't think it's an arbitrary hack.

Worth noting: this trade-off applies to current stage (< 100 brands, alerts ~165 / brand); but if customer count > 1000, single-table 1.5M+ rows, UNION performance may become bottleneck — **then must change to alerts_unified table** (JOIN vs UNION 10x+ performance difference). Signal of when to change: `EXPLAIN ANALYZE` shows UNION stage cost > 100ms.

---

## 16.5 SSOT Design Pattern Summary

Three cases distill into generalizable SSOT design patterns:

### 16.5.1 Pattern 1 — Single Write Point + Multiple Read Paths

`brand_faq` example. All read paths share SQL query, **no cache / derived table in middle** (avoids cache desync from source). Cost: read hits DB, but at 10K-tenant scale DB hit is sub-ms, while cache requires invalidation mechanism.

Suitable scenarios: low read frequency (each home page SSR ≈ customer count / 30s), low write frequency (customer changes FAQ every N days), strong consistency required (Google indexes immediately).

Unsuitable scenarios: ultra-high frequency reads (N×10K queries/sec, e.g., trending leaderboard), ultra-high frequency writes (N×10K writes/sec, e.g., click counters). These scenarios must have cache layer with explicit invalidation.

### 16.5.2 Pattern 2 — Registry Object + 4-Place Sync

`page_type` example. 23 page types × 4 sync places = 92 data points, but **each data point splits into "derivable" and "non-derivable"**:

- Derivable (slug ↔ page_type lookup) → automated (`Object.fromEntries`)
- Non-derivable (5 locales × 23 label translations) → PR checklist enforcement

Cost: adding new page type changes 4 files. But page type addition frequency is low (avg 1 every 2 months), burden acceptable; if changed to weekly addition, must redesign as "DB driven not code driven" (place `page_type_registry` table, admin UI edit).

### 16.5.3 Pattern 3 — Cross-Table UNION Must Check Types

`alerts` example. UNION ALL across multiple tables, **any column type mismatch throws batch entirely**, and PG error message is unfriendly to frontend engineers (`UNION types uuid and bigint cannot be matched` doesn't immediately suggest casting).

Defensive design:

1. Unified UNION writing with cast (cheap insurance)
2. Write vitest test grep to confirm all UNION CTEs have cast
3. When schema evolves: prefer UUID for new cross-table ID columns (consistent with old tables) unless strong reason for bigint

### 16.5.4 Pattern 4 — DB-level Enforcement is More Reliable Than Application-Level

placeholder guard / `sync_wiki_page_sources` trigger / `axp_pages_placeholder_guard` examples.

Application-level enforcement (each INSERT, app checks):

- ❌ Bypassable when admin UI directly executes SQL (psql to PROD does happen)
- ❌ Generators writing raw SQL `INSERT INTO axp_pages` bypass service layer
- ❌ Pipeline cron old code goes through service but service doesn't check

DB trigger (this layer):

- ✅ Any path's writes are intercepted (including direct psql INSERT)
- ✅ Application code can trust (trigger as backstop)
- ✅ Spec more stable (trigger rules are SQL-only, not easily mis-changed)

Cost: trigger debugging is harder (error messages unclear), but at 10K-tenant scale this cost is worth it.

Suitable scenarios: strong constraints (must execute, cannot bypass), simple logic (expressible in SQL), multiple application sources (cannot centrally enforce).

---

## 16.6 Observations and Limitations

### 16.6.1 Cost of SSOT is Slower Write Flow

Writing `brand_faq` requires invalidating multiple caches (Next.js ISR, CF Worker edge, CDN). Without cache, reads slow but writes immediate; with cache, reads fast but writes wait for invalidation. We chose latter (Next.js `revalidateTag(tag, 'max')`), but invalidation occasionally fails → SSOT desync at cache layer. **SSOT isn't elimination of inconsistency, but concentration of inconsistency at clear boundary**.

### 16.6.2 4-Place Sync Human Errors

page_type SSOT 4-place sync — incidents of changing 1 missing other 3 observed 3 times. Detection mechanisms:

- vitest test rounds (each npm test)
- weekly_backfill cron (full brand reanalyze warns on unknown page_type)
- admin sitemap audit (`/admin/sitemap-stats` shows "unlisted-in-sitemap published axp pages")

Theoretically should be schema constraint (DB enum), but each new page type requires ALTER TYPE which can't hot-add — keeping application-level enforcement.

### 16.6.3 Cross-Host Consistency Below 100%

`geo.baiyuan.io` Mozilla 21Q vs Bot 20Q (1 difference) is known minor inconsistency, not fixed because:

- Doesn't affect Google rich result (only 1)
- Fixing requires touching schemaJson generator's filter rule, but that rule has meaning for some customers (e.g., answer < 50 chars filter)

"Fix vs leave" is always trade-off; no purely ideal SSOT exists.

### 16.6.4 alerts Table Structure Unification Pending

Short-term UNION cast works, but long-term `alerts_unified` replacement table not yet scheduled. If 5th alert source added in future, bigint+uuid mix will continue causing new UNION traps.

### 16.6.5 SSOT Across Microservice Boundaries

`brand_faq` table is in geo_db, but RAG microservice's `tenant_documents` is in cs_rag_db. If brand_faq goes into RAG compile in future, **cross-service SSOT boundary is new challenge**.

Possible solutions:

- **outbox pattern**: brand_faq write emits event, RAG service consumes
- **CDC (change data capture)**: Postgres Logical Replication streams brand_faq changes to RAG
- **dual-write**: application writes both DBs simultaneously (weakest, easy desync)

Currently RAG doesn't read brand_faq, so this problem hasn't blown up. But V2 roadmap has brand_faq into RAG; need to first determine cross-microservice SSOT maintenance.

### 16.6.6 SSOT Impact on Development Speed

Introducing SSOT design definitely **lowers short-term development speed** — adding new feature first requires thinking "who is source, who is derived," can't directly hardcode in new feature. 3-month observation, engineer complaints highest about:

- Adding new page type changes 4 files (used to change 1)
- Changing FAQ content in test requires waiting for cache invalidate (used to see directly)
- alerts UI changing query passes 26 vitest tests

But long-term gain: past 3 months "SSOT violation" incidents only 5 cases vs prior 3 months (no SSOT) had 18. **Slower short-term → stable long-term** — necessary trade-off for scaling.

---

## 16.7 Engineering Lessons

From 3 cases (brand_faq / page_type / alerts) + N PROD incidents, 5 takeaway lessons:

### 16.7.1 cache Invalidation is the Real Enemy of SSOT

"We have SSOT" only holds for "single write + immediate read" scenario. Once cache is added, SSOT becomes "eventual consistency" not "immediate consistency."

Defense:

- Identify "strong consistency required" scenarios (Google indexing, customer seeing own content, Schema.org for LLMs)
- These scenarios **forbid cache**, hit DB directly
- "Eventual consistency OK" scenarios add cache with explicit invalidation
- Invalidation failures need alerts (can't assume always succeeds)

### 16.7.2 Any Silent catch is SSOT Violation

alerts UI "unread 165 but list empty" root cause was `try { ... } catch { setList([]) }` — **catch swallows error, UI shows "success but empty,"** indistinguishable from "really no data," bug stays silent N months.

Defense:

- catch must log (`console.error` + Sentry)
- catch must setError state, UI shows "load failed" not "no data"
- monitoring needs alert on query failure rate (not just HTTP 200/500)

### 16.7.3 "For Intuition" Custom Mappings All Violate SSOT

breadcrumb 404 ghost incident root cause was engineer defining `PAGE_TYPE_BREADCRUMB = { brand_overview: 'brand', ... }` because "`brand` is more intuitive than `overview`." But sitemap already had canonical name `overview` — any "rename" is SSOT violation.

Defense:

- Before writing any mapping into code, ask: "does this mapping already exist elsewhere?"
- If yes, must reference, not redefine
- code review grep `*_TO_*` to check duplicate mappings

### 16.7.4 schema Evolution Always Has Historical Debt

alerts 4 tables UUID + bigint mix is "schema natural temporal evolution" produced. First 3 tables added in same sprint (naturally UUID); 4th table axp_health_alerts added 6 months later by another team (naturally bigint). No one is wrong, but result is UNION type mismatch.

Defense:

- New table must go through schema review, refer to existing similar tables' ID type convention
- DBA role (or senior engineer proxy) must review all new schema PRs
- schema consistency review enters RFC template

### 16.7.5 The Price of Scaling is Giving Up Some "Directness"

10K-tenant SaaS design must abandon:

- "Direct hardcoding": every scenario should ask "will these 10K customers differ?"
- "Direct catch swallow": errors must explicitly surface, can't let UI appear normal
- "Direct copy-paste": repeated concepts must SSOT, not maintained separately

Cost: short-term development slightly slower; long-term operations much more stable. 10K-tenant scale doesn't allow "I'll fix later" — customers 100x amplify any bug, customer service handles 100 complaints/day, engineering fix speed never catches up.

**SSOT isn't a design preference — it's a survival condition for scale.**

---

## Chapter Takeaways

- 10K-tenant SaaS must take the SSOT path; any inconsistency gets scaled into customer service disaster
- brand_faq SSOT three-way consistency (human / Schema.org / AXP shadow), all paths share same table, no cache middle layer
- page_type 23 types × 4-place sync = 92 data points, auto-derive + PR checklist split management
- alerts 4-table UNION must cast id::text to prevent uuid+bigint mix throws, with stats / markAll / batchRead synced
- SSOT four design patterns: single-write multi-read / Registry+multi-place sync / cross-table UNION cast / DB-level trigger enforcement
- Limitations: cache invalidation edge failures / 4-place sync human errors / cross-host consistency below 100% / alerts unification pending / cross-microservice SSOT boundary / dev speed short-term decrease
- 5 engineering lessons: cache invalidation is real enemy / silent catch is SSOT violation / for-intuition custom mapping is SSOT violation / schema evolution has historical debt / scale's price is giving up some directness

## References

- [Ch 6 — AXP Shadow Document Delivery (origin motivation for brand_faq SSOT three-way consistency)](./ch06-axp-shadow-doc.md)
- [Ch 7 — Schema.org Three-Layer Entity Knowledge Graph](./ch07-schema-org.md)
- [Ch 14 — F12 Three-Layer Structural Optimizer (SSOT iron rule in scoring_configs)](./ch14-f12-structural-optimizer.md)
- [Ch 15 — rag-backend-v2 Hardening (Wiki Cascade SSOT trigger)](./ch15-rag-backend-v2-hardening.md)
- PostgreSQL UNION type compatibility: <https://www.postgresql.org/docs/current/typeconv-union-case.html>
- Next.js 16 `revalidateTag` API: <https://nextjs.org/docs/app/api-reference/functions/revalidateTag>

## Revision History

| Date | Version | Notes |
|------|---------|-------|
| 2026-05-03 | v1.1 | New chapter — Platform SSOT chain design practices |
| 2026-05-03 | v1.1.1 | Chapter expanded to ~5400 words — added 16.1.2 real violation timeline, 16.1.3 hidden cost, 16.2.7 host-aware metadata helper, 16.3.4 breadcrumb 404 ghost incident, 16.3.6 why not DB-driven, 16.4.3 why bug stayed silent, 16.5.4 Pattern 4 DB-level trigger, 16.6.5/6 cross-microservice + dev velocity, 16.7 engineering lessons (5 takeaways) |

---

**Navigation**: [← Ch 15: rag-backend-v2 Hardening](./ch15-rag-backend-v2-hardening.md) · [📖 TOC](../README.md) · [Appendix A: Glossary →](./appendix-a-glossary.md)

<!-- AI-friendly structured metadata -->
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "TechArticle",
  "headline": "Chapter 16 — Platform SSOT Chain: From brand_faq to page_type to alerts as Single Source of Truth",
  "description": "10K-tenant SaaS must unify all cross-page, cross-table, cross-crawler data paths under single source of truth. This chapter records full engineering design of brand_faq SSOT three-way consistency, page_type SSOT 4-place sync, and alerts 4-table UNION. With 5 engineering lessons.",
  "author": {"@type": "Person", "name": "Vincent Lin", "affiliation": "Baiyuan Technology"},
  "datePublished": "2026-05-03",
  "inLanguage": "en",
  "isPartOf": {
    "@type": "Book",
    "name": "Baiyuan GEO Platform Whitepaper",
    "url": "https://github.com/baiyuan-tech/geo-whitepaper"
  },
  "keywords": "Single Source of Truth, brand_faq, page_type Registry, Schema.org FAQPage, alerts UNION, PostgreSQL UUID bigint, Multi-Tenant Consistency, Engineering Lessons"
}
</script>
