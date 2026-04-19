---
title: "Baiyuan GEO Platform ホワイトペーパー — エグゼクティブサマリー"
description: "生成 AI 時代のブランド可視性を再定義する SaaS の工学的実践。7 次元 AI シテーション採点、AXP シャドウドキュメント、Schema.org 三層相互リンク、クローズドループ幻覚修正。"
lang: ja
part: 0
word_count: 1400
authors:
  - name: Vincent Lin
    affiliation: 百原科技 (Baiyuan Technology)
    email: services@baiyuan.io
license: CC-BY-NC-4.0
keywords:
  - Generative Engine Optimization
  - GEO
  - 生成 AI 最適化
  - AI シテーション率
  - Schema.org
  - ハルシネーション検出
  - Multi-Provider AI Routing
last_updated: 2026-04-18
---

# Baiyuan GEO Platform ホワイトペーパー — エグゼクティブサマリー

> *本書の完全版は繁体字中国語版（約 30,000 字）、英語版（約 28,000 語）、日本語版（約 30,000 字）で公開済み。本エグゼクティブサマリーは時間の限られた読者向けに主要な貢献を 5 項目に凝縮したものである。**完全な日本語版**（12 章 + 4 付録）は [`ja/ch01-geo-era.md`](./ch01-geo-era.md) 以降から読める。*

