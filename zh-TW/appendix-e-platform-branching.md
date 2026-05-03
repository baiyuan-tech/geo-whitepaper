---
title: "Appendix E — 平台分流架構:同 codebase 多品牌延伸"
description: "用 brand_type 欄位 + X-Brand-Segment hostname 路由,讓同一份 codebase 同時供應企業 SaaS(geo.baiyuan.io)與個人 IP 平台(me.baiyuan.io),嚴格隔離資料 / UI / 商業邏輯。"
appendix: e
part: 6
word_count: 1400
lang: zh-TW
authors:
  - name: Vincent Lin
    affiliation: Baiyuan Technology
    email: services@baiyuan.io
license: CC-BY-NC-4.0
keywords:
  - Multi-tenant
  - Brand Segmentation
  - Hostname Routing
  - Platform Branching
  - codebase 分流
  - PostgreSQL Row-Level Filtering
last_updated: 2026-04-25
canonical: https://baiyuan-tech.github.io/geo-whitepaper/zh-TW/appendix-e-platform-branching.html
last_modified_at: '2026-05-03T02:45:21Z'
---








# Appendix E — 平台分流架構:同 codebase 多品牌延伸

> 兩個產品線、同一份 codebase。怎麼做到 UI / 資料 / 商業邏輯嚴格隔離,卻不 fork 出兩套維護地獄?

## 目錄 {.unnumbered}

