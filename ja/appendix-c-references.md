---
title: "付録C — 参考文献"
description: "百元 GEO Platform ホワイトペーパーの完全参考文献リスト。公式仕様、学術論文、技術文書、業界レポート。"
appendix: C
part: 6
lang: ja
authors:
  - name: Vincent Lin
    affiliation: 百元科技 (Baiyuan Technology)
license: CC-BY-NC-4.0
last_updated: 2026-04-18
---

# 付録C — 参考文献

本書で引用した全外部情報源。4カテゴリに整理する。

## 1. 公式仕様と標準

### Schema.org と Linked Data

- Schema.org. *Schema.org Vocabulary Specification*. <https://schema.org/docs/schemas.html>
- W3C. *JSON-LD 1.1: A JSON-based Serialization for Linked Data*. <https://www.w3.org/TR/json-ld11/>
- Schema.org. *ClaimReview*. <https://schema.org/ClaimReview>
- Schema.org. *LocalBusiness*. <https://schema.org/LocalBusiness>
- Schema.org. *Organization*. <https://schema.org/Organization>
- Schema.org. *Service*. <https://schema.org/Service>
- Schema.org. *Person*. <https://schema.org/Person>

### Google エコシステム

- Google Business Profile API. *Overview*. <https://developers.google.com/my-business>
- Google Business Profile API. *Notifications Overview*. <https://developers.google.com/my-business/content/notification-setup>
- Google Maps Platform. *Places API documentation*. <https://developers.google.com/maps/documentation/places/web-service>
- Google Search Central. *How Search works*. <https://www.google.com/search/howsearchworks/>
- Google Search Central. *Fact check article markup*. <https://developers.google.com/search/docs/appearance/structured-data/factcheck>
- Google Search Central. *Google-Extended and generative AI training*. <https://developers.google.com/search/docs/crawling-indexing/overview-google-crawlers>

### AIボット User Agent

- OpenAI. *GPTBot User Agent Documentation*. <https://platform.openai.com/docs/gptbot>
- OpenAI. *OAI-SearchBot User Agent*. <https://platform.openai.com/docs/bots>
- Anthropic. *ClaudeBot documentation*. <https://support.anthropic.com/en/articles/8896518>
- Perplexity. *PerplexityBot*. <https://docs.perplexity.ai/guides/bots>
- Common Crawl. *CCBot User Agent*. <https://commoncrawl.org/ccbot>

### ウェブプロトコル

- IETF. *RFC 9309 — Robots Exclusion Protocol*. <https://www.rfc-editor.org/rfc/rfc9309.html>
- sitemaps.org. *Sitemap Protocol 0.9*. <https://www.sitemaps.org/protocol.html>
- W3C. *HTTP/1.1*. <https://www.rfc-editor.org/rfc/rfc7230>

### オープンライセンス

- Creative Commons. *CC BY-NC 4.0 Legal Code*. <https://creativecommons.org/licenses/by-nc/4.0/legalcode>
- Citation File Format. *CFF 1.2.0 Specification*. <https://github.com/citation-file-format/citation-file-format>

---

## 2. 技術フレームワークと開発ツール

- Cloudflare. *Workers Runtime Documentation*. <https://developers.cloudflare.com/workers/>
- Next.js. *App Router & Server Components*. <https://nextjs.org/docs>
- PostgreSQL. *Row Security Policies*. <https://www.postgresql.org/docs/current/ddl-rowsecurity.html>
- pgvector. *Open-source vector similarity search for Postgres*. <https://github.com/pgvector/pgvector>
- BullMQ. *Queue documentation*. <https://docs.bullmq.io/>
- OpenAI. *Node.js SDK*. <https://github.com/openai/openai-node>
- Anthropic. *TypeScript SDK*. <https://github.com/anthropics/anthropic-sdk-typescript>

---

## 3. 業界調査と統計

- OpenAI. (2025). *Usage & Revenue Update, Q3 2025*. 公式四半期開示。（第1章で引用）
- Perplexity AI. (2025). *Year in Review 2024: Search Volume & Engagement*. 公式ブログ。（第1章で引用）
- SimilarWeb. (2025). *The State of Generative Search: AI Overview Impact on Publisher Traffic*. 調査レポート。（第1章で引用）
- Google. (2024). *Generative AI in Search: Let Google do the searching for you*. 公式発表。

---

## 4. 書籍・単行本

- Kleppmann, M. (2017). *Designing Data-Intensive Applications: The Big Ideas Behind Reliable, Scalable, and Maintainable Systems*. O'Reilly Media. ISBN 978-1449373320.（邦訳：『データ指向アプリケーションデザイン』オライリー・ジャパン）
  - 第8章 *"The Trouble with Distributed Systems"* は第4章 Stale Carry-Forward パターンの一般的参照枠である。
- Nygard, M. T. (2018). *Release It! Design and Deploy Production-Ready Software*（第2版）. Pragmatic Bookshelf. ISBN 978-1680502398.
  - Circuit Breaker、Bulkhead、Timeout パターンは第5章 modelRouter の設計参照である。
- Kim, G., Humble, J., Debois, P., & Willis, J. (2016). *The DevOps Handbook*. IT Revolution. ISBN 978-1942788003.
  - 第9章のクローズドフィードバック設計は、第三の道の *「継続的学習」* 章に示唆を受けている。

---

## 5. 関連学術研究（抜粋）

*本書は包括的な文献レビューを実施していない。以下は著者が読んだ代表的研究であり、読者は [GitHub Issues](https://github.com/baiyuan-tech/geo-whitepaper/issues) を通じて追加を提案されたい。*

### LLMハルシネーション検知

- Manakul, P., Liusie, A., & Gales, M. J. F. (2023). *SelfCheckGPT: Zero-Resource Black-Box Hallucination Detection for Generative Large Language Models*. EMNLP 2023.
- Min, S., Krishna, K., Lyu, X., et al. (2023). *FActScore: Fine-grained Atomic Evaluation of Factual Precision in Long Form Text Generation*. EMNLP 2023.

### ナレッジグラフとエンティティリンキング

- Pan, J. Z., Razniewski, S., Kalo, J.-C., et al. (2024). *Large Language Models and Knowledge Graphs: Opportunities and Challenges*. TGDK 2024.
- Hogan, A., Blomqvist, E., Cochez, M., et al. (2021). *Knowledge Graphs*. ACM Computing Surveys, 54(4).

### 多源 NLI

- Nie, Y., Chen, H., & Bansal, M. (2019). *Combining Fact Extraction and Verification with Neural Semantic Matching Networks*. AAAI 2019.
- Thorne, J., Vlachos, A., Christodoulopoulos, C., & Mittal, A. (2018). *FEVER: A Large-scale Dataset for Fact Extraction and VERification*. NAACL 2018.

### 構造化データと LLM

- Yang, J., Chen, H., Yan, Y., et al. (2024). *Exploring the Role of Structured Data in Large Language Model Fact Verification*. Preprint.

---

## 本書の引用方法

正式な引用形式は [`CITATION.cff`](../CITATION.cff) および [README の引用セクション](../README.md#引用方式) に記載されている。

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

**ナビゲーション**：[← 付録B：API仕様](./appendix-b-api.md) · [📖 目次](../README.md) · [付録D：図目録 →](./appendix-d-figures.md)
