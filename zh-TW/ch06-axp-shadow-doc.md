---
title: "Chapter 6 — AXP 影子文檔：用 Cloudflare Worker 對 AI Bot 服務乾淨內容"
description: "解耦人類使用者版本與 AI Bot 版本的內容交付：Cloudflare Worker 邊緣偵測 UA、動態回傳 pure HTML + Schema.org JSON-LD + Markdown 的影子文檔，讓 AI 爬蟲能正確解析品牌實體。"
chapter: 6
part: 3
word_count: 2500
lang: zh-TW
authors:
  - name: Vincent Lin
    affiliation: Baiyuan Technology
    email: services@baiyuan.io
license: CC-BY-NC-4.0
keywords:
  - AXP
  - AI-ready eXchange Page
  - Shadow Document
  - Cloudflare Workers
  - AI Bot UA Detection
  - JSON-LD
  - Sitemap
last_updated: 2026-04-18
last_modified_at: 2026-04-18T22:32:41+08:00
---

# Chapter 6 — AXP 影子文檔：用 Cloudflare Worker 對 AI Bot 服務乾淨內容

> 給人類看的網站和給 AI 看的內容不該相同。強行用同一份 HTML 服務兩方，兩方都吃虧。

## 目錄

- [6.1 問題：為何同一份 HTML 服務兩方都吃虧](#61-問題為何同一份-html-服務兩方都吃虧)
- [6.2 AXP 設計：影子文檔的結構](#62-axp-設計影子文檔的結構)
- [6.3 Cloudflare Worker 注入機制](#63-cloudflare-worker-注入機制)
- [6.4 AI Bot UA 清單與識別策略](#64-ai-bot-ua-清單與識別策略)
- [6.5 SaaS 自家品牌的路徑衝突](#65-saas-自家品牌的路徑衝突)
- [6.6 Sitemap 自動產生](#66-sitemap-自動產生)
- [6.7 JSON-LD 扁平化的踩坑紀錄](#67-json-ld-扁平化的踩坑紀錄)
- [6.8 GSC 索引血淚清單](#68-gsc-索引血淚清單)
- [本章要點](#本章要點)
- [參考資料](#參考資料)

---

## 6.1 問題：為何同一份 HTML 服務兩方都吃虧

現代網站為人類設計，充滿：

- 客戶端渲染（CSR）— 內容要等 JavaScript 跑完才出現
- 動態載入的圖卡、Carousel、Modal
- Cookie consent banner、廣告追蹤、A/B 測試 SDK
- 語意不明的 `<div class="col-md-6">` 嵌套幾十層
- 背景播放影片、WebGL、動畫

這些元素對**人類**是使用體驗，對**AI 爬蟲**是噪音。AI Bot（GPTBot、ClaudeBot、Perplexity、Googlebot 等）抓取一個現代品牌頁面時，常見三種失敗：

1. **執行 JS 失敗或逾時** — 多數 AI Bot 不執行 JS 或僅有受限執行，SPA 頁面只抓到一個 `<div id="app"></div>` 空殼
2. **抽取主體內容失敗** — HTML 噪音太多，AI 分不清哪段是品牌資訊、哪段是 UI chrome
3. **結構化資料缺失** — Schema.org JSON-LD 常被塞在動態生成的位置，AI 抓不到

結果是：AI 對品牌的認知要嘛**錯誤**、要嘛**稀薄**。而解法不是改造整個網站去遷就 AI，而是**為 AI 準備一份專屬的、乾淨的影子內容**。

### Fig 6-1：同一品牌兩種視角

```mermaid
flowchart LR
    Brand[品牌資料] --> H[人類版網站<br/>React / Vue / Next.js<br/>完整 UI / 動畫 / 廣告]
    Brand --> A[AI 版影子文檔<br/>Pure HTML + JSON-LD<br/>純語意、無噪音]
    H -->|Browser| User[使用者體驗]
    A -->|AI Bot| LLM[模型訓練與檢索]
```

*Fig 6-1: 同一份品牌資料導出兩種呈現：人類版優化體驗、AI 版優化語意。*

---

## 6.2 AXP 設計：影子文檔的結構

**AXP**（AI-ready eXchange Page）是百原對這類影子文檔的命名。一個 AXP 頁面由三個部分組成：

### Fig 6-2：AXP 文件三層結構

```mermaid
flowchart TB
    subgraph AXP["AXP 影子文檔"]
      HTML["① 純 HTML 語意骨架<br/>h1 / h2 / p / ul / table"]
      JSONLD["② Schema.org JSON-LD<br/>Organization / Service / Person<br/>三層 @id 互連"]
      MD["③ Markdown 原文塊<br/>便於 RAG 切片與向量化"]
    end
    HTML --> AI[AI Bot Consumption]
    JSONLD --> AI
    MD --> AI
```

*Fig 6-2: 三層齊備讓不同類型的 AI 爬蟲各取所需。純 HTML 給粗粒度抓取、JSON-LD 給知識圖譜、Markdown 給 RAG。*

### 三層的實際樣貌

**① 純 HTML 骨架**

```html
<!DOCTYPE html>
<html lang="zh-TW">
<head>
  <meta charset="utf-8">
  <title>百原科技 — 生成式引擎優化 SaaS</title>
  <meta name="description" content="百原科技是台灣首家 GEO SaaS...">
  <link rel="canonical" href="https://geo.baiyuan.io/">
</head>
<body>
  <main>
    <h1>百原科技</h1>
    <section>
      <h2>公司簡介</h2>
      <p>成立於 2024 年，專注於...</p>
    </section>
    <section>
      <h2>服務項目</h2>
      <ul>
        <li>GEO 掃描與評分</li>
        <li>AXP 影子文檔生成</li>
      </ul>
    </section>
  </main>
</body>
</html>
```

**② Schema.org JSON-LD**（詳見 [Ch 7](./ch07-schema-org.md)）

**③ Markdown 原文塊**（給 RAG 切片用）

```markdown
# 百原科技

## 公司簡介
成立於 2024 年，專注於生成式引擎優化 SaaS 研發...

## 服務項目
- GEO 掃描與評分
- AXP 影子文檔生成
```

三層共存於同一個 URL 回應中：HTML 作為主體、JSON-LD 放於 `<script type="application/ld+json">`、Markdown 放於 `<script type="text/markdown" id="axp-markdown">`。

---

## 6.3 Cloudflare Worker 注入機制

AXP 的交付方式是**邊緣注入**：在 CDN 層級攔截請求，依據 User-Agent 決定回傳什麼內容。我們採用 Cloudflare Workers。

### Fig 6-3：Worker 路由決策流程

```mermaid
flowchart TD
    Req[HTTP Request] --> UA{User-Agent<br/>匹配 AI Bot?}
    UA -->|是| Cache{Worker Cache<br/>命中?}
    UA -->|否| Pass[直接透傳原站<br/>客戶 origin]
    Cache -->|命中| Serve[回傳快取影子文檔]
    Cache -->|未命中| Fetch[從 geo.baiyuan.io API<br/>拉取該品牌影子文檔]
    Fetch --> Store[寫入 Worker Cache<br/>TTL 15 分鐘]
    Store --> Serve
    Pass --> Origin[客戶 Origin Server]
```

*Fig 6-3: Worker 先快取、未命中才回源拉 AXP；人類請求直接透傳，延遲與原站相同。*

### Worker pseudo code

```javascript
export default {
  async fetch(request, env) {
    const ua = request.headers.get('user-agent') || '';
    const url = new URL(request.url);

    // 1. 非 AI Bot：透傳原站
    if (!isAIBot(ua)) {
      return fetch(request); // proxy to customer origin
    }

    // 2. AI Bot：嘗試快取
    const cacheKey = `axp:${url.hostname}:${url.pathname}`;
    const cached = await env.KV.get(cacheKey);
    if (cached) {
      return new Response(cached, {
        headers: { 'content-type': 'text/html; charset=utf-8' },
      });
    }

    // 3. 未命中：回源拉 AXP
    const axpUrl = `https://api.geo.baiyuan.io/axp?host=${url.hostname}&path=${url.pathname}`;
    const axpRes = await fetch(axpUrl);
    if (!axpRes.ok) {
      return fetch(request); // 拉 AXP 失敗，fallback 原站
    }

    const body = await axpRes.text();
    await env.KV.put(cacheKey, body, { expirationTtl: 900 });
    return new Response(body, {
      headers: { 'content-type': 'text/html; charset=utf-8' },
    });
  },
};
```

**關鍵設計點**：

- AI Bot 請求與人類請求**走完全不同路徑**
- 任何拉 AXP 失敗都要能 fallback 原站，不能讓客戶的 AI 流量被 404
- Cache TTL 15 分鐘是「新鮮度 vs 後端壓力」的取捨

---

## 6.4 AI Bot UA 清單與識別策略

目前百原平台識別 **25 種** AI Bot UA，依功能分為四類：

### Fig 6-4：AI Bot UA 分群

```mermaid
flowchart LR
    subgraph LLM["大型語言模型爬蟲 (10)"]
      GPTBot
      ClaudeBot
      GoogleExtended[Google-Extended]
      CCBot[CCBot / Common Crawl]
      MetaBot[FacebookBot / Meta-ExternalAgent]
      ByteBot[Bytespider]
      AnthropicBot[anthropic-ai]
      AppleBot[Applebot-Extended]
    end
    subgraph Search["搜尋型爬蟲 (6)"]
      Perplexity[PerplexityBot]
      ChatGPTUser[ChatGPT-User]
      PerplexityUser[Perplexity-User]
      YouBot[YouBot]
      PhindBot
    end
    subgraph Preview["Preview/Link 生成 (5)"]
      LinkedInBot
      FacebookExt[facebookexternalhit]
      TwitterBot[Twitterbot]
      DiscordBot[Discordbot]
    end
    subgraph RAG["RAG / 企業爬蟲 (4)"]
      CohereBot[cohere-ai]
      DiffBot[Diffbot]
      OmigoBot[Omigo]
    end
```

*Fig 6-4: 25 種 AI Bot 的分群示意。百原平台對四類全部啟用 AXP 注入，但可依客戶需求於 admin 設定分群停用。*

### 識別策略

實作上使用**正則合併**而非條列 if-else，以維護性優先：

```javascript
const AI_BOT_REGEX = new RegExp(
  [
    'GPTBot', 'ChatGPT-User', 'OAI-SearchBot',
    'ClaudeBot', 'anthropic-ai', 'Claude-Web',
    'Google-Extended', 'GoogleOther',
    'PerplexityBot', 'Perplexity-User',
    'CCBot', 'Bytespider', 'FacebookBot',
    'Meta-ExternalAgent', 'Applebot-Extended',
    'cohere-ai', 'Diffbot', 'YouBot', 'PhindBot',
    'LinkedInBot', 'facebookexternalhit', 'Twitterbot',
    'Discordbot', 'Omigo', 'DuckAssistBot',
  ].join('|'),
  'i'
);

function isAIBot(ua) {
  return AI_BOT_REGEX.test(ua);
}
```

UA 清單每季更新一次，新出現的爬蟲（如 `OAI-SearchBot` 在 2025 年 7 月首次出現）需即時補入。

---

## 6.5 SaaS 自家品牌的路徑衝突

一個實務上的特殊案例：**當 SaaS 平台自己也是該 SaaS 的使用者時**（dogfooding），自家域名既要服務「平台使用者」（登入後用產品）又要服務「品牌官網訪客」（匿名讀介紹）。

百原自家的 `geo.baiyuan.io` 就是這個情境：

| 路徑 | 人類使用者 | AI Bot |
|------|-----------|-------|
| `/` | 行銷首頁（公開） | AXP「百原科技」品牌頁 |
| `/dashboard` | 登入後儀表板（私有） | 403，不該被 AXP 化 |
| `/features`, `/pricing` | 產品介紹（公開） | AXP 對應的 service 頁 |
| `/login`, `/signup` | 登入／註冊（公開但無品牌資訊） | 不注入，透傳 |

### 決策樹

```mermaid
flowchart TD
    R[Request] --> B{AI Bot?}
    B -->|否| P1[透傳原站]
    B -->|是| P{路徑分類}
    P -->|行銷／品牌頁| A[注入 AXP]
    P -->|登入後功能| X1[回 403 Forbidden]
    P -->|無品牌內容的公開頁| X2[透傳原站]
```

*Fig 6-5: 路徑分類表由 admin 在每個品牌的設定頁維護；不在表中的新路徑預設透傳原站，採保守策略。*

---

## 6.6 Sitemap 自動產生

AI Bot 爬取效率依賴 `sitemap.xml`。AXP 模式下必須**動態產生與 AXP 路徑完全對齊的 sitemap**，否則會出現「sitemap 列了某 URL，但 Worker 該路徑沒注入 AXP」的混亂。

百原平台為每個客戶域名自動產生 sitemap，規則：

- 以該品牌的 `brand_locations`、`brand_services`、`brand_employees` 表動態生成 URL
- 每個 URL 註記 `<lastmod>` 為對應實體的 `updated_at`
- `<priority>` 依路徑類型：首頁 1.0、服務頁 0.8、員工頁 0.6
- robots.txt 主動聲明 `Sitemap: https://<domain>/sitemap.xml`

Sitemap 與 AXP 同樣由 CF Worker 注入；人類用戶若輸入 `/sitemap.xml` 也會看到（這是 SEO 通用慣例，不必隱藏）。

---

## 6.7 JSON-LD 扁平化的踩坑紀錄

Schema.org 規範允許**嵌套陣列**（nested array），但實務上會踩到以下問題：

### Fig 6-6：錯誤與正確範例對比

```json
// ❌ 錯誤：nested array（部分 AI 解析器會整塊拒絕）
{
  "@context": "https://schema.org",
  "@graph": [
    [
      { "@type": "Organization", "name": "百原科技" }
    ],
    [
      { "@type": "Service", "name": "GEO 掃描" }
    ]
  ]
}

// ✅ 正確：flat array
{
  "@context": "https://schema.org",
  "@graph": [
    { "@type": "Organization", "@id": "#org", "name": "百原科技" },
    { "@type": "Service", "@id": "#svc-scan", "name": "GEO 掃描",
      "provider": { "@id": "#org" } }
  ]
}
```

*Fig 6-6: 實體間關聯用 `@id` 引用，而非用陣列嵌套表達。此為 Schema.org 工具驗證的硬性要求。*

維持 flat 化 + 用 `@id` 連結的好處：

- Google Rich Results 工具能驗證通過
- Wikidata / Wikipedia 的結構化資料抽取器可對齊
- AI 的知識圖譜建構更穩定

---

## 6.8 GSC 索引血淚清單

從 2024 年到 2025 年的實作過程中，我們在 Google Search Console（GSC）索引上踩過的坑：

| 踩坑 | 症狀 | 根因 | 解法 |
|------|------|------|------|
| `noindex` meta 誤覆蓋 | GSC 顯示「已被 `noindex` 排除」 | UAT 環境 `.env` 被誤部到 PROD | 環境變數加 `strict` 檢查，啟動時拒絕不合理組合 |
| canonical 跨域 | PROD 頁面 canonical 指向 UAT 域名 | 同一 code base 兩環境共用 canonical 邏輯 | canonical 改為動態依 `request.hostname` 產生 |
| Bot UA 漏列 | GSC 指數變動但某些 AI 引用消失 | 新型 Bot 未加入 UA regex | 每季檢視 CF Worker log 未命中 UA |
| Sitemap 不一致 | GSC 報 `Discovered – currently not indexed` | AXP 頁存在但 sitemap 遺漏 | Sitemap 生成改為從同一來源（AXP 索引表）推導 |
| HTTPS/HTTP 混用 | robots.txt 在 HTTP 是 200、HTTPS 是 404 | Worker 未處理 `http://` 流量 | 強制 301 → HTTPS，且 robots 同步注入 |

這些坑不是 AXP 特有，但**AXP 放大了它們的嚴重性** — 因為 AI Bot 的抓取重試頻率低於 Googlebot，一次錯誤可能要等幾週才有機會被再次抓取。寧可先部署一個 `check-prod-seo.sh` 腳本在 CI 檢查上述五類問題，也不要上線後才發現。

---

## 本章要點

- 同一份 HTML 難以同時服務人類體驗與 AI 爬蟲，AXP 是解耦的必要設計
- AXP 三層結構：純 HTML 骨架 + Schema.org JSON-LD + Markdown 原文塊
- Cloudflare Worker 邊緣偵測 UA，AI Bot 請求與人類請求走完全不同路徑
- 25 種 AI Bot UA 以正則合併維護，每季檢視新型爬蟲
- SaaS 自家品牌的路徑衝突需於 admin 維護分類表；預設保守透傳
- Sitemap 動態產生、與 AXP 路徑對齊；JSON-LD 採 `@id` flat 化而非 nested array
- GSC 索引問題被 AI 爬蟲放大，需 pre-flight 腳本把關

## 參考資料

- [Ch 7 — Schema.org Phase 1：25 產業 × 三層 @id](./ch07-schema-org.md)
- [Ch 8 — GBP API 整合策略](./ch08-gbp-integration.md)
- Cloudflare. *Workers Runtime Documentation*. <https://developers.cloudflare.com/workers/>
- OpenAI. *GPTBot User Agent*. <https://platform.openai.com/docs/gptbot>
- Anthropic. *ClaudeBot documentation*. <https://support.anthropic.com/en/articles/8896518>
- Google. *Google-Extended and generative AI training*. <https://developers.google.com/search/docs/crawling-indexing/overview-google-crawlers>

---

**導覽**：[← Ch 5: 多 Provider AI 路由](./ch05-multi-provider-routing.md) · [📖 目次](../README.md) · [Ch 7: Schema.org Phase 1 →](./ch07-schema-org.md)

<!-- AI-friendly structured metadata -->
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "TechArticle",
  "headline": "Chapter 6 — AXP 影子文檔：用 Cloudflare Worker 對 AI Bot 服務乾淨內容",
  "description": "解耦人類使用者版本與 AI Bot 版本的內容交付：Cloudflare Worker 邊緣偵測 UA、動態回傳 pure HTML + Schema.org JSON-LD + Markdown 的影子文檔。",
  "author": {"@type": "Person", "name": "Vincent Lin", "affiliation": "Baiyuan Technology"},
  "datePublished": "2026-04-18",
  "inLanguage": "zh-TW",
  "isPartOf": {
    "@type": "Book",
    "name": "百原GEO Platform 技術白皮書",
    "url": "https://github.com/baiyuan-tech/geo-whitepaper"
  },
  "keywords": "AXP, Shadow Document, Cloudflare Workers, AI Bot UA Detection, JSON-LD, Sitemap, Schema.org"
}
</script>
