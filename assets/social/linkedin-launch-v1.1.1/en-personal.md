---
title: "LinkedIn post — Vincent Lin personal account (en)"
audience: "International engineers, AI/SaaS practitioners, academic researchers"
recommended_post_time: "2026-05-04 21:00 GMT+8 (24h after zh-TW)"
char_count: ~1750
license: CC-BY-NC-4.0
---

# en personal post

## Headline (not shown by LinkedIn, for SEO)

Baiyuan GEO Platform Whitepaper v1.1.1 — 16 chapters, 50K characters, Zenodo DOI

## Post body (copy-paste ready)

---

I finally wrote down the last 2 years of engineering work and deposited it on Zenodo:

📖 *Baiyuan GEO Platform: A Whitepaper on Building a SaaS for Generative Engine Optimization*

DOI: https://doi.org/10.5281/zenodo.19994035
GitHub: https://github.com/baiyuan-tech/geo-whitepaper

This is not a marketing piece. It's a 16-chapter, 50,000-character engineering journal documenting what we actually built and what we actually got wrong.

【What's really inside】

✅ **Seven-dimension AI citation-rate scoring algorithm** — how to quantify whether ChatGPT/Claude/Gemini actually mention your brand correctly

✅ **AXP shadow document delivery** — even when the customer's site has zero SEO, we run Cloudflare Workers on their domain to inject Schema.org / llms.txt / sitemap for AI bots only

✅ **F12 Three-Layer Structural Optimizer** — V1 rule-based scoring + V3.1 dual-engine (AutoGEO [arXiv:2510.11438] + E-GEO [arXiv:2511.20867]) literally line-by-line ported with DB-level placeholder guard preventing fabricated paper_table_ref

✅ **Six-layer hardening for LLM hallucination in our RAG microservice** — real bugs we hit in production:
   • LLM emitting partial UUIDs aborting entire KB compile
   • Docker DNS dual-fallback (single internal DNS broke Google OAuth platform-wide)
   • Reasoning models consuming `maxTokens` budget for reasoning, leaving content empty

✅ **Schema.org three-layer entity knowledge graph** + closed-loop hallucination detection & auto-remediation

✅ **Platform SSOT chain** — how a 10K-tenant SaaS unifies cross-page / cross-table / cross-crawler data under a single source of truth

【What I honestly couldn't solve yet】

Ch 12 / 14–16 each end with an "Open Problems" section:

⚠️ L3 S3 cache layer not deployed yet
⚠️ Phase 1→2→3 escalation logic spec-aligned but never tested in real load
⚠️ Wiki fallback article quality has no end-to-end evaluation
⚠️ ME custom engine's metadata gating is too strict (5 of 17 personal-IP brands fail through to enterprise-toned fallback)

This isn't a polished success story. It's an engineering field journal with the open wounds visible.

【Who might read this】

- Engineers building multi-tenant SaaS (architectural patterns copy-able directly)
- Researchers studying generative AI citation mechanics (cites real arxiv only; DB triggers prevent fabricated metrics)
- AI marketers / brand managers (non-engineering Ch 1 + Ch 11 has 5-brand real data)

Licensed CC BY-NC 4.0. Free to cite, adapt, translate. GitHub issues open for discussion. Each chapter has Schema.org JSON-LD for AI crawlers.

If you work in this space, let's talk.

<!-- markdownlint-disable-next-line MD018 -->
#GenerativeEngineOptimization #GEO #AISearch #SaaS #LLM #OpenScience #Zenodo #SchemaOrg #Multitenant #PostgreSQL #BullMQ #Cloudflare
