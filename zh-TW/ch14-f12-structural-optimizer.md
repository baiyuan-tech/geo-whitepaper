---
title: "Chapter 14 — F12 三層結構優化器:從規則 V1 到雙引擎 v3.1"
description: "F12(TLSO, Three-Layer Structural Optimizer)架構演進史:V1 規則式評分 + LLM rewrite,V3.1 引入 AutoGEO + E-GEO 雙引擎並行 + Hybrid 整合,加上 Plan Gate / Multi-Layer Cache / Tier Priority / Scale Trigger 等 1 萬租戶基礎設施。"
chapter: 14
part: 5
word_count: 4500
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
last_modified_at: '2026-05-03T10:45:10+08:00'
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

---

## 14.1 為什麼需要 F12

Ch 3 的七維度評分回答「品牌引用率是多少」,但**不告訴你「下一篇文章該怎麼寫才會被引用率提升」**。商業客戶實際拿著評分問:

> 「我已經知道 AI 引用率 32 分,我下一篇 blog 要怎麼改才會升到 60?」

F12(代號取自 Web DevTools F12 開發者工具的「結構檢視」隱喻)是這個問題的工程答案:輸入一段內容,輸出**可量化的結構分數**(三層:macro / meso / micro)+ **可直接套用的優化版本**。它跟 Ch 3 評分的差別是:

| | Ch 3 評分 | F12 |
|---|---|---|
| 輸入 | 品牌維度 + AI 平台維度 | 一段內容(URL / 文字 / Markdown) |
| 輸出 | 0–100 分 + 訊號分項 | 三層分數 + 優化後內容 |
| 用途 | 「我品牌健康嗎」 | 「這篇文章如何改才會被引用」 |
| 對象 | brand-level | content-level |

**F12 的核心承諾是:給定同一段內容,系統能產出多個變體並選出 LLM 最可能引用的那個**。

---

## 14.2 V1:三層分析 + LLM Optimizer

### 14.2.1 三層結構

| 層 | 顆粒度 | 檢查內容 |
|---|---|---|
| **Macro** | 文件級 | 是否 ≥1 篇 fact-check / FAQ / comparison?是否有清楚 H1?是否聲明 entity? |
| **Meso** | 段落級 | 段落長度分布、TL;DR 是否在前、是否有 lead-in 句子吸引 LLM 摘錄 |
| **Micro** | 句子級 | 數字 / 日期 / 來源連結密度、實體 (entity) 標記、是否有可被引用的 atomic fact |

每層獨立打分(0–100),最終 `overall = 0.4 × macro + 0.35 × meso + 0.25 × micro`(權重存於 `scoring_configs.f12_thresholds_<brand_type>` SSOT,可由 admin 調整不需改 code)。

V1 的分析器是**純規則式**:正規表達 + AST 解析 + 統計密度,沒打 LLM。每秒可分析 30+ 篇文章,適合 batch 全平台掃描。

### 14.2.2 LLM Optimizer

評分若 < 70,**自動排隊**走 LLM Optimizer:

```text
[低分頁] → 抓 axp_pages.content_md → 餵 LLM
       → 給定:原文 + 三層分析 issues + 可參考 templates
       → 要求:重寫使分數提升 ≥ 20,但 cosine 相似度 ≥ 0.90 不能離題
       → 寫進 axp_page_history(可雙向 rollback)
       → 重跑 analyzer 確認新分數 ≥ 原分數 + 15
       → 寫進 axp_pages 並標 needs_recompile
```

`min_similarity ≥ 0.90` 是 spec 定錨點(從 0.85 升到 0.90),目的是防 LLM 把客戶內容改到「漂亮但偏題」(出現過 case:LLM 把保險產品介紹改成投資建議,違反金管會規定)。

### 14.2.3 axpPageWriter hook 全覆蓋

任何寫進 `axp_pages` 的路徑(9 個 generators + 4 legacy + admin 手動編輯)**全部 hook F12 analyze**,確保新內容立即評分,低於閾值立即排 optimizer。覆蓋驗證:

- ✅ `axpPageWriter.service.js#upsertAxpPage`(主路徑)
- ✅ `hybridCoordinator.service.js`(6 核心類)
- ✅ `factCheckGenerator.js#refreshFactCheckPage`(直 SQL,補了 hook)
- ✅ `axp.controller.js#updatePageContent`(admin 編輯)

