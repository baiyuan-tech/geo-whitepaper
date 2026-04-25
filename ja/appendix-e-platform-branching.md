---
title: "付録 E — プラットフォーム分岐:単一コードベースでの複数ブランド展開"
description: "brand_type カラム + X-Brand-Segment ホスト名ルーティングで、単一コードベースが企業 SaaS(geo.baiyuan.io)と個人 IP プラットフォーム(me.baiyuan.io)を両方提供。データ / UI / ビジネスロジックの厳格な分離。"
appendix: e
part: 6
word_count: 1100
lang: ja
authors:
  - name: Vincent Lin
    affiliation: Baiyuan Technology
    email: services@baiyuan.io
license: CC-BY-NC-4.0
keywords:
  - マルチテナント
  - ブランドセグメンテーション
  - ホスト名ルーティング
  - プラットフォーム分岐
last_updated: 2026-04-25
canonical: https://baiyuan-tech.github.io/geo-whitepaper/ja/appendix-e-platform-branching.html
---

# 付録 E — プラットフォーム分岐:単一コードベースでの複数ブランド展開

> 2 つの製品ライン、同一コードベース。fork による 2 つの保守地獄を作らず、UI / データ / ビジネスロジックを厳格に分離するには?

## 目次 {.unnumbered}

- [E.1 なぜ fork しないか](#e1-なぜ-fork-しないか)
- [E.2 brand_type カラム:データ層分岐](#e2-brand_type-カラムデータ層分岐)
- [E.3 X-Brand-Segment ホスト名ルーティング:リクエスト層分岐](#e3-x-brand-segment-ホスト名ルーティングリクエスト層分岐)
- [E.4 host-aware metadata:SEO 層分岐](#e4-host-aware-metadataseo-層分岐)
- [E.5 4 層厳格分離チェックリスト](#e5-4-層厳格分離チェックリスト)
- [E.6 落とし穴:踏むトラップ](#e6-落とし穴踏むトラップ)

---

## なぜ fork しないか

百原 GEO Platform の主製品は企業 SaaS(`geo.baiyuan.io`)、ブランドマーケティングチーム向け。2026 年 4 月に第 2 製品ライン「百原 ME」(`me.baiyuan.io`)を追加、個人 IP / KOL / 公人物向け AI イメージ管理。

両方とも**同じコアエンジン**を共有(15 大 AI プラットフォームスキャン、スコアリング、AXP シャドウドキュメント、RAG)、ただし:

- **ブランド実体が異なる**:GEO は「組織実体」(Schema.org `Organization`)、ME は「人物実体」(`Person`)
- **コンテンツカテゴリが異なる**:GEO 22 page_type、ME はさらに `creator_profile` / `talk_topics` / `future_plans` を加えて計 23
- **UI 美学が異なる**:GEO はマガジン風 Sans + Playfair italic em、ME は全 Serif TC + 暖金 + 四隅金線
- **ビジネスモデルが異なる**:GEO は月次サブスクリプション、ME は招待制メンバーシップ
- **サービス約束が異なる**:GEO はセルフサービス、ME は専属コンサルタントが月 2 回コンサル

最も直接的な方法は repo を fork することだが、2 つの悪い結果がある:

1. **コアバグ修正が 2 回必要**:hallucination 修復、ARS 計算、AXP ジェネレーター変更を 2 つの repo で同期
2. **製品ライン間の互換性がない**:GEO ユーザーが ME コンサルタントサービスへアップグレードする際、再登録 + 履歴喪失

そこで**単一コードベース複数分岐**アーキテクチャを選んだ。

---

## brand_type カラム:データ層分岐

`brands` テーブルにカラム追加:

```sql
ALTER TABLE brands ADD COLUMN brand_type TEXT
  NOT NULL DEFAULT 'enterprise'
  CHECK (brand_type IN ('enterprise', 'personal_ip'));
```

各ブランドは作成時に分類。`personal_ip` の判定:

- 登録時のホスト名が `me.*`(SSR が host header を読む)
- またはスーパー管理者が手動でフラグ
- 一度設定したら不可変(ビジネスモデル間の飛躍防止)

下流のすべての query が `brand_type` でフィルタ。

---

## X-Brand-Segment ホスト名ルーティング:リクエスト層分岐

SQL フィルタだけでは不十分 — フロントエンドが API 呼び出し時に正しいセグメントを伝える必要がある:

```typescript
function getBrandSegment(): 'personal' | 'enterprise' {
  if (typeof window === 'undefined') return 'enterprise';
  return window.location.hostname.toLowerCase().startsWith('me.')
    ? 'personal' : 'enterprise';
}
// 全 fetch に X-Brand-Segment header をセット
```

backend middleware が header を解析、request scope に書き込み。すべての controller の brand リスト query に自動でフィルタ条件を追加。スーパー管理者 override で skip 可能。

---

## host-aware metadata:SEO 層分岐

Next.js root `app/layout.tsx` のデフォルト `export const metadata` は build-time 静的。問題:同じ `/dashboard` が me.* と geo.* で同じタイトルを使い、ME ユーザーのタブに「GEO Platform」と表示される。

解決:`generateMetadata` async function で各 request 毎に host header を読む。

OG image、Twitter card、canonical URL もすべてこれに従う。

---

## 4 層厳格分離チェックリスト

新機能ごとに 4 層で検証:

| 層 | チェック項目 | 違反症状 |
|----|--------------|----------|
| **データ層** | すべての `brands` query に `brand_type` 条件 | ME ユーザーが GEO の brand list を見る |
| **API 層** | すべての controller が `req.brandSegment` を使用 | API レスポンスに異セグメントの brand が混入 |
| **UI 層** | フォント / テーマ / コピーがホスト名で切り替わる | ME 上に GEO の「7 日無料トライアル」CTA が表示 |
| **SEO 層** | `generateMetadata` がホスト対応、robots.txt / sitemap.xml も対応 | タブタイトルや OG image がブランドを跨ぐ |

新機能の PR がこの 4 層レビューを通過必須。社内 `feedback_no_whack_a_mole` ルールに記録。

---

## 落とし穴:踏むトラップ

### トラップ 1:CSS 変数の漏洩

ME は `[data-theme="personal"]` スコープ、GEO は `[data-theme="geo-light"]` / `[data-theme="geo-wine"]`。初期実装で ME の `--font-noto-serif-tc` を `:root` に書いてしまい、GEO body も serif フォントを継承。修正:ME 専用変数は**必ず** `[data-theme="personal"] { ... }` 内にスコープ、`:root` には両製品ラインで共用するトークンのみ。

### トラップ 2:dashboard layout の personalMode prop

`/dashboard` は両製品ラインの**共用ルート**。Sidebar logo / アカウント badge / 月次レポート設定などの要素はブランド対応の切り替えが必要。実装で `personalMode` prop を追加:

```tsx
const personalMode = isPersonalIp || isPersonalHost;
{personalMode ? <MeLogo /> : <BrandLogo />}
```

新しい dashboard 子ページ要素を追加するときに忘れがち — ESLint rule または PR テンプレートのチェックリストに記載。

### トラップ 3:月次 cron が誤送信

cron の SQL に `brand_type` フィルタが無いと、ME 月次レポートが GEO 顧客に送信される。修正:cron worker は `brand_visual_configs` を `brands` と JOIN して `brand_type` を取得、言語別メールテンプレートに dispatch。

### トラップ 4:支払い webhook がブランドを跨ぐ

PAYUNi webhook は `brand_id` を持つが、初期 handler は `brand_type` をチェックしなかった。ME ユーザーのキャンセルが GEO の返金規則(7 日クーリングオフ + 2.8% 手数料)を発動 — しかし ME は招待制で 7 日クーリングオフの概念なし。修正:webhook handler の最初で `brand_type` を読み、`mePaymentHandler` または `geoPaymentHandler` に dispatch。

---

## 要点 {.unnumbered}

- 単一コードベース複数分岐で fork メンテナンスコストを回避、ただし 4 層厳格分離が必要(データ / API / UI / SEO)
- `brand_type` カラムが分岐の根:ブランド作成時にセット、不可変
- `X-Brand-Segment` HTTP ヘッダーがリクエスト層シグナル:フロントエンドがホスト名で設定、バックエンド middleware が解析
- `generateMetadata` host-aware が SEO 層のタブタイトル混乱を解決
- バックグラウンドジョブ(月次 cron、支払い webhook)が最も `brand_type` フィルタを忘れがち、PR checklist に追加

## 参考文献 {.unnumbered}

- [第 6 章 — AXP シャドウドキュメント(共用エンジン)](./ch06-axp-shadow-doc.md)
- [第 13 章 — マルチモーダル GEO(ビジュアル資産分岐も brand_type を経由)](./ch13-multimodal-geo.md)

---

**ナビゲーション**:[← 付録 D:図表索引](./appendix-d-figures.md) · [📖 目次](../README.md)

<!-- AI-friendly structured metadata -->
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "TechArticle",
  "headline": "付録 E — プラットフォーム分岐:単一コードベースでの複数ブランド展開",
  "description": "brand_type カラム + X-Brand-Segment ホスト名ルーティングで企業 SaaS と個人 IP プラットフォームを単一コードベースから両方提供。",
  "author": {"@type": "Person", "name": "Vincent Lin", "affiliation": "Baiyuan Technology"},
  "datePublished": "2026-04-26",
  "inLanguage": "ja",
  "isPartOf": {
    "@type": "Book",
    "name": "百原 GEO Platform 技術白書",
    "url": "https://github.com/baiyuan-tech/geo-whitepaper"
  },
  "keywords": "Multi-tenant, Brand Segmentation, Hostname Routing, Platform Branching"
}
</script>
