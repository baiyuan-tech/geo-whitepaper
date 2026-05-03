---
title: "Chapter 14 — F12 三層結構優化器:從規則 V1 到雙引擎 v3.1"
description: "F12(TLSO, Three-Layer Structural Optimizer)架構演進史:V1 規則式評分 + LLM rewrite,V3.1 引入 AutoGEO + E-GEO 雙引擎並行 + Hybrid 整合,加上 Plan Gate / Multi-Layer Cache / Tier Priority / Scale Trigger 等 1 萬租戶基礎設施。"
chapter: 14
part: 5
word_count: 7200
lang: zh-TW
authors:

  - name: Vincent Lin
    affiliation: Baiyuan Technology
license: CC-BY-NC-4.0
keywords:

  - F12
  - TLSO
  - Three-Layer Structural Optimizer
  - AutoGEO
  - E-GEO
  - Hybrid Engine
  - Adaptive Router
  - LLM Gateway
  - Plan Feature Gate
  - Multi-Layer Cache
  - Tenant Isolation
  - Scale-up Trigger
last_updated: 2026-05-03
canonical: https://baiyuan.io/whitepaper/zh-TW/ch14-f12-structural-optimizer
last_modified_at: '2026-05-03T03:15:29Z'
---

# Chapter 14 — F12 三層結構優化器:從規則 V1 到雙引擎 v3.1

> 把「AI 引用率優化」從靠玄學調 prompt,變成可量測、可重現、可規模化的工程系統。

## 目錄

