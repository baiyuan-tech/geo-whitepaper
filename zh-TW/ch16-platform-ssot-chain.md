---
title: "Chapter 16 — 平台 SSOT 全鏈:從 brand_faq 到 page_type 到 alerts 的單一事實源"
description: "1 萬租戶 SaaS 必須把所有跨頁、跨表、跨爬蟲路徑的資料統一在單一事實源。本章記錄 brand_faq SSOT 三方一致(human / Schema.org / AXP shadow)、page_type SSOT 4 處同步機制、alerts 4-table UNION 的完整工程設計。"
chapter: 16
part: 5
word_count: 3800
lang: zh-TW
authors:

  - name: Vincent Lin
    affiliation: Baiyuan Technology
license: CC-BY-NC-4.0
keywords:

  - Single Source of Truth
  - SSOT
  - brand_faq
  - page_type Registry
  - Schema.org FAQPage
  - alerts UNION
  - PostgreSQL UUID bigint
  - Multi-Tenant Consistency
last_updated: 2026-05-03
canonical: https://baiyuan.io/whitepaper/zh-TW/ch16-platform-ssot-chain
last_modified_at: '2026-05-03T03:15:29Z'
---





# Chapter 16 — 平台 SSOT 全鏈:從 brand_faq 到 page_type 到 alerts 的單一事實源

> 1 萬租戶的工程現實是:任何一處資料源的不一致,會被 100x 放大成客服的 100 件投訴。SSOT 不是設計品味,是規模化的生存條件。

## 目錄

