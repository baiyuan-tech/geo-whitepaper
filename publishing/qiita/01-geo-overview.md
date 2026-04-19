<!--
Qiita 投稿設定
-------------
Title: 生成AI時代の「検索 SEO」の次——GEO（Generative Engine Optimization）の技術体系
Tags: GEO, AI, LLM, SEO, Architecture
Private: false

投稿URL（投稿後に記入）: https://qiita.com/...
-->

# 生成AI時代の「検索 SEO」の次——GEO（Generative Engine Optimization）の技術体系

## TL;DR

- ChatGPT、Claude、Perplexity、Gemini など生成AIの回答内で**ブランドが言及される確率と正確性**を工学的に最適化する分野を **GEO**（Generative Engine Optimization）と呼ぶ。
- SEO のキーワード順位・被リンクとは**異なる評価次元**が必要であり、単一指標では測れない。
- 本稿では 7 次元スコアリング、シャドウドキュメント（AXP）、Schema.org 三層相互リンク、NLI + ChainPoll によるハルシネーション検知の 4 本柱を手短に紹介する。
- 公開ホワイトペーパー（12 章 + 4 付録、CC BY-NC 4.0）からの抜粋：<https://github.com/baiyuan-tech/geo-whitepaper>

---

## 1. なぜ今 GEO なのか

SEO が数十年かけて構築してきた「クリック経由の検索トラフィック」は、生成AIの登場で地殻変動している。

- ChatGPT：約 25 億クエリ／日（2025 Q3 公式開示）
- Perplexity：月間 7.8 億クエリ
- Google AI Overview：2024 年ロールアウト後、従来 SERP の CTR が高インテント領域で 34〜60% 低下

ユーザが *「B2B の CRM おすすめは？」* と聞いたとき、AI は数社のブランド名を含む段落を生成する。**その段落に載らないブランドは、意思決定経路から事実上消える**。これは「検索順位 5 位以下」とは質的に異なる問題である。なぜなら AI の回答には *クリック先* が存在しないからだ。

GEO は SEO の後継ではなく、**並列の工学問題**である：

| 次元 | SEO | GEO |
|------|-----|-----|
| 評価対象 | SERP 内のキーワード順位 | AI 回答テキスト内の言及 |
| 測定単位 | CTR、被リンク、Core Web Vitals | シテーション率、ポジション、センチメント |
| 失敗モード | 順位下落 | ハルシネーション、モデルバージョンドリフト |
| 修復手段 | コンテンツ・内部リンク | 構造化エンティティ、ClaimReview、RAG 注入 |

---

## 2. 7 次元スコアリング：単一指標の罠を避ける

*「AI シテーション率」* の単一指標だけでは、「2 プラットフォームで末尾に軽く言及」と「7 プラットフォームで冒頭に詳細に言及」が同じ点数になりうる。これを避けるため 7 つの直交次元に分解する：

| 次元 | 測定対象 |
|------|----------|
| **Citation Rate** | 代表インテントクエリ中、ブランドが言及される割合 |
| **Position Quality** | 回答の冒頭／中盤／末尾のどこで言及されるか |
| **Query Coverage** | インテント種別の多様性（ベスト、比較、How-to、推薦） |
| **Platform Breadth** | 監視 15 プラットフォーム中、言及するプラットフォームの割合 |
| **Sentiment** | 各言及のトーン方向 |
| **Content Depth** | 記述の長さと事実密度 |
| **Consistency** | クロスプラットフォーム標準偏差（AI 間合意度の逆数） |

**重み値は意図的に非公開**。これは PageRank が歴史的に重みを公開しなかった思想と同じで、顧客に *指標ゲーミング* ではなく *実質最適化* をさせるためである。

---

## 3. AXP：AIボット専用シャドウドキュメント

現代の顧客サイトは **人間向け** に作られている：クライアントサイド JS、Cookie バナー、トラッカー SDK、アニメーション UI。AIボット（GPTBot、ClaudeBot、PerplexityBot 等 25 種）はこれを清潔にパースできない。

解は **AXP**（AI-ready eXchange Page）：

```
同一 URL
  ├─ 人間リクエスト  → 既存の CSR アプリ（変更なし）
  └─ AIボット（UA 検出） → 純HTML + JSON-LD + Markdown の清潔な文書
```

