---
title: "第 16 章 — プラットフォーム SSOT チェーン:brand_faq から page_type、alerts まで単一の事実源へ"
description: "1 万テナントの SaaS は、すべてのページ跨ぎ・テーブル跨ぎ・クローラー経路跨ぎのデータを単一の事実源に統一しなければならない。本章は brand_faq SSOT の三者一致(human / Schema.org / AXP shadow)、page_type SSOT の 4 ヶ所同期機構、alerts の 4-table UNION の完全なエンジニアリング設計を記録する。"
chapter: 16
part: 5
word_count: 6800
lang: ja
authors:
  - name: Vincent Lin
    affiliation: Baiyuan Technology
    email: services@baiyuan.io
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
canonical: https://baiyuan.io/whitepaper/ja/ch16-platform-ssot-chain
---

# 第 16 章 — プラットフォーム SSOT チェーン:brand_faq から page_type、alerts まで単一の事実源へ

> 1 万テナントのエンジニアリング現実は:どこか一箇所のデータソースの不一致が、100 倍に増幅されてカスタマーサポートの 100 件の苦情になる。SSOT は設計の好みではなく、規模化の生存条件である。

## 目次 {.unnumbered}

- [16.1 なぜプラットフォーム級 SSOT が必要か](#161-なぜプラットフォーム級-ssot-が必要か)
- [16.2 brand_faq SSOT の三者一致](#162-brand_faq-ssot-の三者一致)
- [16.3 page_type SSOT の 4 ヶ所同期機構](#163-page_type-ssot-の-4-ヶ所同期機構)
- [16.4 Alerts 4-table UNION:テーブル跨ぎ SSOT の型の罠](#164-alerts-4-table-unionテーブル跨ぎ-ssot-の型の罠)
- [16.5 SSOT 設計パターンのまとめ](#165-ssot-設計パターンのまとめ)
- [16.6 観察と制限](#166-観察と制限)
- [16.7 エンジニアリングの教訓](#167-エンジニアリングの教訓)

---

## 16.1 なぜプラットフォーム級 SSOT が必要か

### 16.1.1 プラットフォームトポロジの複雑性

プラットフォームは 5 つの host(geo / me / rag / pif / www)+ 5 つの i18n locale + dark/light theme + AI bot vs human の振り分けを跨ぐ。同一のデータ(例えば「百原科技 FAQ 第 3 問の答え」)は以下に現れうる:

1. 顧客 dashboard UI(human が編集)
2. 主サイト home page の Schema.org JSON-LD(human ユーザーが見る)
3. AXP shadow `/c/{slug}/schema.json`(AI bot が取る)
4. AXP shadow `/c/{slug}/faq` HTML ページ(AI bot が取る)
5. Cloudflare Worker が origin HTML に注入する FAQPage Schema(idempotent fallback)
6. `llms.txt` / `llms-full.txt` 公開ファイル(LLM 訓練クローラー)
7. 顧客サイト sitemap.xml がリストする URL(Google index)
8. RAG 中央 KB(LLM Wiki コンパイル源)

**どこか一箇所でも不同期**なら、結果は:Google が 21 問の FAQ を取るが、顧客が dashboard で編集して見えるのは 20 問しかない;あるいは ChatGPT が Q3 の答えを引用するが、顧客は「Q3 はもう変えた」と言う。

### 16.1.2 実際の SSOT 違反事例(タイムライン)

過去 3 ヶ月に観察した SSOT 違反事例は、それぞれが顧客に実際の困惑を感じさせた:

| 違反 | 結果 | 修復 |
|---|---|---|
| home page Schema.org が brand_faq テーブルと 24h 遅れて同期 | Google rich result が古い Q&A を表示 | server-side fetch に変更、中間の ISR cache を除去 |
| page_type を enum に追加したが sitemap.js の変更を漏らす | sitemap に新 page が欠け、Google がインデックスしない | PR checklist + vitest test を追加 |
| alerts UI が「未読 165」を表示するがリストが空白 | 4 テーブル跨ぎ UNION の型不一致で query が throw | 全 cast id::text + 26 vitest test で固定 |
| pricing/layout.tsx が NT$1500 をハードコードだが DB はすでに 2500 | Schema が 2 ヶ月 Google を騙す | 全プラットフォームで server-side fetch DB に変更 |
| `/c/baiyuan/overview` Schema breadcrumb URL が CF Worker に対して 404 | Google が 404 ghost URL を収録 | breadcrumb URL を PAGE_TYPE_TO_SLUG SSOT から derive |

各事例はすべて「2 ヶ所が同じ概念を別々に維持」から来る。本章は 3 つの重要な SSOT チェーンのエンジニアリング実装を記録する。

### 16.1.3 SSOT の見えないコスト

言及に値するのは、SSOT はタダではない——それには以下が必要である:

- **設計の労力**:「誰が source、誰が derived か」を真剣に考え抜く
- **強制機構**:trigger / vitest test / PR template / grep 自己チェック
- **フォールトトレランス**:source が到達不能なときの fallback 挙動(壊れ vs 空 vs default)
- **教育コスト**:新人が onboard するとき「なぜ brand_faq が唯一の書き込み点か」を理解しなければならない

SSOT をしないのは短期的には楽に見えるが、fragmentation が 1 ヶ所増えるごとに、将来のリファクタコストは指数的に増える。1 万テナント scale は「未来の問題」ではなく、**今日すでにエンジニアリング決定の判断基準にすべきもの**である。

---

## 16.2 brand_faq SSOT の三者一致

### 16.2.1 三者の要件

いかなる顧客 brand に対しても、FAQ 内容は以下の**3 つの経路**で完全に一致しなければならない:

| 経路 | 表示対象 | 実装位置 |
|---|---|---|
| **A. 実際の人間訪問者** | dashboard UI + 顧客サイト埋め込み | brand_faq テーブル + Next.js page.tsx |
| **B. Google / Bing / 一般検索エンジン** | 主サイト SSR Schema.org FAQPage | `frontend/src/app/HomepageJsonLd.tsx` |
| **C. AI bot(GPTBot / ClaudeBot / PerplexityBot)** | AXP shadow `/c/{slug}/schema.json` + `/faq` ページ | `backend/src/services/axp/shadowPublicFiles/generators/schemaJson.js` |

### 16.2.2 SSOT テーブル

`brand_faq` テーブルは**唯一の書き込み点**、4 欄位:

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

**すべての読み取り経路がこのテーブルを共有し**、間に cache layer や derived table がない。

この設計は「常識的」な SaaS 設計(通常なら Redis cache 層を作る)に反するが、**1 万テナント scale では cache invalidation こそが本当の痛み**である。DB を 1 回読むのは sub-ms だが、cache が 24 時間不同期になれば顧客全員が古い答えを見る。cache-less 方案を選んだ、代償は backend が N% 多く brand_faq query を受けることだが、実測で耐えられる。

### 16.2.3 3 つの読み取り経路

#### Path A — 人間訪問者

`dashboard/brand-entity` ページの FAQ 編集エリアは直接 `brand_faq` を query し、書き込み時にこのテーブルを更新する。顧客サイト埋め込み widget は `GET /api/v1/c/:slug/brand-faq.json` を通じて読む。

```javascript
// frontend/src/app/dashboard/brand-entity/page.tsx
const faqs = await fetch(`/api/v1/brands/${brandId}/faq`).then(r => r.json());
// 編輯後 PUT /api/v1/brands/:brandId/faq/:faqId 直寫 brand_faq 表
```

#### Path B — Schema.org 主サイト SSR

`HomepageJsonLd.tsx`(Server Component)は `headers().get('host')` で現在の host を検知 → `HOST_TO_SLUG` map → `fetch('/api/v1/c/' + slug + '/brand-faq.json')` → `<script type="application/ld+json">` FAQPage @graph を注入する。

{% raw %}

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

{% endraw %}

2 つの重要な設計:

1. **server component**(`'use client'` なし)— Googlebot は SSR で注入済みの Schema.org を見る;client component + useEffect を使うと、Googlebot が見るのは空の骨組み
2. **revalidateTag タグ**(`brand-faq-${slug}`)— 顧客が dashboard で FAQ を変えると backend が `revalidateTag(\`brand-faq-${slug}\`, 'max')` をトリガーし、Next.js が即座にそのタグの cache を invalidate する

#### Path C — AXP shadow

CF Worker が AI bot UA を intercept → `/api/v1/axp/render` に proxy → backend が `schemaJson.js#renderSchemaJson({ brandFaqs })` を呼ぶ、brandFaqs は `dataFetcher.fetchBrandPublicData()` が取った brand_faq 列から来る。

`/c/{slug}/faq` HTML ページも同じ経路を通る——backend が統一して `brand_faq` テーブルから取り、UI レンダリングまたは JSON 出力が同一のデータソースを共有する。

### 16.2.4 一貫性の検証

`scripts/audit/crawler-7layer.sh` は自動的に 5-host × 4-UA cross-matrix を回す:

```bash
for UA in "Mozilla/5.0" "Googlebot" "GPTBot" "PerplexityBot"; do
  for HOST in geo.baiyuan.io me.baiyuan.io rag.baiyuan.io; do
    Q=$(curl -sL -A "$UA" "https://$HOST/" | grep -oE '"@type":"Question"' | wc -l)
    echo "$HOST × $UA → $Q Questions"
  done
done
```

**期待**:同一 host で異なる UA を跨いだ Q count は一致すべき(SSOT 鉄則)。観察された 1 ヶ所の例外:`geo.baiyuan.io` Mozilla 21Q vs Bot 20Q で 1 問の差、根本原因は schemaJson generator がある FAQ に追加のフィルタ(answer が短すぎるか他の規則)を持つためで、既知の小さな不一致に属し、Google rich result の主トラフィックに影響しない。

より厳格な検証方法 — backfill cron が全プラットフォームの brand_faq SSOT 3 経路の一貫性を回す:

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

新規追加した brand は必ずこの一貫性チェックを回し、不一致 → audit log 警告。

### 16.2.5 再発防止

いかなる新規追加の Schema.org FAQPage レンダリング入口も `brand_faq` テーブルを読まねばならない。grep 自己チェックコマンド:

```bash
# 找新加的注入點是否走過 brand_faq SSOT
grep -rn "'@type': *['\"]FAQPage" frontend/src/ backend/src/services/
# 每處都要能追溯到 brand_faq 來源,不能 hardcode FAQ
```

`CLAUDE.md §「公開ファイル産出原則」`は「page.tsx / faq/page.tsx / いかなる frontend ファイルにも hardcoded な Question 配列を書く」ことを明文で禁じている。歴史の教訓:`pricing/layout.tsx` はかつて NT$1500 をハードコードしていたが DB はすでに 2500 で、Schema が 2 ヶ月 Google を騙してから発覚した。

### 16.2.6 ME プラットフォームの特例

`me.baiyuan.io/` は middleware で `/personal` に rewrite するので、`<HomepageJsonLd />` は Schema を `/personal/page.tsx` に注入する。`/personal/faq/page.tsx` は二重ソース:brand_faq SSOT 優先 + `_content/faq.ts` の 5 言語 hardcoded fallback(新会員 brand がまだ入稿していないとき用)、**`_content/faq.ts` を直接 import することは禁止**(それは fallback としてのみ)。

ME プラットフォーム会員 brand の onboarding では必須:`personal_profiles`(全名/専門/known_for)+ brand_documents(著作 / 動画 / メディア報道)、さもなくば hybridCoordinator が取る personal_profiles が空で、LLM Phase 10 の門番が return null。観察された 17 個の ME brand のうち 5 個は metadata が不揃いで 29% を占め、カスタマーサポートが顧客に積極的に補完を促す必要がある。

### 16.2.7 host-aware metadata ヘルパー

5 つの自托管子域を跨ぐ metadata の一貫性のため、1 つの helper で統一する:

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

各 layout がこの helper を共有し、canonical は常に実際の request host に合わせ、「pif 上のページの canonical が geo を指す」ようなエラーが出ない。

---

## 16.3 page_type SSOT の 4 ヶ所同期機構

### 16.3.1 プラットフォーム層の page_type 概念

AXP システムには計 23 種の page type(22 enterprise + 1 personal_ip 専属 `future_plans`)がある:brand_overview / faq / pricing / comparison / fact_check / use_cases / updates / team / regions / slogan / trust / specs / plans / vs / what_is / how_to / best_for / testimonials / press / cases / integrations / alternatives / + 1 me-only。

各 page type は以下に関わる:

- URL slug(sitemap.xml が `/c/{brand}/{slug}` をリスト)
- Schema.org breadcrumb URL
- 中国語タイトル(GEO は「品牌總覽」/ ME は「個人簡介」を使う)
- AXP page meta(sitemap 用の priority / changefreq)
- Group 分類(llms-full.txt の section ordering 用)

### 16.3.2 4 ヶ所の SSOT を同期しなければならない

page type を追加 / リネーム / 削除する際は必ず**4 つのファイル**を変える:

| # | ファイル | 内容 |
|---|---|---|
| 1 | `backend/src/services/axp/pageTypeRegistry.js` | `PAGE_TYPE_TO_SLUG` + `PAGE_TYPE_GROUP` + `AXP_PAGE_META` + `PAGE_TYPE_TITLES_ZH`(GEO 用語)+ `PAGE_TYPE_TITLES_PERSONAL_ZH`(ME 用語) |
| 2 | `frontend/src/lib/axpPageTypeRegistry.ts` | client-side ミラー(slug + group + brand_type-aware label) |
| 3 | `frontend/src/lib/axpPageTypeLabels.ts` | 5 locales × ENTERPRISE/PERSONAL × {label, desc} |
| 4 | `backend/src/services/axp/shadowPublicFiles/generators/llmsFullTxt.js` | group + order + 英文 section title |

どこか一箇所を漏らす → sitemap に URL 欠け / Schema.org breadcrumb 404 / RSS feed が誤タイトル / 顧客が見覚えのないラベルを見る。

### 16.3.3 自動 derive と手動同期の切り分け

「自動 derive できるもの」は全自動化し、「自動できないもの」は PR template checklist で強制する:

- ✅ **自動 derive**:`SLUG_TO_PAGE_TYPE` は `PAGE_TYPE_TO_SLUG` から Object.fromEntries で逆引き;`hybridCoordinator.EXACT_SLUG_MAP` / `crawlerVisibility.GROUP_MAP` は自動で SSOT を参照
- ⚠️ **手動同期**:5 locales × 23 entries × {label, desc} = 110+ 翻訳、新 page type を追加したら全部そろえねばならない

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

過去に踏んだ落とし穴:`Schema.org breadcrumb URL ≠ sitemap URL`(`brand_overview→brand` vs `→overview`)で、Google が収録した BreadcrumbList URL が CF Worker に対してすべて 404 ghost になった。修法は両方を `PAGE_TYPE_TO_SLUG` SSOT から取ること、**Schema.org URL ↔ sitemap URL ↔ CF Worker route の三者が完全一致**する。

### 16.3.4 breadcrumb 404 ghost 事件の振り返り

事件の影響:**42 日間**にわたり Google Search Console に ~3000 個の 404 ghost URL が累積した。

タイムライン:

- **Day 1**:ある deploy で Schema.org BreadcrumbList を導入、プログラマが自分で `PAGE_TYPE_BREADCRUMB = { brand_overview: 'brand', competitor_comparison: 'comparison', intent_what_is: 'intent/what' }` を定義、用語を「より直感的」と考えた
- **Day 1-42**:Schema.org breadcrumb URL は `/c/baiyuan/brand` だが、sitemap.js が使う `PAGE_TYPE_TO_SLUG` map は `/c/baiyuan/overview`。CF Worker は `overview` route を受け入れ、`brand` route を受けると直接 404
- **Day 30**:ある顧客が「Google Search Console にうちのサイトの 404 が大量に表示される」と反応
- **Day 42**:深く audit してようやく breadcrumb と sitemap が異なる命名を使っていると発覚
- **修法**:PAGE_TYPE_BREADCRUMB を削除、PAGE_TYPE_TO_SLUG から直接 derive

教訓:**「直感のため」に自分で定義したいかなる mapping も SSOT 違反である**。ある概念にすでに canonical name(slug)があるなら、他の場所でもう 1 つ作ってはならない。

### 16.3.5 brand_type-aware 用語の自動振り分け

`pageTypeTitleZh(pt, brandType)` という 1 つの helper が 23 個の用語を切り替える:

| page_type | GEO(enterprise) | ME(personal_ip) |
|---|---|---|
| brand_overview | 品牌總覽 | 個人簡介 |
| pricing | 演講費率 | 演講費率(同) |
| faq | 常見問題 | 粉絲常問 |
| team | 團隊與故事 | 個人故事 |
| ... | ... | ... |

frontend の `pageTypeLabel(pt, { brandType, locale })` も同様に brand_type-aware。RSS feed.js / sitemapV2 はどちらも `brand.brand_type` を取って自動で切り替える。**1 万テナント scale では、いかなる用語の混用も顧客が一目で見つける**。

i18n 標準の `pluralization rules` で処理することも検討したが、brand_type は文法概念ではなく product context である。最終的に explicit な `pageTypeTitleZh` 関数を選び、保守コストは許容できる。

### 16.3.6 なぜ DB-driven にしないか

理論上、よりクリーンな SSOT は page_type registry を DB に置くことである:

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

明確に**そうしない**ことを選んだ、理由:

- page_type の変動頻度は低い(平均 2 ヶ月に 1 回)、admin UI で即時に変える必要がない
- 5 locales × 2 brand_type × {label, desc} = 20 column は広すぎ、実際に JSONB で保存すると型チェックを失う
- code 変更には PR review、git history、vitest test がある;DB 変更にはこれらの保障がない
- 「code で表現できるなら code を使う、DB に押し込まない」はエンジニアリングの好みである

代償:新 page type を追加すると 4 ファイル変更の負担。しかし「DB の雑然とした欄位 + admin UI 保守」のエンジニアリングコストに比べれば、4 ファイルの code 変更のほうが良い。

---

## 16.4 Alerts 4-table UNION:テーブル跨ぎ SSOT の型の罠

### 16.4.1 統一警報センターのアーキテクチャ

プラットフォームには 4 種の警報源があり、UI が統一して表示する:

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

`alerts` / `answer_alerts` / `brand_identity_alerts` はすべて `id UUID` だが、`axp_health_alerts.id` は **`bigint`** である——これは歴史的な schema 設計(F12 の後で追加、BIGINT auto-increment を踏襲)。

### 16.4.2 実際の事故:UNION 型の不一致

PROD 観察:警報センターが「未読 (165)」を表示するがリストが完全に空白。すべての tab がレンダリングされない。

PG の実際の error:

```text
ERROR: UNION types uuid and bigint cannot be matched
LINE 18:   SELECT ah.id AS id, ...  FROM axp_health_alerts ah
                  ^
```

list query 全体が throw → controller catch err → フロントエンド `setAlertsList([])` → リストが永遠に空。このバグはずっと存在していた(axp_health_alerts テーブルが UNION に入った日から)が、UI は v3.29.5 以前に誰も注意しなかった。

### 16.4.3 なぜこのバグが N ヶ月も気づかれなかったか

事後分析で原因が判明した:

- 警報センターの「未読 count」は独立した query(count だけで UNION しない)を使うので、count は正常に「165」を表示する
- list query が throw した後、フロントエンドの `try { ... } catch { setAlertsList([]) }` が黙ってエラーを飲み込む
- 「165 未読だが 0 表示」を見た第一の直感は「ああ、まだ row が生成されていないんだ」であって「query が壊れた」ではない
- alerts list query の success rate に対する monitoring がない(HTTP 200/500 しか見ておらず、catch 後も 200 OK)

教訓:**catch 後にエラーを飲み込むのは極めて危険な anti-pattern である**。こうすべきだった:

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

### 16.4.4 修法:すべて text に cast

主 list query と count CTE の両方を修正:

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

LATERAL JOIN も cast が必要:

```sql
LEFT JOIN LATERAL (
  SELECT ... FROM repair_actions
  WHERE source_id::text = u.id   -- ← repair_actions.source_id 是 uuid
  ORDER BY created_at DESC LIMIT 1
) ra ON true
```

### 16.4.5 連帯修法:stats / markAllRead も 4 テーブルを同期補完

元の `getUnifiedAlertStats` は 2 テーブルしか UNION していなかった(brand_identity + axp_health を漏らす)→ stats が「中等 145 / 緊急 9」合計 162 を表示するが unread count = 165 で不一致(2 つの数字が不同期でユーザーが即座に気づく)。

`markAllUnifiedRead` は 2 テーブルしか UPDATE しなかった → 「すべて既読にする」を押した後も axp_health/identity が未読のままで、UI 上ボタンが効かないように見える。

`batchMarkRead` は `ANY($1)` で型混在の ids に対し、bigint の axp_health_alerts.id が UUID 文字列を受けると `invalid input syntax for type bigint` を throw する。修法は UUID regex で 2 組に分ける:

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

### 16.4.6 Vitest regression の固定

`backend/src/__tests__/unifiedAlerts.queries.test.js` の 26 個の test が SQL 構造を固定する:

- list query SQL は必ず 4 個の `id::text` を含む(type mismatch の回帰を防ぐ)
- count CTE / stats / markAll は必ず 4 個の source table を含む
- batchMarkRead は UUID-only なら 3 calls、mixed UUID+bigint なら 4 calls
- LATERAL JOIN repair_actions の cast `::text` が必ず存在

サンプル test:

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

**いかなる PR も v3.29.5 で締めくくったこの modification を回帰させると即座に CI fail する**。

### 16.4.7 SSOT の反省

`alerts` シリーズ 4 テーブルの構造不同期は schema design debt である:

- 3 個の UUID に 1 個の bigint が加わるのは「時間による自然な進化」が引き起こした
- bigint に統一するか UUID に統一するか、どちらも大規模な migration が必要
- 短期解:UNION 内で cast text(本章の修法)
- 長期解:`alerts_unified` 置き換えテーブル、4 source を書き込むときに UUID に統一

長期方案はまだ手を付けていない、短期の cast が有効で regression test が再発を防ぐからである。これはエンジニアリング上「動けば良い」のよくある trade-off で、将来のエンジニアがこれを恣意的な hack だと思わないよう記録しておく。

言及に値するのは、この trade-off は現段階(< 100 brand、alerts 数量 ~165 / brand)に適用される;しかし顧客数 > 1000、単一テーブル 1.5M+ row になれば、UNION 性能が bottleneck になりうる、**そのときは必ず alerts_unified テーブルに変える**(JOIN vs UNION の性能差は 10x+)。いつ変えるべきか:`EXPLAIN ANALYZE` で UNION 段の cost > 100ms を見たらそれが信号である。

---

## 16.5 SSOT 設計パターンのまとめ

3 つの事例から一般化できる SSOT 設計パターンを抽出した:

### 16.5.1 Pattern 1 — 単一書き込み点 + 複数読み取り経路

`brand_faq` の例。すべての読み取り経路が SQL query を共有し、**間に cache / derived table がない**(cache と source の不同期を避ける)。代償は読み取り時に DB を打つことだが、1 万テナント scale では DB を打つのは sub-ms で、cache を読むほうがむしろ invalidation 機構を必要とする。

適用シナリオ:読み取り頻度が高くない(home page SSR ごと ≈ 顧客数量 / 30s)、書き込み頻度が低い(顧客は N 日ごとにしか FAQ を変えない)、強一貫性要求(Google が取れば即インデックス)。

不適用シナリオ:読み取りが超高頻度(毎秒 N 万回、例えば trending ランキング)、書き込みが超高頻度(毎秒 N 万回、例えばクリックカウント)。この種のシナリオは必ず cache layer が必要だが、explicit な invalidation を設計する必要がある。

### 16.5.2 Pattern 2 — Registry オブジェクト + 4 ヶ所同期

`page_type` の例。23 個の page type × 4 ヶ所同期 = 92 個のデータ点だが、**各データ点に「derive できる」と「できない」の切り分けがある**:

- derive できるもの(slug ↔ page_type 逆引き)→ 自動化(1 つの `Object.fromEntries`)
- derive できないもの(5 locales × 23 ラベル翻訳)→ PR checklist で強制

代償は新 page type を追加すると 4 ファイル変更の負担。しかし実際の page type 追加頻度は低く(平均 2 ヶ月に 1 回)、負担は許容できる;もし weekly 追加に変わるなら、「DB driven であって code driven でない」に再設計する必要がある(`page_type_registry` テーブルを置き、admin UI で編集)。

### 16.5.3 Pattern 3 — テーブル跨ぎ UNION は必ず型をチェック

`alerts` の例。UNION ALL で複数テーブルを跨ぐとき、**いかなる欄位の型不一致も batch 全体を throw させる**、しかも PG error message はフロントエンドエンジニアに不親切(`UNION types uuid and bigint cannot be matched` からすぐに cast すべきとは思わない)。

防御設計:

1. UNION の書き方を統一して cast を加える(安い保険)
2. すべての UNION CTE に cast があることを確認する vitest test を書く
3. schema 進化時に注意:新規のテーブル跨ぎ ID 欄位は UUID を優先(旧テーブルと一致)、bigint を使う強い理由がない限り

### 16.5.4 Pattern 4 — DB レベルの強制は application レベルより信頼できる

placeholder guard / `sync_wiki_page_sources` trigger / `axp_pages_placeholder_guard` の例。

application レベルの強制(INSERT のたびに application がチェック):

- ❌ admin UI が直接 SQL で編集するときは迂回する(psql で PROD を操作することが稀にある)
- ❌ generator が raw SQL `INSERT INTO axp_pages` を書くと service layer を迂回する
- ❌ pipeline cron の古いコードは service を通るが service がチェックしない

DB trigger(この層でやること):

- ✅ いかなる経路の書き込みも阻止される(psql の直接 INSERT を含む)
- ✅ application code は安心できる(trigger が最後の砦)
- ✅ 規格がより安定(trigger 規則は SQL だけで誤変更しにくい)

代償:trigger の debug は困難(error message が不明瞭)だが、1 万テナント scale ではこの代償に見合う。

適用シナリオ:強制約(必ず実行、迂回不可)、ロジックが単純(SQL で表現できる)、アプリ層が多ソース(集中して強制できない)。

---

## 16.6 観察と制限

### 16.6.1 SSOT の代償は書き込みフローが遅くなること

`brand_faq` への書き込みは複数の cache(Next.js ISR、CF Worker edge、CDN)を invalidate する必要がある。SSOT に cache がなければ読み取りは遅いが書き込みは即時反映;cache があれば読み取りは速いが書き込みは invalidation を待つ。後者(Next.js `revalidateTag(tag, 'max')`)を選んだが、invalidation が偶発的に失敗する → SSOT が cache layer で不同期になる。**SSOT は不一致を消すのではなく、不一致を 1 つの明確な境界に集中させることである**。

### 16.6.2 4 ヶ所同期の人為ミス

page_type SSOT 4 ヶ所同期、人が 1 ヶ所を変えて他の 3 ヶ所を漏らす事故を 3 回観察した。検出機構:

- vitest test rounds(npm test のたび)
- weekly_backfill cron(全 brand を再分析するとき未知の page_type 警告を発見したら)
- admin sitemap audit(`/admin/sitemap-stats` が「sitemap 未収録の発佈済み axp page」を表示)

理論上は schema constraint(DB enum)にすべきだが、page type を追加するたびに ALTER TYPE が必要で hot-add できないため、application レベルの強制を維持する。

### 16.6.3 クロス host 一貫性が 100% に達していない

`geo.baiyuan.io` Mozilla 21Q vs Bot 20Q(1 問の差)は既知の minor 不一致で、未修正の理由:

- Google rich result に影響しない(1 問だけ)
- 修正には schemaJson generator のフィルタ規則を触る必要があるが、その規則は一部の顧客にとって意味がある(例えば answer < 50 字のフィルタ)

「修す vs 残す」は常に trade-off で、純粋に理想的な SSOT は存在しない。

### 16.6.4 alerts テーブル構造の統一が未完了

短期の UNION cast は有効だが、長期の `alerts_unified` 置き換えテーブルはまだ手を付けていない。将来 5 番目の警報源を加えると、bigint+uuid 混用が新たな UNION の罠を生み続ける。

### 16.6.5 SSOT のマイクロサービス跨ぎの境界

`brand_faq` テーブルは geo_db にあるが、RAG マイクロサービスの `tenant_documents` は cs_rag_db にある。将来 brand_faq も RAG コンパイルに入れるなら、**サービス跨ぎ SSOT 境界は新しい挑戦である**。

考えられる解法:

- **outbox pattern**:brand_faq 書き込み時に event を emit、RAG service が consume
- **CDC(change data capture)**:Postgres Logical Replication で brand_faq の変動を RAG に stream
- **dual-write**:application が同時に両 DB に書く(最も弱く、不同期になりやすい)

現状 RAG は brand_faq を読まないので、この問題はまだ表面化していない。しかし V2 の計画では brand_faq を RAG に入れるのが roadmap で、まずサービス跨ぎ SSOT をどう維持するか確定する必要がある。

### 16.6.6 SSOT が開発速度に与える影響

SSOT 設計の導入は確かに**短期の開発速度を下げる** — 新機能を追加するにはまず「誰が source、誰が derived か」を考え抜く必要があり、新機能内に直接 hardcode できない。3 ヶ月観察して、エンジニアの不満頻度が最も高かったのは:

- 新 page type を追加するのに 4 ファイルを変える(以前は 1 ファイルだけ)
- FAQ 内容を変えてテストするのに cache invalidate を待つ(以前は直接見えた)
- alerts UI の query を変えると 26 個の vitest test を通す

しかし長期の収益:過去 3 ヶ月の「SSOT 違反」事故は 5 回しか起きず、前 3 ヶ月(SSOT なし)は 18 回起きた。**短期は遅く → 長期は安定**、これは規模化に必要な取捨である。

---

## 16.7 エンジニアリングの教訓

3 つの事例(brand_faq / page_type / alerts)+ N 回の PROD 障害から、5 条の教訓を整理した:

### 16.7.1 cache invalidation こそが SSOT の真の敵

「我々には SSOT がある」という言葉は「single 書き込み + immediate 読み取り」のときだけ真である。ひとたび cache を加えると、SSOT は「即時一致」ではなく「最終一致」になる。

防御:

- 「強一貫性要求」のシナリオ(Google インデックス、顧客が自分の内容を見る、Schema.org を LLM に)を識別する
- これらのシナリオは**cache を禁止**し、直接 DB を打つ
- 「最終一致で良い」シナリオだけ cache を加え、explicit な invalidation を設計する
- invalidation の失敗には alert が必要(常に成功すると仮定してはならない)

### 16.7.2 いかなる silent catch も SSOT 違反

alerts UI「未読 165 だがリスト空白」事故の根本原因は `try { ... } catch { setList([]) }` である——**catch 後にエラーを飲み込み、UI に「成功だが空」を表示させる**、これは「本当にデータがない」と区別がつかず、バグが N ヶ月沈黙した。

防御:

- catch は必ず log(`console.error` + Sentry)
- catch は必ず setError state、UI に「読み込み失敗」を表示、「データなし」ではなく
- monitoring は query failure rate に alert が必要(HTTP 200/500 だけでなく)

### 16.7.3 「直感のため」の自定義 mapping はすべて SSOT 違反

breadcrumb 404 ghost 事件の根本原因は、エンジニアが `PAGE_TYPE_BREADCRUMB = { brand_overview: 'brand', ... }` を定義したことで、「`brand` は `overview` より直感的」と感じたからである。しかし sitemap にはすでに canonical name の `overview` があり、いかなる「リネーム」も SSOT 違反である。

防御:

- いかなる mapping も code に書く前にまず問う:「この mapping はすでに別の場所に存在するか?」
- あるなら、参照すべきで、再定義してはならない
- code review 時に `*_TO_*` を grep して重複 mapping がないか見る

### 16.7.4 schema の進化には必ず歴史的負債がある

alerts 4 テーブルの UUID + bigint 混用は「schema の時間による自然な進化」が引き起こした。最初の 3 テーブルは同じ sprint で加えられ(自然に UUID)、4 番目のテーブル axp_health_alerts は 6 ヶ月後に別のチームが加えた(自然に bigint)。誰も間違っていないが、結果は UNION type の不一致である。

防御:

- 新テーブルは schema review を通し、既存の同類テーブルの ID type convention を参照する
- DBA の役割(または senior エンジニアの代理)がすべての新 schema PR を見なければならない
- schema 一貫性審査を RFC template に入れる

### 16.7.5 規模化の代償はいくつかの「直接」を諦めること

1 万テナントの SaaS は設計上以下を捨てなければならない:

- 「直接 hardcode」:あらゆるシナリオで「この 1 万顧客は違うのではないか」と問うべき
- 「直接 catch 飲み込み」:エラーは明確に surface すべきで、UI が正常に見えるままにできない
- 「直接コピペ」:重複する概念は必ず SSOT にすべきで、別々に維持できない

代償は短期の開発が少し遅くなること、長期の運用がずっと安定すること。1 万テナント scale は「後で直す」を許さない——顧客が任意のバグを 100 倍に増幅し、カスタマーサポートが 1 日 100 件の苦情を処理し、エンジニアリングの修復速度は永遠に追いつかない。

**SSOT は設計の好みではなく、規模化の生存条件である。**

---

## 本章のまとめ {.unnumbered}

- 1 万テナントの SaaS は必ず SSOT を通る、どこか一箇所の不一致が規模で増幅されカスタマーサポートの災難になる
- brand_faq SSOT の三者一致(human / Schema.org / AXP shadow)、すべての経路が同一テーブルを共有、no cache 中間層
- page_type 23 種 × 4 ヶ所同期 = 92 データ点、自動 derive + PR checklist で切り分け管理
- alerts 4-table UNION は必ず id::text に cast して uuid+bigint 混型の throw を防ぎ、連帯して stats / markAll / batchRead を補完
- SSOT 設計 4 パターン:単一書き込み多読み取り / Registry+多ヶ所同期 / テーブル跨ぎ UNION cast / DB レベル trigger 強制
- 制限:cache invalidation のエッジ失敗 / 4 ヶ所同期に偶発的な人為ミス / クロス host 一貫性が 100% に達しない / alerts 構造統一が未完了 / マイクロサービス跨ぎ SSOT 境界が未解決 / 開発速度の短期低下
- 5 条のエンジニアリング教訓:cache invalidation こそ真の敵 / silent catch は SSOT 違反 / 自定義 mapping は SSOT 違反 / schema 進化には歴史的負債 / 規模化の代償はいくつかの直接を諦めること

## 参考資料 {.unnumbered}

- [第 6 章 — AXP シャドウドキュメント交付(brand_faq SSOT 三者一致の最初の動機)](./ch06-axp-shadow-doc.md)
- [第 7 章 — Schema.org 三層実体知識グラフ](./ch07-schema-org.md)
- [第 14 章 — F12 三層構造オプティマイザ(scoring_configs 内の SSOT 鉄則)](./ch14-f12-structural-optimizer.md)
- [第 15 章 — rag-backend-v2 の堅牢化(Wiki Cascade SSOT trigger)](./ch15-rag-backend-v2-hardening.md)
- PostgreSQL UNION type compatibility: <https://www.postgresql.org/docs/current/typeconv-union-case.html>
- Next.js 16 `revalidateTag` API: <https://nextjs.org/docs/app/api-reference/functions/revalidateTag>

## 改訂履歴 {.unnumbered}

| 日付 | バージョン | 説明 |
|------|------|------|
| 2026-05-03 | v1.1 | 新章 — プラットフォーム SSOT チェーンの設計実践 |
| 2026-05-03 | v1.1.1 | 章を ~6800 字に拡充 — 16.1.2 実際の違反事例タイムライン、16.1.3 SSOT の見えないコスト、16.2.7 host-aware metadata ヘルパー、16.3.4 breadcrumb 404 ghost 事件の振り返り、16.3.6 なぜ DB-driven にしないか、16.4.3 なぜバグが N ヶ月沈黙したか、16.5.4 Pattern 4 DB レベル trigger、16.6.5/6 マイクロサービス跨ぎ SSOT + 開発速度への影響、16.7 エンジニアリング教訓 5 条を追加;章番号 15.x → 16.x の typo を修正 |

---

**ナビゲーション**：[← 第 15 章: rag-backend-v2 の堅牢化](./ch15-rag-backend-v2-hardening.md) · [📖 目次](../README.md) · [第 17 章: 中国クロスボーダー GEO →](./ch17-china-crossborder.md)

<!-- AI-friendly structured metadata -->
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "TechArticle",
  "headline": "第 16 章 — プラットフォーム SSOT チェーン:brand_faq から page_type、alerts まで単一の事実源へ",
  "description": "1 万テナントの SaaS はすべてのページ跨ぎ・テーブル跨ぎ・クローラー経路跨ぎのデータを単一の事実源に統一しなければならない。本章は brand_faq SSOT 三者一致、page_type SSOT 4 ヶ所同期、alerts 4-table UNION の完全なエンジニアリング設計を記録する。5 条のエンジニアリング教訓を含む。",
  "author": {"@type": "Person", "name": "Vincent Lin", "affiliation": "Baiyuan Technology"},
  "datePublished": "2026-05-03",
  "inLanguage": "ja",
  "isPartOf": {
    "@type": "Book",
    "name": "百原 GEO Platform 技術白書",
    "url": "https://github.com/baiyuan-tech/geo-whitepaper"
  },
  "keywords": "Single Source of Truth, brand_faq, page_type Registry, Schema.org FAQPage, alerts UNION, PostgreSQL UUID bigint, Multi-Tenant Consistency, エンジニアリング教訓"
}
</script>