漏掉時 weekly backfill cron(`0 2 * * 0`)是安全網。

### 14.2.4 五個 cron 串接

| Cron | 排程 | 用途 |
|------|------|------|
| `f12-weekly-backfill` | `0 2 * * 0` | 全 brand × 22 page F12 重析 + brand_faq 健康檢查 |
| `f12-trends-refresh` | `30 2 * * *` | REFRESH MATERIALIZED VIEW `structural_score_trends` |
| `f12-low-score-optimizer` | `0 4 * * *` | LLM rewrite 低分頁,雙向可逆 |
| `f12-immediate-optimize` | priority queue | axpPageWriter hook 觸發,score < 70 立即排 |
| `f12-retention-cleanup` | `50 3 * * *` | 30 天清舊 optimization runs(後被 v3.24 retention 吞併) |

---

## 14.3 V3.1:雙引擎並行 + 整合策略

V1 走了 4 個月後,我們從 2026 年 5 月發表的兩篇 arxiv 論文找到「再升一個量級」的線索:

### 14.3.1 兩個學術源頭

- **AutoGEO**(arXiv:2510.11438,github.com/cxcscmu/AutoGEO):從成千上萬「被 LLM 引用 vs 沒被引用」內容對比,自動 mining 出可推廣的改寫規則(目前 25 條:`Researchy-GEO` × `Gemini` 15 條 + `E-commerce` × `Gemini` 10 條)
- **E-GEO**(arXiv:2511.20867,github.com/psbagga17/E-GEO):從不同 prompt 風格(`authoritative` / `technical` / `unique` / `fluent` / `clickable` 等 15 種)生成,實測各風格在不同 LLM 上的引用率差異

