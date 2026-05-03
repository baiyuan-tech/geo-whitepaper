---
title: "Chapter 15 — rag-backend-v2 加固:六層 LLM hallucination 失敗模式的防禦設計"
description: "中央 RAG 微服務 (rag-backend-v2) 的工程加固史:從 Wiki Cascade 四層級 cascade、六種 wikiCompiler hallucination 失敗模式的修法 (cutoff / partial UUID / no marker / DNS / LLM timeout / reasoning maxTokens),到 Queue retention 自動 cleanup,完整呈現 LLM-driven 系統的 robustness 設計實踐。"
chapter: 15
part: 5
word_count: 7100
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
last_modified_at: '2026-05-03T04:52:06Z'
---




# Chapter 15 — rag-backend-v2 加固:六層 LLM hallucination 失敗模式的防禦設計

> 在 LLM 不能信賴的世界裡,工程的職責不是強迫 LLM 完美,而是為它每一種可能的失敗形態都備好出路。

## 目錄

- [15.1 為什麼 RAG 需要獨立微服務 + 自家 Wiki](#151-為什麼-rag-需要獨立微服務--自家-wiki)
- [15.2 Wiki Cascade:四層級級聯刪除](#152-wiki-cascade四層級級聯刪除)
- [15.3 六種 wikiCompiler hallucination 失敗 + 對應 patch](#153-六種-wikicompiler-hallucination-失敗--對應-patch)
- [15.4 Worker robustness:從 silent hang 到自動恢復](#154-worker-robustness從-silent-hang-到自動恢復)
- [15.5 1 萬租戶部署模式](#155-1-萬租戶部署模式)
- [15.6 觀察與未解問題](#156-觀察與未解問題)
- [15.7 工程教訓](#157-工程教訓)

---

## 15.1 為什麼 RAG 需要獨立微服務 + 自家 Wiki

### 15.1.1 Karpathy LLM Wiki 概念簡介

Andrej Karpathy 在 2024 年公開分享過一個概念:傳統 RAG 直接把 chunk 餵給 LLM 太「碎」,沒有結構;不如先用 LLM 把所有源文件**編譯成 wiki**,每篇 wiki 有清楚的 slug / page_type / TLDR / tags,query 時先看 wiki(快、便宜、可解釋),wiki 沒答案才 fallback 到 chunk-level RAG。

我們把這個概念**真實實作**到 cs-frontend(企業 RAG 知識庫產品)+ GEO Platform 共用的 rag-backend-v2 微服務,不只當宣傳概念用,實際跑生產:

- 客戶上傳源文件(PDF / Markdown / URL crawl)→ 進 `tenant_documents` 表
- LLM compiler 把所有 ready-to-compile 的 doc 編成 wiki pages → 進 `wiki_pages` 表
- 用戶 query → wiki_query_route LLM 先決定「這 query 屬於哪個 page_type」→ 拿對應 wiki 直接回
- wiki 沒命中 → fallback 到 L2 chunk-level RAG(pgvector + BM25 + RRF)

這個 L1+L2 hybrid 跑出來的效益:同樣 query「百原 GEO 怎麼幫品牌提升 AI 引用率」:

- L2 only:檢索 5 個 chunks → LLM rewrite → 12 秒,LLM cost ≈ $0.008
- L1+L2:wiki 直接回 → 1.5 秒,LLM cost ≈ $0.0012(砍 85%)

但 L1 的成立有一個前提:**Wiki 編譯必須穩定**。實際上,我們 4 月初到 5 月初一個月內,踩了 6 種 LLM hallucination 形態的雷,每一種都讓 wiki 編譯部分或全部失敗。本章記錄這六層加固。

### 15.1.2 為什麼 RAG 是獨立微服務

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

### 15.1.3 跨庫查詢的 cost

`backend/src/config/database.js` 開兩個獨立 PG pool:

```javascript
const geoPool = new Pool({ connectionString: DATABASE_URL });
const csRagPool = new Pool({
  connectionString: DATABASE_URL.replace(/\/geo_db$/, '/cs_rag_db'),
});

async function csRagQuery(text, params) {
  try {
    return await csRagPool.query(text, params);
  } catch (err) {
    console.error('[csRagQuery] failed:', err.message);
    return { rows: [] };  // fail-soft 不擋主流程
  }
}
```

UAT 沒 cs_rag_db(`DATABASE_URL` 對應 single-DB instance)→ csRagQuery 直接 fail,但 try/catch 接住回空陣列,主流程不擋。PROD 雙 DB 都在,csRagQuery 正常工作。

代價:GEO 主後端要跨庫拿 `wiki_link_count` 顯示「N 個 Wiki 頁」必須跑 csRagPool query,延遲 +30ms vs 同庫 JOIN。但相比把兩 DB 合在一起的維運成本,這 30ms 可接受。

### 15.1.4 V1 與 V2 的演進

V1 的 RAG 只有 chunks(L2 向量檢索)+ pgvector + BM25 + RRF。L1 layer 是 2026 年 4 月加的「Karpathy LLM Wiki」概念實作。V1→V2 的決策驅動因素:

- L2-only 平均回應 12 秒,客戶嫌慢(尤其 cs-frontend 的客服場景)
- LLM cost / 月 / brand 約 $80,毛利為負
- 客戶問「為什麼回答這樣」,L2 chunks 太碎沒辦法解釋來源

V2 加 wiki 後,L1 hit 時 1.5 秒回 + 可指 wiki page_id 給「來源」,客戶滿意度提升。但 L1 必須穩定,任何編譯失敗都讓客戶看到「沒有相關資料」假象。

---

## 15.2 Wiki Cascade:四層級級聯刪除

### 15.2.1 為何需要四層 cascade

當客戶在前端按「刪除文件」,我們需要連帶清掉:

```text
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

### 15.2.2 SSOT trigger 的 SQL 實作

migration 201 的 trigger 強制兩者同步:

```sql
CREATE OR REPLACE FUNCTION sync_wiki_page_sources()
RETURNS TRIGGER AS $$
BEGIN
  -- INSERT/UPDATE wiki_pages.source_doc_ids → 同步 junction
  IF TG_TABLE_NAME = 'wiki_pages' AND (TG_OP = 'INSERT' OR TG_OP = 'UPDATE') THEN
    -- 刪除 junction 中不在 source_doc_ids 的 row
    DELETE FROM wiki_page_sources
     WHERE wiki_page_id = NEW.id
       AND NOT (source_doc_id = ANY(NEW.source_doc_ids));
    -- 插入 source_doc_ids 中不在 junction 的 row
    INSERT INTO wiki_page_sources(wiki_page_id, source_doc_id)
    SELECT NEW.id, doc_id
      FROM unnest(NEW.source_doc_ids) AS doc_id
     WHERE NOT EXISTS (
       SELECT 1 FROM wiki_page_sources
        WHERE wiki_page_id = NEW.id AND source_doc_id = doc_id
     );
  END IF;

  -- DELETE wiki_page_sources 後,若 wiki_page 失去最後一個 source → 軟刪
  IF TG_TABLE_NAME = 'wiki_page_sources' AND TG_OP = 'DELETE' THEN
    UPDATE wiki_pages
       SET deleted_at = NOW(), source_doc_ids = '{}'
     WHERE id = OLD.wiki_page_id
       AND NOT EXISTS (
         SELECT 1 FROM wiki_page_sources WHERE wiki_page_id = OLD.wiki_page_id
       );
  END IF;

  RETURN COALESCE(NEW, OLD);
END;
$$ LANGUAGE plpgsql;
```

部署這個 trigger 之前,我們的清資料邏輯散在 7 個地方(application code 4 處 + migration script 3 處),常 desync。trigger 上線後改 1 處不需動 7 處,1 萬租戶 scale 必要。

### 15.2.3 36 個 stale junction entry 的偵錯

trigger 上線時做了一次性 reconciliation:

```sql
-- 偵測 stale junction(junction 指向不存在的 wiki_pages 或 tenant_documents)
SELECT wps.*
  FROM wiki_page_sources wps
  LEFT JOIN wiki_pages wp ON wp.id = wps.wiki_page_id
  LEFT JOIN tenant_documents td ON td.id = wps.source_doc_id
 WHERE wp.id IS NULL OR td.id IS NULL;
-- → 36 row
```

一次性 DELETE 清掉,trigger 之後保證新增不再產生。這 36 個 entry 怎麼來的?Track 下去:

- 12 個是 application code 早期版本 INSERT junction 但忘記更新 array
- 14 個是 cron 重編 wiki 時 race condition(舊 wiki 軟刪但 junction 沒清)
- 10 個是 migration script 在歷史中嘗試 backfill 失敗留下殘骸

trigger 之後三類 case 都死路一條,以下保證:**junction 永遠 reflect array 內容**。

### 15.2.4 UAT vs PROD 行為差異

UAT 沒中央 RAG(`CENTRAL_RAG_URL` unset)→ Phase 1 path 自動 skip,本地 brand_documents 純走 pgvector adapter,**不影響主流程**(有 try/catch 不擋)。PROD 有中央 RAG → 完整四層 cascade。這條對齊「降級邊界清晰」的設計。

---

## 15.3 六種 wikiCompiler hallucination 失敗 + 對應 patch

每一種失敗都來自真實 PROD 觀察,以下用 `Patch N` 編號順序記錄。**這六個 patch 是我們在 2026 年 4 月底 5 月初一個禮拜內連環踩雷修出來的**,每一個都對應一個血淋淋的故障時刻。

### 15.3.1 Patch 1:150K char cutoff bug

**失敗形態**:某 KB 84 個 docs 只有 33 個編進 wiki,**51 個永久 orphan**。

**發現過程**:客戶反應「我上傳了 84 個 FAQ 文件,wiki 顯示 wiki 頁面數只有 33 頁」。一開始懷疑是 LLM 編譯失敗,看 log 全綠;再懷疑 doc 沒 ready,DB 檢查全部 `status='ready'`。最後 grep `wikiCompiler.js` 發現 cutoff 邏輯。

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

### 15.3.2 Patch 2:parseCompiledArticles partial UUID

**失敗形態**:整個 KB 的 compile 中斷,後續 docs 全沒編。PROD log:

```text
[WIKI] Compile error: invalid input syntax for type uuid: "b7bced27"
```

**發現過程**:Patch 1 deploy 後重跑 compile,跑到第 13 個 doc 整批 abort。log 顯示「invalid uuid」但完整 UUID 是 `b7bced27-3e4f-4abc-...`,看起來只截到前 8 字元。

**根因**:LLM(DeepSeek)在 `SOURCE_IDS:` header 行偶爾輸出截斷的 UUID(8 字元 prefix 而非完整 36 字元)。原 code 沒驗證,直接傳給 PG `INSERT INTO wiki_pages(source_doc_ids UUID[])` → PG RAISE → catch 不住 → 整批 compile transaction rollback。

關鍵:**單一 LLM 一次小錯誤,炸掉整個 batch**。1 萬租戶 scale 下,只要 LLM 偶發 hallucinate 一次,就整 KB 重跑。不可接受。

**修法**:正規式驗證 LLM 輸出:

```js
const UUID_RE = /^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$/i;
const sourceDocIds = sourceIdsRaw
  ? sourceIdsRaw.split(',').map(s => s.trim()).filter(s => UUID_RE.test(s))
  : [];
```

filter 後若空陣列,後段「保底」邏輯仍 prepend `doc.id` 真實 UUID,LLM 給 garbage 不影響歸屬正確性。

### 15.3.3 Patch 3:fallbackWrapArticle — 防 LLM marker hallucination

**失敗形態**:LLM 偶爾忽略 prompt 規定的 `---ARTICLE---/---CONTENT---` marker 直接吐純文字。`parseCompiledArticles` 找不到 marker → 回 0 article → 該 doc 永久 orphan。PROD 觀察 `02_醫美業_FAQ_與_Query集.md` 編譯 LLM 輸出 5670 字無 marker。

**發現過程**:Patch 2 deploy 後再跑,84 docs 全部 compile 完成,但 `wiki_pages` 只新增 51 個。grep 對應 doc id,確認某些 doc 沒對應 wiki 頁。檢查 LLM 輸出 log,發現某些 doc 的 LLM 直接吐純文字,連 `---ARTICLE---` 都沒。

**根因**:LLM 輸出 schema 不可信,任何依賴特定 token / marker / JSON shape 的 parser 都會偶發失敗。Anthropic / OpenAI / Gemini 都會偶爾「忘記指令」,DeepSeek 比例偏高(約 5-8%)。

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

admin UI 用 `WHERE 'wiki_fallback' = ANY(tags)` 過濾人工檢視,實測 84 docs KB 有 14 個進 fallback bucket,人工看內容大多 OK 只是格式不規整,signal 還是夠用。

### 15.3.4 Patch 4:Docker DNS 內外雙 fallback(三版演進)

**失敗形態**:rag 容器無法解析 `postgres` 內部 hostname → `pg.Pool.connect("postgres")` SERVFAIL **silent hang**(不 throw)→ wikiCompiler 第一個 `await q(SELECT...)` 永遠等不到 connection → 整個 wiki compile 表面正常但 0 進度。

**發現過程**:Patch 3 deploy 後 PROD 觀察 wiki 編譯「卡住」— admin UI 顯示 `progress: 0/84`,容器 CPU 0.01%,DB 0 active query,沒有任何 error log。30 分鐘沒進度。SSH 進容器 `nslookup postgres` → SERVFAIL。

**根因**:容器 `/etc/resolv.conf` 指向 host `systemd-resolved`(`127.0.0.53`)而不是 Docker 內部 DNS(`127.0.0.11`),Lightsail Ubuntu daemon 的 DNS 配置邊角案例。**SERVFAIL 不是 NXDOMAIN**,pg client 把它解讀為「暫時失敗,持續重試」而不是「域名不存在,放棄」,於是無限等待。

**修法**(三版演進,前兩版都失敗):

| 版本 | 配置 | 結果 |
|---|---|---|
| 初版 | `extra_hosts: ["postgres:172.18.0.10"]` 硬編 IP | 失敗:postgres IP 在 docker recreate 時 shift |
| 第二版 | `dns: ["127.0.0.11"]` only | 災難:解不到 `www.googleapis.com` → Google OAuth 全平台壞 8 小時 |
| 最終版 | `dns: ["127.0.0.11", "1.1.1.1", "8.8.8.8"]` | 通過:內外 hostname 都通 |

第二版的災難尤其慘——把 backend / worker / frontend / rag 全改成 `dns: [127.0.0.11]` 後,容器內 Node.js 解 `www.googleapis.com`(Google OAuth 用)失敗,Google 登入返回 `getaddrinfo EAI_AGAIN`,客戶整整 8 小時無法登入。緊急回滾後追根本因:Docker embedded DNS(127.0.0.11)只管內部 hostname(`postgres` / `redis` / `backend`),外部 hostname 它會 forward 到 host 的 resolv.conf,但 Lightsail Ubuntu 的 systemd-resolved 配錯導致 forward 失敗。

最終版加 `1.1.1.1`(Cloudflare)+ `8.8.8.8`(Google)當外部 fallback,內部 hostname 走 127.0.0.11,外部 hostname 走 fallback,兩者都通。

教訓:**單純設 Docker embedded DNS 不夠,還必須保留外部 fallback**。`docker-compose.prod.yml` 對 backend / worker / frontend / rag 全部都要這樣寫。任何新加容器**「dns: 必須包含內部 + 外部 fallback」**已寫進平台憲法。

### 15.3.5 Patch 5:LLM call 60s timeout

**失敗形態**:wikiCompiler progress 卡在「`AI 編譯 (13/84):官網-FAQ`」10+ 分鐘,CPU 0.01%,DB 0 active query。`aiCall` 在等 DeepSeek 永不返回的 stream。

**發現過程**:Patch 4 deploy 後 DNS 通了,但客戶反應某次編譯「卡在第 13 個」。SSH 進容器 `top` 看 node process 0% CPU,`netstat` 看到一個 `ESTABLISHED` 連線到 DeepSeek API 但 0 byte traffic 5 分鐘。

**根因**:OpenAI SDK 預設 `timeout: 600_000ms`(10 分鐘)+ `maxRetries: 2`,理論上 20 分鐘卡死;Gemini SDK(`@google/generative-ai`)沒提供原生 timeout / abort API。DeepSeek 偶有 `ECONNRESET` / 半開連線 / 低速回應,預設 timeout 撐不住。

更糟的是,**streaming response 中斷不會丟錯**——SDK 認為「stream 還沒結束,繼續等」,於是永遠等不到 `[DONE]` event。

**修法**(三層 timeout):

```js
// 1) OpenAI client constructor 60s + maxRetries=1
new OpenAI({ apiKey, timeout: 60_000, maxRetries: 1 });

// 2) per-request abort signal(防 streaming 卡 chunk)
client.chat.completions.create(params, { signal: AbortSignal.timeout(60_000) });

// 3) Gemini Promise.race + 60s timeout(SDK 無 abort)
function withTimeout(promise, ms, label) {
  return Promise.race([
    promise,
    new Promise((_, reject) =>
      setTimeout(() => reject(new Error(`${label} timeout ${ms}ms`)), ms)
    ),
  ]);
}
withTimeout(genModel.generateContent(...), 60_000, `gemini/${model}`);
```

修後:單一 doc 失敗最多卡 60s,wikiCompiler per-doc try/catch 接住 → 走 Patch 3 fallback 或跳下一 doc,**整批 84 docs 不再被單一 LLM hang 拖垮**。1 萬租戶 scale 安全。

這個 patch 後來變成平台憲法**「LLM call 必有 ≤60s timeout」**——適用 rag-backend-v2 + geo-saas backend + frontend 所有 LLM call site。違反此鐵律的後果:單一 LLM 半開連線就能拖垮整批工作流,1 萬 brand 同時失敗。

### 15.3.6 Patch 6:wiki_query_route maxTokens 200 → 2000

**失敗形態**:Wiki L1 全平台 miss,大多查詢 fallback L2,回「沒有相關資料」。

**發現過程**:2026-04-30 把 PROD primary LLM 從 `deepseek-v3-direct` 換到 `deepseek-v4-flash`(reasoning model 比較聰明,理論上 wiki query 路由更準),但隔天客戶反應「問什麼都答不出來」。grep wiki query log 全空。

**根因**:reasoning model 的 `message.content` 之外另吃 `reasoning_tokens`(可佔 token budget 70%+)。`wikiCompiler.js` 兩處 `wiki_query_route` call 用 `maxTokens: 200` 對 deepseek-v3 夠,但 V4 reasoning **把 200 token 全部吃進 reasoning,content 永遠空字串** → JSON parse fail → wiki query 路由失敗 → fallback L2 也回「沒命中」。

最詭異的是:**沒有 error**,LLM API 200 OK 回 `{ content: "", reasoning_tokens: 198 }`,只是 content 空。原本的 try/catch 抓不到。

**修法**:兩處 `maxTokens` 200 → 2000。實測 2000 token 給 reasoning + content 都夠,wiki query routing 恢復正常。

**1 萬租戶影響**:任何啟用 wiki + 用 V4-style reasoning model(DeepSeek V4 / OpenAI o1-o4 / Anthropic 含 thinking)的租戶,Wiki L1 全失靈,fallback L2 雖能勉強回答但成本高且品質差。這條教訓推廣為平台憲法:**換 reasoning model 後必查所有 maxTokens 配置**,任何 maxTokens < 1500 在 reasoning model 上都要驗證 content 不為空。

### 15.3.7 全部 6 patch 的整體效果

部署完整 6 patch 後 PROD 觀察:

| 指標 | Before | After |
|---|---|---|
| 平台 wiki orphan rate | 45%(156/348) | ≤25%(預期) |
| 5ce78a51 KB(84 docs)compile 完成率 | 39%(33/84) | 100%(84/84) |
| 單 doc LLM hang 卡死整批機率 | ~20% | <1% |
| Wiki L1 hit rate | 12%(reasoning model 後)| 65% |

orphan rate 從 45% 降到 ≤25% 但**還沒降到 0%** 的原因:Patch 3 的 fallback 算「保底解決」但不算「完美編譯」,進 fallback bucket 的 doc 仍標 `confidence='low'`,在某些 admin 計算口徑下算 partial orphan。完全消除 orphan 需要 LLM 永遠不漏 marker(不可能)或全部 doc 都用結構化 schema(JSON output mode,部分 LLM 支援),這是 V2 工作。

### 15.3.8 Patch 順序的因果鏈

回頭看,六個 patch 的發現順序非常巧合——前一個 patch 修好之前,看不到後一個 patch 的 bug:

```text
Patch 1 (cutoff)        → 修了才看到所有 doc 都進編譯
        ↓
Patch 2 (partial UUID)  → 修了才看到後續 doc 都編完
        ↓
Patch 3 (no marker)     → 修了才看到 wiki page 真實生成
        ↓
Patch 4 (DNS)           → 修了才看到生成過程不卡死
        ↓
Patch 5 (LLM timeout)   → 修了才看到單 doc 失敗不拖整批
        ↓
Patch 6 (maxTokens)     → 修了才看到 wiki query 真實命中
```

每一個 patch 都解開「下游錯覺正常其實不正常」的層,這是 LLM-driven 系統 debug 的典型形態:**錯誤被 silent swallow 多層,要剝洋蔥一層一層才看見**。

---

## 15.4 Worker robustness:從 silent hang 到自動恢復

WikiCompiler 失敗只是冰山一角。整個 BullMQ worker 體系也踩了同類陷阱。

### 15.4.1 Defensive guard pattern

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

### 15.4.2 Queue retention worker

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

實測一次性 cleanup 結果:

```text
[QueueRetention] Cleaned 2026-05-02 02:30
  monthly-report-queue: 7 failed, 312 completed
  ab-testing-queue: 30 failed, 845 completed
  geo-closed-loop-queue: 4 failed, 198 completed
  geo-rlhf-queue: 0 failed, 67 completed
  geo-axp-queue: 2 failed, 630 completed
  ...
  Total: 43 failed + 2052 completed cleaned
```

清乾淨後 admin UI 恢復實時狀態,任何新失敗立即可見。1 萬租戶 scale 預估 daily cleanup ≥ 1 萬筆,retention worker 必跑。

### 15.4.3 KNOWN_QUEUES 的維護

retention worker 的 `KNOWN_QUEUES` 陣列必須含全平台所有 queue 名,否則新 queue 會永遠不被清:

```js
const KNOWN_QUEUES = [
  'geo-axp-queue',
  'geo-axp-immediate',
  'geo-axp-bootstrap',
  'monthly-report-queue',
  'tier-a-visual-monthly',
  'billing-cycle-queue',
  'payment-dunning-queue',
  'reverse-scan-queue',
  'ab-testing-queue',
  'geo-closed-loop-queue',
  'geo-rlhf-queue',
  'geo-f12-optimize-queue',
  'queue-retention-queue',  // self
  // ... 其他 11 個
];
```

漏寫某 queue → 失敗永久累積。為避免漏掉,我們加了 admin UI 的 `/admin/scan-frequencies` 頁面 — 從 BullMQ 列出所有 queue 名 vs `KNOWN_QUEUES` array,差集顯示「⚠ 未納入 retention」紅標。新 queue 加完忘記更新 KNOWN_QUEUES → admin 立即看到。

### 15.4.4 LLM call site timeout 全平台對齊

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

我們把這個 grep 加進 pre-commit hook(規畫中),違反 → block commit。1 萬租戶 scale 不能容忍任何漏網。

---

## 15.5 1 萬租戶部署模式

### 15.5.1 Patches 目錄結構

`rag-backend-v2` 不在 geo-saas git repo(獨立微服務),patch 紀錄保留於 `docs/rag-backend-v2-patches/`:

```text
docs/rag-backend-v2-patches/
├── README.md                          ← 部署順序 + verification block
├── wikiCompiler.js                    ← Patch 1, 2, 3, 6 都改這個
├── modelRouter.js                     ← Patch 5 改這個
├── docker-compose.rag.yml             ← Patch 4 DNS 設定
└── verification-block.sh              ← 一鍵驗證 6 patch live
```

每次 RAG 容器重建必須三個檔一起 scp(漏一個 patch 就會回退)。

### 15.5.2 部署 + 驗證 block

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

期望輸出 `1 / 2 / 3 / 1 / 8 / ≥3`。任一不符即 patch 回退,立即重 scp + rebuild。每次 PROD 容器重建必須帶這個 verification block,否則 patch 會默默丟失重蹈覆轍。

### 15.5.3 1 萬租戶 readiness 驗證

部署 patches 之外,1 萬租戶 readiness 還需:

- **L1 hit rate monitoring**:`f12_cache_metrics` 每小時聚合,目標 ≥65%。低於 50% → admin alert
- **per-tenant LLM cost cap**:`f12_billing_records` 月度聚合,starter 超 $1 / pro 超 $50 → email warn
- **fallback bucket size**:`SELECT COUNT(*) FROM wiki_pages WHERE 'wiki_fallback' = ANY(tags)` > 100 / brand → admin review
- **orphan tenant_documents**:`SELECT COUNT(*) FROM tenant_documents td WHERE NOT EXISTS (SELECT 1 FROM wiki_page_sources WHERE source_doc_id=td.id)` > 10% → 重新 trigger compile

這 4 個指標都進 `/admin/rag-health` admin dashboard,daily cron 算出後寫 alert。

### 15.5.4 RAG 容器重建的歷史

從 2026-04-23 到 2026-05-02 短短 9 天內,rag 容器 recreate 了 11 次:

| 日期 | 原因 | Patch |
|---|---|---|
| 04-23 | 升級 DeepSeek SDK | — |
| 04-25 | 加 Wiki Cascade trigger | migration 170 |
| 04-27 | 加 sync_wiki_page_sources trigger | migration 201 |
| 04-29 | Patch 1 (cutoff) | wikiCompiler.js |
| 04-30 | Patch 2 (UUID) | wikiCompiler.js |
| 04-30 | 切 V4 reasoning(Patch 6 之前) | platform_llm_config |
| 05-01 | Patch 3 (no marker) | wikiCompiler.js |
| 05-01 | Patch 4 v1 (extra_hosts IP) | docker-compose.rag.yml |
| 05-01 | Patch 4 v2 (dns 127.0.0.11 only) | docker-compose.rag.yml |
| 05-01 | Patch 4 v3 (dns + fallback) | docker-compose.rag.yml + .prod.yml |
| 05-02 | Patch 5 + 6 | modelRouter.js + wikiCompiler.js |

每次重建都是 ~3 分鐘 downtime(during build & recreate),11 次累計 ~33 分鐘。下次再大改建議走 blue-green deployment(雙容器交替)減 downtime,目前 quota 不夠開兩容器同時跑。

---

## 15.6 觀察與未解問題

### 15.6.1 LLM 不可信任的本質

六層 patch 看似已涵蓋常見失敗模式,但 LLM 演進速度快過 patch 速度:

- 換 model(deepseek-v3 → v4-flash)會帶來新 hallucination 形態(`maxTokens` 對 reasoning 不夠)
- 不同 prompt 格式對不同 model 失敗模式不同(GPT-4 不會漏 marker,但會吐 prefix「Here's the response:」)
- streaming vs non-streaming 行為差異(streaming 可能中斷後不重發)

這意味著 LLM-driven 系統的 robustness 設計必須**永遠假設 LLM 會以新方式失敗**,每加一個新 model 都要重跑一輪 hallucination test。

### 15.6.2 fallback wrap article 的品質

Patch 3 的低信心 fallback 雖然救起了客戶 doc,但 `confidence='low'` + `tags=['wiki_fallback']` 的 wiki page 在 L1 retrieval 時實際品質如何,目前**沒做端到端 evaluation**。理論上應該:

- 不參與 wiki_query_route 的「你問什麼類別」LLM 路由(避免騙 user)
- 在 admin UI 顯示「⚠ 待審視」標記
- 累計 N 個失敗後系統 alert admin

第三點還沒做。

進一步地,fallback 內容的「來源綁定」雖然有(`sourceDocIds`),但「LLM 重組品質」沒驗證——可能 LLM 把客戶兩個不相關 doc 內容混在一起。這需要人工或 LLM-as-judge 抽樣審視,V2 工作。

### 15.6.3 多 LLM 路由 vs 單 LLM 加固

理論上,如果同一 query 同時跑兩個不同 LLM(DeepSeek + Gemini),交叉驗證 marker / UUID / 內容,可以解掉大部分 hallucination 問題。但成本翻倍且延遲倍增,是否值得?目前數據不夠決策,留 V2 評估。

具體量化:單一 wiki compile call 約 $0.0003(DeepSeek V4 flash),雙 LLM 變 $0.0006 + max latency 1.5s → 3s。84 docs KB compile $0.025 → $0.05。1 萬租戶月 100 KB recompile = $50/月,可接受。但需要先做 A/B test 驗證雙 LLM 真能降 hallucination(可能兩 LLM 偶爾犯同樣錯)。

### 15.6.4 Wiki Cascade 的反向操作

刪除是 cascade 自動處理,但**還原**(undo soft-delete)沒做。客戶誤刪文件 → wiki_pages 軟刪 → 30 天後自動 hard delete。如果客戶 7 天後想要回來,目前要客服手工 SQL UPDATE。這是常見需求但工程沒排上。

完整實作需要:

- Soft delete 期間給客戶「30 天內可還原」 UI
- 還原時逆向 cascade:wiki_page 還原 → junction 還原 → tenant_document 還原 → brand_document 還原
- 4 層每一層都要有 `deleted_at IS NOT NULL` 檢查 + restore 函式
- admin 強制 hard delete 流程(GDPR 合規,客戶要求徹底刪除)

工程量約 1-2 週,V2 排上。

### 15.6.5 Wiki 編譯失敗的可恢復性

V1 編譯失敗 → DB 寫 `last_compile_error`,UI 顯示「編譯失敗」。但**沒重試機制** — admin 必須手動點按鈕。1 萬租戶 readiness 應該:

- 失敗自動 retry 1 次(若是 LLM timeout 類短暫錯誤)
- N 次連續失敗 → admin alert,而不是默默死掉
- failure 分類:transient(timeout / 5xx)→ 自動 retry;permanent(invalid input / over quota)→ 不 retry

也是 V2 工作。

### 15.6.6 跨 RAG 微服務的 idempotent guarantee

`ragDeleteDocument` 失敗後 `brand_documents` 已軟刪但 `tenant_documents` 沒清,造成 orphan tenant_document(無 brand_document 對應)。目前靠 daily cron 抓 orphan 重新清一次,但這條 cron 可能漏跑或失敗。

更乾淨的設計是 outbox pattern:

```sql
-- brand_documents.delete 時寫 outbox row
INSERT INTO outbox_events(event_type, payload, status)
VALUES ('rag_delete_document', '{"central_doc_id": "..."}', 'pending');

-- 獨立 worker 讀 outbox,call ragDeleteDocument,成功才標 'done'
-- 失敗自動 retry,長期失敗 alert
```

但要重構整個 cascade flow 才能套 outbox,V2 工作量大,目前 daily cron 暫時夠用。

---

## 15.7 工程教訓

從 6 個 patch + Worker robustness 演進(2 週,5 次 PROD 故障),整理出 5 條教訓:

### 15.7.1 LLM 失敗永遠 silent 比 vocal 多

六個 patch 沒一個是 LLM 直接 throw error。全部都是「看似成功但實際空 / 截斷 / 卡住」:

- Patch 1:cutoff 默默砍掉 60% doc,沒任何 warning
- Patch 2:LLM 吐 partial UUID,PG throw 但被 transaction 吞掉
- Patch 3:LLM 漏 marker,parser 回 0 article 沒 throw
- Patch 4:DNS SERVFAIL,pg-pool 永遠等不 throw
- Patch 5:streaming 中斷,SDK 永遠等 [DONE] 不 throw
- Patch 6:reasoning maxTokens 不夠,API 200 OK 回空 content 不 throw

**LLM-driven 系統 debug 的第一條規則:不能信任「沒看到 error」,必須驗證「output 真的是預期 shape」**。每個 LLM call 後加 schema validate(UUID_RE / marker / minLength)是基本動作。

### 15.7.2 silent hang 是分散式系統的隱形殺手

DNS SERVFAIL / streaming 中斷 / reasoning content 空——三種情況都讓系統「看起來正常運作」但 0 進度。傳統的「失敗會 throw」假設破功。

防禦這類失敗的工程模式:

- **timeout 必加**:任何外部 call(LLM / DB / 內部 service)都要有 timeout,寧可手動 retry 也不要等永久
- **進度心跳**:long-running task 每 10 秒寫一次 heartbeat,沒 heartbeat 視同卡死
- **silent 比 throw 危險**:寫 try/catch 時記得「成功路徑也要驗證 output」,不只 catch error

### 15.7.3 三層 fallback 比一層 retry 可靠

Patch 3 的 fallback 設計是「**LLM 失敗→ 寬鬆解析 → 仍失敗 → 用 doc title 保底包裝**」三層,任一層成功就保底。對比一層 retry(失敗 retry 3 次,還失敗就 throw),三層 fallback 永遠有出路:

- 第一層理想:LLM 完美輸出,直接 parse
- 第二層寬鬆:LLM 漏 marker / 加 prefix,寬鬆 regex 抓
- 第三層保底:全失敗,用 doc title + content 包成 single article

代價:fallback bucket 的 wiki page 品質低,需 admin 看。但相比「LLM 失敗 → doc 變孤兒」,值得。

1 萬租戶 scale 不能容忍「永久孤兒」,任何客戶上傳的內容都必須**有出口**。

### 15.7.4 Patch 順序揭示「下游錯覺正常」原理

回顧六個 patch 發現順序——每修好一個都會「揭露」下一個:

```text
看到 33/84 完成 → 修 cutoff → 看到 全部 doc 進編譯
看到 全部 doc 進編譯 → 看到第 13 個就斷 → 修 UUID → 看到 全部 doc 完成
看到 全部 doc 完成 → 看到 wiki page 數不對 → 修 marker → 看到 wiki 真實生成
看到 wiki 真實生成 → 看到 編譯卡住 → 修 DNS → 看到 編譯運行
看到 編譯運行 → 看到 卡在第 13 個 → 修 LLM timeout → 看到 編譯穩定
看到 編譯穩定 → 看到 wiki query 全 miss → 修 maxTokens → 看到 wiki 真實命中
```

每一層都是「**上一層 silent failure 把下一層遮起來了**」。debug 必須一層一層剝開,**急著一次解全部會錯估根因**。

### 15.7.5 patches/ 目錄是微服務 SSOT

rag-backend-v2 不在 geo-saas git repo,但 patch 進去後容易丟失。我們設計 `docs/rag-backend-v2-patches/` 當 SSOT:

- 每次 PROD scp 都從這目錄取
- README 寫部署順序 + verification block
- 每次容器重建必跑 verification 檢查 patch live

**如果不寫 verification block,下次重建忘記帶就回退**。我們踩過一次:某次升級 RAG 重 build,忘了 scp wikiCompiler.js,Patch 6 的 maxTokens=2000 回到 200,客戶反應「Wiki 又不命中」。grep `'maxTokens: 2000'` count=0 直接知道 patch 沒帶。從此每次 RAG container recreate 必跑 verification,不會再踩。

---

## 本章要點

- rag-backend-v2 獨立微服務 + 自家 cs_rag_db + Wiki Cascade 4 層級聯刪除確保資料一致性
- 六種 LLM hallucination 失敗模式對應六層 patch:cutoff / partial UUID / no marker / DNS / LLM timeout / reasoning maxTokens
- Patch 順序揭示「下游錯覺正常」原理:每修好一層才看到下一層 silent failure
- Worker robustness 三件套:defensive guard(entity 預檢)、per-item try/catch、daily retention auto-cleanup(43 failed + 2052 completed 一次清完)
- 1 萬租戶部署必走 patches/ 目錄 SSOT,每次容器重建必跑 verification block
- 未解:LLM 演進速度 > patch 速度;fallback 品質未 eval;多 LLM 交叉驗證待評估;wiki 還原機制缺失;失敗重試缺失;outbox pattern 待重構
- 5 條工程教訓:LLM 失敗 silent > vocal / silent hang 隱形殺手 / 三層 fallback > 一層 retry / Patch 順序原理 / patches/ SSOT 必跑 verification

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
| 2026-05-03 | v1.1.1 | 章節擴充至 ~7100 字 — 加 15.1.1/3 Karpathy 概念與 V1→V2 演進、15.2.2/3 SSOT trigger SQL 與 36 entry 偵錯、15.3 每 patch 加發現過程、15.3.8 Patch 順序因果鏈、15.4.3 KNOWN_QUEUES 維護、15.5.4 RAG 11 次重建史、15.6.6 outbox pattern、15.7 工程教訓 5 條;修正章節編號 14.x → 15.x typo |

---

**導覽**:[← Ch 14: F12 三層結構優化器](./ch14-f12-structural-optimizer.md) · [📖 目次](../README.md) · [Ch 16: 平台 SSOT 全鏈 →](./ch16-platform-ssot-chain.md)

<!-- AI-friendly structured metadata -->
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "TechArticle",
  "headline": "Chapter 15 — rag-backend-v2 加固:六層 LLM hallucination 失敗模式的防禦設計",
  "description": "中央 RAG 微服務 (rag-backend-v2) 工程加固史:Wiki Cascade 4 層級聯、6 種 wikiCompiler hallucination patch、Worker defensive guard 與 Queue retention,完整呈現 LLM-driven 系統 robustness 設計實踐。含 5 條工程教訓。",
  "author": {"@type": "Person", "name": "Vincent Lin", "affiliation": "Baiyuan Technology"},
  "datePublished": "2026-05-03",
  "inLanguage": "zh-TW",
  "isPartOf": {
    "@type": "Book",
    "name": "百原GEO Platform 技術白皮書",
    "url": "https://github.com/baiyuan-tech/geo-whitepaper"
  },
  "keywords": "rag-backend-v2, LLM Wiki Compiler, Wiki Cascade, LLM Hallucination Hardening, Docker DNS, LLM Timeout, BullMQ Retention, 工程教訓"
}
</script>
