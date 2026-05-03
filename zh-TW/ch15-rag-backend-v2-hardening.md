---
title: "Chapter 15 — rag-backend-v2 加固:六層 LLM hallucination 失敗模式的防禦設計"
description: "中央 RAG 微服務 (rag-backend-v2) 的工程加固史:從 Wiki Cascade 四層級 cascade、六種 wikiCompiler hallucination 失敗模式的修法 (cutoff / partial UUID / no marker / DNS / LLM timeout / reasoning maxTokens),到 Queue retention 自動 cleanup,完整呈現 LLM-driven 系統的 robustness 設計實踐。"
chapter: 15
part: 5
word_count: 4200
lang: zh-TW
authors:
  - name: Vincent Lin
    affiliation: Baiyuan Technology
license: CC-BY-NC-4.0
keywords:
  - rag-backend-v2
  - LLM Wiki Compiler
  - Wiki Cascade
  - LLM Hallucination Hardening
  - Docker DNS
  - LLM Timeout
  - AbortSignal
  - Multilingual Prompts
  - BullMQ Retention
  - PostgreSQL pgvector
last_updated: 2026-05-03
canonical: https://baiyuan.io/whitepaper/zh-TW/ch15-rag-backend-v2-hardening
last_modified_at: '2026-05-03T10:25:18+08:00'
---


# Chapter 15 — rag-backend-v2 加固:六層 LLM hallucination 失敗模式的防禦設計

> 在 LLM 不能信賴的世界裡,工程的職責不是強迫 LLM 完美,而是為它每一種可能的失敗形態都備好出路。

## 目錄

