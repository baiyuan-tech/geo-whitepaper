---
title: "Appendix C — 參考文獻"
description: "百原GEO Platform 白皮書引用的官方規範、學術論文、技術文件、產業報告全集。"
appendix: C
part: 6
lang: zh-TW
authors:
  - name: Vincent Lin
    affiliation: Baiyuan Technology
license: CC-BY-NC-4.0
last_updated: 2026-04-18
last_modified_at: 2026-04-18T17:00:53Z
---

# Appendix C — 參考文獻

本書引用的所有外部資料來源。分為四類。

## 1. 官方規範與標準

### Schema.org 與 Linked Data

- Schema.org. *Schema.org Vocabulary Specification*. <https://schema.org/docs/schemas.html>
- W3C. *JSON-LD 1.1: A JSON-based Serialization for Linked Data*. <https://www.w3.org/TR/json-ld11/>
- Schema.org. *ClaimReview*. <https://schema.org/ClaimReview>
- Schema.org. *LocalBusiness*. <https://schema.org/LocalBusiness>
- Schema.org. *Organization*. <https://schema.org/Organization>
- Schema.org. *Service*. <https://schema.org/Service>
- Schema.org. *Person*. <https://schema.org/Person>

### Google 生態

- Google Business Profile API. *Overview*. <https://developers.google.com/my-business>
- Google Business Profile API. *Notifications Overview*. <https://developers.google.com/my-business/content/notification-setup>
- Google Maps Platform. *Places API documentation*. <https://developers.google.com/maps/documentation/places/web-service>
- Google Search Central. *How Search works*. <https://www.google.com/search/howsearchworks/>
- Google Search Central. *Fact check article markup*. <https://developers.google.com/search/docs/appearance/structured-data/factcheck>
- Google Search Central. *Google-Extended and generative AI training*. <https://developers.google.com/search/docs/crawling-indexing/overview-google-crawlers>

### AI Bot User Agents

- OpenAI. *GPTBot User Agent Documentation*. <https://platform.openai.com/docs/gptbot>
- OpenAI. *OAI-SearchBot User Agent*. <https://platform.openai.com/docs/bots>
- Anthropic. *ClaudeBot documentation*. <https://support.anthropic.com/en/articles/8896518>
- Perplexity. *PerplexityBot*. <https://docs.perplexity.ai/guides/bots>
- Common Crawl. *CCBot User Agent*. <https://commoncrawl.org/ccbot>

### 網站通訊協定

- IETF. *RFC 9309 — Robots Exclusion Protocol*. <https://www.rfc-editor.org/rfc/rfc9309.html>
- sitemaps.org. *Sitemap Protocol 0.9*. <https://www.sitemaps.org/protocol.html>
- W3C. *HTTP/1.1*. <https://www.rfc-editor.org/rfc/rfc7230>

### 開放授權

- Creative Commons. *CC BY-NC 4.0 Legal Code*. <https://creativecommons.org/licenses/by-nc/4.0/legalcode>
- Citation File Format. *CFF 1.2.0 Specification*. <https://github.com/citation-file-format/citation-file-format>

---

## 2. 技術框架與開發工具

- Cloudflare. *Workers Runtime Documentation*. <https://developers.cloudflare.com/workers/>
- Next.js. *App Router & Server Components*. <https://nextjs.org/docs>
- PostgreSQL. *Row Security Policies*. <https://www.postgresql.org/docs/current/ddl-rowsecurity.html>
- pgvector. *Open-source vector similarity search for Postgres*. <https://github.com/pgvector/pgvector>
- BullMQ. *Queue documentation*. <https://docs.bullmq.io/>
- OpenAI. *Node.js SDK*. <https://github.com/openai/openai-node>
- Anthropic. *TypeScript SDK*. <https://github.com/anthropics/anthropic-sdk-typescript>

---

## 3. 產業研究與統計資料