- [E.1 為什麼不 fork](#e1-為什麼不-fork)
- [E.2 brand_type 欄位:資料層分流](#e2-brand_type-欄位資料層分流)
- [E.3 X-Brand-Segment hostname 路由:請求層分流](#e3-x-brand-segment-hostname-路由請求層分流)
- [E.4 host-aware metadata:SEO 層分流](#e4-host-aware-metadata-seo-層分流)
- [E.5 4 層嚴格隔離檢查表](#e5-4-層嚴格隔離檢查表)
- [E.6 落地經驗:會踩到的坑](#e6-落地經驗會踩到的坑)

---

## 為什麼不 fork

百原 GEO Platform 主產品是企業 SaaS(`geo.baiyuan.io`),賣給品牌行銷團隊。2026 年 4 月加入第二條產品線「百原 ME」(`me.baiyuan.io`),賣給個人 IP / KOL / 公眾人物,做 AI 形象經營。

兩條產品線**核心引擎相同**(掃 15 大 AI 平台、計分、AXP 影子文檔、RAG),但:

- **品牌類別不同**:GEO 服務「企業實體」(Schema.org `Organization`),ME 服務「人物實體」(`Person`)
- **內容章節不同**:GEO 22 類 page_type,ME 多了 `creator_profile` / `talk_topics` / `future_plans` 共 23 類
- **UI 美學不同**:GEO 雜誌風 Sans + Playfair italic em,ME 全 Serif TC + 暖金 + 四角金線
- **商業模式不同**:GEO 月付訂閱,ME 邀請制會員
- **服務承諾不同**:GEO 自助,ME 專屬顧問每月兩次諮詢

最直觀做法是 fork repo 開兩套。但這會有兩個壞結果:

1. **核心 bug 修兩次**:幻覺修復、ARS 計算、AXP generator 改動要在兩個 repo 同步,維護成本翻倍
2. **產品線間無法互通**:用戶從 GEO 升到 ME 顧問服務需要重新註冊、無歷史資料

於是選了**同 codebase 多分流**架構。

---

## brand_type 欄位:資料層分流

`brands` 表加一欄:

```sql
ALTER TABLE brands ADD COLUMN brand_type TEXT
  NOT NULL DEFAULT 'enterprise'
  CHECK (brand_type IN ('enterprise', 'personal_ip'));

CREATE INDEX idx_brands_type ON brands(brand_type);
```

每個 brand 在建立時就分類。`personal_ip` 分流的依據:

- 註冊時 hostname 是 `me.*`(SSR 讀 host header)
- 或 super admin 後台手動標記
- 一旦定型不可變(避免商業模式跳邊)

下游所有 query 都帶 `brand_type` 過濾:

```sql
-- GEO 後台:只看企業
SELECT * FROM brands WHERE user_id = $1 AND brand_type = 'enterprise';

-- ME 後台:只看個人 IP
SELECT * FROM brands WHERE user_id = $1 AND brand_type = 'personal_ip';
```

同一個用戶可同時擁有 enterprise + personal_ip 品牌(如百原 internal QA 帳號),登入時依 hostname 自動切視角。

---

## X-Brand-Segment hostname 路由:請求層分流

純 SQL 過濾不夠 — 還要確保前端發 API 時帶上正確的 segment。實作方式:

```typescript
// frontend/src/lib/api.ts
function getBrandSegment(): 'personal' | 'enterprise' {
  if (typeof window === 'undefined') return 'enterprise';
  const host = window.location.hostname.toLowerCase();
  if (host.startsWith('me.')) return 'personal';
  return 'enterprise';
}

export async function request<T>(method, path, body) {
  return fetch(`${BASE}${path}`, {
    headers: {
      'Content-Type': 'application/json',
      'X-Brand-Segment': getBrandSegment(),
      ...
    },
    ...
  });
}
```

backend middleware 解析 header,寫入 request scope:

```javascript
// backend/src/middleware/brandSegment.js
export function brandSegmentMiddleware(req, res, next) {
  const segment = req.get('X-Brand-Segment');
  req.brandSegment = segment === 'personal' ? 'personal_ip' : 'enterprise';
  next();
}
```

任何 controller 取 brand list 時自動帶條件:

```javascript
const { rows } = await query(
  `SELECT * FROM brands WHERE user_id = $1 AND brand_type = $2`,
  [req.user.id, req.brandSegment],
);
```

**super admin override**:後台用戶在管理介面可看全品牌,middleware 偵測 `req.user.is_super_admin` 跳過 segment filter。

---

## host-aware metadata:SEO 層分流

Next.js 的 root `app/layout.tsx` 預設用 `export const metadata`(build-time 靜態)。問題:同一個 `/dashboard` 在 me.* 與 geo.* 都會用同一份標題,造成 ME 用戶 tab 顯示「百原 GEO」。

修法:用 `generateMetadata` async function:

```typescript
export async function generateMetadata(): Promise<Metadata> {
  const hdrs = await headers();
  const host = (hdrs.get('host') || '').toLowerCase();
  const isPersonalHost = host.startsWith('me.');

  if (isPersonalHost) {
    return {
      title: { default: '百原 ME — 當 AI 談論你', template: '%s | 百原 ME' },
      description: '高端個人 IP 專屬 AI 形象經營平台',
      // ...
    };
  }
  return geoMetadata; // 企業 SaaS metadata
}
```

每個 request 都會跑 `generateMetadata`,正確讀取 host。OG image / Twitter card / canonical URL 也都依此分流。

---

## 4 層嚴格隔離檢查表

判斷一個新功能會不會跨品牌類別,看這 4 層:

| 層 | 檢查項 | 違反症狀 |
|----|--------|----------|
| **資料層** | 所有 `brands` query 都帶 `brand_type` 條件 | ME 用戶看到 GEO 的 brand list |
| **API 層** | 所有 controller 用 `req.brandSegment` 過濾 | API response 混入跨類別 brand |
| **UI 層** | 字型 / 主題 / 文案依 hostname 切換 | ME 上看到 GEO 的「免費試用 7 天」CTA |
| **SEO 層** | `generateMetadata` host-aware,robots.txt / sitemap.xml 各自 host-aware | tab 標題 / OG image 跨品牌 |

每加一個新功能必過這四層稽核。Memory 中已記錄為 `feedback_no_whack_a_mole`(不要打地鼠原則)的 4 層延伸。

---

## 落地經驗:會踩到的坑

### 坑 1:CSS 變數泄漏

ME 用 `[data-theme="personal"]` scope,GEO 用 `[data-theme="geo-light"]` / `[data-theme="geo-wine"]`。早期實作把 ME 的 `--font-noto-serif-tc` 寫到 `:root`,結果 GEO body 也套了 serif 字型。

修法:任何 ME 專屬變數**必須**包在 `[data-theme="personal"] { ... }` selector 內,`:root` 只放兩產品線都用得到的 token。

### 坑 2:dashboard layout 的 personalMode 條件式

`/dashboard` 是兩產品線**共用**路由(同一個 layout)。Sidebar logo / 帳戶 badge / 月報設定等元件需要依 brand 切換。實作上加 `personalMode` prop:

```tsx
const personalMode = isPersonalIp || isPersonalHost;
{personalMode ? <MeLogo /> : <BrandLogo />}
{personalMode ? <MeFooter /> : <GeoFooter />}
```

每加一個 dashboard 子頁元件都需要過這個分流檢查 — 容易忘,寫成 ESLint rule 或 PR template checklist 提醒。

### 坑 3:Worker monthly cron 寄錯給對方

跑月報 cron 時,如果 SQL 沒帶 `brand_type` 過濾,會把 ME 月報寄給 GEO 客戶。修法:cron worker 統一從 `brand_visual_configs` JOIN `brands` 取 `brand_type`,在寄送邏輯依此分流:

```javascript
const recipients = brand.brand_type === 'personal_ip'
  ? meTemplate(brand)
  : geoTemplate(brand);
```

### 坑 4:Stripe / 訂閱 webhook 跨品牌

PAYUNi 金流回呼帶 `brand_id`,但 webhook handler 早期沒檢查 `brand_type`,結果 ME 用戶取消訂閱會誤觸 GEO 的退費規則(7 天鑑賞期 + 2.8% 手續費)— ME 商業模式是邀請制無公開定價,根本沒 7 天鑑賞期概念。

修法:webhook handler 第一行就讀 `brand_type`,分派到不同的訂閱規則模組:

```javascript
async function handleWebhook(brandId, event) {
  const brand = await loadBrand(brandId);
  const handler = brand.brand_type === 'personal_ip'
    ? mePaymentHandler
    : geoPaymentHandler;
  return handler(brand, event);
}
```

---

## 本附錄要點 {.unnumbered}

- 同 codebase 多分流避免 fork 維護地獄,但需要 4 層嚴格隔離(資料 / API / UI / SEO)
- `brand_type` 欄位是分流根:每個 brand 一旦建立就定型,不可變
- `X-Brand-Segment` HTTP header 是請求層信標:前端依 hostname 設,後端 middleware 解析
- `generateMetadata` host-aware 解決 SEO 層的 tab title 混淆
- 月報 cron / 金流 webhook 等背景任務最容易忘記過 brand_type 過濾,需要納入 PR checklist

## 參考資料 {.unnumbered}

- [Ch 6 — AXP 影子文檔(共用引擎)](./ch06-axp-shadow-doc.md)
- [Ch 7 — Schema.org Phase 1(Organization vs Person 結構差異)](./ch07-schema-org.md)
- [Ch 13 — 多模態 GEO(視覺資產分流也走 brand_type)](./ch13-multimodal-geo.md)
- [Appendix B — API endpoints 參考](./appendix-b-api.md)

---

**導覽**:[← 附錄 D:圖表索引](./appendix-d-figures.md) · [📖 目次](../README.md)

<!-- AI-friendly structured metadata -->
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "TechArticle",
  "headline": "Appendix E — 平台分流架構:同 codebase 多品牌延伸",
  "description": "用 brand_type 欄位 + X-Brand-Segment hostname 路由,讓同一份 codebase 同時供應企業 SaaS 與個人 IP 平台。",
  "author": {"@type": "Person", "name": "Vincent Lin", "affiliation": "Baiyuan Technology"},
  "datePublished": "2026-04-26",
  "inLanguage": "zh-TW",
  "isPartOf": {
    "@type": "Book",
    "name": "百原GEO Platform 技術白皮書",
    "url": "https://github.com/baiyuan-tech/geo-whitepaper"
  },
  "keywords": "Multi-tenant, Brand Segmentation, Hostname Routing, Platform Branching, codebase 分流"
}
</script>