- [14.1 為什麼 RAG 需要獨立微服務 + 自家 Wiki](#141-為什麼-rag-需要獨立微服務--自家-wiki)
- [14.2 Wiki Cascade:四層級級聯刪除](#142-wiki-cascade四層級級聯刪除)
- [14.3 六種 wikiCompiler hallucination 失敗 + 對應 patch](#143-六種-wikicompiler-hallucination-失敗--對應-patch)
- [14.4 Worker robustness:從 silent hang 到自動恢復](#144-worker-robustness從-silent-hang-到自動恢復)
- [14.5 1 萬租戶部署模式](#145-1-萬租戶部署模式)
- [14.6 觀察與未解問題](#146-觀察與未解問題)

---

## 14.1 為什麼 RAG 需要獨立微服務 + 自家 Wiki

GEO 主後端跟 `rag-backend-v2` 雖然在同一個 PostgreSQL instance 內,但**兩個 DB 完全分離**:

| | `geo_db` | `cs_rag_db` |
|---|---|---|
| 服務 | GEO Platform 主 backend | rag-backend-v2 微服務 |
| 用途 | brands / users / axp_pages / ground_truths | tenant_documents / chunks / wiki_pages |
| 角色 | 業務邏輯 + 多租戶資料 | 向量檢索 + LLM Wiki 編譯 |
| 跨庫查詢 | `csRagQuery(text, params)` 獨立 pool | — |

切分的理由有三:

1. **資料量級不同**:RAG 的 `chunks` 表(向量 embedding)會隨客戶量爆量,單表 100M row 不少見,跟業務表混在一起會傷主庫效能
2. **更新節奏不同**:RAG 文件是「上傳即不動」,主庫業務資料是「秒級更新」
3. **降級邊界清晰**:RAG 掛了,主平台還能跑(只是 AI 回答品質低);主庫掛了,什麼都壞 — 給 RAG 獨立 surface 讓 fault isolation 可行

### 14.1.1 Wiki 的角色

V1 RAG 只有 chunks(L2 向量檢索)+ pgvector + BM25 + RRF。L1 layer 是 2026 年 4 月加的「**Karpathy LLM Wiki**」概念實作:把 tenant 上傳的源文件先**LLM 編譯**成結構化 wiki pages(每頁有 slug / page_type / TLDR / tags),先用 wiki 回答(快、便宜、可解釋),失敗才 fallback 到 L2。

實測影響:同樣 query「百原 GEO 怎麼幫品牌提升 AI 引用率」:

- L2 only:檢索 5 個 chunks → LLM rewrite → 12 秒
- L1+L2:wiki 直接回 → 1.5 秒 + LLM cost 砍 80%

但 L1 的成立有一個前提:**Wiki 編譯必須穩定**。實際上,我們 4 月初到 5 月初一個月內,踩了 6 種 LLM hallucination 形態的雷,每一種都讓 wiki 編譯部分或全部失敗。本章記錄這六層加固。

---

## 14.2 Wiki Cascade:四層級級聯刪除

當客戶在前端按「刪除文件」,我們需要連帶清掉:

```
brand_documents (geo_db)
        │ via central_doc_id UUID
        ▼
tenant_documents (cs_rag_db)        ← LLM Wiki 編譯來源
        │ via wiki_page_sources junction
        ▼
wiki_page_sources                   ← 多對多關聯表
        │ via SSOT 同步 trigger
        ▼
wiki_pages.source_doc_ids           ← 被引用的 wiki page
```

四層全自動 cascade 透過:

1. `brand_documents.central_doc_id` 欄位(migration 170)— 上傳時 capture 中央回傳的 doc id 寫進此欄
2. DELETE handler:`ragDeleteDocument(tenantId, central_doc_id)` → 中央 `trg_source_soft_delete` cascade trigger 自動清 junction
3. junction 失去最後一個 source 的 wiki_pages 自動軟刪
4. `migration 201:sync_wiki_page_sources trigger` — 強制 `wiki_pages.source_doc_ids` array 與 `wiki_page_sources` junction 同步(SSOT)

設計關鍵:**junction 是 SSOT,array 是冗餘 derivation**。原本兩者各自維護,常 desync(觀察到 36 stale junction entries),migration 201 後 trigger 強制兩者一致,客戶端「N 個 Wiki 頁」UI 顯示就永遠正確。

### 14.2.1 UAT vs PROD 行為差異

UAT 沒中央 RAG(`CENTRAL_RAG_URL` unset)→ Phase 1 path 自動 skip,本地 brand_documents 純走 pgvector adapter,**不影響主流程**(有 try/catch 不擋)。PROD 有中央 RAG → 完整四層 cascade。這條對齊「降級邊界清晰」的設計。

---

## 14.3 六種 wikiCompiler hallucination 失敗 + 對應 patch

每一種失敗都來自真實 PROD 觀察,以下用 `Patch N` 編號順序記錄:

### 14.3.1 Patch 1:150K char cutoff bug

**失敗形態**:某 KB 84 個 docs 只有 33 個編進 wiki,**51 個永久 orphan**。

**根因**:`compileWiki()` function 開頭有一段 cutoff 邏輯,從 `compileIncremental` 借過來但**邏輯不適用**:

```js
// Buggy 原版
let totalChars = 0;
const selectedDocs = [];
for (const doc of docs) {
  if (totalChars + (doc.content?.length || 0) > 150000) break;  // 砍超過 150K 後的全部 doc
  selectedDocs.push(doc);
  totalChars += doc.content?.length || 0;
}
```

問題在於 main `compileWiki` 對每份 doc 跑**獨立 LLM call**,沒有 aggregate token 限制理由。但 `compileIncremental` 是把多 doc 拼在一個 prompt 裡,有 prompt token 限制。code 借過來時沒看清楚使用情境。

且 `ORDER BY created_at ASC` 讓**早期 doc 先進** selectedDocs,新 doc 永遠被砍。1 萬租戶 scale:84 docs × 4.5K avg = 378K → 只編 33 docs → 51 docs 永遠 orphan。

**修法**:刪掉 cutoff,全部 ready wiki source 都納入(per-doc LLM call 為 sequential,不會 burst):

```js
// Patched
const selectedDocs = docs;
const totalChars = docs.reduce((s, d) => s + (d.content?.length || 0), 0);
```

### 14.3.2 Patch 2:parseCompiledArticles partial UUID

**失敗形態**:整個 KB 的 compile 中斷,後續 docs 全沒編。PROD log:

```
[WIKI] Compile error: invalid input syntax for type uuid: "b7bced27"
```

**根因**:LLM(DeepSeek)在 `SOURCE_IDS:` header 行偶爾輸出截斷的 UUID(8 字元 prefix 而非完整 36 字元)。原 code 沒驗證,直接傳給 PG `INSERT INTO wiki_pages(source_doc_ids UUID[])` → PG RAISE → catch 不住 → 整批 compile transaction rollback。

**修法**:正規式驗證 LLM 輸出:

```js
const UUID_RE = /^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$/i;
const sourceDocIds = sourceIdsRaw
  ? sourceIdsRaw.split(',').map(s => s.trim()).filter(s => UUID_RE.test(s))
  : [];
```

filter 後若空陣列,後段「保底」邏輯仍 prepend `doc.id` 真實 UUID,LLM 給 garbage 不影響歸屬正確性。

### 14.3.3 Patch 3:fallbackWrapArticle — 防 LLM marker hallucination

**失敗形態**:LLM 偶爾忽略 prompt 規定的 `---ARTICLE---/---CONTENT---` marker 直接吐純文字。`parseCompiledArticles` 找不到 marker → 回 0 article → 該 doc 永久 orphan。PROD 觀察 `02_醫美業_FAQ_與_Query集.md` 編譯 LLM 輸出 5670 字無 marker。

**根因**:LLM 輸出 schema 不可信,任何依賴特定 token / marker / JSON shape 的 parser 都會偶發失敗。

**修法**:寫 fallback wrap helper,marker 缺失但有 ≥50 字內容時保底包成 single article:

```js
function fallbackWrapArticle(text, doc) {
  const trimmed = (text || '').trim();
  if (trimmed.length < 50) return null;
  return {
    title: doc.title.replace(/\.md$/i, '').slice(0, 100),
    slug: deriveSlug(doc.title) || `fallback-${doc.id.slice(0, 8)}`,
    pageType: heuristicPageType(doc.title),  // faq/product/policy/concept
    confidence: 'low',                        // 標 low 提示 admin 人工檢視
    tags: ['wiki_fallback'],                  // admin 用此 tag 過濾
    content: trimmed,
    sourceDocIds: [doc.id],                   // 與原 doc 雙向綁定
    // ...
  };
}
```

關鍵設計:**寧可保底用低信心包裝(tags=wiki_fallback)+ admin 警示,也不能讓客戶上傳的源文件變孤兒**。`compileWiki` per-doc loop 與 `compileIncremental` 批次 hook 雙路徑都套用。

### 14.3.4 Patch 4:Docker DNS 內外雙 fallback

**失敗形態**:rag 容器無法解析 `postgres` 內部 hostname → `pg.Pool.connect("postgres")` SERVFAIL **silent hang**(不 throw)→ wikiCompiler 第一個 `await q(SELECT...)` 永遠等不到 connection → 整個 wiki compile 表面正常但 0 進度。

**根因**:容器 `/etc/resolv.conf` 指向 host `systemd-resolved`(`127.0.0.53`)而不是 Docker 內部 DNS(`127.0.0.11`),Lightsail Ubuntu daemon 的 DNS 配置邊角案例。

**修法**(三版演進,前兩版都失敗):

| 版本 | 配置 | 結果 |
|---|---|---|
| 初版 | `extra_hosts: ["postgres:172.18.0.10"]` 硬編 IP | 失敗:postgres IP 在 docker recreate 時 shift |
| 第二版 | `dns: ["127.0.0.11"]` only | 災難:解不到 `www.googleapis.com` → Google OAuth 全平台壞 |
| 最終版 | `dns: ["127.0.0.11", "1.1.1.1", "8.8.8.8"]` | 通過:內外 hostname 都通 |

教訓:**單純設 Docker embedded DNS 不夠,還必須保留外部 fallback**。`docker-compose.prod.yml` 對 backend / worker / frontend / rag 全部都要這樣寫。

### 14.3.5 Patch 5:LLM call 60s timeout

**失敗形態**:wikiCompiler progress 卡在「`AI 編譯 (13/84):官網-FAQ`」10+ 分鐘,CPU 0.01%,DB 0 active query。`aiCall` 在等 DeepSeek 永不返回的 stream。

**根因**:OpenAI SDK 預設 `timeout: 600_000ms`(10 分鐘)+ `maxRetries: 2`,理論上 20 分鐘卡死;Gemini SDK(`@google/generative-ai`)沒提供原生 timeout / abort API。DeepSeek 偶有 `ECONNRESET` / 半開連線 / 低速回應,預設 timeout 撐不住。

**修法**(三層 timeout):

```js
// 1) OpenAI client constructor 60s + maxRetries=1
new OpenAI({ apiKey, timeout: 60_000, maxRetries: 1 });

// 2) per-request abort signal
client.chat.completions.create(params, { signal: AbortSignal.timeout(60_000) });

// 3) Gemini Promise.race + 60s timeout(SDK 無 abort)
withTimeout(genModel.generateContent(...), 60_000, 'gemini/${model}')
```

修後:單一 doc 失敗最多卡 60s,wikiCompiler per-doc try/catch 接住 → 走 Patch 3 fallback 或跳下一 doc,**整批 84 docs 不再被單一 LLM hang 拖垮**。1 萬租戶 scale 安全。

### 14.3.6 Patch 6:wiki_query_route maxTokens 200 → 2000

**失敗形態**:Wiki L1 全平台 miss,大多查詢 fallback L2,回「沒有相關資料」。

**根因**:PROD primary LLM 自 2026-04-30 切到 `deepseek-v4-flash`(reasoning model)。reasoning model 的 `message.content` 之外另吃 `reasoning_tokens`。`wikiCompiler.js` 兩處 `wiki_query_route` call 用 `maxTokens: 200` 對 deepseek-v3 夠,但 V4 reasoning **把 200 token 全部吃進 reasoning,content 永遠空字串** → JSON parse fail → wiki query 路由失敗。

**修法**:兩處 `maxTokens` 200 → 2000。

**1 萬租戶影響**:任何啟用 wiki + 用 V4-style reasoning model(DeepSeek V4 / OpenAI o1-o4 / Anthropic 含 thinking)的租戶,Wiki L1 全失靈,fallback L2 雖能勉強回答但成本高且品質差。這條教訓推廣:**換 reasoning model 後必查所有 maxTokens 配置**。

### 14.3.7 全部 6 patch 的整體效果

部署完整 6 patch 後 PROD 觀察:

| 指標 | Before | After |
|---|---|---|
| 平台 wiki orphan rate | 45%(156/348) | ≤25%(預期) |
| 5ce78a51 KB(84 docs)compile 完成率 | 39%(33/84) | 100%(84/84) |
| 單 doc LLM hang 卡死整批機率 | ~20% | <1% |
| Wiki L1 hit rate | 12%(reasoning model 後)| 65% |

---

## 14.4 Worker robustness:從 silent hang 到自動恢復

WikiCompiler 失敗只是冰山一角。整個 BullMQ worker 體系也踩了同類陷阱。

### 14.4.1 Defensive guard pattern

每個 cron worker 業務邏輯前必先檢查 entity 存在:

```js
async (job) => {
  const { brandId } = job.data;

  // 預檢 — brand 被刪後 cron 還在跑會 throw → BullMQ 累積 failed jobs → admin UI 永遠紅
  const { rows: [b] } = await query('SELECT id FROM brands WHERE id = $1', [brandId]);
  if (!b) {
    console.warn(`[Worker] Brand ${brandId} not found — removing orphan scheduler`);
    const q = new Queue(QUEUE_NAME, { connection });
    await q.removeJobScheduler(`monthly-${brandId}`);  // 自動清孤兒 scheduler
    await q.close();
    return { skipped: true, reason: 'brand_not_found', brandId };
  }

  // ... 業務邏輯
}
```

關鍵點:**檢查不過 → return skipped 不是 throw**。throw 算 failed,return 算 completed,UI 永遠綠。同時順手 `removeJobScheduler` 把 orphan scheduler 也清掉。

PROD 觀察套用範圍:`monthly.worker.js`(brand 被刪)、`abTesting.worker.js`(experiment 被刪 → FK violation)、`closedLoop sentinel`(table 被改名 `brand_prompts → multilingual_prompts`)。

### 14.4.2 Queue retention worker

`backend/src/workers/queueRetention.worker.js` 每日 02:30 跑:

```js
// 24 個 known queue,daily cleanup
for (const name of KNOWN_QUEUES) {
  const q = new Queue(name, { connection });
  await q.clean(7 * 24 * 3600 * 1000, 500, 'failed');     // 7 天 failed
  await q.clean(30 * 24 * 3600 * 1000, 500, 'completed'); // 30 天 completed
  await q.close();
}
```

不清的後果:per-brand cron 每天產生大量 jobs,失敗的若不清會撐到 Redis OOM,且 admin UI(`/admin/scan-frequencies`)永遠紅燈累積看不出新狀態。

### 14.4.3 LLM call site timeout 全平台對齊

除了 wikiCompiler,geo-saas backend 同步加上 LLM timeout(防同類 silent hang):

- `services/modelRouter.service.js` Anthropic SDK `timeout: 60000, maxRetries: 0`(原預設 600s)
- `services/modelRouter.service.js` Gemini call 加 `Promise.race` + 60s timeout
- `services/tier-a/visualEnrichment.service.js` Anthropic Vision API fetch 加 `signal: AbortSignal.timeout(60_000)`

驗證指令(commit 前自查):

```bash
grep -rn "fetch(.*api\.\(deepseek\|openai\|anthropic\|googleapis\|xiaomimimo\)" backend/src/services/ \
  | grep -v "AbortSignal\|signal:"
# 應為空,有 = 違反 LLM timeout 鐵律
```

---

## 14.5 1 萬租戶部署模式

`rag-backend-v2` 不在 geo-saas git repo(獨立微服務),patch 紀錄保留於 `docs/rag-backend-v2-patches/`,每次 RAG 容器重建必須:

```bash
# 1. 從 patches/ 取最新版
scp wikiCompiler.js     ubuntu@PROD:/home/ubuntu/rag-backend-v2/src/services/
scp modelRouter.js      ubuntu@PROD:/home/ubuntu/rag-backend-v2/src/services/
scp docker-compose.rag.yml  ubuntu@PROD:/home/ubuntu/geo-saas/

# 2. 重 build + recreate
ssh ubuntu@PROD
cd /home/ubuntu/rag-backend-v2
sudo docker build -t geo-saas-prod-rag:latest .

cd /home/ubuntu/geo-saas
sudo docker compose -f docker-compose.rag.yml --project-name geo-saas-prod up -d --force-recreate rag

# 3. 驗證 6 個 patch 全 live
sudo docker exec geo-saas-prod-rag-1 sh -c '
  grep -c "selectedDocs = docs" /app/src/services/wikiCompiler.js     # patch 1: 1
  grep -c UUID_RE /app/src/services/wikiCompiler.js                   # patch 2: 2
  grep -c fallbackWrapArticle /app/src/services/wikiCompiler.js       # patch 3: 3
  grep -c postgres /etc/hosts                                         # patch 4 prep: 1+
  grep -c LLM_CALL_TIMEOUT_MS /app/src/services/modelRouter.js        # patch 5: 8
  grep -c "maxTokens: 2000" /app/src/services/wikiCompiler.js         # patch 6: ≥3
'
```

每次 PROD 容器重建必須帶這個 verification block,否則 patch 會默默丟失重蹈覆轍。

---

## 14.6 觀察與未解問題

### 14.6.1 LLM 不可信任的本質

六層 patch 看似已涵蓋常見失敗模式,但 LLM 演進速度快過 patch 速度:

- 換 model(deepseek-v3 → v4-flash)會帶來新 hallucination 形態(`maxTokens` 對 reasoning 不夠)
- 不同 prompt 格式對不同 model 失敗模式不同(GPT-4 不會漏 marker,但會吐 prefix「Here's the response:」)
- streaming vs non-streaming 行為差異(streaming 可能中斷後不重發)

這意味著 LLM-driven 系統的 robustness 設計必須**永遠假設 LLM 會以新方式失敗**,每加一個新 model 都要重跑一輪 hallucination test。

### 14.6.2 fallback wrap article 的品質

Patch 3 的低信心 fallback 雖然救起了客戶 doc,但 `confidence='low'` + `tags=['wiki_fallback']` 的 wiki page 在 L1 retrieval 時實際品質如何,目前**沒做端到端 evaluation**。理論上應該:

- 不參與 wiki_query_route 的「你問什麼類別」LLM 路由(避免騙 user)
- 在 admin UI 顯示「⚠ 待審視」標記
- 累計 N 個失敗後系統 alert admin

第三點還沒做。

### 14.6.3 多 LLM 路由 vs 單 LLM 加固

理論上,如果同一 query 同時跑兩個不同 LLM(DeepSeek + Gemini),交叉驗證 marker / UUID / 內容,可以解掉大部分 hallucination 問題。但成本翻倍且延遲倍增,是否值得?目前數據不夠決策,留 V2 評估。

### 14.6.4 Wiki Cascade 的反向操作

刪除是 cascade 自動處理,但**還原**(undo soft-delete)沒做。客戶誤刪文件 → wiki_pages 軟刪 → 30 天後自動 hard delete。如果客戶 7 天後想要回來,目前要客服手工 SQL UPDATE。這是常見需求但工程沒排上。

### 14.6.5 Wiki 編譯失敗的可恢復性

V1 編譯失敗 → DB 寫 `last_compile_error`,UI 顯示「編譯失敗」。但**沒重試機制** — admin 必須手動點按鈕。1 萬租戶 readiness 應該:

- 失敗自動 retry 1 次(若是 LLM timeout 類短暫錯誤)
- N 次連續失敗 → admin alert,而不是默默死掉

也是 V2 工作。

---

## 本章要點

- rag-backend-v2 獨立微服務 + 自家 cs_rag_db + Wiki Cascade 4 層級聯刪除確保資料一致性
- 六種 LLM hallucination 失敗模式對應六層 patch:cutoff / partial UUID / no marker / DNS / LLM timeout / reasoning maxTokens
- Worker robustness 三件套:defensive guard(entity 預檢)、per-item try/catch、daily retention auto-cleanup
- 1 萬租戶部署必走 patches/ 目錄 SSOT,每次容器重建必跑 verification block
- 未解:LLM 演進速度 > patch 速度;fallback 品質未 eval;多 LLM 交叉驗證待評估;wiki 還原機制缺失;失敗重試缺失

## 參考資料

- [Ch 5 — 多 Provider 路由:錯誤處理層](./ch05-multi-provider-routing.md)
- [Ch 6 — AXP 影子文檔交付](./ch06-axp-shadow-doc.md)
- [Ch 14 — F12 三層結構優化器](./ch14-f12-structural-optimizer.md)
- BullMQ Job Cleaning API: <https://docs.bullmq.io/guide/queues/removing-jobs>
- Docker DNS configuration: <https://docs.docker.com/network/#dns-services>
- Karpathy on LLM Wiki concept(2024 公開分享)

## 修訂記錄

| 日期 | 版本 | 說明 |
|------|------|------|
| 2026-05-03 | v1.1 | 新章 — 六層 LLM hallucination patch + Wiki Cascade + Worker robustness |

---

**導覽**:[← Ch 14: F12 三層結構優化器](./ch14-f12-structural-optimizer.md) · [📖 目次](../README.md) · [Ch 16: 平台 SSOT 全鏈 →](./ch16-platform-ssot-chain.md)

<!-- AI-friendly structured metadata -->
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "TechArticle",
  "headline": "Chapter 15 — rag-backend-v2 加固:六層 LLM hallucination 失敗模式的防禦設計",
  "description": "中央 RAG 微服務 (rag-backend-v2) 工程加固史:Wiki Cascade 4 層級聯、6 種 wikiCompiler hallucination patch、Worker defensive guard 與 Queue retention,完整呈現 LLM-driven 系統 robustness 設計實踐。",
  "author": {"@type": "Person", "name": "Vincent Lin", "affiliation": "Baiyuan Technology"},
  "datePublished": "2026-05-03",
  "inLanguage": "zh-TW",
  "isPartOf": {
    "@type": "Book",
    "name": "百原GEO Platform 技術白皮書",
    "url": "https://github.com/baiyuan-tech/geo-whitepaper"
  },
  "keywords": "rag-backend-v2, LLM Wiki Compiler, Wiki Cascade, LLM Hallucination Hardening, Docker DNS, LLM Timeout, BullMQ Retention"
}
</script>