Cloudflare Worker がエッジで User-Agent を検出し、AIボットには **3 層の清潔な文書**（pure HTML + Schema.org JSON-LD + RAG 向け Markdown）に差し替える。顧客サイトのコードは一切変更しない。

実地観察：AXP 展開 2 週間以内に AIボットトラフィックが **3〜5 倍** に増加、シテーション率改善は **2〜3 週間遅れ** で追従する（ボットが取り込んでからコーパス統合までのラグ）。

---

## 4. Schema.org 三層相互リンク

Schema.org は *「リッチリザルト用のタグ」* として扱われがちだが、GEO では **AI ナレッジグラフのエンティティ同定層** として使う：

```
Organization (brand)
  └─ hasService → Service
       └─ providedBy → Person (Physician / Attorney / ...)
  └─ sameAs → [Wikipedia, Wikidata, LinkedIn, Google Business Profile]
```

**@id** による三層相互リンク + **sameAs** で外部権威ノードへの参照を構築することで、AI がエンティティを「同一物」と認識する確率を上げる。

25 業種の実地観察：Schema.org 完成度 **60% が認識しきい値**、80% 超は収益逓減。「完成度 100% 追求」の顧客不安は大半が無駄で、80% + コンテンツ品質のほうが 100% 機械的充填より強い。

---

## 5. NLI + ChainPoll：ハルシネーションの閉ループ検知

AI が *「この会社は 2018 年創業」* と誤った情報を出したとき、単純な *「Ground Truth と文字列比較」* では：

1. GT カバレッジが低い（全事実を手動でキーバリュー化は不可能）
2. 完全一致は厳しすぎる（「2018 年設立」「2018 年創業」「創業 7 年」全て正しい）

解として **NLI（Natural Language Inference）三値分類**を主機構に置く：

| クラス | アクション |
|--------|------------|
| `entailment` | 情報源が主張を支持 → 通過 |
| `contradiction` | 情報源が主張と矛盾 → ハルシネーション認定 |
| `neutral` | 肯定も否定もしない → **判定しない**（これが重要） |
| `opinion` | 主観的価値判断 → スキップ |

核心原理：**`neutral` はハルシネーションではない**。「情報源に記載がない」と「事実が誤っている」を区別しないと、修復サイクル自体が汚染される（AI が「訂正」された捏造内容を学習してしまう）。

信頼度 0.5〜0.8 の曖昧帯には **ChainPoll**（同一プロンプトで LLM を 3 回呼び多数決）を発動してノイズを抑える。

検出後は **ClaimReview**（Schema.org 標準）を生成し、AXP + RAG + GBP LocalPosts の 3 経路に注入。2 層スキャン（センチネル 4h + フル 24h）で収束を検証する。

---

## 6. これから GEO を始めるエンジニアへ

優先順位の推奨：

1. **AXP 導入**（1 週間）— Cloudflare Worker で UA 検出、既存サイトを触らずに AIボット経路を分岐
2. **Schema.org 完成度 60% 超え**（2〜4 週間）— Organization + Service + Person の三層、sameAs に Wikipedia/Wikidata/LinkedIn
3. **7 次元モニタリング**（運用開始）— 15 プラットフォームに対する定期スキャン、どの次元で弱いかを可視化
4. **ハルシネーション検知**（問題発生後）— NLI で分類、contradiction のみ修復

一気に全部やる必要はない。まず AXP で AIボットに *見てもらえる状態* を作ることが最重要。

---

## 参考

- **完全版ホワイトペーパー**（12 章 + 4 付録、CC BY-NC 4.0）：<https://github.com/baiyuan-tech/geo-whitepaper>
- 日本語版：<https://github.com/baiyuan-tech/geo-whitepaper/tree/main/ja>
- 英語版：<https://github.com/baiyuan-tech/geo-whitepaper/tree/main/en>
- PDF ダウンロード：<https://github.com/baiyuan-tech/geo-whitepaper/releases/latest>

誤記・質問・追加提案は [GitHub Issues](https://github.com/baiyuan-tech/geo-whitepaper/issues)（テンプレート提供）まで。

---

*本稿は百元科技（Baiyuan Technology）の技術ホワイトペーパーからの抜粋である。SaaS プロダクト：<https://geo.baiyuan.io>*