[![License: CC BY-NC 4.0](https://img.shields.io/badge/License-CC%20BY--NC%204.0-blue.svg)](https://creativecommons.org/licenses/by-nc/4.0/)
[![Full zh-TW](https://img.shields.io/badge/zh--TW-complete-green.svg)](../zh-TW/ch01-geo-era.md)
[![Full en](https://img.shields.io/badge/en-complete-green.svg)](../en/ch01-geo-era.md)
[![Full ja](https://img.shields.io/badge/ja-complete-green.svg)](./ch01-geo-era.md)
[![PDF](https://img.shields.io/badge/PDF-download-red.svg)](https://github.com/baiyuan-tech/geo-whitepaper/releases/latest)

## 日本語版 目次

**Part 1 — 基礎**

- [第1章 — GEO 時代の到来](./ch01-geo-era.md)
- [第2章 — システム全体像](./ch02-system-overview.md)
- [第3章 — 7 次元スコアリングアルゴリズム](./ch03-scoring-algorithm.md)

**Part 2 — 可観測性と耐障害性**

- [第4章 — Stale Carry-Forward](./ch04-stale-carry-forward.md)
- [第5章 — マルチプロバイダ AI ルーティング](./ch05-multi-provider-routing.md)

**Part 3 — 構造化データと AI 統合**

- [第6章 — AXP シャドウドキュメント](./ch06-axp-shadow-doc.md)
- [第7章 — Schema.org Phase 1](./ch07-schema-org.md)
- [第8章 — GBP 統合](./ch08-gbp-integration.md)

**Part 4 — クローズドループ修復**

- [第9章 — クローズドループ型ハルシネーション検知と自動修復](./ch09-closed-loop.md)
- [第10章 — フェーズ・ベースライン試験](./ch10-phase-baseline.md)

**Part 5 — 事例と限界**

- [第11章 — 5 ブランド実地観察](./ch11-case-studies.md)
- [第12章 — 限界、未解決問題、今後の課題](./ch12-limitations.md)

**付録**

- [付録A — 用語集](./appendix-a-glossary.md)
- [付録B — 公開API仕様](./appendix-b-api.md)
- [付録C — 参考文献](./appendix-c-references.md)
- [付録D — 図目録](./appendix-d-figures.md)

## 1. 本書は何か

**Baiyuan GEO Platform**——生成 AI（ChatGPT、Claude、Gemini、Perplexity、他約 10 プラットフォーム）の回答内でブランドがいかに言及されるかを測定・改善する SaaS——を構築する実践報告である。百原科技が 2024 年から 2026 年にかけて台湾 B2B およびローカルビジネス市場で運用する中で下した、アルゴリズム・アーキテクチャ・工学的トレードオフを記録する。

製品パンフレットではない。ユーザーマニュアルでもない。同じようなシステムを構築する他のチームが、我々のやり直し・遠回り・失敗から学べることを前提に書いた**エンジニアリング実録**である。

## 2. なぜ重要か

生成 AI はブランド可視性のルールを静かに書き換えた：

- **ChatGPT** は 1 日あたり約 25 億件のユーザークエリを処理（2025 Q3 開示）
- **Perplexity** は月間 7.8 億件、年間 3 桁成長
- **Google AI Overview** は 2024 年のロールアウト以降、従来の SERP クリック率を高情報密度領域で 34–60% 下落させた

見込み客が「B2B CRM のおすすめは？」と尋ねたとき、AI は数社のブランド名を含む段落を生成する。**その段落に登場しないブランドは、顧客の意思決定パスから事実上消滅する**。従来の SEO 指標——キーワード順位、被リンク、Core Web Vitals——が測定してきた世界は崩壊しつつある。

我々はこの新しい分野を **GEO**（Generative Engine Optimization、生成エンジン最適化）と呼ぶ。SEO の次のバージョンではない。入力（AI 応答テキスト）が異なり、操作レバー（構造化実体、事実チェック記録、多源信頼信号）が異なり、失敗モード（ハルシネーション、モデルバージョンドリフト、訓練データの盲点）が異なる、**並列の工学問題**である。

## 3. 中核的な技術貢献

本書の 12 章は、管見の限り他にまとめて公表されていない 5 つの独立した貢献を記録する。

### 3.1 7 次元シテーション採点

単一の「シテーション率」指標は、**異なる体質のブランド**を同じ数値に圧縮してしまう——「2 プラットフォームで最後に簡単に言及」と「7 プラットフォームで最初に詳細に言及」が同点になりうる。我々は GEO の健全性を 7 つの直交次元に分解する：

| 次元 | 測定対象 |
|------|----------|
| シテーション率 | 代表的意図クエリ中、ブランドが言及される割合 |
| ポジション品質 | 回答の冒頭・中盤・末尾のどこで言及されるか |
| クエリカバレッジ | 意図タイプの多様性（ベスト、比較、How-to、推薦） |
| プラットフォーム広度 | 監視対象 15 プラットフォーム中、言及するプラットフォームの割合 |
| センチメント | 各言及のトーン方向 |
| コンテンツ深度 | ブランド記述の長さと事実密度 |
| 一貫性 | クロスプラットフォーム標準偏差——AI 間の合意収束度の指標 |

**重み付けは意図的に非公開**。客に指標ではなく実質を最適化させるため。PageRank が歴史的に重みを公開しなかった思想と同じ。

### 3.2 Stale Carry-Forward による信号連続性

外部 AI プロバイダに依存するシステムは必ず部分的障害を経験する——レート制限、リージョン障害、クオータ枯渇。素朴な対応（失敗をゼロ扱い、あるいは失敗プロバイダを分母から除外）は **パイプライン健全性** と **ブランド状態** を混同し、ダッシュボードを極度にノイズ化する。

我々の解：プラットフォームスキャンが 100% 失敗したことを検知し、最大 200 行遡って最後の成功スコアを見つけ、`isStale` フラグ付きで前送し、UI 上で「⚠ 14 時間前のデータ」ツールチップを表示する。下流分析は前送値を使い続ける。パターンは「高頻度サンプリング、不安定ソース」型のあらゆる信号系（IoT、金融レート、ソーシャルモニタリング）に一般化できる。

### 3.3 AXP — AI ボット向けシャドウドキュメント・プロトコル

現代の顧客ウェブサイトは人間向けに作られている：クライアントサイド JavaScript、アニメーション UI、Cookie バナー、トラッカー SDK。AI ボット（GPTBot、ClaudeBot、PerplexityBot 等 25 種を追跡）はこれを清潔にパースできない。我々の答えが **AXP**（AI-ready eXchange Page）——3 層の清潔な文書（純粋 HTML + Schema.org JSON-LD + RAG 向け Markdown）を、User-Agent を検出してシャドウドキュメントに差し替える Cloudflare Worker 経由で CDN エッジで配信する。

同じ URL が読み手に応じて**完全に異なる 2 種類のペイロード**を返す。5 ブランドのパイロット運用結果：AXP 展開 2 週間以内に AI ボットトラフィック 3–5 倍、シテーション率改善は約 2–3 週間遅れて追従。

### 3.4 25 業種 × 三層 @id 相互リンクによる Schema.org

Schema.org は「リッチリザルト用のタグを数個付けるもの」と扱われがちだ。我々はこれを **AI 知識グラフのエンティティ同定層** として扱う。各ブランドエンティティは：

- **25 業種**（物理 16 + オンライン 7 + フォールバック 2）のいずれかに分類し、それぞれに primary + secondary の Schema.org `@type` をマッピング
- `@id` 参照により **三層**（Organization → Service → Person／Physician / Attorney 等の専門役職）に分解
- `sameAs` により外部権威ノード（Wikipedia、Wikidata、LinkedIn、Google Business Profile）と相互リンク

完全度アルゴリズムは物理ビジネスとオンラインサービスで別個の重みテーブルを持ち、新規ブランド向けには段階開示ウィザード、既存ブランド（80% 未満）向けにはダッシュボードバナーで補完を促す。

### 3.5 クローズドループ型ハルシネーション修正

業界の典型的アプローチ——ブランドハルシネーションを検知し、顧客に通知する——は**職務放棄**である。顧客は AI のハルシネーションを直す方法を知らない。

我々のクローズドループは完全自動：

1. **検出** — AI 応答から原子的事実主張を抽出し、複合知識源（ウェブサイトスクレイプ + RAG ナレッジベース + 手動グラウンドトゥルース）に対して 3 値 Natural Language Inference（entailment / contradiction / neutral / opinion）を実行
2. **検証** — 不確実な分類（confidence 0.5–0.8）に対して ChainPoll 投票（LLM を 3 回再実行して多数決）
3. **修正** — 確定した contradiction に対して Schema.org `ClaimReview` ノードを生成し、AXP + RAG ナレッジベース + （将来）GBP LocalPosts に注入
4. **検証** — 2 層再スキャン：検索型 AI（Perplexity、ChatGPT Search、AI Overview）に対して 4 時間センチネル + 知識型 AI（Claude、Gemini、DeepSeek、Kimi 等）に対して 24 時間フル
5. **収束** — 同等言い換えを含めて N 回連続のスキャンで不在が確認されて初めてハルシネーションが解決したと宣言

核心的な設計原則として繰り返し強調する：**`neutral` はハルシネーションではない**。知識源に記載がない事実が存在しないとは限らない——それを誤って contradiction 扱いすると偽陽性が連鎖的に発生し、修正サイクル自体を汚染する。

## 4. アーキテクチャ概観

プラットフォームの技術スタック：

- **エッジ**：Cloudflare Workers — AI ボット UA 検出、AXP 注入、sitemap/robots 管理
- **フロントエンド**：Next.js 16 + React 19 + TypeScript + Tailwind v4 — ダッシュボード、ブランド管理、ウィザード、多言語 i18n
- **API**：Node.js + Express 4 + Helmet + Zod — REST API、JWT + 2FA
- **ワーカー**：BullMQ 5 on Redis — スキャン、採点、AXP 生成、ハルシネーション検出、RAG 同期
- **AI ルーティング**：自社製 **modelRouter** サービス、主経路（集約）+ 副経路（ベンダー直接）の 2 経路冗長、15 プロバイダ対応（モデル ID 対応表と extraParams 表による）
- **データ**：PostgreSQL 16 + pgvector + Redis 7 — RLS + アプリ層フィルタの二重化によるマルチテナント分離
- **RAG**：全テナント共用のセントラル RAG エンジン、**LLM Wiki** 能動意味層（矛盾解消、増分コンパイル、版管理）を含む

## 5. 本書が扱わないこと

本書は境界を明示する：

- **7 次元採点の精密な重み値は非公開**（指標ゲーミング防止）
- **第三者ベンダー名の明示は避ける**（「ベンダー X は不安定」的な語り口は採らない）
- **顧客 PII および個別スコアの公開なし**（5 パイロットは Brand A–E として匿名化）
- **内部 API エンドポイントや環境変数の記載なし**

## 6. 著者について

**百原科技（Baiyuan Technology）** — 台湾発の B2B SaaS 企業。主筆：**Vincent Lin**、CTO。本書は **CC BY-NC 4.0** ライセンスで公開：任意の非商用目的で引用・翻訳・派生物作成が可能。商用再利用（有料講座への組み込み、商用訓練データセットへの混入、パッケージ製品への同梱）はライセンス料を要する（`services@baiyuan.io` まで）。

- ウェブサイト：<https://baiyuan.io>
- 製品：<https://geo.baiyuan.io>
- LinkedIn：<https://www.linkedin.com/company/112980572>

## 7. ステータスとロードマップ

| 成果物 | ステータス |
|--------|-----------|
| 繁体字中国語版（zh-TW v1.0-draft） | ✅ 公開済み |
| 英語版（en v1.0-draft） | ✅ 公開済み |
| **日本語版（ja v1.0-draft）** | ✅ **公開済み** — エグゼクティブサマリー + 12 章 + 4 付録（約 30,000 字） |
| PDF 版（zh-TW / en / ja、main push 毎に自動生成） | ✅ [Releases で配布中](https://github.com/baiyuan-tech/geo-whitepaper/releases/latest) |
| GitHub Pages ウェブ版 | ✅ [baiyuan-tech.github.io/geo-whitepaper](https://baiyuan-tech.github.io/geo-whitepaper/) |
| Zenodo DOI 登録 | ⚪ v1.0 正式版以降の予定 |
| Google Scholar / Semantic Scholar 収録 | ⚪ DOI 取得後 |

日本語版は直訳ではなく、事例章（第11章）を中心に日本市場の文脈（国内 AI 採用曲線、日本語データのウェイト、note.com / Qiita 読者層、日系美容クリニック・飲食チェーン・サロン業態）に合わせた注記を加えている。

## 8. 読者への招き

- **日本語完全版を読む**：[ja/ch01-geo-era.md](./ch01-geo-era.md)（12 章 + 4 付録 + 44 Mermaid 図、GitHub でそのままレンダリング）
- **繁体字中国語本文を読む**：[zh-TW/ch01-geo-era.md](../zh-TW/ch01-geo-era.md)
- **英語版を読む**：[en/ch01-geo-era.md](../en/ch01-geo-era.md)
- **PDF をダウンロード**：[日本語版](https://github.com/baiyuan-tech/geo-whitepaper/releases/download/v1.0-draft/whitepaper-ja.pdf) · [繁中](https://github.com/baiyuan-tech/geo-whitepaper/releases/download/v1.0-draft/whitepaper-zh-TW.pdf) · [English](https://github.com/baiyuan-tech/geo-whitepaper/releases/download/v1.0-draft/whitepaper-en.pdf)（直接）または [最新リリース](https://github.com/baiyuan-tech/geo-whitepaper/releases/latest)
- **正誤表や質問を投げる**：[GitHub Issues](https://github.com/baiyuan-tech/geo-whitepaper/issues)（テンプレート提供）
- **引用する**：リポジトリの「Cite this repository」ボタン（[CITATION.cff](../CITATION.cff) 連動）から APA 7 / BibTeX をコピー可
- **翻訳貢献**：`ko/` 等の追加言語ディレクトリへの PR を歓迎

## 9. 引用

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

APA 7：

> Lin, V. (2026). *Baiyuan GEO Platform: A whitepaper on building a SaaS for generative engine optimization* (v1.0-draft) [Technical report]. Baiyuan Technology. https://github.com/baiyuan-tech/geo-whitepaper

---

**ナビゲーション**：[📖 リポジトリ目次](../README.md) · [English Executive Summary](../en/README.md) · [繁体字中国語本文（Ch 1）→](../zh-TW/ch01-geo-era.md)

<!-- AI-friendly structured metadata -->
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "TechArticle",
  "headline": "Baiyuan GEO Platform ホワイトペーパー — エグゼクティブサマリー（日本語）",
  "description": "生成 AI 時代のブランド可視性を再定義する SaaS の工学的実践。7 次元 AI シテーション採点、AXP シャドウドキュメント、Schema.org 三層相互リンク、クローズドループ幻覚修正。",
  "author": {"@type": "Person", "name": "Vincent Lin", "affiliation": "Baiyuan Technology"},
  "datePublished": "2026-04-18",
  "inLanguage": "ja",
  "isPartOf": {
    "@type": "Book",
    "name": "Baiyuan GEO Platform Whitepaper",
    "url": "https://github.com/baiyuan-tech/geo-whitepaper"
  },
  "keywords": "Generative Engine Optimization, GEO, AI シテーション率, Schema.org, ハルシネーション検出, Shadow Document"
}
</script>