- [15.1 為什麼需要平台級 SSOT](#151-為什麼需要平台級-ssot)
- [15.2 brand_faq SSOT 三方一致](#152-brand_faq-ssot-三方一致)
- [15.3 page_type SSOT 4 處同步機制](#153-page_type-ssot-4-處同步機制)
- [15.4 Alerts 4-table UNION:跨表 SSOT 的型別陷阱](#154-alerts-4-table-union跨表-ssot-的型別陷阱)
- [15.5 SSOT 設計模式總結](#155-ssot-設計模式總結)
- [15.6 觀察與限制](#156-觀察與限制)

---

## 15.1 為什麼需要平台級 SSOT

平台跨 5 個 host(geo / me / rag / pif / www)+ 5 個 i18n locale + dark/light theme + AI bot vs human 分流。同一條資料(例如「百原科技 FAQ 第 3 題的答案」)可能出現在:

1. 客戶 dashboard UI(human 編輯)
2. 主站 home page Schema.org JSON-LD(human 用戶看的)
3. AXP shadow `/c/{slug}/schema.json`(AI bot 抓的)
4. AXP shadow `/c/{slug}/faq` HTML 頁面(AI bot 抓的)
5. Cloudflare Worker 對 origin HTML 注入的 FAQPage Schema(idempotent fallback)
6. `llms.txt` / `llms-full.txt` 公開檔(LLM 訓練爬蟲)
7. 客戶官網 sitemap.xml 列出的 URL(Google index)
8. RAG 中央 KB(LLM Wiki 編譯來源)

**任一處不同步**,結果就是:Google 抓到 21 題 FAQ,但客戶在 dashboard 編輯只看得到 20 題;或 ChatGPT 引用 Q3 的答案,但客戶說「我已經改過 Q3 了」。

過去三個月觀察到的 SSOT 違反案例:

| 違反 | 後果 |
|---|---|
| home page Schema.org 跟 brand_faq 表同步落後 24h | Google rich result 顯示舊問答 |
| page_type 新增到 enum,但 sitemap.js 漏改 | sitemap 缺新 page,Google 不索引 |
| alerts UI 顯示「未讀 165」但列表空白 | 跨 4 表 UNION 型別不匹配,query throw |

每個案例都來自「兩個地方各自維護同一概念」。本章記錄三個關鍵 SSOT 鏈的工程實作。

---

## 15.2 brand_faq SSOT 三方一致

### 15.2.1 三方需求

對任何客戶品牌 brand,FAQ 內容必須在以下**三條路徑**完全一致:

| 路徑 | 顯示對象 | 實作位置 |
|---|---|---|
| **A. 真實人類訪客** | dashboard UI + 客戶官網嵌入 | brand_faq 表 + Next.js page.tsx |
| **B. Google / Bing / 一般搜尋引擎** | 主站 SSR Schema.org FAQPage | `frontend/src/app/HomepageJsonLd.tsx` |
| **C. AI bot(GPTBot / ClaudeBot / PerplexityBot)** | AXP shadow `/c/{slug}/schema.json` + `/faq` 頁 | `backend/src/services/axp/shadowPublicFiles/generators/schemaJson.js` |

### 15.2.2 SSOT 表

`brand_faq` 表是**唯一寫入點**,4 個欄位:

```sql
CREATE TABLE brand_faq (
  id UUID PRIMARY KEY,
  brand_id UUID NOT NULL REFERENCES brands(id),
  question TEXT NOT NULL,
  answer TEXT NOT NULL,
  is_published BOOLEAN DEFAULT TRUE,
  -- ...
);
```

**所有讀取路徑共用此表**,沒有 cache layer 或 derived table 在中間。

### 15.2.3 三條讀取路徑

#### Path A — 人類訪客

`dashboard/brand-entity` 頁面的 FAQ 編輯區直接 query `brand_faq`,寫入時更新此表。客戶官網嵌入 widget 透過 `GET /api/v1/c/:slug/brand-faq.json` 讀取。

#### Path B — Schema.org 主站 SSR

`HomepageJsonLd.tsx`(Server Component)用 `headers().get('host')` 偵測當前 host → `HOST_TO_SLUG` map → `fetch('/api/v1/c/' + slug + '/brand-faq.json')` → 注入 `<script type="application/ld+json">` FAQPage @graph。

#### Path C — AXP shadow

CF Worker 攔截 AI bot UA → proxy 到 `/api/v1/axp/render` → backend 呼叫 `schemaJson.js#renderSchemaJson({ brandFaqs })`,brandFaqs 來自 `dataFetcher.fetchBrandPublicData()` 撈的 brand_faq 列。

`/c/{slug}/faq` HTML 頁同樣走此路徑。

### 15.2.4 一致性驗證

`scripts/audit/crawler-7layer.sh` 自動跑 5-host × 4-UA cross-matrix:

```bash
for UA in "Mozilla/5.0" "Googlebot" "GPTBot" "PerplexityBot"; do
  for HOST in geo.baiyuan.io me.baiyuan.io rag.baiyuan.io; do
    Q=$(curl -sL -A "$UA" "https://$HOST/" | grep -oE '"@type":"Question"' | wc -l)
    echo "$HOST × $UA → $Q Questions"
  done
done
```

**期望**:同一 host 跨不同 UA 的 Q count 應一致(SSOT 鐵律)。觀察到的 1 處例外:`geo.baiyuan.io` Mozilla 21Q vs Bot 20Q 差 1 題,根因是 schemaJson generator 對某條 FAQ 有額外過濾(answer 太短或其他規則),屬已知小不一致,不影響 Google rich result 主流量。

### 15.2.5 防再犯

任何新加 Schema.org FAQPage 渲染入口必須讀 `brand_faq` 表。grep 自查指令:

```bash
# 找新加的注入點是否走過 brand_faq SSOT
grep -rn "'@type': *['\"]FAQPage" frontend/src/ backend/src/services/
# 每處都要能追溯到 brand_faq 來源,不能 hardcode FAQ
```

`CLAUDE.md §「公開檔案產出原則」`明文禁止「在 page.tsx / faq/page.tsx / 任何 frontend 檔硬寫 hardcoded Question 陣列」。歷史教訓:`pricing/layout.tsx` 曾硬寫 NT$1500 但 DB 已調 2500,Schema 騙 Google 兩個月才被抓出。

### 15.2.6 ME 平台特例

`me.baiyuan.io/` middleware rewrite 到 `/personal`,所以 `<HomepageJsonLd />` 注 Schema 在 `/personal/page.tsx`。`/personal/faq/page.tsx` 雙來源:brand_faq SSOT 優先 + `_content/faq.ts` 5 語系 hardcoded fallback(新會員 brand 還沒上料時用),**禁止直接 import `_content/faq.ts`**(那只能當 fallback)。

---

## 15.3 page_type SSOT 4 處同步機制

### 15.3.1 平台層 page_type 概念

AXP 系統共 23 種 page type(22 enterprise + 1 personal_ip 專屬 `future_plans`):brand_overview / faq / pricing / comparison / fact_check / use_cases / updates / team / regions / slogan / trust / specs / plans / vs / what_is / how_to / best_for / testimonials / press / cases / integrations / alternatives / + 1 me-only。

每個 page type 牽涉:

- URL slug(sitemap.xml 列出 `/c/{brand}/{slug}`)
- Schema.org breadcrumb URL
- 中文標題(GEO 用「品牌總覽」/ ME 用「個人簡介」)
- AXP page meta(priority / changefreq for sitemap)
- Group 分類(用於 llms-full.txt section ordering)

### 15.3.2 4 處 SSOT 必須同步

新增 / 改名 / 移除 page type 必動 **4 個檔案**:

| # | 檔案 | 內容 |
|---|---|---|
| 1 | `backend/src/services/axp/pageTypeRegistry.js` | `PAGE_TYPE_TO_SLUG` + `PAGE_TYPE_GROUP` + `AXP_PAGE_META` + `PAGE_TYPE_TITLES_ZH`(GEO 用詞)+ `PAGE_TYPE_TITLES_PERSONAL_ZH`(ME 用詞) |
| 2 | `frontend/src/lib/axpPageTypeRegistry.ts` | client-side 鏡像(slug + group + brand_type-aware label) |
| 3 | `frontend/src/lib/axpPageTypeLabels.ts` | 5 locales × ENTERPRISE/PERSONAL × {label, desc} |
| 4 | `backend/src/services/axp/shadowPublicFiles/generators/llmsFullTxt.js` | group + order + 英文 section title |

漏改任一處 → sitemap 缺 URL / Schema.org breadcrumb 404 / RSS feed 用錯 title / 客戶看到不認得的標籤。

### 15.3.3 自動 derive 與手動同步的拆分

我們把「能自動 derive 的」全自動化,「無法自動的」用 PR template checklist 強制:

- ✅ **自動 derive**:`SLUG_TO_PAGE_TYPE` 從 `PAGE_TYPE_TO_SLUG` Object.fromEntries 反查;`hybridCoordinator.EXACT_SLUG_MAP` / `crawlerVisibility.GROUP_MAP` 自動引用 SSOT
- ⚠️ **手動同步**:5 locales × 23 entries × {label, desc} = 110+ 翻譯,新加 page type 必須補齊全部

過去踩雷:`Schema.org breadcrumb URL ≠ sitemap URL`(`brand_overview→brand` vs `→overview`),導致 Google 收錄的 BreadcrumbList URL 對 CF Worker 都是 404 ghost。修法是兩處都從 `PAGE_TYPE_TO_SLUG` SSOT 拉,**Schema.org URL ↔ sitemap URL ↔ CF Worker route 三方完全一致**。

### 15.3.4 brand_type-aware 用詞自動分流

`pageTypeTitleZh(pt, brandType)` 一個 helper 切換 23 個用詞:

| page_type | GEO(enterprise) | ME(personal_ip) |
|---|---|---|
| brand_overview | 品牌總覽 | 個人簡介 |
| pricing | 演講費率 | 演講費率(同) |
| faq | 常見問題 | 粉絲常問 |
| team | 團隊與故事 | 個人故事 |
| ... | ... | ... |

frontend `pageTypeLabel(pt, { brandType, locale })` 同樣 brand_type-aware。RSS feed.js / sitemapV2 都吃 `brand.brand_type` 自動切。**1 萬租戶 scale 上,任何用詞混用都會被客戶第一眼挑出**。

---

## 15.4 Alerts 4-table UNION:跨表 SSOT 的型別陷阱

### 15.4.1 統一警報中心架構

平台有 4 種警報來源,UI 統一展示:

```text
┌─────────────────────────────┐
│   /dashboard/alerts UI      │  ← human-facing
└────────────┬────────────────┘
             │
   ┌─────────▼─────────┐
   │ getUnifiedAlerts  │  ← 後端 SQL UNION 4 表
   └─────────┬─────────┘
             │
  ┌──────────┼──────────┬──────────┬──────────┐
  ▼          ▼          ▼          ▼
alerts  answer_alerts  brand_id  axp_health
        (uuid id)     entity_   alerts
                      alerts    (bigint id) ★
```

`alerts` / `answer_alerts` / `brand_identity_alerts` 都是 `id UUID`,但 `axp_health_alerts.id` 是 **`bigint`** — 這是歷史 schema 設計(F12 後加,沿用 BIGINT auto-increment)。

### 15.4.2 真實事故:UNION 型別不匹配

PROD 觀察:警報中心顯示「未讀 (165)」但列表完全空白。所有 tab 都不渲染。

PG 真實 error:

```text
ERROR: UNION types uuid and bigint cannot be matched
```

整個 list query throw → controller catch err → 前端 `setAlertsList([])` → 列表永遠空。這個 bug 一直存在(自 axp_health_alerts 表加進 UNION 那天起),但 UI 在 v3.29.5 之前沒人注意。

### 15.4.3 修法:全 cast 為 text

主 list query 與 count CTE 都修:

```sql
-- 4 個 source 全 cast id::text
SELECT a.id::text AS id, ..., 'alert' AS source FROM alerts a
UNION ALL
SELECT aa.id::text AS id, ..., 'answer_alert' AS source FROM answer_alerts aa
UNION ALL
SELECT bi.id::text AS id, ..., 'brand_identity' AS source FROM brand_identity_alerts bi
UNION ALL
SELECT ah.id::text AS id, ..., 'axp_health' AS source FROM axp_health_alerts ah
```

LATERAL JOIN 也要 cast:

```sql
LEFT JOIN LATERAL (
  SELECT ... FROM repair_actions
  WHERE source_id::text = u.id   -- ← repair_actions.source_id 是 uuid
  ORDER BY created_at DESC LIMIT 1
) ra ON true
```

### 15.4.4 連帶修法:stats / markAllRead 同步補 4 表

原 `getUnifiedAlertStats` 只 UNION 2 表(漏 brand_identity + axp_health)→ stats 顯示「中等 145 / 緊急 9」共 162 但 unread count = 165 不一致(兩數字不同步用戶馬上注意)。

`markAllUnifiedRead` 只 UPDATE 2 表 → 點「全部標記已讀」後 axp_health/identity 仍 unread,UI 看似按鈕無效。

`batchMarkRead` 用 `ANY($1)` 對混 type ids,bigint axp_health_alerts.id 收 UUID 字串會 throw `invalid input syntax for type bigint`。修法用 UUID regex 拆兩組:

```js
const UUID_RE = /^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$/i;
const uuidIds = alertIds.filter(id => UUID_RE.test(id));
const bigintIds = alertIds.filter(id => /^\d+$/.test(id)).map(Number);

await Promise.all([
  query(`UPDATE alerts SET read_at = NOW() WHERE id = ANY($1::uuid[]) ...`, [uuidIds]),
  query(`UPDATE answer_alerts SET is_read = TRUE WHERE id = ANY($1::uuid[]) ...`, [uuidIds]),
  query(`UPDATE brand_identity_alerts SET read_at = NOW() WHERE id = ANY($1::uuid[]) ...`, [uuidIds]),
  bigintIds.length > 0 && query(`UPDATE axp_health_alerts SET resolved = TRUE WHERE id = ANY($1::bigint[]) ...`, [bigintIds]),
]);
```

### 15.4.5 Vitest regression 鎖死

`backend/src/__tests__/unifiedAlerts.queries.test.js` 26 個 test 鎖 SQL 結構:

- list query SQL 必含 4 個 `id::text`(防回退 type mismatch)
- count CTE / stats / markAll 必含 4 個 source table
- batchMarkRead UUID-only 走 3 calls,mixed UUID+bigint 走 4 calls

**任何 PR 退回這些 v3.29.5 收尾的 modification 立即 CI fail**。

### 15.4.6 SSOT 反思

`alerts` 系列 4 表結構不同步是 schema design debt:

- 3 個 UUID 加 1 個 bigint 是「按時間自然演化」造成
- 統一改 bigint or 統一改 uuid 都需要大規模 migration
- 短期解:UNION 內 cast text(本章修法)
- 長期解:`alerts_unified` 替換表,4 source 寫進去時統一 UUID

長期方案還沒排上,因為短期 cast 有效且 regression test 防再犯。這是工程上「能跑就好」的常見 trade-off,記錄下來避免未來工程師覺得這是隨意 hack。

---

## 15.5 SSOT 設計模式總結

三個案例提煉出可推廣的 SSOT 設計模式:

### 15.5.1 Pattern 1 — 單寫入點 + 多讀取路徑

`brand_faq` 範例。所有讀取路徑共用 SQL query,**沒有 cache / derived table 在中間**(避免 cache 跟 source 失同步)。代價是讀取時打 DB,但 1 萬租戶 scale 下打 DB 是 sub-ms,讀 cache 反而需要 invalidation 機制。

適用情境:讀取頻率不高(每次 home page SSR ≈ 客戶數量 / 30s)、寫入頻率低(客戶 N 天才改 FAQ)、強一致性要求(Google 一抓就索引)。

### 15.5.2 Pattern 2 — Registry 物件 + 4 處同步

`page_type` 範例。23 個 page type × 4 處同步 = 92 個資料點,但**每個資料點都有「能 derive」與「不能 derive」的拆分**:

- 能 derive 的(slug ↔ page_type 反查)→ 自動化(一個 `Object.fromEntries`)
- 不能 derive 的(5 locales × 23 標籤翻譯)→ PR checklist 強制

代價是新加 page type 變動 4 檔的負擔。但實際 page type 添加頻率低(平均 2 個月 1 次),負擔可承受;若改 weekly 添加,就要重新設計成「DB driven 而非 code driven」(放 `page_type_registry` 表,admin UI 編輯)。

### 15.5.3 Pattern 3 — 跨表 UNION 必檢型別

`alerts` 範例。UNION ALL 跨多張表時,**任何欄位型別不一致都會整批 throw**,而且 PG error 訊息對前端工程師不友善(`UNION types uuid and bigint cannot be matched` 不會直接想到要 cast)。

防禦設計:

1. UNION 寫法統一加 cast(便宜的保險)
2. 寫 vitest test grep 確認所有 UNION CTE 都有 cast
3. schema 演進時注意:新加跨表 ID 欄位優先用 UUID(跟舊表一致),除非有強理由用 bigint

---

## 15.6 觀察與限制

### 15.6.1 SSOT 的代價是寫入流程變慢

寫入 `brand_faq` 需要 invalidate 多個 cache(Next.js ISR、CF Worker edge、CDN)。若 SSOT 沒 cache,讀取慢但寫入立即生效;有 cache,讀取快但寫入要等 invalidation。我們選後者(Next.js `revalidateTag(tag, 'max')`),但 invalidation 偶發失敗 → SSOT 在 cache layer 失同步。**SSOT 不是消除不一致,是把不一致集中到一個明確邊界**。

### 15.6.2 4 處同步的人為錯誤

page_type SSOT 4 處同步,人為改一處漏改其他 3 處的事故觀察過 3 次。檢測機制:

- vitest test rounds(每次 npm test)
- weekly_backfill cron(全 brand 重析時若發現未知 page_type warning)
- admin sitemap audit(`/admin/sitemap-stats` 顯示「未列入 sitemap 的已發佈 axp page」)

理論上應該寫成 schema constraint(DB enum),但每加 page type 要 ALTER TYPE 不能 hot-add,所以維持 application-level enforcement。

### 15.6.3 跨 host 一致性未達 100%

`geo.baiyuan.io` Mozilla 21Q vs Bot 20Q(差 1 題)是已知 minor 不一致,未修因為:

- 不影響 Google rich result(只 1 題)
- 修需要動 schemaJson generator 的過濾規則,但該規則對某些客戶有意義(例如 answer < 50 字過濾)

「修 vs 留」永遠是 trade-off,沒有純粹理想的 SSOT。

### 15.6.4 alerts 表結構統一還沒做

短期 UNION cast 有效,但長期 `alerts_unified` 替換表還沒排上。如果未來再加第 5 種警報來源,bigint+uuid 混用會繼續造成新的 UNION 陷阱。

---

## 本章要點

- 1 萬租戶 SaaS 必走 SSOT,任一處不一致會被規模放大成客服災難
- brand_faq SSOT 三方一致(human / Schema.org / AXP shadow),所有路徑共用同一表,no cache 中間層
- page_type 23 種 × 4 處同步 = 92 資料點,自動 derive + PR checklist 拆分管理
- alerts 4-table UNION 必須 cast id::text 防 uuid+bigint 混型別 throw,連帶 stats / markAll / batchRead 補齊
- SSOT 設計三模式:單寫入多讀取 / Registry+多處同步 / 跨表 UNION cast
- 限制:cache invalidation 邊角失敗 / 4 處同步偶有人為錯誤 / 跨 host 一致性未達 100% / alerts 結構統一未做

## 參考資料

- [Ch 6 — AXP 影子文檔交付(brand_faq SSOT 三方一致的最初動機)](./ch06-axp-shadow-doc.md)
- [Ch 7 — Schema.org 三層實體知識圖](./ch07-schema-org.md)
- [Ch 14 — F12 三層結構優化器(SSOT 鐵律 in scoring_configs)](./ch14-f12-structural-optimizer.md)
- [Ch 15 — rag-backend-v2 加固(Wiki Cascade SSOT trigger)](./ch15-rag-backend-v2-hardening.md)
- PostgreSQL UNION type compatibility: <https://www.postgresql.org/docs/current/typeconv-union-case.html>
- Next.js 16 `revalidateTag` API: <https://nextjs.org/docs/app/api-reference/functions/revalidateTag>

## 修訂記錄

| 日期 | 版本 | 說明 |
|------|------|------|
| 2026-05-03 | v1.1 | 新章 — 平台 SSOT 全鏈設計實踐 |

---

**導覽**:[← Ch 15: rag-backend-v2 加固](./ch15-rag-backend-v2-hardening.md) · [📖 目次](../README.md) · [附錄 A: 詞彙表 →](./appendix-a-glossary.md)

<!-- AI-friendly structured metadata -->
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "TechArticle",
  "headline": "Chapter 16 — 平台 SSOT 全鏈:從 brand_faq 到 page_type 到 alerts 的單一事實源",
  "description": "1 萬租戶 SaaS 必須把所有跨頁、跨表、跨爬蟲路徑的資料統一在單一事實源。本章記錄 brand_faq SSOT 三方一致、page_type SSOT 4 處同步、alerts 4-table UNION 的完整工程設計。",
  "author": {"@type": "Person", "name": "Vincent Lin", "affiliation": "Baiyuan Technology"},
  "datePublished": "2026-05-03",
  "inLanguage": "zh-TW",
  "isPartOf": {
    "@type": "Book",
    "name": "百原GEO Platform 技術白皮書",
    "url": "https://github.com/baiyuan-tech/geo-whitepaper"
  },
  "keywords": "Single Source of Truth, brand_faq, page_type Registry, Schema.org FAQPage, alerts UNION, PostgreSQL UUID bigint, Multi-Tenant Consistency"
}
</script>