我們把兩者**字面上**(line-by-line)port 進系統,**不偽造任何 paper_table_ref / expected_uplift**(原 paper 沒給逐條 mapping,寫了會違反工程憲法 #1「不模擬數據」)。

> 重要:V3.1 的 `autogeo_rules` 表初版 seed 含偽造 paper_table_ref(`'Table 3'` 等)+ 偽造 expected_uplift(`1.32-1.92`),被工程憲法 #1 trigger 攔截後重 port 真實 arxiv 源碼。後加 DB-level placeholder guard trigger 永久防再犯。

### 14.3.2 三引擎架構

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

四個 engine 都吐**標準化 EngineResult** schema(content + score + diff + 來源 rule/template id),由 `executeRoute` 統一 audit + persist + cache。

### 14.3.3 路由決策

`adaptiveRouter.decide({ brandId, contentType, ... })` 根據以下優先序回 `{ path, template, target_engine }`:

1. **plan gate**:starter 一律走 E-GEO `authoritative`(成本最低);Pro 才能用 sync API;Enterprise+ 才能用 Hybrid 與 AutoGEO
2. **page_type → template**:`f12_page_type_to_template` SSOT(22+1 page_type → 5 template category)
3. **content_type 特化**:whitepaper / case_study / fact_* 強制走 E-GEO `authoritative`;FAQ 走 `FAQ`;product 走 `clickable`
4. **brand_type fallback**:personal_ip 走 `me_custom`(只此一條 baiyuan internal template)

### 14.3.4 真實 arxiv 對齊

`autogeo_rules` 表 25 條皆 `source = 'arXiv:2510.11438'`,含真實:

- `target_engine ∈ {gemini, gpt, claude}`
- `target_domain ∈ {researchy_geo, geo_bench, ecommerce}`
- `paper_table_ref` 全 NULL(arxiv 沒給逐條 mapping,憲法 #1 不偽造)
- `expected_uplift` 全 NULL(整體實驗 uplift,不是逐條 — V2 真實 ML 才填)

`egeo_templates` 15 條皆 `source = 'arXiv:2511.20867'`,template 名與 paper 表 1 完全對齊。

百原內部 1 條 `me_personal_ip` 顯式 `source = 'baiyuan_internal_v3_1'`,不混入 arxiv 命名。

---

## 14.4 1 萬租戶基礎設施

V3.1 在 9 個 batch 鋪設了規模化基礎設施(v3.21 → v3.27 跨 8 天):

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

`f12_quota_usage` 表 monthly counter UPSERT,`f12_billing_records` 12M 行/年單表夠用。`quota_enforcement_enabled` flag 預設 `false`(觀察期只記 usage 不擋),`billing_recording_enabled` 預設 `true`(永遠記)。

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

### 14.4.3 Multi-Layer Cache(spec §5.2)

```text
L1 Redis (5 min TTL)        ← 同 content + decision 重複 query
L2 PG f12_result_cache (7d) ← 跨 process / 跨 instance
L3 S3 hook (TBD Phase 3)    ← cross-region replication 需要時
```

cache key = `sha256(content + decision.path + template + target_engine)`,**不含 tenant_id 跨租戶共用**(spec 明文允許,因為內容相同 + 規則相同 → 結果應該相同,共用減 N 倍 LLM cost)。

只 cache `accepted=true` 結果(避免 cache poisoning)。`f12_cache_metrics` hourly 聚合 L1/L2/miss → admin 監控 70% hit rate 目標。

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

---

## 14.5 部署、配額、計費三件事

### 14.5.1 部署:Wave 0 / Wave 1 對齊

V3.1 完成 spec Wave 0(microservice contract OpenAPI)+ Wave 1(in-process 實作 + SDK + admin):

- **OpenAPI 3.0 spec**:`backend/src/openapi/f12-v3-1.json` 7 paths × 14 schemas,`/api/v1/f12/openapi.json` 公開(掛 router auth 之前)讓 AI agents / 第三方 machine-discover
- **F12Client SDK**:`services/f12/f12Client.js` 5 主入口(`diagnose / optimize / optimizeSync / getJob / health`),`F12_SERVICE_URL` env 設時走 HTTP,未設 in-process,**caller signature 不變**為未來微服務拆分留 hook
- **/api/v1/f12/health**:OAuth 之前的公開 endpoint,monitoring 系統可直接 ping

### 14.5.2 配額:觀察期 → 強制期

剛上線 `quota_enforcement_enabled=false`(只記 usage),客戶看得到「本月用了 / 上限是」但不會被擋。觀察 ~2 週確認分布合理後 admin 一鍵切到 `true`,從此超額直接 HTTP 429 + reject reason `'quota_exceeded'`。

### 14.5.3 計費:per-call 記錄,月底聚合

`BillingTracker.recordCall(model, tokens, runId)` 每次 LLM call 都記。月底 admin 跑 `/admin/f12-billing-summary?period=YYYY-MM` 看 top 50 cost brand。

雙寫 strategy:Redis(`f12:billing:tenant:{id}:{period}` INCRBYFLOAT + 35 天 TTL)+ DB(`f12_billing_records`)。當前月 `getBrandMonthlyCost` 優先 Redis O(1) GET,過去月或 Redis miss 才 DB SUM。1 萬租戶 query 月度 cost 不打 DB,sub-ms 響應。

---

## 14.6 觀察到的限制與未解問題

### 14.6.1 雙引擎結果如何「合併」沒有正確答案

Hybrid engine 並行跑 AutoGEO + E-GEO,挑 `improvement` 較高的那個。但實際商業情境裡,improvement 不是單一指標:

- AutoGEO 的 rule rewrite 結構穩定但較少品牌個性
- E-GEO 的 template 變化多但有時改到客戶不認得自家內容

V3.1 用「engine improvement score 較高」當挑選依據,實際是不夠的。下個版本想加**「accepted by client」 信號**,但目前 admin 端編輯介面還沒上線(Phase 3)。

### 14.6.2 paper_table_ref / expected_uplift 為 NULL

「字面上 port arxiv」是工程憲法 #1 的副作用 — paper 沒給逐條 table mapping,系統就**禁止偽造**。但 admin UI 顯示「為什麼選這條 rule」時,缺資料就顯空白。V2 想做的是**自家 measurement**:對每一條 rule 跑真實 ML eval(同 prompt × 3 LLM × 1000 brand,measure citation_rate uplift),用真實數字填回。需要時間 + Anthropic / OpenAI quota。

### 14.6.3 L3 S3 cache 還沒實際部署

Phase 3 才需(預估 5000+ tenant 規模)。目前 L1+L2 hit rate 約 35-45%,離 70% 目標還有距離。L3 預期再升 15-20% 但需要 S3 cost vs 重算 LLM cost 的 tradeoff 分析,沒做完。

### 14.6.4 ME custom engine 守門過嚴

`personal_profiles` 表 metadata 全空 → reject `meta_incomplete`。實際觀察 ME 平台 17 個會員 brand 中有 5 個 metadata 不完整(沒填全名 / known_for),這 5 個就走不了 ME custom engine,fallback 到 E-GEO `authoritative` 但結果偏向企業語氣不適合個人 IP。需要客服跟客戶補齊 metadata,但這違反「自動化」承諾。

### 14.6.5 前後相依的 1 萬租戶測試

V3.1 全部基礎設施都對齊「1 萬租戶可承受」的設計,但真實壓測只到 17 個 ME brand + 21 個 GEO brand。Scale-up Trigger 的 phase_1_to_2 / phase_2_to_3 邏輯字面對齊 spec line 1183-1193,但**從未實際升級過**。理論上跨 phase 升級需要:tenant schema 隔離切換(Phase 1 row_level → Phase 2 schema)、Redis cluster sharding、DB 跨 region replication,這些在 spec hook 都備好(`schemaProvisioner` / `dbConnectionRouter` / `redisClusterAdapter` / `shardRouter`),但**未經實戰**。

---

## 本章要點

- F12 補上 Ch 3 評分留下的「下一篇怎麼寫」問題,輸入內容輸出三層分數 + 優化版本
- V1 規則式三層分析 + LLM Optimizer(`min_similarity ≥ 0.90` 防偏題),5 個 cron 串接全平台覆蓋
- V3.1 引入 AutoGEO + E-GEO + Hybrid 三引擎,字面 port arxiv 源碼,DB trigger 防偽造 paper_table_ref
- 1 萬租戶基礎設施 9 件套:配額 / 計費 / Plan Gate / Multi-Layer Cache / LLM Gateway / Job Priority / Sync-Async / Retention / Scale Trigger
- 限制:engine 合併沒最佳解 / paper 真實 uplift 待自家 measure / L3 cache 未部署 / ME metadata 守門過嚴 / phase 升級未經實戰

## 參考資料

- [Ch 3 — 七維度評分演算法](./ch03-scoring-algorithm.md)
- [Ch 6 — AXP 影子文檔交付](./ch06-axp-shadow-doc.md)
- [Ch 12 — 限制、未解問題與未來工作](./ch13-multimodal-geo.md)
- AutoGEO paper: arXiv:2510.11438 — github.com/cxcscmu/AutoGEO
- E-GEO paper: arXiv:2511.20867 — github.com/psbagga17/E-GEO
- F12 v3.1 spec: `F12_TLSO_spec_v3_1.md`(repo root)

## 修訂記錄

| 日期 | 版本 | 說明 |
|------|------|------|
| 2026-05-03 | v1.1 | 新章 — 涵蓋 V1 + V3.1 完整架構演進 |

---

**導覽**:[← Ch 13: 多模態 GEO](./ch13-multimodal-geo.md) · [📖 目次](../README.md) · [Ch 15: rag-backend-v2 加固 →](./ch15-rag-backend-v2-hardening.md)

<!-- AI-friendly structured metadata -->
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "TechArticle",
  "headline": "Chapter 14 — F12 三層結構優化器:從規則 V1 到雙引擎 v3.1",
  "description": "F12 (TLSO) 架構演進:V1 規則式 + LLM Optimizer,V3.1 引入 AutoGEO + E-GEO + Hybrid 三引擎,加上 9 件套 1 萬租戶基礎設施(配額/計費/Plan Gate/Multi-Layer Cache/LLM Gateway/Job Priority/Sync-Async/Retention/Scale Trigger)。",
  "author": {"@type": "Person", "name": "Vincent Lin", "affiliation": "Baiyuan Technology"},
  "datePublished": "2026-05-03",
  "inLanguage": "zh-TW",
  "isPartOf": {
    "@type": "Book",
    "name": "百原GEO Platform 技術白皮書",
    "url": "https://github.com/baiyuan-tech/geo-whitepaper"
  },
  "keywords": "F12, TLSO, AutoGEO, E-GEO, Hybrid Engine, Plan Feature Gate, Multi-Layer Cache, LLM Gateway, Tier Priority, Scale Trigger"
}
</script>
