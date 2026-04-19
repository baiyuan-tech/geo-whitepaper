<!--
Qiita 投稿設定
-------------
Title: AIボット専用シャドウドキュメント（AXP）をCloudflare Workerで配信する実装
Tags: Cloudflare, CloudflareWorkers, AI, SEO, GEO
Private: false

投稿URL（投稿後に記入）: https://qiita.com/...
-->

# AIボット専用シャドウドキュメント（AXP）をCloudflare Workerで配信する実装

## TL;DR

- 現代の顧客サイトは人間向け（CSR、Cookie、トラッカー SDK）で、GPTBot・ClaudeBot・PerplexityBot はまともにパースできない。
- 解：**同一 URL** で人間には通常サイト、AIボットには清潔な HTML + JSON-LD + Markdown を返す **シャドウドキュメント**。
- エッジ（Cloudflare Worker）で User-Agent を検出して経路分岐する。顧客サイトのコードは一切変更しない。
- 展開 2 週間以内に AIボットトラフィック 3〜5 倍、シテーション率改善は 2〜3 週間遅れで追従する。

---

## 1. なぜ既存サイトでは駄目なのか

生成AIのクローラ（GPTBot、ClaudeBot、PerplexityBot、CCBot 等）は、人間向けに最適化された現代サイトを取り込もうとすると以下で詰まる：

- **クライアントサイド JavaScript** — 初期 HTML が空骨、実コンテンツは後から DOM 注入
- **Cookie バナー / モーダル** — 本文より先に同意ダイアログが占有
- **トラッカー SDK** — Google Tag Manager、ホットジャー等のノイズでトークン予算を食う
- **アニメーション UI** — 意味論的に重要でない装飾が HTML の大半
- **ログイン壁、レート制限** — ボットが遅延 or ブロックされる

結果：クローラはしばしば本文を取りこぼし、ブランドが「AI の世界に存在しない」状態になる。

---

## 2. AXP の設計

**AXP**（AI-ready eXchange Page）は **3 層** の清潔な文書である：

```text
layer 1: pure HTML（CSR / JS 無し、semantic tag 中心）
layer 2: Schema.org JSON-LD（@graph で構造化エンティティ）
layer 3: Markdown（RAG インジェスト用、オプション）
```

人間が普通に URL を叩けば既存の CSR アプリが返り、AIボットが叩けば AXP が返る。**どちらも URL は同じ**（AI の世界ではカノニカル URL が重要）。

### 2.1 経路分岐

Cloudflare Worker が **エッジで User-Agent を判定** する。擬似コード：

```javascript
export default {
  async fetch(request, env) {
    const ua = request.headers.get('User-Agent') || '';

    if (isAIBot(ua)) {
      // AXP 経路
      const axp = await fetchAXP(request, env);
      if (axp.ok) return axp;
      // フォールバック：オリジンに渡す（404 を起こさない）
    }

    // 人間経路
    return fetch(request);
  }
};

function isAIBot(ua) {
  return /GPTBot|ChatGPT-User|OAI-SearchBot|ClaudeBot|Claude-Web|PerplexityBot|CCBot|Google-Extended|Applebot-Extended/i.test(ua);
}
```

**重要な設計ポイント**：

- AIボットリクエストと人間リクエストは**完全に異なる経路**を通る
- AXP 取得失敗時は**必ずオリジンにフォールバック**（AI トラフィックを 404 させない）
- Cache TTL 15 分で「鮮度 vs バックエンド負荷」のバランス

### 2.2 監視対象の UA リスト（25 種、4 グループ）

- **AI 訓練コーパス系**：GPTBot、ClaudeBot、CCBot、Google-Extended、Applebot-Extended
- **AI 検索系**：OAI-SearchBot、PerplexityBot、ChatGPT-User、Claude-Web
- **RAG / 企業利用系**：AnthropicBot、Meta-ExternalAgent
- **その他**：AI2Bot、cohere-ai、Omgilibot、YouBot、Bytespider 等

UA リストは月次で更新する（新しい AIボットが頻出する）。

---

## 3. AXP の中身：JSON-LD が主役

人間向けサイトの `<head>` に JSON-LD を注入する古典的手法では、CSR の都合で AIボットに届かないことがある。AXP はそれを前提に、JSON-LD を**文書本文の先頭**に置く：