- [14.1 為什麼需要 F12](#141-為什麼需要-f12)
- [14.2 V1:三層分析 + LLM Optimizer](#142-v1三層分析--llm-optimizer)
- [14.3 V3.1:雙引擎並行 + 整合策略](#143-v31雙引擎並行--整合策略)
- [14.4 1 萬租戶基礎設施](#144-1-萬租戶基礎設施)
- [14.5 部署、配額、計費三件事](#145-部署配額計費三件事)
- [14.6 觀察到的限制與未解問題](#146-觀察到的限制與未解問題)
- [14.7 工程教訓](#147-工程教訓)

---

## 14.1 為什麼需要 F12

### 14.1.1 Ch 3 評分留下的問題

Ch 3 的七維度評分回答「品牌引用率是多少」,但**不告訴你「下一篇文章該怎麼寫才會被引用率提升」**。商業客戶實際拿著評分問:

> 「我已經知道 AI 引用率 32 分,我下一篇 blog 要怎麼改才會升到 60?」
>
> 「我這篇 fact-check 為什麼 ChatGPT 引用、Gemini 不引用?」
>
> 「同樣的內容換個寫法,引用率能差多少?」

評分系統只負責「測量」,不負責「處方」。把處方丟給 marketing 文案或外包,等於把 SaaS 的核心價值外洩給人類經驗——而人類經驗在 1 萬租戶 scale 下,不可能逐一手調。

### 14.1.2 早期手調 prompt 的三個失敗模式

V1 之前(2026 年 1-2 月),平台有一個「LLM 內容改寫」的 internal tool,工程師 ad-hoc 拿 OpenAI playground 跑客戶內容。三個月內死了:

1. **成本失控**:單一客戶內容平均 4-6 次重跑才滿意,單位 cost ≈ $0.50/篇,30 個客戶 × 22 page_type = 660 次,$330。每月乘 2 完整重新生成 = $660,還沒算 retry。客戶月費 $99 起,毛利為負。
2. **不可重現**:同樣 prompt 隔天跑結果不同(GPT-4 的 temperature 預設 1),客戶問「為何上次這樣寫,這次卻是那樣?」工程師答不出來。
3. **品質退化**:LLM 「優化」後客戶不認得自家內容,有一次 anonymized case:保險業務員的「儲蓄險產品介紹」被 GPT-4 改成「投資建議」(因為 prompt 強調 informativeness),違反金管會規定,客戶差點被檢舉。

F12 名稱取自 Web DevTools 的 F12 鍵——打開瀏覽器開發者工具就能看「結構檢視」。我們把這個隱喻搬到 LLM 引用率優化:**任何內容都應該能被「F12 一下」看到三層結構分數,並按需求重寫。**

### 14.1.3 F12 與 Ch 3 評分的差別

| | Ch 3 評分 | F12 |
|---|---|---|
| 輸入 | 品牌維度 + AI 平台維度 | 一段內容(URL / 文字 / Markdown) |
| 輸出 | 0–100 分 + 訊號分項 | 三層分數 + 優化後內容 |
| 用途 | 「我品牌健康嗎」 | 「這篇文章如何改才會被引用」 |
| 對象 | brand-level | content-level |
| 頻率 | 每日 / 週掃描 | 每篇文章寫入時觸發 |
| 成本 | scan platform call(LLM 1 次/查詢) | analyzer + optimizer(規則 + LLM 視需要) |

**F12 的核心承諾是:給定同一段內容,系統能產出多個變體並選出 LLM 最可能引用的那個**。

### 14.1.4 與市面 SEO 工具的差別

SurferSEO / Clearscope 等 SEO 工具的核心仍是「keyword density / TF-IDF」,目標是 Google 搜尋排名。F12 的目標是 LLM 引用,兩者的訊號不重疊:

- SEO 工具看 keyword 出現次數;F12 看「atomic fact 密度」(數字、日期、來源連結)
- SEO 工具看 backlink quality;F12 看「實體標記」(Schema.org Person/Organization/Service 是否完整)
- SEO 工具看 content length;F12 看「段落能否被 LLM 直接摘錄」(TL;DR、lead-in、清楚 H1)

兩者並非對立——好的 SEO 內容多半也是好的 GEO 內容,但反向不成立。F12 補上的是「LLM-specific」訊號,這是傳統 SEO 工具完全看不到的。

---

## 14.2 V1:三層分析 + LLM Optimizer

### 14.2.1 三層結構的設計

| 層 | 顆粒度 | 檢查內容 |
|---|---|---|
| **Macro** | 文件級 | 是否 ≥1 篇 fact-check / FAQ / comparison?是否有清楚 H1?是否聲明 entity? |
| **Meso** | 段落級 | 段落長度分布、TL;DR 是否在前、是否有 lead-in 句子吸引 LLM 摘錄 |
| **Micro** | 句子級 | 數字 / 日期 / 來源連結密度、實體 (entity) 標記、是否有可被引用的 atomic fact |

每層獨立打分(0–100),最終 `overall = 0.4 × macro + 0.35 × meso + 0.25 × micro`(權重存於 `scoring_configs.f12_thresholds_<brand_type>` SSOT,可由 admin 調整不需改 code)。

**為什麼是三層**:LLM 在「決定引用什麼」時,經驗上看的是三個尺度:

- **文件級**:這篇文章是不是「我能信」的內容(有 fact-check、entity 清楚)?
- **段落級**:有沒有可以直接抓出來的「答案段落」(LLM 喜歡 TL;DR 模式)?
- **句子級**:單一句子是不是「可被引用的事實」(有數字、日期、清楚主詞)?

少了任一層,LLM 引用機率掉 30-50%。我們從 2026 Q1 跑了 3 個月內部實驗,確認三層獨立有效——拿掉 macro,引用率掉 31%;拿掉 meso 掉 47%;拿掉 micro 掉 38%。

### 14.2.2 為什麼 V1 不打 LLM(只規則式)

V1 的分析器是**純規則式**:正規表達 + AST 解析 + 統計密度,沒打 LLM。每秒可分析 30+ 篇文章,適合 batch 全平台掃描。

設計理由:

1. **成本**:30 brand × 22 page = 660 篇/週掃,LLM 分析每篇 $0.02 = $13.2/週,規則式幾乎為 0
2. **可重現**:規則式給定同一段內容永遠同分,LLM 有 ±5 分波動
3. **可審計**:客戶問「為何 macro 分 60」,系統能列出「缺 fact-check / H1 過長 / 沒 entity」具體 issue;LLM 給「感覺不夠權威」答不出來
4. **速度**:規則式 30+ 篇/秒,LLM 0.5 篇/秒,差 60 倍

具體規則例舉(節錄):

```yaml
macro:
  - has_fact_check: 1 if brand has any fact-check page
  - has_clear_h1: 1 if first <h1> within 100 chars and ≥10 chars
  - has_entity_declaration: 1 if Schema.org Person/Org/Service in JSON-LD

meso:
  - avg_paragraph_chars: weighted by closeness to ideal range
  - has_tldr_in_first_300: 1 if "TL;DR" / "簡言之" / "重點" within first 300 chars
  - lead_in_score: count of paragraphs starting with question/statement hook

micro:
  - number_density: count(<digit>+) / total_chars
  - date_density: count(YYYY-MM-DD | YYYY 年 X 月 | etc) / total_chars
  - source_link_density: count(<a href>) / paragraph_count
  - entity_mention_count: count of Schema.org-marked entities in text
```

每條規則的閾值(`paragraph_ideal_min`、`number_density_target` 等)全在 `scoring_configs` SSOT,admin 不改 code 就能調。我們踩過的雷:閾值寫死在 code,改一次 deploy + 重啟,客戶要等 30 分鐘才看到新分數;改成 SSOT 後,即時生效。

### 14.2.3 LLM Optimizer 的設計

評分若 < 70(`scoring_configs.f12_score_thresholds.low_score`),**自動排隊**走 LLM Optimizer:

```text
[低分頁] → 抓 axp_pages.content_md → 餵 LLM
       → 給定:原文 + 三層分析 issues + 可參考 templates
       → 要求:重寫使分數提升 ≥ 20,但 cosine 相似度 ≥ 0.90 不能離題
       → 寫進 axp_page_history(可雙向 rollback)
       → 重跑 analyzer 確認新分數 ≥ 原分數 + 15
       → 寫進 axp_pages 並標 needs_recompile
```

LLM prompt 結構(簡化版):

```text
你是 GEO 內容優化專家。下面是一篇分數 X 的內容,issue 為:[列表]
請改寫,目標:
1. 修補列出的 issue
2. 保持原意(別偏題,別加客戶沒說的事實)
3. 段落結構偏好 TL;DR + 條列 + 來源連結

可參考的 5 個 template 風格:[列表]

原文:
[content_md]

改寫後輸出 JSON:{ rewritten: "...", reasoning: "..." }
```

`min_similarity ≥ 0.90` 是 spec 定錨點(從 0.85 升到 0.90,migration 186)。原本 0.85 看似合理,但實測踩到兩個雷:

1. **保險→投資建議偏題雷**:某保險業務員的 axp_pages 分數低,Optimizer 為了「提升 informativeness」,把「儲蓄險月繳 5000、20 年期、保證利率 1.8%」改成「投資理財建議:推薦您把儲蓄險改成 ETF 投資」。cosine 0.86,過 0.85 門檻,但內容已違法。
2. **個人 IP→企業介紹漂移雷**:某 personal_ip brand 的「個人專長」被 Optimizer 改成「公司提供以下服務」,語氣完全不對。cosine 0.88,但 brand_type 從個人變企業。

升到 0.90 後,前者降到 0.83 被擋,後者降到 0.89 也被擋。代價是部分合理改寫被擋掉(約 12% false negative),但相比「違法內容上線」的風險,寬限度可以接受。

### 14.2.4 雙向可逆(rollback)的設計

`axp_page_history` 表保存每次 Optimizer 改寫前的 snapshot,admin UI 可一鍵 rollback。設計動機:

V1 早期沒有 history 表,某次 cron rerun 時 bug 把 30 個 brand 的 fact-check 頁全部改寫,結果發現 Optimizer 因為 RAG 失效拿到空 ground_truth,改寫出空泛內容。當時無法 rollback,只能 force-refresh RAG + 重跑 pipeline,30 brand × 22 page = 660 LLM call 全部重跑,5 小時後才復原。

加 `axp_page_history` 後,同類事件 30 秒內就能 rollback。代價是 DB 多 ~40 GB / 年(`content_md` 平均 8 KB × 30 brand × 22 page × 365 天 × 1.5 重跑率),但相比一次故障 5 小時的 downtime,值得。

### 14.2.5 axpPageWriter hook 全覆蓋

任何寫進 `axp_pages` 的路徑(9 個 generators + 4 legacy + admin 手動編輯)**全部 hook F12 analyze**,確保新內容立即評分,低於閾值立即排 optimizer。覆蓋驗證:

- ✅ `axpPageWriter.service.js#upsertAxpPage`(主路徑)
- ✅ `hybridCoordinator.service.js`(6 核心類)
- ✅ `factCheckGenerator.js#refreshFactCheckPage`(直 SQL,補了 hook)
- ✅ `axp.controller.js#updatePageContent`(admin 編輯)

漏掉時 weekly backfill cron(`0 2 * * 0`)是安全網。

**為什麼 hook 全覆蓋這麼難**:9 個 generators 不是同一個團隊在同一個 sprint 寫的——`overviewGenerator` 是 V1 寫的、`comparisonGenerator` 是 V2 寫的、`factCheckGenerator` 是 V2.5 寫的、`featuresGenerator` 是 V3 寫的。每個 generator 的「寫入 axp_pages」邏輯都略有不同(有的用 service,有的直 SQL,有的繞過 service 直接 INSERT)。

我們用了一週 grep 全 codebase 把「寫進 axp_pages」的所有路徑列出來:

```bash
grep -rn "INSERT INTO axp_pages\|UPDATE axp_pages\|axp_pages.*INSERT\|axp_pages.*UPDATE" backend/src/
```

找到 13 個入口,逐一補 F12 hook。最後再加一個 weekly backfill 當安全網——萬一未來新增 generator 漏 hook,backfill 會在週日 02:00 全平台重析,把漏掉的補上。實測 weekly backfill 一次跑 ~12 分鐘(30 brand × 22 page = 660 篇,約 1 篇/秒)。

### 14.2.6 五個 cron 的時序設計

| Cron | 排程 | 用途 |
|------|------|------|
| `f12-weekly-backfill` | `0 2 * * 0` | 全 brand × 22 page F12 重析 + brand_faq 健康檢查 |
| `f12-trends-refresh` | `30 2 * * *` | REFRESH MATERIALIZED VIEW `structural_score_trends` |
| `f12-low-score-optimizer` | `0 4 * * *` | LLM rewrite 低分頁,雙向可逆 |
| `f12-immediate-optimize` | priority queue | axpPageWriter hook 觸發,score < 70 立即排 |
| `f12-retention-cleanup` | `50 3 * * *` | 30 天清舊 optimization runs(後被 v3.24 retention 吞併) |

時間挑選邏輯:

- **02:00 UTC = 台灣 10:00**:平台流量最低時段(台灣客戶剛上班但還沒大量操作)。weekly backfill 跑 12 分鐘,影響可忽略。
- **02:30 UTC**:trends MV refresh 跑 ~30 秒,要排在 backfill 之後(後者寫資料,前者讀)。
- **03:50 UTC**:retention cleanup 在 trends 之後(後者依賴歷史資料)。
- **04:00 UTC = 台灣 12:00**:LLM Optimizer 跑全平台,選此時段是因 Anthropic / OpenAI 在 UTC 04:00 通常 RPM peak 已過(美西凌晨,沒人寫 prompt)。實測同樣 prompt 在 UTC 04:00 vs UTC 14:00,後者平均慢 1.8 倍(rate limit 排隊)。

priority queue 是即時觸發,不在 cron 表內——axpPageWriter hook 偵測到 score < 70 的新內容會立即 enqueue,worker 1-2 分鐘內處理完。

---

## 14.3 V3.1:雙引擎並行 + 整合策略

V1 走了 4 個月後,我們從 2026 年發表的兩篇 arxiv 論文找到「再升一個量級」的線索。

### 14.3.1 兩個學術源頭

**AutoGEO**(arXiv:2510.11438,github.com/cxcscmu/AutoGEO):

由 CMU 團隊發表,核心方法是**從成千上萬「被 LLM 引用 vs 沒被引用」內容對比,自動 mining 出可推廣的改寫規則**。他們的實驗範圍:

- 25 條 rule,分兩個 dataset:
  - Researchy-GEO(學術內容)+ Gemini:15 條
  - E-commerce(電商內容)+ Gemini:10 條
- 每條 rule 的形式為 LLM-readable instruction(例:「在開頭加上一句明確聲明此文討論的主題」)
- 實驗 LLM 主要是 Gemini-2.0-flash + GPT-4o + Claude-3.5

優點:**真實對比實驗 derived 出來**,不是憑直覺
缺點:rule 集合相對窄(只 25 條),且大部分針對 Gemini

**E-GEO**(arXiv:2511.20867,github.com/psbagga17/E-GEO):

由獨立研究員發表,核心方法是**從不同 prompt 風格(authoritative / technical / unique / fluent / clickable / diverse / quality / competitive / trick / format / FAQ / advertisement / language / minimalist / storytelling)生成**,實測各風格在不同 LLM 上的引用率差異。

- 15 個 template(=15 個 prompt 風格)
- 每個 template 是一段「重寫 instruction」(例:authoritative = 「以權威語氣重寫,加上來源、數據、引用」)
- 實驗範圍橫跨 GPT-4o / Claude-3.5 / Gemini-2.0 / DeepSeek-V3

優點:**template 多樣**,適配不同 use case
缺點:**沒給逐 template 的 uplift 數值**(只給整體實驗 macro 結論)

### 14.3.2 為何兩篇都 port,而非選一篇

兩篇方法論互補:

- AutoGEO 的 rule 是**「客觀對比實驗 derived」**,適合 batch 自動化(F12 大部分 case)
- E-GEO 的 template 是**「人工定義的 prompt 風格」**,適合「需要特定語氣」的 case(例:客戶要 authoritative tone for whitepaper)

我們把兩者**字面上**(line-by-line)port 進系統,**不偽造任何 paper_table_ref / expected_uplift**(原 paper 沒給逐條 mapping,寫了會違反工程憲法 #1「不模擬數據」)。

> ⚠️ 重要:V3.1 的 `autogeo_rules` 表初版 seed 含偽造 paper_table_ref(`'Table 3'` 等)+ 偽造 expected_uplift(`1.32-1.92`),被工程憲法 #1 trigger 攔截後重 port 真實 arxiv 源碼。後加 DB-level placeholder guard trigger 永久防再犯。詳見 14.3.5。

### 14.3.3 三引擎架構

```text
                 ┌──────────────────┐
                 │ adaptiveRouter   │ ← 依 brand tier / page_type / use case 路由
                 └─────────┬────────┘
                           │
        ┌──────────────────┼──────────────────┐
        ▼                  ▼                  ▼
  ┌──────────┐       ┌──────────┐       ┌──────────┐
  │ AutoGEO  │       │ E-GEO    │       │ Hybrid   │ ← parallel call,挑 improvement 高的
  │ engine   │       │ engine   │       │ engine   │
  └──────────┘       └──────────┘       └──────────┘
        │                  │                  │
        └──────────────────┼──────────────────┘
                           ▼
                    ┌─────────────┐
                    │ ME custom   │ ← personal_ip 專屬,從 personal_profiles 守門
                    │ engine      │
                    └─────────────┘
```

四個 engine 都吐**標準化 EngineResult** schema:

```typescript
interface EngineResult {
  engine: 'autogeo' | 'egeo' | 'hybrid' | 'me_custom';
  rule_or_template_id: string;         // arxiv source 或 internal id
  source: string;                       // 'arXiv:2510.11438' 等
  original_content: string;
  rewritten_content: string;
  diff: { additions, deletions, similarity };
  scores: { before, after, delta };
  llm_call: { provider, model, tokens, cost };
  accepted: boolean;                    // optimizer 自決(score 提升 ≥ 15 + similarity ≥ 0.90)
}
```

由 `executeRoute` 統一 audit + persist + cache。Hybrid engine 的「並行」實作:

```javascript
const [autogeoResult, egeoResult] = await Promise.allSettled([
  Promise.race([autogeoEngine.optimize(input), timeout(30_000)]),
  Promise.race([egeoEngine.optimize(input), timeout(30_000)]),
]);

const candidates = [autogeoResult, egeoResult]
  .filter(r => r.status === 'fulfilled' && r.value.accepted)
  .map(r => r.value);

if (candidates.length === 0) {
  return { engine: 'hybrid', accepted: false, reason: 'all_engines_failed' };
}

return candidates.reduce((best, c) =>
  c.scores.delta > best.scores.delta ? c : best
);
```

`Promise.allSettled` 確保單一 engine 失敗不影響另一個;30 秒 timeout 防 LLM 卡死(對齊平台憲法「LLM call 必有 ≤60s timeout」)。

### 14.3.4 路由決策

`adaptiveRouter.decide({ brandId, contentType, ... })` 根據以下優先序回 `{ path, template, target_engine }`:

1. **plan gate**:starter 一律走 E-GEO `authoritative`(成本最低);Pro 才能用 sync API;Enterprise+ 才能用 Hybrid 與 AutoGEO
2. **page_type → template**:`f12_page_type_to_template` SSOT(22+1 page_type → 5 template category)
3. **content_type 特化**:whitepaper / case_study / fact_* 強制走 E-GEO `authoritative`;FAQ 走 `FAQ`;product 走 `clickable`
4. **brand_type fallback**:personal_ip 走 `me_custom`(只此一條 baiyuan internal template)

完整 decision tree:

```text
adaptiveRouter.decide(input)
├── if brand.tier === 'starter':
│     return { path: 'egeo', template: 'authoritative' }
├── if input.contentType in ['whitepaper', 'case_study', 'fact_check']:
│     return { path: 'egeo', template: 'authoritative' }   ← 強制 authoritative
├── if input.contentType === 'faq':
│     return { path: 'egeo', template: 'FAQ' }
├── if brand.brand_type === 'personal_ip':
│     return { path: 'me_custom' }                         ← ME 守門
├── if brand.tier in ['enterprise', 'group']:
│     return { path: 'hybrid' }                            ← 雙引擎並行
└── default (Pro tier general content):
      template = mapPageTypeToTemplate(input.page_type);
      return { path: 'egeo', template };
```

**為什麼 plan gate 在 router + executeRoute 雙層**:router 端決策時就擋住,但 executeRoute 端是 defensive 第二層——萬一 router bug 或 SSOT config 改錯,executeRoute 會再次檢查 brand.tier vs path,不符合直接 reject。曾踩過一次:某次改 router config 漏改 starter 的 path,starter brand 跑了 3 次 hybrid call(3× LLM cost),executeRoute 第二層攔不住。加了 defensive plan_gate 後同類事件無法發生。

### 14.3.5 placeholder guard trigger 的故事

V3.1 的 `autogeo_rules` 表初版 seed(migration 189)出現了一個重大事件——**偽造 paper_table_ref**。

當時 spec 寫「每條 rule 對應 paper Table 3 第 N 行」,工程師依直覺填入 `'Table 3'` / `'Table 4'`,但實際翻 arxiv:2510.11438 paper 沒給「rule X 對應 Table Y 第 Z 行」的 mapping。同時 `expected_uplift` 欄位也填了 `1.32-1.92` 範圍,但 paper 實驗只給整體實驗 uplift,不是逐 rule。

工程憲法 #1「不模擬數據」trigger 攔截:

```sql
CREATE OR REPLACE FUNCTION autogeo_rules_placeholder_guard()
RETURNS trigger AS $$
BEGIN
  IF NEW.paper_table_ref IS NOT NULL
     AND NEW.source NOT LIKE 'arXiv_%'
     AND NEW.source NOT LIKE 'arXiv:%' THEN
    RAISE EXCEPTION 'paper_table_ref must be NULL or source must be arXiv (憲法 #1)';
  END IF;

  -- 6 patterns 偵測 placeholder
  IF NEW.rule_description ~ '(待填入|To be filled|Actual Data Pending|TODO:|FIXME:|XXX)' THEN
    RAISE EXCEPTION 'placeholder pattern detected in rule_description (憲法 #1)';
  END IF;

  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_autogeo_rules_placeholder_guard
BEFORE INSERT OR UPDATE ON autogeo_rules
FOR EACH ROW EXECUTE FUNCTION autogeo_rules_placeholder_guard();
```

migration 191 用 `TRUNCATE ... RESTART IDENTITY CASCADE` 清掉偽造 seed,migration 192 加 trigger,並重 port 真實 arxiv 源碼。`autogeo_rules` 表 25 條皆 `source = 'arXiv:2510.11438'`,含真實:

- `target_engine ∈ {gemini, gpt, claude}`
- `target_domain ∈ {researchy_geo, geo_bench, ecommerce}`
- `paper_table_ref` 全 NULL(arxiv 沒給逐條 mapping,憲法 #1 不偽造)
- `expected_uplift` 全 NULL(整體實驗 uplift,不是逐條 — V2 真實 ML 才填)

`egeo_templates` 15 條皆 `source = 'arXiv:2511.20867'`,template 名與 paper 表 1 完全對齊(authoritative / technical / unique / fluent / clickable / diverse / quality / competitive / trick / format / FAQ / advertisement / language / minimalist / storytelling)。

百原內部 1 條 `me_personal_ip` 顯式 `source = 'baiyuan_internal_v3_1'`,不混入 arxiv 命名。

**這個 trigger 是「事後諸葛」**——已經被偽造一次才裝上的。但裝上後,任何未來 admin 編輯 / 新 generator / cron 故障都進不來,1 萬租戶 scale 下這是必要的硬體層保障。

---

## 14.4 1 萬租戶基礎設施

V3.1 在 9 個 batch 鋪設了規模化基礎設施(v3.21 → v3.27 跨 8 天)。每個 batch 解決一個瓶頸,9 個合在一起讓系統從「能跑 30 brand」進化到「設計上能跑 10000 brand」。

### 14.4.1 配額與計費(spec §4.3 + §4.4)

```text
TenantQuotaService(monthly_optimizations / max_content_size_kb)
  starter:  100 ops, 50 KB
  pro:      1000, 200
  enterprise: 10000, 1000
  group:    ∞, 5000

BillingTracker(每次 LLM call 記 cost)
  9 model × per-token pricing(claude_haiku $0.0003 / opus $0.015 / 4o-mini $0.0002 ...)
  in/out 倍率 4×
```

`f12_quota_usage` 表 monthly counter UPSERT(`tenant_id+brand_id+action+period` 唯一),`f12_billing_records` 12M 行/年單表夠用(1 萬 × 100 opt × 12 月)。`quota_enforcement_enabled` flag 預設 `false`(觀察期只記 usage 不擋),`billing_recording_enabled` 預設 `true`(永遠記)。

「觀察期」設計:剛上線給客戶看「本月用了 X / 上限 Y」但不擋,觀察 ~2 週確認分布合理(不會出現 starter 客戶月用 5000 次的 outlier)後才切到 enforcement。實測觀察期數據:

- starter 平均月 18 ops(中位數 5),95th percentile 47——遠低於 100 上限
- pro 平均月 234 ops,95th = 612——低於 1000
- enterprise 平均 1820 ops,95th = 4500——低於 10000

切到 enforcement 後 0 個客戶被擋,代表配額設計合理。

### 14.4.2 Plan Feature Gate(spec §9)

`f12_plan_features` SSOT 4 tier × 6 flag(`scoring_configs` 表內 JSONB):

```yaml
egeo_template_count:    starter=0    pro=5     enterprise=15  group=15
autogeo_rules:          'none'       'none'    'full'         'full'
engine_specialization:  false        false     true           true
hybrid_engine:          false        false     true           true
sync_api:               false        true      true           true
```

非 Enterprise 走 `whitepaper` / `case_study` / `fact-*` 強制改走 E-GEO `authoritative`(不再 AutoGEO);只 Enterprise+ 可走 Hybrid + 三引擎特化。`adaptiveRouter` 內 plan gate + `executeRoute` defensive plan_gate 雙層攔截。

設計動機:starter / pro 客戶月費低($99 / $499),不能讓他們用最貴的引擎(Hybrid 一次 call 兩個 LLM),否則毛利為負。enterprise+ 月費 $2000+,可以承擔 Hybrid 成本。

`egeo_template_count` 設計:starter 0(只能用 default authoritative)、pro 5(authoritative / FAQ / clickable / fluent / diverse)、enterprise+ 全 15。客戶看到「啟用更多 template 升級至 Enterprise」的 upsell。

### 14.4.3 Multi-Layer Cache(spec §5.2)

```text
L1 Redis (5 min TTL)        ← 同 content + decision 重複 query
L2 PG f12_result_cache (7d) ← 跨 process / 跨 instance
L3 S3 hook (TBD Phase 3)    ← cross-region replication 需要時
```

cache key = `sha256(content + decision.path + template + target_engine)`,**不含 tenant_id 跨租戶共用**(spec 明文允許,因為內容相同 + 規則相同 → 結果應該相同,共用減 N 倍 LLM cost)。

只 cache `accepted=true` 結果(避免 cache poisoning)。`f12_cache_metrics` hourly 聚合 L1/L2/miss → admin 監控 70% hit rate 目標。

**跨租戶共用的隱私邊界**:乍看 sketchy(A 客戶的內容 cache,B 客戶能拿到嗎?),實際分析:

- cache key 包含**整段 content** 的 sha256,只有 content 完全相同才命中
- 客戶 A 的內容 99.9% 不會跟客戶 B 完全相同(content 是 KB 等級,sha256 等同)
- 命中的唯一場景:同一條 generic content(如 FAQ template「什麼是 GEO?」)被多個客戶用——這種 case 共用合理(沒有客戶私有資訊)

實測 1 個月 cache hit rate 38%(L1+L2),低於 70% 目標。原因:`content` 帶 brand-specific 資訊(brand name、entity 等),即使邏輯相同,內容已 unique 化。Phase 3 想做的是「**規範化 cache key**」——把 brand-specific token 替換成 `<<BRAND_NAME>>` placeholder 後再 hash,理論上能讓相似 brand 共用 cache。這是 V2 工作。

### 14.4.4 LLM Gateway(spec §5.3)

per-model 平台層 RPM 限制(in-memory sliding window 60s):

```yaml
claude_haiku:  10000 / min
sonnet:         2000
opus:            500
gpt-4o:         5000
gpt-4o-mini:   10000
gemini-flash:  20000
qwen_direct:   30000
deepseek_v4:    8000
```

5 個 F12 LLM call site 全走 `gatewayAiCall`:autogeo / egeo / meCustom / V1 optimizer / hybrid 透過前兩者連帶。`opts.billing` 提供時自動 record(向後兼容)。

**為什麼平台層 RPM 限制必要**:Anthropic / OpenAI 的 API key 是 per-organization,1 萬租戶共用一把 key,單一客戶 burst 會耗掉全平台 quota。Gateway 在 LLM call 之前先檢查當前 RPM,超過則排隊或 reject。

實測場景:某次 admin UI bug 導致一個 brand 重複觸發 100 次 hybrid optimize,每次 2 個 LLM call,200 個 call 同時打 Claude Sonnet。沒 Gateway 時直接超 Anthropic 30 RPM 限制,全平台所有 brand 都被擋;有 Gateway 時擋在我方,只影響該 brand,其他繼續正常。

**限制**:in-memory bucket 是 per-process,單 backend container ok;多 instance 需 Redis sliding-window(已留 hook,未實作)。1 萬租戶 scale 預期 backend 至少 5-10 個 instance,Phase 3 必須切 Redis-based。

### 14.4.5 其他 1 萬租戶 hook

| 機制 | 作用 |
|------|------|
| **Tier-based Job Priority**(`group=1` 最高 → `starter=4`) | BullMQ priority,大客戶優先處理 |
| **Sync/Async Dual API** | `/optimize` 異步 return jobId / `/optimize/sync` Pro+ 限,內容 ≤5KB |
| **Concurrent Jobs Enforcement** | 防 starter 無限堆積拖垮 worker(`starter=1`,`group=100`) |
| **Per-Tenant API RPM** | Redis sliding-window per brand,fail-open(Redis 失效時不擋) |
| **Retention TTL SSOT** | 6 表 daily 05:00 cron 清過期 row,`scoring_configs.f12_retention_config` 可 admin 調 |
| **Budget Alerts** | tier threshold(starter $1 / group $5000 月)warn@80%,critical@block_pct |
| **Scale-up Trigger** | daily 04:30 snapshot 8 維度 metrics(tenant_count / monthly_ops / p99 / cache_hit / cost),對齊 spec 升 phase 1→2→3 |
| **Tenant Isolation Strategy** | 寫 spec/actual 雙欄位,enterprise/group 客戶 isolation gap 給 admin 看 |
| **L3 S3 Cache hook**(Phase 3 預留) | `F12_S3_CACHE_BUCKET` env 設時動態 import `@aws-sdk/client-s3` |

完整 SSOT 在 `scoring_configs.f12_*` 系列 14 個 key,admin UI `/dashboard/admin/f12-dashboard` 整合 4 個 stat card + 4 個 detail section。

**互依關係**:9 件套不是獨立的——Concurrent Jobs Enforcement 防 worker 過載 → Job Priority 確保高 tier 客戶不被低 tier 擠掉 → Per-Tenant RPM 防 burst → LLM Gateway 防整平台爆 quota → Cache 減 LLM cost → Quota / Billing 管錢 → Budget Alerts 防錢失控 → Scale-up Trigger 提示何時要升 phase。任一缺失都會在 1 萬 tenant scale 下出事。

---

## 14.5 部署、配額、計費三件事

### 14.5.1 部署:Wave 0 / Wave 1 對齊

V3.1 完成 spec Wave 0(microservice contract OpenAPI)+ Wave 1(in-process 實作 + SDK + admin):

- **OpenAPI 3.0 spec**:`backend/src/openapi/f12-v3-1.json` 7 paths × 14 schemas,`/api/v1/f12/openapi.json` 公開(掛 router auth 之前)讓 AI agents / 第三方 machine-discover
- **F12Client SDK**:`services/f12/f12Client.js` 5 主入口(`diagnose / optimize / optimizeSync / getJob / health`),`F12_SERVICE_URL` env 設時走 HTTP,未設 in-process,**caller signature 不變**為未來微服務拆分留 hook
- **/api/v1/f12/health**:OAuth 之前的公開 endpoint,monitoring 系統可直接 ping

部署順序(時序):

1. **v3.21**(2026-04-23):TenantQuotaService + BillingTracker + ME custom engine 真實邏輯 + LLM Gateway
2. **v3.22**(2026-04-25):Plan Feature Gate + Multi-Layer Cache + Tier Priority + Sync/Async Dual API
3. **v3.23**(2026-04-27):F12Client SDK + OpenAPI + Tenant Isolation hook + L3 S3 hook + Scale-up Trigger
4. **v3.24**(2026-04-29):Concurrent Jobs + Per-Tenant RPM + Retention + Budget Alerts
5. **v3.25**(2026-04-30):Retention dedup + Redis Billing cache + Admin Dashboard
6. **v3.26**(2026-05-01):AI bot whitelist SSOT(對齊 7-layer audit fix)
7. **v3.27**(2026-05-02):Phase 2-3 規模化基礎設施 hook(schema/db provisioner、Redis cluster、shard router、multi-region)

每個 batch 完整流程:遷移 SQL → 業務邏輯 → 規格 test → admin UI → CLAUDE.md 文件 → push PROD。

### 14.5.2 配額:觀察期 → 強制期

剛上線 `quota_enforcement_enabled=false`(只記 usage),客戶看得到「本月用了 / 上限是」但不會被擋。觀察 ~2 週確認分布合理後 admin 一鍵切到 `true`,從此超額直接 HTTP 429 + reject reason `'quota_exceeded'`。

切換前準備:

1. **客戶通知**:UI 顯示「本月用了 X / Y,下月 Y/Y 起超額將被擋,請聯繫升級方案」
2. **客服培訓**:超額被擋的客戶會直接 ticket,客服需熟悉「升級方案 / 月底重置 / 緊急配額調整」三種應對
3. **monitoring 加紅燈**:admin dashboard 顯示「過去 24h 被擋的 brand 數量」,> 5 即發 alert(如果是 bug 而非真實超額,要快速回退)
4. **回退方案**:一鍵切回 `false` 5 秒生效(SSOT in DB,不需 deploy)

### 14.5.3 計費:per-call 記錄,月底聚合

`BillingTracker.recordCall(model, tokens, runId)` 每次 LLM call 都記。月底 admin 跑 `/admin/f12-billing-summary?period=YYYY-MM` 看 top 50 cost brand。

雙寫 strategy:Redis(`f12:billing:tenant:{id}:{period}` INCRBYFLOAT + 35 天 TTL)+ DB(`f12_billing_records`)。當前月 `getBrandMonthlyCost` 優先 Redis O(1) GET,過去月或 Redis miss 才 DB SUM。1 萬租戶 query 月度 cost 不打 DB,sub-ms 響應。

35 天 TTL 是有意設計:當月 + 上個月覆蓋,客戶 7 號要看上個月帳單也能秒查。35 天後自動過期,永遠以 DB 為真相源。

monthly batch 跑「成本 vs 月費」對比,毛利為負的 brand 自動 alert(可能是定價錯 / 客戶誤用 / bug),admin review。

---

## 14.6 觀察到的限制與未解問題

### 14.6.1 雙引擎結果如何「合併」沒有正確答案

Hybrid engine 並行跑 AutoGEO + E-GEO,挑 `improvement` 較高的那個。但實際商業情境裡,improvement 不是單一指標:

- AutoGEO 的 rule rewrite 結構穩定但較少品牌個性
- E-GEO 的 template 變化多但有時改到客戶不認得自家內容

V3.1 用「engine improvement score 較高」當挑選依據,實際是不夠的。下個版本想加**「accepted by client」 信號**,但目前 admin 端編輯介面還沒上線(Phase 3)。

### 14.6.2 paper_table_ref / expected_uplift 為 NULL

「字面上 port arxiv」是工程憲法 #1 的副作用 — paper 沒給逐條 table mapping,系統就**禁止偽造**。但 admin UI 顯示「為什麼選這條 rule」時,缺資料就顯空白。

V2 想做的是**自家 measurement**:對每一條 rule 跑真實 ML eval(同 prompt × 3 LLM × 1000 brand,measure citation_rate uplift),用真實數字填回。需要時間 + Anthropic / OpenAI quota:

- 25 rule × 3 LLM × 100 brand × 5 重複 = 37500 LLM call
- Claude Sonnet $0.003/1k tokens,平均 5k tokens/call = $0.015/call
- 總 cost ≈ $562
- 時間:Anthropic RPM 2000/min,跑滿 ≈ 18 分鐘

成本不高,但需要 1000 brand 的真實 baseline citation rate 資料。目前只 30 brand,等 brand 數量足夠才能跑——Phase 3 預期 5000+ brand 時做。

### 14.6.3 L3 S3 cache 還沒實際部署

Phase 3 才需(預估 5000+ tenant 規模)。目前 L1+L2 hit rate 約 35-45%,離 70% 目標還有距離。L3 預期再升 15-20% 但需要 S3 cost vs 重算 LLM cost 的 tradeoff 分析,沒做完。

粗估:L3 S3 PUT $0.005/1000 obj、GET $0.0004/1000 obj、storage $0.023/GB/月。1 萬 brand × 100 ops × 5 KB / op = 5 GB,storage $0.115/月可忽略。但每次 GET 0.4 ms latency,vs L1 Redis 0.05 ms,L3 慢 8 倍。Phase 3 必須 L3 in cross-region(US + JP + EU),latency 攤勻。

### 14.6.4 ME custom engine 守門過嚴

`personal_profiles` 表 metadata 全空 → reject `meta_incomplete`。實際觀察 ME 平台 17 個會員 brand 中有 5 個 metadata 不完整(沒填全名 / known_for),這 5 個就走不了 ME custom engine,fallback 到 E-GEO `authoritative` 但結果偏向企業語氣不適合個人 IP。需要客服跟客戶補齊 metadata,但這違反「自動化」承諾。

下一版考慮:**LLM 從 brand_documents 自動推斷 personal_profiles 缺失欄位**,但需要 high-confidence guard 防 hallucination。例如「Tina Chou 的 jobTitle 是?」LLM 從她的著作 / 媒體報導推斷,推斷結果 confidence > 0.95 才填。

### 14.6.5 前後相依的 1 萬租戶測試

V3.1 全部基礎設施都對齊「1 萬租戶可承受」的設計,但真實壓測只到 17 個 ME brand + 21 個 GEO brand。Scale-up Trigger 的 phase_1_to_2 / phase_2_to_3 邏輯字面對齊 spec line 1183-1193,但**從未實際升級過**。理論上跨 phase 升級需要:tenant schema 隔離切換(Phase 1 row_level → Phase 2 schema)、Redis cluster sharding、DB 跨 region replication,這些在 spec hook 都備好(`schemaProvisioner` / `dbConnectionRouter` / `redisClusterAdapter` / `shardRouter`),但**未經實戰**。

未來 1 年的工作項:

- **synthetic load test**:寫一個 generator 模擬 1000 brand × 100 ops/月,跑滿 1 個月看 Scale-up Trigger 是否如預期觸發
- **Phase 2 dry-run**:在 staging 環境真實切 schema isolation,測 query 路徑、跨 schema JOIN 是否還 work
- **Phase 3 cross-region sketch**:測 Cloudflare Workers 如何路由到最近 region

這些工作都不能在 production 直接做,需 staging 環境 + 額外硬體預算。

### 14.6.6 跨租戶 cache 的隱私邊界(新加)

14.4.3 提到 cache key 不含 tenant_id,跨租戶共用減 LLM cost。乍看可能引發隱私顧慮,但設計時已分析:

- **content 完全相同才命中**:cache key 是 `sha256(content + decision)`,content 是 KB 等級,2 個客戶寫一模一樣 content 機率為 0(除非抄襲)
- **真實命中場景**:generic FAQ template「什麼是 GEO?」答案被多個客戶用——這種 case 共用合理(沒客戶私有資訊)
- **未來風險**:如果 cache key normalization(把 brand name token 替換成 placeholder)上線,跨 brand hit rate 會升,但需嚴格保證 normalization 沒洩漏資訊
- **法規對齊**:GDPR / 個資法不禁止「相同公開內容的 LLM 結果共用」,因為 content 本身已是 brand 自願公開資訊

V2 計畫加 audit log:每次 cache hit 記「from brand A → to brand B」,客戶可 opt-out 跨租戶共用(換來自己更高 LLM cost)。

### 14.6.7 cache key 算法的隱含假設(新加)

cache key = `sha256(content + decision.path + template + target_engine)`,隱含假設:**LLM call 是 deterministic**(同 input → 同 output)。實際:

- `temperature=0` 大部分 deterministic,但 OpenAI / Anthropic 都不保證 100%(infrastructure 層有微小 noise)
- 同樣 prompt 跑 100 次,~95% identical / ~5% 字面不同但語意相同

cache hit 時直接回舊結果,客戶看不出差別,因為 95% deterministic 已經是很高的一致性。但未來如果 LLM API 改成「明確 non-deterministic」(例如 OpenAI Sora / Gemini Veo 影片生成),cache 必須加 versioning 跟 invalidation。

實作上有預留:`f12_result_cache` 有 `cache_version` 欄位,Phase 3 LLM 換代時可一鍵 bump version 失效全平台 cache。

---

## 14.7 工程教訓

從 V1→V3.1 的演進(7 個月,5 個 PROD 故障),整理出 5 條教訓:

### 14.7.1 字面 port arxiv 比自家發明更安全

我們最初想自己發明 GEO 規則(基於 internal 觀察),但很快發現:

- 缺乏對比實驗(自家 30 brand 不夠 derived 規則)
- 容易 overfit 到自家客戶風格
- 沒有學術 reference 客戶不買單(「為什麼這樣寫?」「我們覺得好」←弱)

切到「字面 port arxiv 源碼」後:

- ✅ 25 + 15 條 rule/template 立即可用
- ✅ 客戶看到「arXiv:2510.11438」立即信服
- ✅ DB-level placeholder guard 永久防偽造

但代價是 paper_table_ref / expected_uplift 為 NULL,UI 顯示空白。寧可空白也不偽造——這是憲法 #1 的成本。

### 14.7.2 SSOT 全平台比寫死在 code 重要

V1 早期所有閾值寫死 code(`const LOW_SCORE_THRESHOLD = 70`),改一次 deploy + 重啟。客戶問「為何我這頁 65 分卻沒被優化」,工程師查 code 發現是 `hard-coded 70`,改成 65 就 deploy。整個流程 30 分鐘。

切到 SSOT (`scoring_configs.f12_score_thresholds.low_score`) 後:

- admin UI 改一個數字 5 秒生效
- 不同 brand_type 可不同閾值(GEO 70、ME 75)
- 異常時一鍵回退(SQL UPDATE 上一個值)

代價是多一個 indirection 層,新人看 code 要多 grep 一次。但相比 deploy 速度,值得。

### 14.7.3 DB-level 強制比 application-level 可靠

placeholder guard 原本想做在 application-level(每次 INSERT 前 application 檢查):

- ❌ admin UI 直接走 SQL 編輯時繞過(走 psql 操作 PROD 偶爾會發生)
- ❌ generator 寫 raw SQL `INSERT INTO axp_pages` 繞過 service layer
- ❌ pipeline cron 老 code 走 service 但 service 沒檢查

切到 DB trigger 後:

- ✅ 任何路徑寫入都被攔截(包括 psql 直接 INSERT)
- ✅ application code 可以放心(trigger 是兜底)
- ✅ 規格更穩定(trigger 規則只有 SQL,不容易誤改)

代價:trigger debug 困難(error message 不清),但 1 萬租戶 scale 下這個代價值得。

### 14.7.4 觀察期是 SaaS 部署的標配

quota_enforcement_enabled / billing_recording_enabled 等 flag 都預設「觀察期」(只記不擋),確認分布合理才切「強制期」。觀察期 ≥ 2 週,期間:

- 看真實 usage 分布
- 找出 outlier(可能是 bug 也可能是真實用量)
- 通知客戶「下月起會擋」
- 客服培訓

V1 早期沒這個機制,直接上線 enforcement,結果:

- 第 1 週就有 3 個客戶超額被擋,但其實是 admin 配額設錯(寫 100 寫成 10)
- 客服沒培訓,客戶 ticket 答不出來
- 最後一鍵回退 enforcement,客戶體驗已壞

加觀察期後同類事件再沒發生。

### 14.7.5 hook 全覆蓋 + safety net 缺一不可

axpPageWriter F12 hook 必須 100% 覆蓋,但 100% 覆蓋難保證(13 個入口,新 generator 隨時加):

- **hook 100%**:第一道防線,即時觸發 F12 analyze
- **weekly backfill cron**:第二道防線,週日 02:00 全平台重析,漏掉的補上
- **admin manual trigger**:第三道防線,admin 看到異常可手動 trigger 單 brand 重析

三層缺一不可——只靠 hook 漏覆蓋 → 永久漏分析;只靠 backfill 不及時(最壞 7 天才反應);只靠 manual 不規模化。

V1 早期只做 hook,某次 fact-check generator 漏 hook,4 個 brand 的 fact-check 頁分數異常低 3 個月才被發現。加 backfill 後同類事件最多 7 天必然反應。

---

## 本章要點

- F12 補上 Ch 3 評分留下的「下一篇怎麼寫」問題,輸入內容輸出三層分數 + 優化版本
- V1 規則式三層分析 + LLM Optimizer(`min_similarity ≥ 0.90` 防偏題),5 個 cron 串接全平台覆蓋
- V3.1 引入 AutoGEO + E-GEO + Hybrid 三引擎,字面 port arxiv 源碼,DB trigger 防偽造 paper_table_ref
- 1 萬租戶基礎設施 9 件套:配額 / 計費 / Plan Gate / Multi-Layer Cache / LLM Gateway / Job Priority / Sync-Async / Retention / Scale Trigger
- 限制:engine 合併沒最佳解 / paper 真實 uplift 待自家 measure / L3 cache 未部署 / ME metadata 守門過嚴 / phase 升級未經實戰 / 跨租戶 cache 隱私邊界 / cache key 假設 LLM deterministic
- 5 條工程教訓:字面 port arxiv > 自家發明 / SSOT > 寫死 code / DB-level > application-level / 觀察期是標配 / hook + safety net 缺一不可

## 參考資料

- [Ch 3 — 七維度評分演算法](./ch03-scoring-algorithm.md)
- [Ch 6 — AXP 影子文檔交付](./ch06-axp-shadow-doc.md)
- [Ch 12 — 限制、未解問題與未來工作](./ch12-limitations.md)
- AutoGEO paper: arXiv:2510.11438 — github.com/cxcscmu/AutoGEO
- E-GEO paper: arXiv:2511.20867 — github.com/psbagga17/E-GEO
- F12 v3.1 spec: `F12_TLSO_spec_v3_1.md`(repo root)

## 修訂記錄

| 日期 | 版本 | 說明 |
|------|------|------|
| 2026-05-03 | v1.1 | 新章 — 涵蓋 V1 + V3.1 完整架構演進 |
| 2026-05-03 | v1.1.1 | 章節擴充至 ~7200 字 — 加 14.1.2 早期手調失敗模式、14.2.4 雙向可逆設計、14.2.5 hook 全覆蓋難度、14.2.6 cron 時序、14.3.5 placeholder guard 故事、14.4 互依關係、14.5 部署順序、14.6.6/7 cache 隱私與假設、14.7 工程教訓 5 條 |

---

**導覽**:[← Ch 13: 多模態 GEO](./ch13-multimodal-geo.md) · [📖 目次](../README.md) · [Ch 15: rag-backend-v2 加固 →](./ch15-rag-backend-v2-hardening.md)

<!-- AI-friendly structured metadata -->
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "TechArticle",
  "headline": "Chapter 14 — F12 三層結構優化器:從規則 V1 到雙引擎 v3.1",
  "description": "F12 (TLSO) 架構演進:V1 規則式 + LLM Optimizer,V3.1 引入 AutoGEO + E-GEO + Hybrid 三引擎,加上 9 件套 1 萬租戶基礎設施(配額/計費/Plan Gate/Multi-Layer Cache/LLM Gateway/Job Priority/Sync-Async/Retention/Scale Trigger)。含 5 條工程教訓。",
  "author": {"@type": "Person", "name": "Vincent Lin", "affiliation": "Baiyuan Technology"},
  "datePublished": "2026-05-03",
  "inLanguage": "zh-TW",
  "isPartOf": {
    "@type": "Book",
    "name": "百原GEO Platform 技術白皮書",
    "url": "https://github.com/baiyuan-tech/geo-whitepaper"
  },
  "keywords": "F12, TLSO, AutoGEO, E-GEO, Hybrid Engine, Plan Feature Gate, Multi-Layer Cache, LLM Gateway, Tier Priority, Scale Trigger, 工程教訓"
}
</script>
