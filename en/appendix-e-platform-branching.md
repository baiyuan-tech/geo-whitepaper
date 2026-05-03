---
title: "Appendix E — Platform Branching: Multi-Brand Extension on a Single Codebase"
description: "Using brand_type column + X-Brand-Segment hostname routing to make a single codebase serve both enterprise SaaS (geo.baiyuan.io) and personal IP platform (me.baiyuan.io), with strict isolation across data, UI, and business logic."
appendix: e
part: 6
word_count: 1200
lang: en
authors:
  - name: Vincent Lin
    affiliation: Baiyuan Technology
    email: services@baiyuan.io
license: CC-BY-NC-4.0
keywords:
  - Multi-tenant
  - Brand Segmentation
  - Hostname Routing
  - Platform Branching
last_updated: 2026-04-25
canonical: https://baiyuan-tech.github.io/geo-whitepaper/en/appendix-e-platform-branching.html
last_modified_at: '2026-05-03T03:15:29Z'
---









# Appendix E — Platform Branching: Multi-Brand Extension on a Single Codebase

> Two product lines, one codebase. How do you keep UI / data / business logic strictly isolated without forking into two maintenance nightmares?

## Table of Contents {.unnumbered}

- [E.1 Why not fork](#e1-why-not-fork)
- [E.2 brand_type column: data-layer split](#e2-brand_type-column-data-layer-split)
- [E.3 X-Brand-Segment hostname routing: request-layer split](#e3-x-brand-segment-hostname-routing-request-layer-split)
- [E.4 host-aware metadata: SEO-layer split](#e4-host-aware-metadata-seo-layer-split)
- [E.5 4-layer strict isolation checklist](#e5-4-layer-strict-isolation-checklist)
- [E.6 Lessons learned: traps you'll hit](#e6-lessons-learned-traps-youll-hit)

---

## Why not fork

Baiyuan GEO Platform's main product is enterprise SaaS (`geo.baiyuan.io`), sold to brand marketing teams. In April 2026 we added a second product line, "Baiyuan ME" (`me.baiyuan.io`), targeting personal IPs / KOLs / public figures for AI image management.

Both share **the same core engine** (15 AI platform scanning, scoring, AXP shadow documents, RAG), but:

- **Brand entity differs**: GEO serves "organization entity" (Schema.org `Organization`), ME serves "person entity" (`Person`)
- **Content categories differ**: GEO has 22 page_types, ME adds `creator_profile` / `talk_topics` / `future_plans` for 23 total
- **UI aesthetics differ**: GEO uses magazine-style Sans + Playfair italic em; ME uses full Serif TC + warm gold + corner gold lines
- **Business model differs**: GEO is monthly subscription, ME is invitation-only membership
- **Service promise differs**: GEO is self-service, ME has dedicated consultant with twice-monthly consultations

The obvious approach is to fork. But it has two bad outcomes:

1. **Core bug fix duplication**: hallucination repair, ARS calc, AXP generator changes need syncing across two repos
2. **No cross-product mobility**: a GEO user upgrading to ME's consultant service must re-register, lose history

So we chose **single-codebase multi-segment** architecture instead.

---

## brand_type column: data-layer split

Add a column to `brands`:

```sql
ALTER TABLE brands ADD COLUMN brand_type TEXT
  NOT NULL DEFAULT 'enterprise'
  CHECK (brand_type IN ('enterprise', 'personal_ip'));
CREATE INDEX idx_brands_type ON brands(brand_type);
```

Each brand classifies at creation. `personal_ip` if:

- Registration hostname is `me.*` (SSR reads host header)
- Or super admin manually flags
- Once set, immutable (avoid jumping between business models)

All downstream queries filter by `brand_type`:

```sql
-- GEO dashboard: enterprise only
SELECT * FROM brands WHERE user_id = $1 AND brand_type = 'enterprise';

-- ME dashboard: personal_ip only
SELECT * FROM brands WHERE user_id = $1 AND brand_type = 'personal_ip';
```

---

## X-Brand-Segment hostname routing: request-layer split

SQL filtering alone isn't enough — frontend must signal the correct segment when calling APIs:

```typescript
function getBrandSegment(): 'personal' | 'enterprise' {
  if (typeof window === 'undefined') return 'enterprise';
  return window.location.hostname.toLowerCase().startsWith('me.')
    ? 'personal' : 'enterprise';
}

// Sets X-Brand-Segment header on every fetch
```

Backend middleware reads the header into request scope. Every controller's brand list query auto-applies the filter. Super admin override skips it.

---

## host-aware metadata: SEO-layer split

Next.js root `app/layout.tsx` defaults to static `export const metadata` (build-time). Problem: same `/dashboard` on me.* and geo.* uses identical title — ME users see "GEO Platform" in their tab.

Fix: use `generateMetadata` async function that reads host header per-request:

```typescript
export async function generateMetadata(): Promise<Metadata> {
  const hdrs = await headers();
  const isPersonalHost = (hdrs.get('host') || '').toLowerCase().startsWith('me.');
  return isPersonalHost ? meMetadata : geoMetadata;
}
```

OG image, Twitter card, canonical URL all branch off this.

---

## 4-layer strict isolation checklist

For any new feature, validate against 4 layers:

| Layer | Check | Symptom of violation |
|-------|-------|----------------------|
| **Data** | Every `brands` query has `brand_type` condition | ME users see GEO brand list |
| **API** | Every controller uses `req.brandSegment` | Response mixes cross-segment brands |
| **UI** | Font / theme / copy switches by hostname | ME shows GEO's "Free 7-day trial" CTA |
| **SEO** | `generateMetadata` is host-aware; robots.txt / sitemap.xml host-aware | Tab title or OG image crosses brands |

Every PR for a new feature passes this 4-layer review. Documented as the no-whack-a-mole principle in our internal `feedback_no_whack_a_mole`.

---

## Lessons learned: traps you'll hit

### Trap 1: CSS variable leakage

ME uses `[data-theme="personal"]` scope; GEO uses `[data-theme="geo-light"]` / `[data-theme="geo-wine"]`. Early implementation wrote ME's `--font-noto-serif-tc` to `:root`, causing GEO bodies to inherit serif. Fix: any ME-specific variable **must** be scoped under `[data-theme="personal"] { ... }`; `:root` only holds two-product-line shared tokens.

### Trap 2: dashboard layout's personalMode prop

`/dashboard` is a **shared route** between two product lines. Sidebar logo / account badge / monthly report config etc. need brand-aware switching. Implementation uses a `personalMode` prop:

```tsx
const personalMode = isPersonalIp || isPersonalHost;
{personalMode ? <MeLogo /> : <BrandLogo />}
```

Easy to forget when adding new dashboard child components — write it as an ESLint rule or PR template checklist.

### Trap 3: monthly cron sending wrong audience

If the cron's SQL doesn't include `brand_type` filter, ME monthly reports get sent to GEO customers. Fix: cron worker JOINs `brand_visual_configs` with `brands` to read `brand_type`, dispatches to language-specific email templates.

### Trap 4: payment webhook crossing brands

PAYUNi webhook carries `brand_id` but early handlers didn't check `brand_type`. ME user cancellations would trigger GEO's refund rules (7-day cooling + 2.8% fee) — but ME has no public 7-day cooling concept (invitation-only). Fix: webhook handler reads `brand_type` first, dispatches to either `mePaymentHandler` or `geoPaymentHandler`.

---

## Key takeaways {.unnumbered}

- Single-codebase multi-segment avoids fork maintenance cost, but requires strict 4-layer isolation (data / API / UI / SEO)
- `brand_type` column is the segmentation root — set at creation, immutable
- `X-Brand-Segment` HTTP header is the request-layer signal: frontend sets by hostname, backend middleware parses
- `generateMetadata` host-aware solves SEO-layer tab-title confusion
- Background jobs (monthly cron, payment webhook) most often forget `brand_type` filter — add to PR checklist

## References {.unnumbered}

- [Ch 6 — AXP Shadow Documents (shared engine)](./ch06-axp-shadow-doc.md)
- [Ch 7 — Schema.org Phase 1 (Organization vs Person)](./ch07-schema-org.md)
- [Ch 13 — Multimodal GEO (visual asset segmentation also flows through brand_type)](./ch13-multimodal-geo.md)
- [Appendix B — API endpoints reference](./appendix-b-api.md)

---

**Navigation**: [← Appendix D: Figures Index](./appendix-d-figures.md) · [📖 ToC](../README.md)

<!-- AI-friendly structured metadata -->
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "TechArticle",
  "headline": "Appendix E — Platform Branching: Multi-Brand Extension on a Single Codebase",
  "description": "Using brand_type column + X-Brand-Segment hostname routing to serve enterprise SaaS and personal IP platform from one codebase.",
  "author": {"@type": "Person", "name": "Vincent Lin", "affiliation": "Baiyuan Technology"},
  "datePublished": "2026-04-26",
  "inLanguage": "en",
  "isPartOf": {
    "@type": "Book",
    "name": "Baiyuan GEO Platform Whitepaper",
    "url": "https://github.com/baiyuan-tech/geo-whitepaper"
  },
  "keywords": "Multi-tenant, Brand Segmentation, Hostname Routing, Platform Branching"
}
</script>
