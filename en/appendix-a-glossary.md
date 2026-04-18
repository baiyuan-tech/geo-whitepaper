---
title: "Appendix A — Glossary (Full)"
description: "Complete glossary of Baiyuan GEO Platform whitepaper terminology. GEO, AXP, Schema.org, RAG, Closed-Loop, and other defined terms with first-usage chapter references."
appendix: A
part: 6
lang: en
authors:
  - name: Vincent Lin
    affiliation: Baiyuan Technology
license: CC-BY-NC-4.0
last_updated: 2026-04-18
last_modified_at: 2026-04-19T01:23:04+08:00
---

# Appendix A — Glossary (Full)

Alphabetical. Each entry provides: **definition + first chapter mention + related chapters**.

## A

- **@id (Schema.org)** — JSON-LD unique identifier string used to reference entities within `@graph` or across documents. First in: [§6.7](./ch06-axp-shadow-doc.md#67-json-ld-flattening-pitfalls). Core use: [§7.3](./ch07-schema-org.md#73-three-layer-id-interlinking).
- **AI Bot** — umbrella term for crawlers that ingest web content for LLM training or retrieval (GPTBot, ClaudeBot, PerplexityBot, etc.). First in: [§6.3](./ch06-axp-shadow-doc.md#63-cloudflare-worker-injection).
- **AI Citation Rate** — fraction of representative intent queries in which the brand is proactively mentioned by an AI (0–100%). The book's core metric. First in: [§1.2](./ch01-geo-era.md#12-ai-citation-rate-a-brand-new-and-life-or-death-metric). Deep dive: [§3.2.1](./ch03-scoring-algorithm.md#321-citation-rate).
- **aggregateRating** — Schema.org property carrying aggregate review scores. Synced from GBP in our platform. [§8.4](./ch08-gbp-integration.md#84-field-mapping-table).
- **AXP (AI-ready eXchange Page)** — Baiyuan coinage. A "shadow document" served to AI bots — pure HTML + Schema.org JSON-LD + Markdown, decoupled from the human-facing website. Full treatment: [Ch 6](./ch06-axp-shadow-doc.md).
- **App-level filter** — application-layer query-level filtering; in our multi-tenant design, the second line of defense alongside RLS. [§2.5](./ch02-system-overview.md#25-multi-tenant-data-isolation).

## B

- **BullMQ** — Redis-backed Node.js job-queue library; the backbone of our background-task infrastructure. [§2.4](./ch02-system-overview.md#24-technology-stack).
- **Brand Entity** — umbrella term for structured brand data rooted at a Schema.org Organization / LocalBusiness node. [Ch 7](./ch07-schema-org.md).

## C

- **ChainPoll** — a verification method in which the same claim is sent to the LLM 3 times with the same prompt and the majority result is taken. Used in the NLI confidence 0.5–0.8 uncertain band. [§9.3](./ch09-closed-loop.md#93-primary-detection-nli-classification--chainpoll).
- **CID (Customer ID)** — Google Maps internal numeric identifier; cannot be converted to Place ID without a Places API call. [§7.7](./ch07-schema-org.md#77-gbp-url-parser).
- **ClaimReview** — a Schema.org property for annotating whether a claim is true. Used by the platform to publicly declare a hallucination has been corrected. [§9.6](./ch09-closed-loop.md#96-remediation-claimreview-generation-and-multi-path-injection).
- **Closed-Loop** — the detect → compare → decide → generate ClaimReview → inject AXP/RAG → rescan cycle. [Ch 9](./ch09-closed-loop.md).
- **Cloudflare Workers** — edge-compute runtime used for AI-bot UA detection and AXP injection. [§6.3](./ch06-axp-shadow-doc.md#63-cloudflare-worker-injection).
- **Common Crawl** — publicly available large-scale web scrape; a primary pretraining source for most major LLMs. [§1.3](./ch01-geo-era.md#13-geo-is-not-an-extension-of-seo--it-is-an-independent-discipline).
- **Consistency** — one of the seven dimensions; inverse of cross-platform standard deviation of the other six dimensions. [§3.2.7](./ch03-scoring-algorithm.md#327-consistency).
- **Content Depth** — one of the seven dimensions; the depth of the description that accompanies a brand mention. [§3.2.6](./ch03-scoring-algorithm.md#326-content-depth).
- **Core Web Vitals** — Google's page-experience metrics (LCP, FID, CLS). Material for SEO; barely relevant in GEO. [§1.3](./ch01-geo-era.md#13-geo-is-not-an-extension-of-seo--it-is-an-independent-discipline).

## D

- **DIRECT_MODEL_ID_MAP** — translation table between aggregator model IDs and vendor-native model IDs. [§5.3.1](./ch05-multi-provider-routing.md#531-model-ids-differ-between-aggregator-and-native-layers).
- **Dogfooding** — using one's own product internally. Brand E in our case studies is this scenario. [§11.1](./ch11-case-studies.md#111-brand-portraits-anonymized).

## E

- **entailment / contradiction / neutral / opinion** — the four NLI classification outputs. Only `contradiction` is flagged as a hallucination; `neutral` is explicitly *not* one. [§9.3](./ch09-closed-loop.md#93-primary-detection-nli-classification--chainpoll).
- **extraParams** — vendor-specific required API parameters, e.g., Qwen3 requires `enable_thinking: false`. [§5.3.2](./ch05-multi-provider-routing.md#532-extraparams-divergence).

## F

- **Fingerprint** — hash of an AXP or Schema.org artifact, used to determine whether AI has ingested updated content. [Ch 9](./ch09-closed-loop.md) related.
- **FTID (Feature ID)** — Google Maps internal location identifier. [§7.7](./ch07-schema-org.md#77-gbp-url-parser).

## G

- **GBP (Google Business Profile)** — Google's business listing; source of truth for physical businesses. [Ch 8](./ch08-gbp-integration.md).
- **GEO (Generative Engine Optimization)** — the discipline and practice of improving a brand's mention frequency, position, and narrative accuracy inside generative-AI responses. [Ch 1](./ch01-geo-era.md).
- **GPTBot** — OpenAI's web crawler for ChatGPT training-corpus collection. [§6.4](./ch06-axp-shadow-doc.md#64-ai-bot-ua-list-and-detection-strategy).
- **Ground Truth (GT)** — factual ground truth. In this book: the authoritative reference set used to compare candidate claims (live-scraped website / RAG / manual GT — three tiers). [§9.3](./ch09-closed-loop.md#93-primary-detection-nli-classification--chainpoll).

## H

- **Hallucination** — AI-generated content that does not match reality. Split in this book into 5 types: entity misidentification, business fact error, industry miscategory, temporal/spatial error, attribute error. [§9.2](./ch09-closed-loop.md#92-five-types-of-ai-hallucination).

## I

- **industry_code** — 25-industry classification enum used by the platform. [§7.2](./ch07-schema-org.md#72-industry-specialized-type-across-25-categories).
- **Intent Query** — a representative user question generated dynamically based on a brand's industry. [§2.2.1](./ch02-system-overview.md#221-scan-module--monitoring).
- **is_physical** — flag on a brand entity indicating physical vs online business; drives Schema.org field weights. [§7.4](./ch07-schema-org.md#74-physical-vs-online-divergent-field-weights).
- **isStale** — marker set under Stale Carry-Forward indicating a platform's score is a historical carried-forward value. [§4.3](./ch04-stale-carry-forward.md#43-stale-carry-forward-design).

## J

- **JSON-LD** — JavaScript Object Notation for Linked Data; the dominant serialization format for Schema.org. [§6.2](./ch06-axp-shadow-doc.md#62-axp-the-structure-of-a-shadow-document), [Ch 7](./ch07-schema-org.md).

## K

- **kgmid (Knowledge Graph Machine ID)** — Google Knowledge Graph entity identifier. Used in `sameAs`.
- **Knowledge Graph** — structured network of entities and relationships. Built via `@id` interlinking in Schema.org. [§7.3](./ch07-schema-org.md#73-three-layer-id-interlinking).

## L

- **Layer 1 sentinel scan** — 4-hour cadence lightweight verification against search-type AI. [§9.7](./ch09-closed-loop.md#97-two-tier-rescan-loop).
- **Layer 2 full scan** — 24-hour cadence deep scan with seven-dimension scoring + fingerprint comparison. [§9.7](./ch09-closed-loop.md#97-two-tier-rescan-loop).
- **LLM Wiki** — Baiyuan coinage. An active semantic-knowledge layer maintained by LLMs, sitting at L1 inside the RAG architecture. [§9.5](./ch09-closed-loop.md#95-l1-llm-wiki-an-active-semantic-layer).
- **LocalBusiness** — Schema.org `@type` commonly used for physical businesses. [§7.2](./ch07-schema-org.md#72-industry-specialized-type-across-25-categories).

## M

- **modelRouter** — in-house multi-AI-provider routing service providing a primary/fallback dual-path abstraction. [Ch 5](./ch05-multi-provider-routing.md).

## N

- **NLI (Natural Language Inference)** — the task of classifying the entailment relationship between two passages. Our primary hallucination-detection mechanism. [§9.3](./ch09-closed-loop.md#93-primary-detection-nli-classification--chainpoll).

## O

- **OpenAI-compatible API** — aggregator endpoints that wrap multiple AI vendors behind OpenAI's SDK interface. [§5.2](./ch05-multi-provider-routing.md#52-modelrouter-architecture).

## P

- **pgvector** — PostgreSQL extension for vector embeddings and similarity search. [§2.4](./ch02-system-overview.md#24-technology-stack).
- **Phase Baseline Testing** — longitudinal test using a fixed 20-question set re-asked across different time points. [Ch 10](./ch10-phase-baseline.md).
- **Place ID** — Google Maps unique location identifier; primary key of the Places API. [§7.7](./ch07-schema-org.md#77-gbp-url-parser).
- **Position Quality** — one of the seven dimensions; weighted average of positions at which the brand appears in AI responses. [§3.2.2](./ch03-scoring-algorithm.md#322-position-quality).
- **Platform Breadth** — one of the seven dimensions; platforms with at least one mention ÷ total monitored platforms. [§3.2.4](./ch03-scoring-algorithm.md#324-platform-breadth).

## Q

- **Query Coverage** — one of the seven dimensions; diversity of intent-query types in which the brand is mentioned. [§3.2.3](./ch03-scoring-algorithm.md#323-query-coverage).
- **QPM (Queries Per Minute)** — API quota unit. GBP API default is 300 QPM. [§8.5](./ch08-gbp-integration.md#85-sync-frequency-and-quota).

## R

- **RAG (Retrieval-Augmented Generation)** — generation augmented by retrieval. Our platform uses a centralized shared RAG SaaS architecture. [§9.4](./ch09-closed-loop.md#94-central-shared-rag-saas-infrastructure).
- **RLS (Row-Level Security)** — PostgreSQL row-level access control; first line of defense in our multi-tenant isolation. [§2.5](./ch02-system-overview.md#25-multi-tenant-data-isolation).

## S

- **sameAs** — Schema.org property pointing to the same entity on other platforms (Wikipedia, Wikidata, LinkedIn, GBP). [§7.3](./ch07-schema-org.md#73-three-layer-id-interlinking).
- **Schema.org** — the structured-data vocabulary jointly promoted by Google, Bing, Yahoo, and Yandex. [Ch 7](./ch07-schema-org.md).
- **Sentiment Score** — one of the seven dimensions; aggregated tone of mentions (positive / neutral / negative). [§3.2.5](./ch03-scoring-algorithm.md#325-sentiment-score).
- **Sentinel Scan** — see Layer 1.
- **SERP (Search Engine Results Page)** — the core battlefield of classical SEO. [§1.1](./ch01-geo-era.md#11-generative-search-has-taken-over-user-habit).
- **Sitemap** — XML URL list that helps crawlers discover content. [§6.6](./ch06-axp-shadow-doc.md#66-automatic-sitemap-generation).
- **Smoke Test** — minimal validation run on fallback paths at service startup, to ensure they are actually reachable. [§5.6.1](./ch05-multi-provider-routing.md#561-startup-smoke-test).
- **Stale Carry-Forward** — Baiyuan coinage. When a platform's scan fails 100%, carry forward the last successful value from history and mark it as stale; used to maintain signal continuity. [Ch 4](./ch04-stale-carry-forward.md).

## T

- **Tenant ID (X-Tenant-ID)** — multi-tenant HTTP header used for tenant filtering in the central shared RAG. [§9.4](./ch09-closed-loop.md#94-central-shared-rag-saas-infrastructure).

## U

- **UA (User-Agent)** — HTTP request header; the basis on which AXP injection decides AI-bot vs human. [§6.4](./ch06-axp-shadow-doc.md#64-ai-bot-ua-list-and-detection-strategy).

## W

- **Webhook** — server-initiated event notification. GBP API **does not provide** webhooks. [§8.6](./ch08-gbp-integration.md#86-remedy-for-no-webhook).
- **Wikidata** — open knowledge graph; a key target for Schema.org `sameAs`. [§7.3](./ch07-schema-org.md#73-three-layer-id-interlinking).
- **Wizard** — linear seven-step Schema.org onboarding flow for new brands. [§7.6](./ch07-schema-org.md#76-dual-entry-points-wizard--edit).

---

## Standards cross-reference

| Standard | Role | Chapters |
|----------|------|----------|
| [Schema.org](https://schema.org/) | structured-data vocabulary | [Ch 6, 7, 9](./ch07-schema-org.md) |
| [JSON-LD 1.1](https://www.w3.org/TR/json-ld11/) | Linked Data serialization | [Ch 6, 7](./ch07-schema-org.md) |
| [OpenGraph](https://ogp.me/) | meta tags for link previews | [Ch 6](./ch06-axp-shadow-doc.md) |
| [robots.txt](https://www.rfc-editor.org/rfc/rfc9309.html) | Robots Exclusion Protocol | [Ch 6](./ch06-axp-shadow-doc.md) |
| [sitemaps.xml](https://www.sitemaps.org/protocol.html) | Sitemap Protocol 0.9 | [§6.6](./ch06-axp-shadow-doc.md#66-automatic-sitemap-generation) |

---

**Navigation**: [← Ch 12: Limitations](./ch12-limitations.md) · [📖 Index](../README.md) · [Appendix B: Public API Spec →](./appendix-b-api.md)