- OpenAI. (2025). *Usage & Revenue Update, Q3 2025*. 官方季度揭露（引用於 Ch 1）
- Perplexity AI. (2025). *Year in Review 2024: Search Volume & Engagement*. 官方部落格（引用於 Ch 1）
- SimilarWeb. (2025). *The State of Generative Search: AI Overview Impact on Publisher Traffic*. 研究報告（引用於 Ch 1）
- Google. (2024). *Generative AI in Search: Let Google do the searching for you*. 官方公告

---

## 4. 書籍與專書

- Kleppmann, M. (2017). *Designing Data-Intensive Applications: The Big Ideas Behind Reliable, Scalable, and Maintainable Systems*. O'Reilly Media. ISBN 978-1449373320.
  - 第 8 章「The Trouble with Distributed Systems」對部分失敗處理的討論是 Ch 4 Stale Carry-Forward 的通用參考框架。
- Nygard, M. T. (2018). *Release It! Design and Deploy Production-Ready Software* (2nd ed.). Pragmatic Bookshelf. ISBN 978-1680502398.
  - Circuit Breaker、Bulkhead、Timeout 三個 pattern 是 Ch 5 modelRouter 的設計參考。
- Kim, G., Humble, J., Debois, P., & Willis, J. (2016). *The DevOps Handbook*. IT Revolution. ISBN 978-1942788003.
  - 閉環回饋設計（Ch 9）受第三之路「持續學習」章節啟發。

---

## 5. 相關學術研究（選錄）

*本書撰寫時未做完整文獻回顧。以下為作者閱讀過、與本書主題相關的代表性研究；歡迎讀者於 [GitHub Issues](https://github.com/baiyuan-tech/geo-whitepaper/issues) 補充推薦。*

### LLM 幻覺偵測

- Manakul, P., Liusie, A., & Gales, M. J. F. (2023). *SelfCheckGPT: Zero-Resource Black-Box Hallucination Detection for Generative Large Language Models*. EMNLP 2023.
- Min, S., Krishna, K., Lyu, X., et al. (2023). *FActScore: Fine-grained Atomic Evaluation of Factual Precision in Long Form Text Generation*. EMNLP 2023.

### 知識圖譜與 Entity Linking

- Pan, J. Z., Razniewski, S., Kalo, J.-C., et al. (2024). *Large Language Models and Knowledge Graphs: Opportunities and Challenges*. TGDK 2024.
- Hogan, A., Blomqvist, E., Cochez, M., et al. (2021). *Knowledge Graphs*. ACM Computing Surveys, 54(4).

### 多源證據的 NLI

- Nie, Y., Chen, H., & Bansal, M. (2019). *Combining Fact Extraction and Verification with Neural Semantic Matching Networks*. AAAI 2019.
- Thorne, J., Vlachos, A., Christodoulopoulos, C., & Mittal, A. (2018). *FEVER: A Large-scale Dataset for Fact Extraction and VERification*. NAACL 2018.

### 結構化資料對 LLM 的影響

- Yang, J., Chen, H., Yan, Y., et al. (2024). *Exploring the Role of Structured Data in Large Language Model Fact Verification*. 預印本。

---

## 如何引用本書

本書的正式引用格式請見 [`CITATION.cff`](../CITATION.cff) 或 [README 的引用方式段落](../README.md#引用方式)。

```bibtex
@techreport{lin2026baiyuangeo,
  author      = {Lin, Vincent},
  title       = {Baiyuan GEO Platform: A Whitepaper on Building a SaaS for Generative Engine Optimization},
  institution = {Baiyuan Technology},
  year        = {2026},
  url         = {https://github.com/baiyuan-tech/geo-whitepaper},
  note        = {v1.0}
}
```

---

**導覽**：[← 附錄 B: 公開 API 規格](./appendix-b-api.md) · [📖 目次](../README.md) · [附錄 D: 配圖總表 →](./appendix-d-figures.md)
