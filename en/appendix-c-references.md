---
title: "Appendix C — References"
description: "Complete reference list for the Baiyuan GEO Platform whitepaper. Official specs, academic papers, technical docs, industry reports."
appendix: C
part: 6
lang: en
authors:
  - name: Vincent Lin
    affiliation: Baiyuan Technology
license: CC-BY-NC-4.0
last_updated: 2026-04-18
last_modified_at: '2026-04-19T01:50:45Z'
---





# Appendix C — References

Every external source cited in this book. Grouped into four categories.

## 1. Official specifications and standards

### Schema.org and Linked Data

- Schema.org. *Schema.org Vocabulary Specification*. <https://schema.org/docs/schemas.html>
- W3C. *JSON-LD 1.1: A JSON-based Serialization for Linked Data*. <https://www.w3.org/TR/json-ld11/>
- Schema.org. *ClaimReview*. <https://schema.org/ClaimReview>
- Schema.org. *LocalBusiness*. <https://schema.org/LocalBusiness>
- Schema.org. *Organization*. <https://schema.org/Organization>
- Schema.org. *Service*. <https://schema.org/Service>
- Schema.org. *Person*. <https://schema.org/Person>

### Google ecosystem

- Google Business Profile API. *Overview*. <https://developers.google.com/my-business>
- Google Business Profile API. *Notifications Overview*. <https://developers.google.com/my-business/content/notification-setup>
- Google Maps Platform. *Places API documentation*. <https://developers.google.com/maps/documentation/places/web-service>
- Google Search Central. *How Search works*. <https://www.google.com/search/howsearchworks/>
- Google Search Central. *Fact check article markup*. <https://developers.google.com/search/docs/appearance/structured-data/factcheck>
- Google Search Central. *Google-Extended and generative AI training*. <https://developers.google.com/search/docs/crawling-indexing/overview-google-crawlers>

### AI bot User Agents

- OpenAI. *GPTBot User Agent Documentation*. <https://platform.openai.com/docs/gptbot>
- OpenAI. *OAI-SearchBot User Agent*. <https://platform.openai.com/docs/bots>
- Anthropic. *ClaudeBot documentation*. <https://support.anthropic.com/en/articles/8896518>
- Perplexity. *PerplexityBot*. <https://docs.perplexity.ai/guides/bots>
- Common Crawl. *CCBot User Agent*. <https://commoncrawl.org/ccbot>

### Web protocols

- IETF. *RFC 9309 — Robots Exclusion Protocol*. <https://www.rfc-editor.org/rfc/rfc9309.html>
- sitemaps.org. *Sitemap Protocol 0.9*. <https://www.sitemaps.org/protocol.html>
- W3C. *HTTP/1.1*. <https://www.rfc-editor.org/rfc/rfc7230>

### Open licenses

- Creative Commons. *CC BY-NC 4.0 Legal Code*. <https://creativecommons.org/licenses/by-nc/4.0/legalcode>
- Citation File Format. *CFF 1.2.0 Specification*. <https://github.com/citation-file-format/citation-file-format>

---

## 2. Technical frameworks and development tools

- Cloudflare. *Workers Runtime Documentation*. <https://developers.cloudflare.com/workers/>
- Next.js. *App Router & Server Components*. <https://nextjs.org/docs>
- PostgreSQL. *Row Security Policies*. <https://www.postgresql.org/docs/current/ddl-rowsecurity.html>
- pgvector. *Open-source vector similarity search for Postgres*. <https://github.com/pgvector/pgvector>
- BullMQ. *Queue documentation*. <https://docs.bullmq.io/>
- OpenAI. *Node.js SDK*. <https://github.com/openai/openai-node>
- Anthropic. *TypeScript SDK*. <https://github.com/anthropics/anthropic-sdk-typescript>

---

## 3. Industry research and statistics

- OpenAI. (2025). *Usage & Revenue Update, Q3 2025*. Official quarterly disclosure. (Cited in Ch 1)
- Perplexity AI. (2025). *Year in Review 2024: Search Volume & Engagement*. Official blog. (Cited in Ch 1)
- SimilarWeb. (2025). *The State of Generative Search: AI Overview Impact on Publisher Traffic*. Research report. (Cited in Ch 1)
- Google. (2024). *Generative AI in Search: Let Google do the searching for you*. Official announcement.

---

## 4. Books and monographs

- Kleppmann, M. (2017). *Designing Data-Intensive Applications: The Big Ideas Behind Reliable, Scalable, and Maintainable Systems*. O'Reilly Media. ISBN 978-1449373320.
  - Chapter 8, *"The Trouble with Distributed Systems,"* is the general reference framework for Ch 4's Stale Carry-Forward pattern.
- Nygard, M. T. (2018). *Release It! Design and Deploy Production-Ready Software* (2nd ed.). Pragmatic Bookshelf. ISBN 978-1680502398.
  - Circuit Breaker, Bulkhead, and Timeout patterns are the design references for Ch 5's modelRouter.
- Kim, G., Humble, J., Debois, P., & Willis, J. (2016). *The DevOps Handbook*. IT Revolution. ISBN 978-1942788003.
  - The closed-feedback design of Ch 9 is informed by the *"continuous learning"* chapter of the Third Way.

---

## 5. Related academic work (selected)

*This book did not perform a comprehensive literature review. The papers below are representative works the author has read; readers are welcome to suggest additions via [GitHub Issues](https://github.com/baiyuan-tech/geo-whitepaper/issues).*

### LLM hallucination detection

- Manakul, P., Liusie, A., & Gales, M. J. F. (2023). *SelfCheckGPT: Zero-Resource Black-Box Hallucination Detection for Generative Large Language Models*. EMNLP 2023.
- Min, S., Krishna, K., Lyu, X., et al. (2023). *FActScore: Fine-grained Atomic Evaluation of Factual Precision in Long Form Text Generation*. EMNLP 2023.

### Knowledge graphs and entity linking

- Pan, J. Z., Razniewski, S., Kalo, J.-C., et al. (2024). *Large Language Models and Knowledge Graphs: Opportunities and Challenges*. TGDK 2024.
- Hogan, A., Blomqvist, E., Cochez, M., et al. (2021). *Knowledge Graphs*. ACM Computing Surveys, 54(4).

### Multi-source NLI

- Nie, Y., Chen, H., & Bansal, M. (2019). *Combining Fact Extraction and Verification with Neural Semantic Matching Networks*. AAAI 2019.
- Thorne, J., Vlachos, A., Christodoulopoulos, C., & Mittal, A. (2018). *FEVER: A Large-scale Dataset for Fact Extraction and VERification*. NAACL 2018.

### Structured data and LLMs

- Yang, J., Chen, H., Yan, Y., et al. (2024). *Exploring the Role of Structured Data in Large Language Model Fact Verification*. Preprint.

---

## How to cite this book

The canonical citation formats are in [`CITATION.cff`](../CITATION.cff) and in the [citation section of the README](../README.md#引用方式).

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

---

**Navigation**: [← Appendix B: API Spec](./appendix-b-api.md) · [📖 Index](../README.md) · [Appendix D: Figure Catalog →](./appendix-d-figures.md)
