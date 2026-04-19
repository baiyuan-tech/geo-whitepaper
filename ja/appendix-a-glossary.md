---
title: "付録A — 用語集（完全版）"
description: "百元 GEO Platform ホワイトペーパーの用語完全集。GEO、AXP、Schema.org、RAG、クローズドループほか、初出章への参照付き。"
appendix: A
part: 6
lang: ja
authors:
  - name: Vincent Lin
    affiliation: 百元科技 (Baiyuan Technology)
license: CC-BY-NC-4.0
last_updated: 2026-04-18
---

# 付録A — 用語集（完全版）

アルファベット順。各エントリは **定義 + 初出章 + 関連章** を提供する。

## A

- **@id (Schema.org)** — JSON-LD 一意識別子文字列、`@graph` 内またはドキュメント間でエンティティ参照に使用。初出：[§6.7](./ch06-axp-shadow-doc.md#67-json-ld-flattening-pitfalls)。中核利用：[§7.3](./ch07-schema-org.md#73-three-layer-id-interlinking)。
- **AI ボット** — LLM学習または検索用にウェブコンテンツを取込むクローラの総称（GPTBot、ClaudeBot、PerplexityBot 等）。初出：[§6.3](./ch06-axp-shadow-doc.md#63-cloudflare-worker-injection)。
- **AI シテーション率** — 代表的インテントクエリのうちAIが能動的にブランドを言及する割合（0〜100%）。本書の中核指標。初出：[§1.2](./ch01-geo-era.md#12-ai-citation-rate-a-brand-new-and-life-or-death-metric)。詳述：[§3.2.1](./ch03-scoring-algorithm.md#321-citation-rate)。
- **aggregateRating** — Schema.org プロパティで集計レビュースコアを保持。当プラットフォームではGBPから同期。[§8.4](./ch08-gbp-integration.md#84-field-mapping-table)。
- **AXP (AI-ready eXchange Page)** — 百元造語。AIボット向けに提供される *「シャドウドキュメント」* — 純HTML + Schema.org JSON-LD + Markdown、人向けサイトから疎結合。詳解：[第6章](./ch06-axp-shadow-doc.md)。
- **アプリ層フィルタ** — アプリケーション層クエリレベルのフィルタリング、マルチテナント設計ではRLSと並ぶ第二防衛線。[§2.5](./ch02-system-overview.md#25-multi-tenant-data-isolation)。

## B

- **BullMQ** — Redisバックエンドの Node.js ジョブキューライブラリ、バックグラウンドタスク基盤の中核。[§2.4](./ch02-system-overview.md#24-technology-stack)。
- **ブランドエンティティ** — Schema.org Organization / LocalBusiness ノードを根とする構造化ブランドデータの総称。[第7章](./ch07-schema-org.md)。

## C

- **ChainPoll** — 同一主張を同一プロンプトで LLM に3回送信し多数決を取る検証手法。NLI信頼度 0.5〜0.8 の曖昧帯で使用。[§9.3](./ch09-closed-loop.md#93-primary-detection-nli-classification--chainpoll)。
- **CID (Customer ID)** — Google Maps 内部数値識別子、Places API 呼出無しでは Place ID に変換不可。[§7.7](./ch07-schema-org.md#77-gbp-url-parser)。
- **ClaimReview** — 主張が真か否かを注釈する Schema.org プロパティ。プラットフォームがハルシネーション訂正を公的宣言するために使用。[§9.6](./ch09-closed-loop.md#96-remediation-claimreview-generation-and-multi-path-injection)。
- **クローズドループ** — 検知 → 比較 → 判定 → ClaimReview生成 → AXP/RAG注入 → 再スキャン の循環。[第9章](./ch09-closed-loop.md)。
- **Cloudflare Workers** — AIボット UA 検知および AXP 注入に用いるエッジ実行環境。[§6.3](./ch06-axp-shadow-doc.md#63-cloudflare-worker-injection)。
- **Common Crawl** — 公開大規模ウェブスクレイプ、大多数のメジャーLLMの主要事前学習源。[§1.3](./ch01-geo-era.md#13-geo-is-not-an-extension-of-seo--it-is-an-independent-discipline)。
- **Consistency（一貫性）** — 7次元の一つ、他6次元のプラットフォーム跨ぎ標準偏差の逆数。[§3.2.7](./ch03-scoring-algorithm.md#327-consistency)。
- **Content Depth（コンテンツ深度）** — 7次元の一つ、ブランド言及に伴う記述の深さ。[§3.2.6](./ch03-scoring-algorithm.md#326-content-depth)。
- **Core Web Vitals** — Google のページ体験指標（LCP、FID、CLS）。SEO では重要だが GEO ではほぼ無関係。[§1.3](./ch01-geo-era.md#13-geo-is-not-an-extension-of-seo--it-is-an-independent-discipline)。

## D

- **DIRECT_MODEL_ID_MAP** — アグリゲータモデルIDとベンダネイティブモデルIDの変換表。[§5.3.1](./ch05-multi-provider-routing.md#531-model-ids-differ-between-aggregator-and-native-layers)。
- **ドッグフーディング** — 自社プロダクトの社内利用。事例のBrand Eが該当。[§11.1](./ch11-case-studies.md#111-brand-portraits-anonymized)。

## E

- **entailment / contradiction / neutral / opinion** — NLI 4値分類出力。`contradiction` のみハルシネーション認定、`neutral` は明示的にハルシネーションではない。[§9.3](./ch09-closed-loop.md#93-primary-detection-nli-classification--chainpoll)。
- **extraParams** — ベンダ固有の必須APIパラメータ、例：Qwen3 は `enable_thinking: false` を要求。[§5.3.2](./ch05-multi-provider-routing.md#532-extraparams-divergence)。

## F

- **フィンガープリント** — AXP または Schema.org アーティファクトのハッシュ、AIが更新内容を取込んだかの判定に使用。[第9章](./ch09-closed-loop.md)関連。
- **FTID (Feature ID)** — Google Maps 内部ロケーション識別子。[§7.7](./ch07-schema-org.md#77-gbp-url-parser)。

## G

- **GBP (Google Business Profile)** — Google のビジネス掲載情報、実店舗の信頼源。[第8章](./ch08-gbp-integration.md)。
- **GEO (Generative Engine Optimization)** — 生成AI応答内でのブランドの言及頻度・位置・記述正確性を向上させる学問と実践。[第1章](./ch01-geo-era.md)。
- **GPTBot** — OpenAI のウェブクローラ、ChatGPT 学習コーパス収集用。[§6.4](./ch06-axp-shadow-doc.md#64-ai-bot-ua-list-and-detection-strategy)。
- **Ground Truth (GT)** — 事実の地盤真実。本書では候補主張比較用の信頼参照セット（ライブスクレイプされた公式サイト / RAG / 手動 GT の3階層）。[§9.3](./ch09-closed-loop.md#93-primary-detection-nli-classification--chainpoll)。

## H

- **ハルシネーション** — 現実と一致しないAI生成コンテンツ。本書では5分類：エンティティ誤認、事業事実誤り、業種誤分類、時間/地理誤り、属性誤り。[§9.2](./ch09-closed-loop.md#92-five-types-of-ai-hallucination)。

## I

- **industry_code** — プラットフォームが使用する25業種分類 enum。[§7.2](./ch07-schema-org.md#72-industry-specialized-type-across-25-categories)。
- **インテントクエリ** — ブランド業種に基づき動的生成される代表的ユーザ質問。[§2.2.1](./ch02-system-overview.md#221-scan-module--monitoring)。
- **is_physical** — ブランドエンティティに付与する実店舗 vs オンライン事業フラグ、Schema.org フィールド重みを駆動。[§7.4](./ch07-schema-org.md#74-physical-vs-online-divergent-field-weights)。
- **isStale** — Stale Carry-Forward 下でプラットフォームスコアが履歴持越値であることを示すマーカー。[§4.3](./ch04-stale-carry-forward.md#43-stale-carry-forward-design)。

## J

- **JSON-LD** — JavaScript Object Notation for Linked Data、Schema.org の主流シリアライゼーション形式。[§6.2](./ch06-axp-shadow-doc.md#62-axp-the-structure-of-a-shadow-document), [第7章](./ch07-schema-org.md)。

## K

- **kgmid (Knowledge Graph Machine ID)** — Google ナレッジグラフ・エンティティ識別子。`sameAs` で使用。
- **ナレッジグラフ** — エンティティと関係性の構造化ネットワーク。Schema.org の `@id` 相互リンクで構築。[§7.3](./ch07-schema-org.md#73-three-layer-id-interlinking)。

## L

- **Layer 1 センチネルスキャン** — 4時間サイクルの軽量検証、検索型AIを対象。[§9.7](./ch09-closed-loop.md#97-two-tier-rescan-loop)。
- **Layer 2 フルスキャン** — 24時間サイクルの深掘スキャン、7次元スコア + フィンガープリント比較。[§9.7](./ch09-closed-loop.md#97-two-tier-rescan-loop)。
- **LLM Wiki** — 百元造語。LLMによって能動的に維持される意味知識層、RAGアーキテクチャ内L1に位置。[§9.5](./ch09-closed-loop.md#95-l1-llm-wiki-an-active-semantic-layer)。
- **LocalBusiness** — 実店舗に汎用される Schema.org `@type`。[§7.2](./ch07-schema-org.md#72-industry-specialized-type-across-25-categories)。

## M

- **modelRouter** — 自社開発のマルチAIプロバイダルーティングサービス、プライマリ/フォールバック2経路抽象を提供。[第5章](./ch05-multi-provider-routing.md)。

## N

- **NLI（自然言語推論、Natural Language Inference）** — 2パッセージ間の含意関係を分類するタスク、当方のハルシネーション検知主機構。[§9.3](./ch09-closed-loop.md#93-primary-detection-nli-classification--chainpoll)。

## O

- **OpenAI互換API** — 複数AIベンダを OpenAI SDK インタフェースでラップするアグリゲータエンドポイント。[§5.2](./ch05-multi-provider-routing.md#52-modelrouter-architecture)。

## P

- **pgvector** — PostgreSQL のベクトル埋込・類似検索拡張。[§2.4](./ch02-system-overview.md#24-technology-stack)。
- **フェーズ・ベースライン試験** — 固定20問セットを異なる時点で再試問する縦断試験。[第10章](./ch10-phase-baseline.md)。
- **Place ID** — Google Maps 一意ロケーション識別子、Places API の主キー。[§7.7](./ch07-schema-org.md#77-gbp-url-parser)。
- **Position Quality（位置品質）** — 7次元の一つ、AI応答内でブランドが出現する位置の加重平均。[§3.2.2](./ch03-scoring-algorithm.md#322-position-quality)。
- **Platform Breadth（プラットフォーム広度）** — 7次元の一つ、少なくとも1回言及したプラットフォーム数 ÷ 監視対象総プラットフォーム数。[§3.2.4](./ch03-scoring-algorithm.md#324-platform-breadth)。

## Q

- **Query Coverage（クエリ網羅）** — 7次元の一つ、ブランドが言及されるインテントクエリ種別の多様性。[§3.2.3](./ch03-scoring-algorithm.md#323-query-coverage)。
- **QPM (Queries Per Minute)** — API クォータ単位、GBP API デフォルトは 300 QPM。[§8.5](./ch08-gbp-integration.md#85-sync-frequency-and-quota)。

## R

- **RAG（検索拡張生成、Retrieval-Augmented Generation）** — 検索で拡張する生成。当プラットフォームは中央共有 RAG SaaS アーキテクチャを採用。[§9.4](./ch09-closed-loop.md#94-central-shared-rag-saas-infrastructure)。
- **RLS (Row-Level Security)** — PostgreSQL の行レベルアクセス制御、マルチテナント隔離の第一防衛線。[§2.5](./ch02-system-overview.md#25-multi-tenant-data-isolation)。

## S

- **sameAs** — 他プラットフォーム（Wikipedia、Wikidata、LinkedIn、GBP）上の同一エンティティを指す Schema.org プロパティ。[§7.3](./ch07-schema-org.md#73-three-layer-id-interlinking)。
- **Schema.org** — Google、Bing、Yahoo、Yandex が共同推進する構造化データ語彙。[第7章](./ch07-schema-org.md)。
- **Sentiment Score（センチメントスコア）** — 7次元の一つ、言及の集計トーン（肯定 / 中立 / 否定）。[§3.2.5](./ch03-scoring-algorithm.md#325-sentiment-score)。
- **センチネルスキャン** — Layer 1 参照。
- **SERP (Search Engine Results Page)** — 古典的 SEO の中核戦場。[§1.1](./ch01-geo-era.md#11-generative-search-has-taken-over-user-habit)。
- **サイトマップ** — クローラのコンテンツ発見を助けるXML URLリスト。[§6.6](./ch06-axp-shadow-doc.md#66-automatic-sitemap-generation)。
- **スモークテスト** — サービス起動時にフォールバック経路上で実行される最小限検証、実到達可能性を保証。[§5.6.1](./ch05-multi-provider-routing.md#561-startup-smoke-test)。
- **Stale Carry-Forward** — 百元造語。プラットフォームのスキャンが100%失敗したとき、履歴から直近成功値を持ち越し stale とマーク、信号連続性を維持する。[第4章](./ch04-stale-carry-forward.md)。

## T

- **Tenant ID (X-Tenant-ID)** — マルチテナント HTTP ヘッダ、中央共有 RAG のテナントフィルタに使用。[§9.4](./ch09-closed-loop.md#94-central-shared-rag-saas-infrastructure)。

## U

- **UA (User-Agent)** — HTTP リクエストヘッダ、AXP 注入が AIボット vs 人間を判定する基礎。[§6.4](./ch06-axp-shadow-doc.md#64-ai-bot-ua-list-and-detection-strategy)。

## W

- **Webhook** — サーバ発イベント通知。GBP API は**提供しない**。[§8.6](./ch08-gbp-integration.md#86-remedy-for-no-webhook)。
- **Wikidata** — 公開ナレッジグラフ、Schema.org `sameAs` の重要ターゲット。[§7.3](./ch07-schema-org.md#73-three-layer-id-interlinking)。
- **ウィザード** — 新規ブランド向けの線形7ステップ Schema.org オンボーディングフロー。[§7.6](./ch07-schema-org.md#76-dual-entry-points-wizard--edit)。

---

## 標準参照表

| 標準 | 役割 | 章 |
|----------|------|----------|
| [Schema.org](https://schema.org/) | 構造化データ語彙 | [第6, 7, 9章](./ch07-schema-org.md) |
| [JSON-LD 1.1](https://www.w3.org/TR/json-ld11/) | Linked Data シリアライゼーション | [第6, 7章](./ch07-schema-org.md) |
| [OpenGraph](https://ogp.me/) | リンクプレビュー用メタタグ | [第6章](./ch06-axp-shadow-doc.md) |
| [robots.txt](https://www.rfc-editor.org/rfc/rfc9309.html) | Robots Exclusion Protocol | [第6章](./ch06-axp-shadow-doc.md) |
| [sitemaps.xml](https://www.sitemaps.org/protocol.html) | Sitemap Protocol 0.9 | [§6.6](./ch06-axp-shadow-doc.md#66-automatic-sitemap-generation) |

---

**ナビゲーション**：[← 第12章：限界](./ch12-limitations.md) · [📖 目次](../README.md) · [付録B：公開API仕様 →](./appendix-b-api.md)