```html
<!DOCTYPE html>
<html lang="ja">
<head>
  <meta charset="UTF-8">
  <title>Acme Corp — B2B SaaS for Marketing Automation</title>
  <meta name="description" content="...">
</head>
<body>
  <script type="application/ld+json">
  {
    "@context": "https://schema.org",
    "@graph": [
      {
        "@type": "Organization",
        "@id": "https://acme.example/#org",
        "name": "Acme Corp",
        "url": "https://acme.example",
        "sameAs": [
          "https://en.wikipedia.org/wiki/Acme_Corp",
          "https://www.wikidata.org/wiki/Q12345",
          "https://www.linkedin.com/company/acme"
        ]
      },
      {
        "@type": "Service",
        "@id": "https://acme.example/#svc-marketing-automation",
        "provider": {"@id": "https://acme.example/#org"},
        "serviceType": "Marketing Automation Platform"
      }
    ]
  }
  </script>

  <main>
    <h1>Acme Corp</h1>
    <p>Acme Corp is a B2B SaaS company ...</p>
    <!-- 以降 semantic HTML 本文 -->
  </main>
</body>
</html>
```

**設計原則**：

- `@graph` にフラットに並べ、`@id` で相互参照する（ネスト配列にしない）
- `sameAs` で外部権威ノード（Wikipedia、Wikidata、LinkedIn、GBP）と結ぶ
- 本文 `<main>` は装飾なしの semantic HTML のみ

---

## 4. sitemap.xml と robots.txt の連携

AIボットは **sitemap.xml** を最初に叩く。AXP モードでは sitemap と AXP パスが**完全一致**していなければならない：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">
  <url>
    <loc>https://acme.example/</loc>
    <lastmod>2026-04-19T00:00:00Z</lastmod>
  </url>
  <url>
    <loc>https://acme.example/services/marketing-automation</loc>
    <lastmod>2026-04-15T00:00:00Z</lastmod>
  </url>
</urlset>
```

sitemap も Cloudflare Worker がエッジで合成する（人間が `/sitemap.xml` を叩いても見える）。

`robots.txt` では積極的に歓迎する：

```text
User-agent: GPTBot
Allow: /

User-agent: ClaudeBot
Allow: /

Sitemap: https://acme.example/sitemap.xml
```

**注**：`Google-Extended` と `Applebot-Extended` は訓練用途のオプトアウト用。通常の Googlebot・Applebot と別物なので、ブロックしたければこちらだけ Disallow にする。

---

## 5. SaaS 自社ブランドのパス競合

自社が SaaS 提供者（`app.example.com` で顧客ダッシュボード稼働）の場合、`app.example.com/dashboard` を AXP 化してはいけない（認証壁の内側なので AI に見せるべきでない）。

**パスカテゴリ判定ツリー**：

```text
リクエストパス
  ├─ /admin/*, /dashboard/*, /api/*  → オリジンへ（AXP 化しない）
  ├─ /blog/*, /docs/*, /pricing, /   → AXP 化
  └─ デフォルト                       → オリジンへ（安全側）
```

一律 AXP 化すると認証ページや管理画面の内部構造が AI に漏れるリスクがある。**allowlist 方式**を推奨。

---

## 6. 観察データ（匿名化集計）

5 ブランド × 6 週間の実地運用から：

- **AXP 展開後 2 週間で AIボットトラフィック 3〜5 倍**（主に GPTBot、ClaudeBot、PerplexityBot）
- **シテーション率改善は 2〜3 週間遅れ**（ボット取込 → コーパス統合のラグ）
- **GSC インデックスも同期増加**（AXP の清潔な HTML は従来検索にも親和的）

ただし：**ボットトラフィック上昇 ≠ シテーション率上昇**。AXP 内コンテンツが意味的に薄ければ AI は扱う材料が無い。AXP は「見てもらうためのインフラ」であり銀の弾丸ではない。

---

## 7. 実装の落とし穴

- **CSR の SPA を AXP 化するのが最も難しい** — サーバサイドで同じデータから HTML を再構築する必要があり、アプリケーションロジックの二重化になりがち。DB から直接引く AXP generator を別系統で持つのが正解。
- **JSON-LD の @id を URL で揃える** — `#org` や `#svc-xxx` のフラグメント記法で、エンティティ間参照が sameAs と混在しないよう整理する。
- **Cache TTL の調整** — 短すぎるとバックエンド負荷、長すぎると更新が AI に届かない。**15 分** が初期値として妥当。
- **エッジログの保持** — どの UA がどのパスを叩いたかを記録しないと、後で「AI は見に来ているのか？」が分からない。Cloudflare の Logpush は必須。

---

## 参考

- **完全版ホワイトペーパー 第 6 章（AXP 詳解）**：<https://github.com/baiyuan-tech/geo-whitepaper/blob/main/ja/ch06-axp-shadow-doc.md>
- リポジトリ全体：<https://github.com/baiyuan-tech/geo-whitepaper>（CC BY-NC 4.0、12 章 + 4 付録）
- GPTBot 仕様：<https://platform.openai.com/docs/gptbot>
- ClaudeBot：<https://support.anthropic.com/en/articles/8896518>
- PerplexityBot：<https://docs.perplexity.ai/guides/bots>

---

*本稿は百元科技（Baiyuan Technology）のホワイトペーパー第 6 章の抜粋である。SaaS プロダクト：<https://geo.baiyuan.io>*
