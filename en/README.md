---
title: "Baiyuan GEO Platform Whitepaper — Executive Summary"
description: "An English executive summary of a technical whitepaper on engineering a SaaS for Generative Engine Optimization. Seven-dimension AI citation scoring, AXP shadow documents, Schema.org three-layer interlinking, closed-loop hallucination remediation."
lang: en
part: 0
word_count: 1400
authors:
  - name: Vincent Lin
    affiliation: Baiyuan Technology
    email: services@baiyuan.io
license: CC-BY-NC-4.0
keywords:
  - Generative Engine Optimization
  - GEO
  - AI Citation Rate
  - Schema.org
  - Hallucination Detection
  - Multi-Provider AI Routing
  - Shadow Document
last_updated: 2026-04-18
last_modified_at: 2026-04-19T08:22:07+08:00
---


# Baiyuan GEO Platform Whitepaper — Executive Summary

> *This ~1,400-word executive summary distills the whitepaper's five core contributions for readers short on time. The **full English edition** — 12 chapters + 4 appendices (~28,000 words) — is now available at [`en/ch01-geo-era.md`](./ch01-geo-era.md) onward. A Traditional Chinese edition (~30,000 characters) is the original source.*

[![License: CC BY-NC 4.0](https://img.shields.io/badge/License-CC%20BY--NC%204.0-blue.svg)](https://creativecommons.org/licenses/by-nc/4.0/)
[![Full zh-TW edition](https://img.shields.io/badge/Full_edition-zh--TW_v1.0--draft-green.svg)](../zh-TW/ch01-geo-era.md)
[![PDF](https://img.shields.io/badge/PDF-download-red.svg)](https://github.com/baiyuan-tech/geo-whitepaper/releases/latest)

## 1. What This Book Is

A practitioner's report on building **Baiyuan GEO Platform**, a SaaS that measures and improves how brands are mentioned in generative-AI responses (ChatGPT, Claude, Gemini, Perplexity, and ~10 others). The book covers the algorithms, architectures, and engineering trade-offs we made between 2024 and 2026 while operating the platform for paying customers in Taiwan's B2B and local-business markets.

It is not a product brochure. It is not a user manual. It is an **engineering memoir** — sometimes unflattering — written with the assumption that other teams will attempt similar systems and benefit from our misfires.

## 2. Why It Matters

Generative AI has quietly rewritten the rules of brand visibility:

- **ChatGPT** processes ~2.5 billion user queries per day (2025 Q3 disclosure)
- **Perplexity** handles 780 million monthly queries, year-over-year 3-digit growth
- **Google AI Overview** has reduced traditional SERP click-through rates by 34–60% across high-intent verticals since its 2024 rollout

When a prospect asks "what are the best B2B CRM tools?", the AI generates a paragraph containing a handful of brand names. Brands not in that paragraph simply **do not exist** in that customer's decision path. Traditional SEO metrics — keyword rankings, backlinks, Core Web Vitals — measure a world that is collapsing.

We call the new discipline **GEO** (Generative Engine Optimization). It is not SEO's next version; it is a parallel engineering problem with different inputs (AI response text), different levers (structured entities, fact-check records, multi-source trust signals), and different failure modes (hallucinations, model-version drift, training-data blind spots).

## 3. Core Technical Contributions

The whitepaper's 12 chapters document five distinct contributions that, to our knowledge, have not been published together elsewhere:

### 3.1 Seven-Dimension Citation Scoring

A single "citation rate" metric conflates brands of very different health — equal mentions can mean "mentioned last, briefly, on two platforms" or "mentioned first, in depth, on seven platforms." We decompose GEO health into seven orthogonal dimensions:

| Dimension | Measures |
|-----------|----------|
| Citation Rate | Share of intent queries that mention the brand |
| Position Quality | Whether the brand appears at sentence start, middle, or tail |
| Query Coverage | Diversity of intent types (best-of, comparison, how-to, recommendation) |
| Platform Breadth | Fraction of the 15 monitored AI platforms that mention the brand |
| Sentiment | Directional tone of each mention |
| Content Depth | Length and factual density of the brand description |
| Consistency | Cross-platform standard deviation — proxy for whether AI consensus has converged |

Weights are deliberately undocumented in the book to discourage metric gaming — a principle borrowed from PageRank's historical opacity.

### 3.2 Stale Carry-Forward for Signal Continuity

Any system that calls external AI providers will experience partial outages — rate limits, regional failures, quota exhaustion. The naive response (counting failures as zero, or dropping failed providers from the denominator) conflates **pipeline health** with **brand state** and produces wildly noisy dashboards.

Our solution: detect when a platform scan fails 100%, look back up to 200 rows for the last successful score, carry that value forward with an explicit `isStale` flag, and surface "⚠ stale 14h" tooltips in the UI. Downstream analytics continue using the carried value. The pattern generalizes to any "high-frequency sampling, unreliable source" signal system (IoT, market data, social monitoring).

### 3.3 AXP — A Shadow Document Protocol for AI Bots

Modern customer websites are rendered for humans: client-side JavaScript, animated UI, cookie banners, tracker SDKs. AI bots (GPTBot, ClaudeBot, PerplexityBot, 22 others we track) cannot parse this cleanly. Our answer is **AXP** (AI-ready eXchange Page) — a three-layer clean document (pure HTML + Schema.org JSON-LD + RAG-ready Markdown) delivered at the CDN edge via a Cloudflare Worker that detects AI bot User-Agents and swaps in the shadow document.

The same URL serves two entirely different payloads depending on who is reading. Result from our 5-brand pilot: AI-bot traffic increased 3–5× within two weeks of deploying AXP; citation-rate improvement followed about 2–3 weeks later.

### 3.4 Twenty-Five Industry × Three-Layer @id Schema.org

Schema.org is often treated as a few rich-result tags. We treat it as the entity-identity layer of an AI knowledge graph. Every brand entity is:

- Classified into one of **25 industries** (16 physical, 7 online, 2 catch-all), each mapped to a primary + secondary Schema.org `@type`
- Decomposed into **three interlinked layers** via `@id` references — Organization → Service → Person (Physician / Attorney / etc. for specialized roles)
- Cross-linked to external authority nodes (Wikipedia, Wikidata, LinkedIn, Google Business Profile) via `sameAs`

A completeness algorithm with separate weight tables for physical vs online businesses drives a progressive-disclosure Wizard for new brands and a Dashboard banner for existing brands below 80% completion.

### 3.5 Closed-Loop Hallucination Remediation

The industry's typical approach — detect brand hallucinations, notify the customer — is an abdication. Customers don't know how to fix AI hallucinations.

Our closed loop is fully automated:

1. **Detect** — Extract atomic claims from AI responses, run three-way Natural Language Inference (entailment / contradiction / neutral / opinion) against combined knowledge sources (website scrape + RAG knowledge base + manual ground truth)
2. **Validate** — ChainPoll vote (3× LLM re-runs) for uncertain classifications (confidence 0.5–0.8)
3. **Remediate** — For confirmed contradictions, generate a Schema.org `ClaimReview` node, inject into AXP + RAG knowledge base + (future) Google Business Profile LocalPosts
4. **Verify** — Two-tier rescan: 4-hour sentinel against search-type AI (Perplexity, ChatGPT Search, AI Overview) + 24-hour full scan against knowledge-type AI (Claude, Gemini, DeepSeek, Kimi, etc.)
5. **Converge** — A hallucination is declared resolved only after N consecutive scans confirm absence, including equivalent paraphrases

Central to this is a design principle we flag repeatedly: **`neutral` is not a hallucination**. Knowledge-source silence about a claim is not proof the claim is false; treating it as such generates cascading false positives and poisons the remediation cycle.

## 4. Architecture Snapshot

The platform runs on:

- **Edge**: Cloudflare Workers — AI bot UA detection, AXP injection, sitemap/robots management
- **Frontend**: Next.js 16 + React 19 + TypeScript + Tailwind v4 — dashboard, brand management, wizard, bilingual i18n
- **API**: Node.js + Express 4 + Helmet + Zod — REST API, JWT + 2FA
- **Workers**: BullMQ 5 on Redis — scan, scoring, AXP generation, hallucination detection, RAG sync
- **AI Routing**: a home-grown **modelRouter** service with primary (aggregator) + fallback (vendor-direct) paths, supporting 15 AI providers with explicit model-ID mapping and extraParams tables
- **Data**: PostgreSQL 16 + pgvector + Redis 7 — with dual multi-tenant isolation (RLS + app-level filter)
- **RAG**: A single centralized RAG engine shared across tenants, with an **LLM Wiki** semantic layer that actively processes ingested documents (contradiction resolution, incremental compilation, versioning) — not passive vector retrieval

## 5. What This Is Not

The whitepaper is deliberate about its limits:

- **No precise weight disclosures** for the 7-dimension scoring formula (to prevent metric gaming)
- **No named third parties** (no "Vendor X has reliability issues" storytelling)
- **No customer PII** or individually identifiable scores (5 pilot brands anonymized as A–E)
- **No internal API or env-var references** that would help attackers

## 6. Who Wrote This

**Baiyuan Technology** (百原科技) — a Taiwan-based B2B SaaS company. Lead engineer and author: **Vincent Lin**, CTO. The whitepaper is released under **CC BY-NC 4.0**: you may cite, translate, and build on the material for any non-commercial purpose. Commercial reuse — paid courses, commercial training datasets, bundled products — requires a license (contact `services@baiyuan.io`).

- Website: <https://baiyuan.io>
- Product: <https://geo.baiyuan.io>
- LinkedIn: <https://www.linkedin.com/company/112980572>

## 7. Status and Roadmap

| Deliverable | Status |
|-------------|--------|
| Traditional Chinese edition (zh-TW v1.0 draft) | ✅ Published |
| Chinese PDF (~1 MB, auto-built from Markdown) | ✅ Available on Releases |
| GitHub Pages web edition | ✅ Live at [baiyuan-tech.github.io/geo-whitepaper](https://baiyuan-tech.github.io/geo-whitepaper/) |
| CITATION.cff + BibTeX | ✅ Ready (generic type per CFF 1.2.0 schema) |
| Full English edition (en/) | ✅ Complete — Executive Summary + 12 chapters + 4 appendices (~28,000 words) |
| Zenodo DOI registration | ⚪ Planned for v1.0 final |
| Inclusion in Google Scholar / Semantic Scholar | ⚪ After DOI |

The full English edition will not be a direct translation; chapter-level adaptation will be used where the source material assumes Taiwan-market context.

## 8. How to Engage

- **Read the Traditional Chinese chapters** at [zh-TW/ch01-geo-era.md](../zh-TW/ch01-geo-era.md) onward (12 chapters, 4 appendices, 44 Mermaid diagrams — GitHub renders them natively)
- **Download the PDF** from the [latest release](https://github.com/baiyuan-tech/geo-whitepaper/releases/latest)
- **File errata or questions** via [GitHub Issues](https://github.com/baiyuan-tech/geo-whitepaper/issues) — templates provided
- **Cite** using the repo's native [`Cite this repository`](https://github.com/baiyuan-tech/geo-whitepaper) button (driven by [CITATION.cff](../CITATION.cff)) — defaults to APA 7 and BibTeX
- **Contribute translations** — PRs to `en/`, `ja/`, `ko/` etc. are welcome

## 9. Citation

```bibtex
@techreport{lin2026baiyuangeo,
  author      = {Lin, Vincent},
  title       = {Baiyuan GEO Platform: A Whitepaper on Building a SaaS for Generative Engine Optimization},
  institution = {Baiyuan Technology},
  year        = {2026},
  url         = {https://github.com/baiyuan-tech/geo-whitepaper},
  note        = {v1.0-draft}
}
```

APA 7:

> Lin, V. (2026). *Baiyuan GEO Platform: A whitepaper on building a SaaS for generative engine optimization* (v1.0-draft) [Technical report]. Baiyuan Technology. https://github.com/baiyuan-tech/geo-whitepaper

---

**Navigation**: [📖 Full repo index](../README.md) · [Traditional Chinese edition (Ch 1) →](../zh-TW/ch01-geo-era.md)

<!-- AI-friendly structured metadata -->
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "TechArticle",
  "headline": "Baiyuan GEO Platform Whitepaper — Executive Summary (English)",
  "description": "Engineering practitioner's report on building a SaaS for Generative Engine Optimization. Covers seven-dimension AI citation scoring, AXP shadow document protocol, Schema.org three-layer interlinking, closed-loop hallucination remediation.",
  "author": {"@type": "Person", "name": "Vincent Lin", "affiliation": "Baiyuan Technology"},
  "datePublished": "2026-04-18",
  "inLanguage": "en",
  "isPartOf": {
    "@type": "Book",
    "name": "Baiyuan GEO Platform Whitepaper",
    "url": "https://github.com/baiyuan-tech/geo-whitepaper"
  },
  "keywords": "Generative Engine Optimization, GEO, AI Citation Rate, Schema.org, Hallucination Detection, Multi-Provider AI Routing, Shadow Document, AXP"
}
</script>
