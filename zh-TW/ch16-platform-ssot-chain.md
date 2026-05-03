---
title: "Chapter 16 — 平台 SSOT 全鏈:從 brand_faq 到 page_type 到 alerts 的單一事實源"
description: "1 萬租戶 SaaS 必須把所有跨頁、跨表、跨爬蟲路徑的資料統一在單一事實源。本章記錄 brand_faq SSOT 三方一致(human / Schema.org / AXP shadow)、page_type SSOT 4 處同步機制、alerts 4-table UNION 的完整工程設計。"
chapter: 16
part: 5
word_count: 6800
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
last_modified_at: '2026-05-03T04:34:12Z'
---

# Chapter 16 — 平台 SSOT 全鏈:從 brand_faq 到 page_type 到 alerts 的單一事實源

> 1 萬租戶的工程現實是:任何一處資料源的不一致,會被 100x 放大成客服的 100 件投訴。SSOT 不是設計品味,是規模化的生存條件。

## 目錄

- [16.1 為什麼需要平台級 SSOT](#161-為什麼需要平台級-ssot)
- [16.2 brand_faq SSOT 三方一致](#162-brand_faq-ssot-三方一致)
- [16.3 page_type SSOT 4 處同步機制](#163-page_type-ssot-4-處同步機制)
- [16.4 Alerts 4-table UNION:跨表 SSOT 的型別陷阱](#164-alerts-4-table-union跨表-ssot-的型別陷阱)
- [16.5 SSOT 設計模式總結](#165-ssot-設計模式總結)
- [16.6 觀察與限制](#166-觀察與限制)
- [16.7 工程教訓](#167-工程教訓)

---

## 16.1 為什麼需要平台級 SSOT

### 16.1.1 平台拓樸的複雜性

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

### 16.1.2 真實 SSOT 違反案例(時間線)

過去三個月觀察到的 SSOT 違反案例,每一個都讓客戶感受過真實困惑:

| 違反 | 後果 | 修復 |
|---|---|---|
| home page Schema.org 跟 brand_faq 表同步落後 24h | Google rich result 顯示舊問答 | 改 server-side fetch,移除中間 ISR cache |
| page_type 新增到 enum,但 sitemap.js 漏改 | sitemap 缺新 page,Google 不索引 | 加 PR checklist + vitest test |
| alerts UI 顯示「未讀 165」但列表空白 | 跨 4 表 UNION 型別不匹配,query throw | 全 cast id::text + 26 vitest test 鎖死 |
| pricing/layout.tsx hardcoded NT$1500 但 DB 已調 2500 | Schema 騙 Google 兩個月 | 全平台改 server-side fetch DB |
| `/c/baiyuan/overview` Schema breadcrumb URL 對 CF Worker 404 | Google 收錄 404 ghost URL | breadcrumb URL 改從 PAGE_TYPE_TO_SLUG SSOT derive |

每個案例都來自「兩個地方各自維護同一概念」。本章記錄三個關鍵 SSOT 鏈的工程實作。

### 16.1.3 SSOT 的隱形成本

值得提的是,SSOT 不是免費的——它需要:

- **設計工夫**:認真想清楚「誰是 source、誰是 derived」
- **強制機制**:trigger / vitest test / PR template / grep 自查
- **容錯處理**:source 不可達時的 fallback 行為(壞 vs 空 vs default)
- **教育成本**:新人 onboard 時必須理解「為什麼 brand_faq 是唯一寫入點」

不做 SSOT 看起來短期省事,但每多一處 fragmentation,未來重構成本指數成長。1 萬租戶 scale 不是「未來的問題」,是**今天就必須拿來當工程決策的判準**。

---

## 16.2 brand_faq SSOT 三方一致

### 16.2.1 三方需求

對任何客戶品牌 brand,FAQ 內容必須在以下**三條路徑**完全一致:

| 路徑 | 顯示對象 | 實作位置 |
|---|---|---|
| **A. 真實人類訪客** | dashboard UI + 客戶官網嵌入 | brand_faq 表 + Next.js page.tsx |
| **B. Google / Bing / 一般搜尋引擎** | 主站 SSR Schema.org FAQPage | `frontend/src/app/HomepageJsonLd.tsx` |
| **C. AI bot(GPTBot / ClaudeBot / PerplexityBot)** | AXP shadow `/c/{slug}/schema.json` + `/faq` 頁 | `backend/src/services/axp/shadowPublicFiles/generators/schemaJson.js` |

### 16.2.2 SSOT 表

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

這個設計違反「常識性」 SaaS 設計(常見會做 Redis cache 層),但**1 萬租戶 scale 下 cache invalidation 才是真正的痛**。讀 DB 一次 sub-ms,但 cache 失同步 24 小時客戶都看到舊答案。我們選 cache-less 方案,代價是 backend 多承受 N% 的 brand_faq query,實測可承受。

### 16.2.3 三條讀取路徑

#### Path A — 人類訪客

`dashboard/brand-entity` 頁面的 FAQ 編輯區直接 query `brand_faq`,寫入時更新此表。客戶官網嵌入 widget 透過 `GET /api/v1/c/:slug/brand-faq.json` 讀取。

```javascript
// frontend/src/app/dashboard/brand-entity/page.tsx
const faqs = await fetch(`/api/v1/brands/${brandId}/faq`).then(r => r.json());
// 編輯後 PUT /api/v1/brands/:brandId/faq/:faqId 直寫 brand_faq 表
```

#### Path B — Schema.org 主站 SSR

`HomepageJsonLd.tsx`(Server Component)用 `headers().get('host')` 偵測當前 host → `HOST_TO_SLUG` map → `fetch('/api/v1/c/' + slug + '/brand-faq.json')` → 注入 `<script type="application/ld+json">` FAQPage @graph。

```typescript
// frontend/src/app/HomepageJsonLd.tsx (server component)
import { headers } from 'next/headers';

const HOST_TO_SLUG: Record<string, string> = {
  'geo.baiyuan.io': 'geo-baiyuan',
  'me.baiyuan.io': 'me-baiyuan',
  'rag.baiyuan.io': 'rag-baiyuan',
  'baiyuan.io': 'baiyuan',           // www 主站
  'www.baiyuan.io': 'baiyuan',
  // 新加自托管子域必須補這條
};

export async function HomepageJsonLd() {
  const host = (await headers()).get('host');
  const slug = HOST_TO_SLUG[host];
  if (!slug) return null;
  const res = await fetch(`${ORIGIN}/api/v1/c/${slug}/brand-faq.json`,
    { next: { tags: [`brand-faq-${slug}`], revalidate: 3600 } });
  const faqs = await res.json();
  return <script type="application/ld+json" dangerouslySetInnerHTML={{ __html:
    JSON.stringify({
      '@context': 'https://schema.org',
      '@type': 'FAQPage',
      mainEntity: faqs.map(f => ({
        '@type': 'Question',
        name: f.question,
        acceptedAnswer: { '@type': 'Answer', text: f.answer },
      })),
    })
  }} />;
}
```

兩個關鍵設計:

1. **server component**(無 `'use client'`)— Googlebot 看到 SSR 已注入的 Schema.org;若用 client component + useEffect,Googlebot 看到的是空骨架
2. **revalidateTag 標籤**(`brand-faq-${slug}`)— 客戶在 dashboard 改 FAQ 時 backend 觸發 `revalidateTag(\`brand-faq-${slug}\`, 'max')`,Next.js 立即 invalidate 該 tag 的 cache

#### Path C — AXP shadow

CF Worker 攔截 AI bot UA → proxy 到 `/api/v1/axp/render` → backend 呼叫 `schemaJson.js#renderSchemaJson({ brandFaqs })`,brandFaqs 來自 `dataFetcher.fetchBrandPublicData()` 撈的 brand_faq 列。

`/c/{slug}/faq` HTML 頁同樣走此路徑——backend 統一從 `brand_faq` 表撈,UI 渲染或 JSON 輸出共用同一資料來源。

### 16.2.4 一致性驗證

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

更嚴格的驗證方式 — backfill cron 跑全平台 brand_faq SSOT 3 路徑一致性:

```bash
# Path A: brand_faq 表 / Path B: /c/:slug/schema.json / Path C: /c/:slug/sitemap.xml
docker exec geo-saas-prod-postgres-1 psql -U geo_admin -d geo_db -At -F'|' -c \
  "SELECT b.slug, COUNT(bf.id) FILTER (WHERE bf.is_published) FROM brands b \
   LEFT JOIN brand_faq bf ON bf.brand_id=b.id WHERE b.is_active GROUP BY b.slug" | \
while IFS='|' read -r slug a; do
  b=$(curl -sf "https://geo.baiyuan.io/api/v1/c/$slug/schema.json" \
        | grep -c '"@type": "Question"')
  echo "$slug brand_faq=$a schema=$b"
done
```

新加新增 brand 必跑這個一致性檢查,不一致 → audit log 警告。

### 16.2.5 防再犯

任何新加 Schema.org FAQPage 渲染入口必須讀 `brand_faq` 表。grep 自查指令:

```bash
# 找新加的注入點是否走過 brand_faq SSOT
grep -rn "'@type': *['\"]FAQPage" frontend/src/ backend/src/services/
# 每處都要能追溯到 brand_faq 來源,不能 hardcode FAQ
```

`CLAUDE.md §「公開檔案產出原則」`明文禁止「在 page.tsx / faq/page.tsx / 任何 frontend 檔硬寫 hardcoded Question 陣列」。歷史教訓:`pricing/layout.tsx` 曾硬寫 NT$1500 但 DB 已調 2500,Schema 騙 Google 兩個月才被抓出。

### 16.2.6 ME 平台特例

`me.baiyuan.io/` middleware rewrite 到 `/personal`,所以 `<HomepageJsonLd />` 注 Schema 在 `/personal/page.tsx`。`/personal/faq/page.tsx` 雙來源:brand_faq SSOT 優先 + `_content/faq.ts` 5 語系 hardcoded fallback(新會員 brand 還沒上料時用),**禁止直接 import `_content/faq.ts`**(那只能當 fallback)。

ME 平台會員 brand 的 onboarding 必填:`personal_profiles`(全名/專業/known_for)+ brand_documents(著作 / 影片 / 媒體報導),否則 hybridCoordinator 撈 personal_profiles 是空,LLM Phase 10 守門 return null。觀察到 17 個 ME brand 中 5 個 metadata 不齊,佔 29%,客服需主動跟客戶補齊。

### 16.2.7 host-aware metadata 工具

跨 5 自托管子域 metadata 一致性,我們用一個 helper 統一:

```typescript
// frontend/src/lib/hostAwareMetadata.ts
const HOST_TO_SITE_NAME: Record<string, string> = {
  'geo.baiyuan.io': '百原 GEO',
  'me.baiyuan.io': '百原 ME',
  'rag.baiyuan.io': '百原 RAG',
  'baiyuan.io': '百原科技',
  'www.baiyuan.io': '百原科技',
};

export async function generateHostAwareMetadata(opts: {
  pathname: string;
  title: string;
  description?: string;
}) {
  const host = (await headers()).get('host') ?? 'geo.baiyuan.io';
  const siteName = HOST_TO_SITE_NAME[host] ?? '百原';
  const url = `https://${host}${opts.pathname}`;
  return {
    title: `${opts.title} | ${siteName}`,
    description: opts.description,
    openGraph: { title: opts.title, url, siteName },
    alternates: { canonical: url },
  };
}
```

每個 layout 共用此工具,canonical 永遠對齊真實 request host,不會出現「pif 上的頁面 canonical 指 geo」這種錯。

---

## 16.3 page_type SSOT 4 處同步機制

### 16.3.1 平台層 page_type 概念

AXP 系統共 23 種 page type(22 enterprise + 1 personal_ip 專屬 `future_plans`):brand_overview / faq / pricing / comparison / fact_check / use_cases / updates / team / regions / slogan / trust / specs / plans / vs / what_is / how_to / best_for / testimonials / press / cases / integrations / alternatives / + 1 me-only。

每個 page type 牽涉:

- URL slug(sitemap.xml 列出 `/c/{brand}/{slug}`)
- Schema.org breadcrumb URL
- 中文標題(GEO 用「品牌總覽」/ ME 用「個人簡介」)
- AXP page meta(priority / changefreq for sitemap)
- Group 分類(用於 llms-full.txt section ordering)

### 16.3.2 4 處 SSOT 必須同步

新增 / 改名 / 移除 page type 必動 **4 個檔案**:

| # | 檔案 | 內容 |
|---|---|---|
| 1 | `backend/src/services/axp/pageTypeRegistry.js` | `PAGE_TYPE_TO_SLUG` + `PAGE_TYPE_GROUP` + `AXP_PAGE_META` + `PAGE_TYPE_TITLES_ZH`(GEO 用詞)+ `PAGE_TYPE_TITLES_PERSONAL_ZH`(ME 用詞) |
| 2 | `frontend/src/lib/axpPageTypeRegistry.ts` | client-side 鏡像(slug + group + brand_type-aware label) |
| 3 | `frontend/src/lib/axpPageTypeLabels.ts` | 5 locales × ENTERPRISE/PERSONAL × {label, desc} |
| 4 | `backend/src/services/axp/shadowPublicFiles/generators/llmsFullTxt.js` | group + order + 英文 section title |

漏改任一處 → sitemap 缺 URL / Schema.org breadcrumb 404 / RSS feed 用錯 title / 客戶看到不認得的標籤。

### 16.3.3 自動 derive 與手動同步的拆分

我們把「能自動 derive 的」全自動化,「無法自動的」用 PR template checklist 強制:

- ✅ **自動 derive**:`SLUG_TO_PAGE_TYPE` 從 `PAGE_TYPE_TO_SLUG` Object.fromEntries 反查;`hybridCoordinator.EXACT_SLUG_MAP` / `crawlerVisibility.GROUP_MAP` 自動引用 SSOT
- ⚠️ **手動同步**:5 locales × 23 entries × {label, desc} = 110+ 翻譯,新加 page type 必須補齊全部

```javascript
// 自動 derive 範例(pageTypeRegistry.js)
const PAGE_TYPE_TO_SLUG = {
  brand_overview: 'overview',
  faq: 'faq',
  pricing: 'pricing',
  // ... 23 entries
};

// 反查 map 從 derive,不手動維護
const SLUG_TO_PAGE_TYPE = Object.fromEntries(
  Object.entries(PAGE_TYPE_TO_SLUG).map(([k, v]) => [v, k])
);
```

過去踩雷:`Schema.org breadcrumb URL ≠ sitemap URL`(`brand_overview→brand` vs `→overview`),導致 Google 收錄的 BreadcrumbList URL 對 CF Worker 都是 404 ghost。修法是兩處都從 `PAGE_TYPE_TO_SLUG` SSOT 拉,**Schema.org URL ↔ sitemap URL ↔ CF Worker route 三方完全一致**。

### 16.3.4 breadcrumb 404 ghost 事件回顧

事件影響:**42 天**期間 Google Search Console 累積 ~3000 個 404 ghost URL。

時間線:

- **Day 1**:某次 deploy 加入 Schema.org BreadcrumbList,程式員自己定義 `PAGE_TYPE_BREADCRUMB = { brand_overview: 'brand', competitor_comparison: 'comparison', intent_what_is: 'intent/what' }` 用語想得「比較直觀」
- **Day 1-42**:Schema.org breadcrumb URL 是 `/c/baiyuan/brand`,但 sitemap.js 用 `PAGE_TYPE_TO_SLUG` map 是 `/c/baiyuan/overview`。CF Worker 接受 `overview` route,接到 `brand` route 直接 404
- **Day 30**:某客戶反應「Google Search Console 顯示我家網站很多 404」
- **Day 42**:深入 audit 才發現 breadcrumb 跟 sitemap 用不同命名
- **修法**:刪掉 PAGE_TYPE_BREADCRUMB,直接從 PAGE_TYPE_TO_SLUG derive

教訓:**任何「為了直觀」自定義的 mapping 都是 SSOT 違反**。如果一個概念已經有 canonical name(slug),其他地方不能再造一份。

### 16.3.5 brand_type-aware 用詞自動分流

`pageTypeTitleZh(pt, brandType)` 一個 helper 切換 23 個用詞:

| page_type | GEO(enterprise) | ME(personal_ip) |
|---|---|---|
| brand_overview | 品牌總覽 | 個人簡介 |
| pricing | 演講費率 | 演講費率(同) |
| faq | 常見問題 | 粉絲常問 |
| team | 團隊與故事 | 個人故事 |
| ... | ... | ... |

frontend `pageTypeLabel(pt, { brandType, locale })` 同樣 brand_type-aware。RSS feed.js / sitemapV2 都吃 `brand.brand_type` 自動切。**1 萬租戶 scale 上,任何用詞混用都會被客戶第一眼挑出**。

我們考慮過用 i18n 標準的 `pluralization rules` 處理,但 brand_type 不是文法概念,是 product context。最後選擇 explicit `pageTypeTitleZh` 函式,維護成本可接受。

### 16.3.6 為什麼不改 DB-driven

理論上更乾淨的 SSOT 是把 page_type registry 放 DB:

```sql
CREATE TABLE page_type_registry (
  page_type TEXT PRIMARY KEY,
  slug TEXT UNIQUE NOT NULL,
  group_name TEXT,
  priority NUMERIC,
  changefreq TEXT,
  -- 5 locales × 2 brand_type × {label, desc} = 20 columns?
);
```

我們明確選**不這樣做**,理由:

- page_type 變動頻率低(平均 2 個月 1 次),不需要 admin UI 即時改
- 5 locales × 2 brand_type × {label, desc} = 20 column 太寬,實際存 JSONB 又失型別檢查
- code 改有 PR review、git history、vitest test;DB 改沒這些保障
- 「能用 code 表達就用 code,不要往 DB 塞」是工程偏好

代價:新加 page type 變動 4 檔的負擔。但相比「DB 雜湊欄位 + admin UI 維護」的工程成本,4 檔 code 改更好。

---

## 16.4 Alerts 4-table UNION:跨表 SSOT 的型別陷阱

### 16.4.1 統一警報中心架構

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

### 16.4.2 真實事故:UNION 型別不匹配

PROD 觀察:警報中心顯示「未讀 (165)」但列表完全空白。所有 tab 都不渲染。

PG 真實 error:

```text
ERROR: UNION types uuid and bigint cannot be matched
LINE 18:   SELECT ah.id AS id, ...  FROM axp_health_alerts ah
                  ^
```

整個 list query throw → controller catch err → 前端 `setAlertsList([])` → 列表永遠空。這個 bug 一直存在(自 axp_health_alerts 表加進 UNION 那天起),但 UI 在 v3.29.5 之前沒人注意。

### 16.4.3 為什麼這個 bug 存在 N 個月沒人發現

事後分析發現原因:

- 警報中心的「未讀 count」用獨立 query(只 count 不 UNION),所以 count 正常顯示「165」
- list query throw 後,前端 `try { ... } catch { setAlertsList([]) }` 默默吞錯誤
- 看到「165 未讀但 0 顯示」第一直覺是「啊應該還沒生成 row」而不是「query 壞了」
- 沒有 monitoring 對 alerts list query 的 success rate(只看 HTTP 200/500,catch 後仍 200 OK)

教訓:**catch 後吞錯誤是極危險的 anti-pattern**。應該:

```js
// 錯誤示範
try { setAlertsList(await fetch(...)) } catch { setAlertsList([]) }

// 正確
try { setAlertsList(await fetch(...)) }
catch (err) {
  console.error('[alerts list] failed:', err);
  setError(err);  // UI 顯示「載入失敗,請重試」
  Sentry.captureException(err);  // 上報 monitoring
}
```

### 16.4.4 修法:全 cast 為 text

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

### 16.4.5 連帶修法:stats / markAllRead 同步補 4 表

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

### 16.4.6 Vitest regression 鎖死

`backend/src/__tests__/unifiedAlerts.queries.test.js` 26 個 test 鎖 SQL 結構:

- list query SQL 必含 4 個 `id::text`(防回退 type mismatch)
- count CTE / stats / markAll 必含 4 個 source table
- batchMarkRead UUID-only 走 3 calls,mixed UUID+bigint 走 4 calls
- LATERAL JOIN repair_actions cast `::text` 必存在

範例 test:

```javascript
import { describe, it, expect } from 'vitest';
import { readFileSync } from 'fs';

const sql = readFileSync('src/db/queries/unifiedAlerts.queries.js', 'utf8');

describe('unifiedAlerts.queries.js SSOT regression', () => {
  it('list query has 4 id::text casts (防回退 UNION type mismatch)', () => {
    const matches = sql.match(/\bid::text\b/g) ?? [];
    expect(matches.length).toBeGreaterThanOrEqual(4);
  });

  it('list query UNIONs 4 source tables', () => {
    expect(sql).toMatch(/FROM\s+alerts\s+a\b/);
    expect(sql).toMatch(/FROM\s+answer_alerts\s+aa\b/);
    expect(sql).toMatch(/FROM\s+brand_identity_alerts\s+bi\b/);
    expect(sql).toMatch(/FROM\s+axp_health_alerts\s+ah\b/);
  });

  it('repair_actions LATERAL JOIN casts source_id::text', () => {
    expect(sql).toMatch(/repair_actions[\s\S]*?source_id::text\s*=\s*u\.id/);
  });

  // ... 23 more tests
});
```

**任何 PR 退回這些 v3.29.5 收尾的 modification 立即 CI fail**。

### 16.4.7 SSOT 反思

`alerts` 系列 4 表結構不同步是 schema design debt:

- 3 個 UUID 加 1 個 bigint 是「按時間自然演化」造成
- 統一改 bigint or 統一改 uuid 都需要大規模 migration
- 短期解:UNION 內 cast text(本章修法)
- 長期解:`alerts_unified` 替換表,4 source 寫進去時統一 UUID

長期方案還沒排上,因為短期 cast 有效且 regression test 防再犯。這是工程上「能跑就好」的常見 trade-off,記錄下來避免未來工程師覺得這是隨意 hack。

值得提的是,這個 trade-off 適用於現階段(< 100 brand,alerts 數量 ~165 / brand);但若客戶數 > 1000,單表 1.5M+ row,UNION 性能可能成 bottleneck,**那時必須改 alerts_unified 表**(JOIN vs UNION 效能差 10x+)。何時該改:可從 `EXPLAIN ANALYZE` 看 UNION 階段 cost > 100ms 即為信號。

---

## 16.5 SSOT 設計模式總結

三個案例提煉出可推廣的 SSOT 設計模式:

### 16.5.1 Pattern 1 — 單寫入點 + 多讀取路徑

`brand_faq` 範例。所有讀取路徑共用 SQL query,**沒有 cache / derived table 在中間**(避免 cache 跟 source 失同步)。代價是讀取時打 DB,但 1 萬租戶 scale 下打 DB 是 sub-ms,讀 cache 反而需要 invalidation 機制。

適用情境:讀取頻率不高(每次 home page SSR ≈ 客戶數量 / 30s)、寫入頻率低(客戶 N 天才改 FAQ)、強一致性要求(Google 一抓就索引)。

不適用情境:讀取極高頻(每秒 N 萬次,例如 trending 排行)、寫入極高頻(每秒 N 萬次,例如點擊計數)。這類場景必須有 cache layer,但要設計 explicit invalidation。

### 16.5.2 Pattern 2 — Registry 物件 + 4 處同步

`page_type` 範例。23 個 page type × 4 處同步 = 92 個資料點,但**每個資料點都有「能 derive」與「不能 derive」的拆分**:

- 能 derive 的(slug ↔ page_type 反查)→ 自動化(一個 `Object.fromEntries`)
- 不能 derive 的(5 locales × 23 標籤翻譯)→ PR checklist 強制

代價是新加 page type 變動 4 檔的負擔。但實際 page type 添加頻率低(平均 2 個月 1 次),負擔可承受;若改 weekly 添加,就要重新設計成「DB driven 而非 code driven」(放 `page_type_registry` 表,admin UI 編輯)。

### 16.5.3 Pattern 3 — 跨表 UNION 必檢型別

`alerts` 範例。UNION ALL 跨多張表時,**任何欄位型別不一致都會整批 throw**,而且 PG error 訊息對前端工程師不友善(`UNION types uuid and bigint cannot be matched` 不會直接想到要 cast)。

防禦設計:

1. UNION 寫法統一加 cast(便宜的保險)
2. 寫 vitest test grep 確認所有 UNION CTE 都有 cast
3. schema 演進時注意:新加跨表 ID 欄位優先用 UUID(跟舊表一致),除非有強理由用 bigint

### 16.5.4 Pattern 4 — DB-level 強制比 application-level 可靠

placeholder guard / `sync_wiki_page_sources` trigger / `axp_pages_placeholder_guard` 範例。

application-level 強制(每次 INSERT 前 application 檢查):

- ❌ admin UI 直接走 SQL 編輯時繞過(走 psql 操作 PROD 偶爾會發生)
- ❌ generator 寫 raw SQL `INSERT INTO axp_pages` 繞過 service layer
- ❌ pipeline cron 老 code 走 service 但 service 沒檢查

DB trigger(本層做的):

- ✅ 任何路徑寫入都被攔截(包括 psql 直接 INSERT)
- ✅ application code 可以放心(trigger 是兜底)
- ✅ 規格更穩定(trigger 規則只有 SQL,不容易誤改)

代價:trigger debug 困難(error message 不清),但 1 萬租戶 scale 下這個代價值得。

適用情境:強約束(必須執行,不可繞過)、邏輯簡單(SQL 表達得出)、應用層多來源(無法集中強制)。

---

## 16.6 觀察與限制

### 16.6.1 SSOT 的代價是寫入流程變慢

寫入 `brand_faq` 需要 invalidate 多個 cache(Next.js ISR、CF Worker edge、CDN)。若 SSOT 沒 cache,讀取慢但寫入立即生效;有 cache,讀取快但寫入要等 invalidation。我們選後者(Next.js `revalidateTag(tag, 'max')`),但 invalidation 偶發失敗 → SSOT 在 cache layer 失同步。**SSOT 不是消除不一致,是把不一致集中到一個明確邊界**。

### 16.6.2 4 處同步的人為錯誤

page_type SSOT 4 處同步,人為改一處漏改其他 3 處的事故觀察過 3 次。檢測機制:

- vitest test rounds(每次 npm test)
- weekly_backfill cron(全 brand 重析時若發現未知 page_type warning)
- admin sitemap audit(`/admin/sitemap-stats` 顯示「未列入 sitemap 的已發佈 axp page」)

理論上應該寫成 schema constraint(DB enum),但每加 page type 要 ALTER TYPE 不能 hot-add,所以維持 application-level enforcement。

### 16.6.3 跨 host 一致性未達 100%

`geo.baiyuan.io` Mozilla 21Q vs Bot 20Q(差 1 題)是已知 minor 不一致,未修因為:

- 不影響 Google rich result(只 1 題)
- 修需要動 schemaJson generator 的過濾規則,但該規則對某些客戶有意義(例如 answer < 50 字過濾)

「修 vs 留」永遠是 trade-off,沒有純粹理想的 SSOT。

### 16.6.4 alerts 表結構統一還沒做

短期 UNION cast 有效,但長期 `alerts_unified` 替換表還沒排上。如果未來再加第 5 種警報來源,bigint+uuid 混用會繼續造成新的 UNION 陷阱。

### 16.6.5 SSOT 跨微服務的邊界

`brand_faq` 表在 geo_db,但 RAG 微服務的 `tenant_documents` 在 cs_rag_db。如果未來 brand_faq 也要進 RAG 編譯,**跨服務 SSOT 邊界是新的挑戰**。

可能解法:

- **outbox pattern**:brand_faq 寫入時 emit event,RAG service consume
- **CDC(change data capture)**:Postgres Logical Replication 把 brand_faq 變動 stream 給 RAG
- **dual-write**:application 同時寫兩 DB(最弱,容易失同步)

目前 RAG 沒讀 brand_faq,所以這問題還沒爆出來。但 V2 規畫中,brand_faq 進 RAG 是 roadmap,需要先確定跨微服務 SSOT 怎麼維護。

### 16.6.6 SSOT 對開發速度的影響

導入 SSOT 設計確實**降低短期開發速度** — 加新功能要先想清楚「誰是 source、誰是 derived」,不能直接在新功能裡 hardcode。觀察 3 個月,工程師抱怨頻率最高的是:

- 加新 page type 要動 4 檔(過去只動 1 檔)
- 改 FAQ 內容測試要等 cache invalidate(過去直接看)
- alerts UI 改 query 要過 26 個 vitest test

但長期收益:過去 3 個月「SSOT 違反」事故只發生 5 次,前 3 個月(無 SSOT)發生 18 次。**短期慢 → 長期穩**,這是規模化必要的取捨。

---

## 16.7 工程教訓

從三個案例(brand_faq / page_type / alerts)+ N 次 PROD 故障,整理出 5 條教訓:

### 16.7.1 cache invalidation 是 SSOT 的真正敵人

「我們有 SSOT」這句話只在「single 寫入 + immediate 讀取」時為真。一旦加 cache,SSOT 就變成「最終一致」而非「立即一致」。

防禦:

- 識別「強一致性需求」場景(Google 索引、客戶看到自己內容、Schema.org 對 LLM)
- 這些場景**禁止 cache**,直接打 DB
- 「最終一致即可」場景才加 cache,並設計 explicit invalidation
- invalidation 失敗要有 alert(不能假設它永遠成功)

### 16.7.2 任何 silent catch 都是違反 SSOT

alerts UI 「未讀 165 但列表空白」事故的根因是 `try { ... } catch { setList([]) }`——**catch 後吞錯誤,讓 UI 顯示「成功但空」**,看起來跟「真的沒資料」沒區別,bug 沉默 N 個月。

防禦:

- catch 必 log(`console.error` + Sentry)
- catch 必 setError state,UI 顯示「載入失敗」而非「無資料」
- monitoring 對 query failure rate 要有 alert(不只看 HTTP 200/500)

### 16.7.3 「為了直觀」的自定義 mapping 都是 SSOT 違反

breadcrumb 404 ghost 事件的根因是工程員定義 `PAGE_TYPE_BREADCRUMB = { brand_overview: 'brand', ... }`,因為他覺得「`brand` 比 `overview` 更直觀」。但 sitemap 已經有 canonical name 是 `overview`,任何「重新命名」都是 SSOT 違反。

防禦:

- 任何 mapping 寫入 code 前先問:「這 mapping 已經存在於別處嗎?」
- 如果有,必須引用而非重新定義
- code review 時 grep `*_TO_*` 看有無重複 mapping

### 16.7.4 schema 演化必有歷史債

alerts 4 表 UUID + bigint 混用是「schema 按時間自然演化」造成。前 3 表加在同一 sprint(自然用 UUID),第 4 表 axp_health_alerts 加在 6 個月後另一團隊(自然用 bigint)。沒人是錯的,但結果就是 UNION type 不匹配。

防禦:

- 新加表要走 schema review,參考既有同類表的 ID type convention
- DBA 角色(或 senior 工程師代理)必須看過所有新 schema PR
- schema 一致性審查放進 RFC template

### 16.7.5 規模化的代價是放棄一些「直接」

1 萬租戶 SaaS 設計上必須拋棄:

- 「直接 hardcode」:任何場景都應該 ask 「這 1 萬個客戶會不會不一樣」
- 「直接 catch 吞」:錯誤要明確 surface,不能讓 UI 看似正常
- 「直接複製貼上」:重複的概念必須 SSOT,不能各自維護

代價是短期開發慢一點,長期維運穩很多。1 萬租戶 scale 不允許「我事後再修」——客戶 100x 放大任何 bug,客服一天處理 100 件投訴,工程修補的速度永遠跟不上。

**SSOT 不是設計品味,是規模化的生存條件。**

---

## 本章要點

- 1 萬租戶 SaaS 必走 SSOT,任一處不一致會被規模放大成客服災難
- brand_faq SSOT 三方一致(human / Schema.org / AXP shadow),所有路徑共用同一表,no cache 中間層
- page_type 23 種 × 4 處同步 = 92 資料點,自動 derive + PR checklist 拆分管理
- alerts 4-table UNION 必須 cast id::text 防 uuid+bigint 混型別 throw,連帶 stats / markAll / batchRead 補齊
- SSOT 設計四模式:單寫入多讀取 / Registry+多處同步 / 跨表 UNION cast / DB-level trigger 強制
- 限制:cache invalidation 邊角失敗 / 4 處同步偶有人為錯誤 / 跨 host 一致性未達 100% / alerts 結構統一未做 / 跨微服務 SSOT 邊界待解 / 開發速度短期下降
- 5 條工程教訓:cache invalidation 是真敵人 / silent catch 違反 SSOT / 自定義 mapping 是 SSOT 違反 / schema 演化必有歷史債 / 規模化的代價是放棄一些直接

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
| 2026-05-03 | v1.1.1 | 章節擴充至 ~6800 字 — 加 16.1.2 真實違反案例時間線、16.1.3 SSOT 隱形成本、16.2.7 host-aware metadata 工具、16.3.4 breadcrumb 404 ghost 事件回顧、16.3.6 為何不改 DB-driven、16.4.3 為何 bug 沉默 N 個月、16.5.4 Pattern 4 DB-level trigger、16.6.5/6 跨微服務 SSOT + 開發速度影響、16.7 工程教訓 5 條;修正章節編號 15.x → 16.x typo |

---

**導覽**:[← Ch 15: rag-backend-v2 加固](./ch15-rag-backend-v2-hardening.md) · [📖 目次](../README.md) · [附錄 A: 詞彙表 →](./appendix-a-glossary.md)

<!-- AI-friendly structured metadata -->
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "TechArticle",
  "headline": "Chapter 16 — 平台 SSOT 全鏈:從 brand_faq 到 page_type 到 alerts 的單一事實源",
  "description": "1 萬租戶 SaaS 必須把所有跨頁、跨表、跨爬蟲路徑的資料統一在單一事實源。本章記錄 brand_faq SSOT 三方一致、page_type SSOT 4 處同步、alerts 4-table UNION 的完整工程設計。含 5 條工程教訓。",
  "author": {"@type": "Person", "name": "Vincent Lin", "affiliation": "Baiyuan Technology"},
  "datePublished": "2026-05-03",
  "inLanguage": "zh-TW",
  "isPartOf": {
    "@type": "Book",
    "name": "百原GEO Platform 技術白皮書",
    "url": "https://github.com/baiyuan-tech/geo-whitepaper"
  },
  "keywords": "Single Source of Truth, brand_faq, page_type Registry, Schema.org FAQPage, alerts UNION, PostgreSQL UUID bigint, Multi-Tenant Consistency, 工程教訓"
}
</script>
