---
title: "Appendix A — 詞彙表（完整版）"
description: "百原GEO Platform 白皮書術語全集，涵蓋 GEO / AXP / Schema.org / RAG / Closed-Loop 等所有專有名詞定義與章節引用。"
appendix: A
part: 6
lang: zh-TW
authors:
  - name: Vincent Lin
    affiliation: Baiyuan Technology
license: CC-BY-NC-4.0
last_updated: 2026-04-18
last_modified_at: '2026-04-21T06:52:21Z'
---












# Appendix A — 詞彙表（完整版）

依字母／漢字筆劃排序。每條列出：**定義 + 本書首次出現章節 + 相關章節**。

## A

- **@id（Schema.org）** — JSON-LD 中實體的唯一識別字串，用於在 `@graph` 內或跨文件互相引用。本書首見：[§6.7](./ch06-axp-shadow-doc.md#67-json-ld-扁平化的踩坑紀錄)。核心應用：[§7.3](./ch07-schema-org.md#73-三層-id-互連知識圖)。
- **AI Bot** — 泛指抓取網路內容以訓練或檢索的機器程式，如 GPTBot、ClaudeBot、PerplexityBot。本書首見：[§6.3](./ch06-axp-shadow-doc.md#63-cloudflare-worker-注入機制)。
- **AI Citation Rate（AI 引用率）** — 在代表性意圖查詢中，品牌被 AI 主動提及的比例（0–100%）。本書核心指標。首見：[§1.2](./ch01-geo-era.md#12-ai-引用率一個嶄新且決定生死的指標)。詳解：[§3.2.1](./ch03-scoring-algorithm.md#321-citation-rate--被提及率)。
- **aggregateRating** — Schema.org property，聚合評分值。本平台從 GBP 同步。[§8.4](./ch08-gbp-integration.md#84-欄位對應表)。
- **AXP（AI-ready eXchange Page）** — 百原造詞。給 AI Bot 讀的影子文檔，pure HTML + Schema.org JSON-LD + Markdown 三層結構，與人類版網站解耦。詳述於 [Ch 6](./ch06-axp-shadow-doc.md)。
- **App-level filter** — 應用層查詢條件。本書指多租戶隔離的第二層防線（與 RLS 搭配）。[§2.5](./ch02-system-overview.md#25-多租戶資料隔離)。

## B

- **BullMQ** — Redis 為底層的 Node.js 任務隊列函式庫。本平台的背景任務執行骨架。[§2.4](./ch02-system-overview.md#24-技術棧)。
- **Brand Entity（品牌實體）** — 以 Schema.org Organization / LocalBusiness 為根節點的結構化品牌資料總稱。[Ch 7](./ch07-schema-org.md)。

## C

- **ChainPoll** — 對同一 claim 以同一 prompt 呼叫 LLM 3 次取多數決的驗證方法；用於 NLI confidence 0.5–0.8 不確定地帶。[§9.3](./ch09-closed-loop.md#93-偵測主機制nli-分類--chainpoll-投票)。
- **CID（Customer ID）** — Google Maps 內部流水號，需透過 Places API 反查成 Place ID。[§7.7](./ch07-schema-org.md#77-gbp-url-parser)。
- **ClaimReview** — Schema.org 正式 property，用於標注「某聲明是否為真」。本平台用於對外宣告幻覺修復結果。[§9.6](./ch09-closed-loop.md#96-修復claimreview-生成與多路徑注入)。
- **Closed-Loop（閉環修復）** — 偵測 → 比對 → 判定 → 生成 ClaimReview → 注入 AXP/RAG → 重掃驗證的自動化循環。[Ch 9](./ch09-closed-loop.md)。
- **Cloudflare Workers** — 邊緣計算環境，本平台用於 AI Bot UA 偵測與 AXP 注入。[§6.3](./ch06-axp-shadow-doc.md#63-cloudflare-worker-注入機制)。
- **Common Crawl** — 公開的大規模網頁抓取資料集，是多數 LLM 預訓練的主要資料來源之一。[§1.3](./ch01-geo-era.md#13-geo-不是-seo-的延伸是獨立新學科)。
- **Consistency（跨平台一致性）** — 七維度之一，同一品牌在不同 AI 平台各維度分數的標準差之倒數。[§3.2.7](./ch03-scoring-algorithm.md#327-consistency--跨平台一致性)。
- **Content Depth（內容深度）** — 七維度之一，AI 提及品牌時所附帶描述的深度。[§3.2.6](./ch03-scoring-algorithm.md#326-content-depth--內容深度)。
- **Core Web Vitals** — Google 的網頁體驗指標（LCP、FID、CLS）。SEO 時代重要，GEO 時代幾無關聯。[§1.3](./ch01-geo-era.md#13-geo-不是-seo-的延伸是獨立新學科)。

## D

- **DIRECT_MODEL_ID_MAP** — 將聚合中轉的 model ID 對應到廠商原生 model ID 的對照表。[§5.3.1](./ch05-multi-provider-routing.md#531-model-id-在聚合層與原生層不一致)。
- **Dogfooding** — 自家產品給自家用。本平台的 Brand E 即為範例。[§11.1](./ch11-case-studies.md#111-品牌畫像匿名)。

## E

- **entailment / contradiction / neutral / opinion** — NLI（自然語言推論）四分類。contradiction 才標記為幻覺；neutral ≠ 幻覺。[§9.3](./ch09-closed-loop.md#93-偵測主機制nli-分類--chainpoll-投票)。
- **extraParams** — 各 AI 廠商原生 API 特有的必填參數。例：Qwen3 需 `enable_thinking: false`。[§5.3.2](./ch05-multi-provider-routing.md#532-extraparams-差異)。

## F

- **Fingerprint（指紋）** — 對 AXP 或 Schema.org 產物的雜湊值，用於判斷內容是否被 AI 抓到並整合。[Ch 9](./ch09-closed-loop.md) 相關。
- **FTID（Feature ID）** — Google Maps 內部地點識別碼。[§7.7](./ch07-schema-org.md#77-gbp-url-parser)。

## G

- **GBP（Google Business Profile）** — Google 商家檔案，實體商家的事實主控源。[Ch 8](./ch08-gbp-integration.md)。
- **GEO（Generative Engine Optimization）** — 生成式引擎優化。提升品牌在生成式 AI 回答中被提及比例、位置、敘事準確度的學科。[Ch 1](./ch01-geo-era.md)。
- **GPTBot** — OpenAI 的網頁爬蟲，用於 ChatGPT 訓練語料收集。[§6.4](./ch06-axp-shadow-doc.md#64-ai-bot-ua-清單與識別策略)。
- **Ground Truth (GT)** — 事實真相。本書指用於比對幻覺候選的權威資料集合（官網即時抓 / RAG / 手動 GT 三層）。[§9.3](./ch09-closed-loop.md#93-偵測主機制nli-分類--chainpoll-投票)。

## H

- **Hallucination（幻覺）** — AI 生成的、與事實不符的內容。本書分五類：實體錯置 / 業務錯誤 / 產業誤分 / 時空錯誤 / 屬性錯誤。[§9.2](./ch09-closed-loop.md#92-ai-幻覺的五種類型)。

## I

- **industry_code** — 百原 25 類產業分類 enum。[§7.2](./ch07-schema-org.md#72-25-類產業特化-type-的設計)。
- **Intent Query（意圖查詢）** — 依品牌產業動態生成的代表性使用者問題。[§2.2.1](./ch02-system-overview.md#221-scan-module--監測)。
- **is_physical** — 品牌的實體／線上屬性旗標，影響 Schema.org 欄位權重。[§7.4](./ch07-schema-org.md#74-實體商家-vs-線上服務欄位權重的分歧)。
- **isStale** — Stale Carry-Forward 下的標記欄位，表示該平台的分數為歷史帶入值。[§4.3](./ch04-stale-carry-forward.md#43-stale-carry-forward-設計)。

## J

- **JSON-LD** — JavaScript Object Notation for Linked Data，Schema.org 的主流序列化格式。[§6.2](./ch06-axp-shadow-doc.md#62-axp-設計影子文檔的結構)、[Ch 7](./ch07-schema-org.md)。

## K

- **kgmid（Knowledge Graph Machine ID）** — Google 知識圖譜的實體識別。在 `sameAs` 中使用。
- **Knowledge Graph（知識圖譜）** — 實體與關係構成的結構化網絡。Schema.org 的 `@id` 互連建立此圖。[§7.3](./ch07-schema-org.md#73-三層-id-互連知識圖)。

## L

- **Layer 1 哨兵掃描** — 4 小時週期、針對搜尋型 AI 的輕量驗證。[§9.7](./ch09-closed-loop.md#97-兩層掃描閉環)。
- **Layer 2 完整掃描** — 24 小時週期、含七維度評分與指紋比對的深度掃描。[§9.7](./ch09-closed-loop.md#97-兩層掃描閉環)。
- **LLM Wiki** — 百原造詞。由 LLM 主動處理與維護的語意知識層，位於 RAG 系統的 L1 層。[§9.5](./ch09-closed-loop.md#95-l1-llm-wiki被動檢索之上的主動語意層)。
- **LocalBusiness** — Schema.org @type，實體商家的主要分類。[§7.2](./ch07-schema-org.md#72-25-類產業特化-type-的設計)。

## M

- **modelRouter** — 百原自研的多 AI Provider 路由服務，主／備雙路徑抽象層。[Ch 5](./ch05-multi-provider-routing.md)。

## N

- **NLI（Natural Language Inference）** — 自然語言推論，判斷兩段文字的蘊含關係。本平台幻覺偵測主機制。[§9.3](./ch09-closed-loop.md#93-偵測主機制nli-分類--chainpoll-投票)。

## O

- **OpenAI-compatible API** — 以 OpenAI API 規格為介面的聚合中轉。多 AI 廠商可用同一 SDK 呼叫。[§5.2](./ch05-multi-provider-routing.md#52-modelrouter-架構)。

## P

- **pgvector** — PostgreSQL 向量擴充，支援 embedding 與相似度查詢。[§2.4](./ch02-system-overview.md#24-技術棧)。
- **Phase 基線測試** — 以固定 20 題在不同時間點重測同一 AI，縱向追蹤認知演變。[Ch 10](./ch10-phase-baseline.md)。
- **Place ID** — Google Maps 的地點唯一識別。Places API 的 primary key。[§7.7](./ch07-schema-org.md#77-gbp-url-parser)。
- **Position Quality（位置品質）** — 七維度之一，品牌在 AI 回答中出現位置的加權平均。[§3.2.2](./ch03-scoring-algorithm.md#322-position-quality--位置品質)。
- **Platform Breadth（平台覆蓋度）** — 七維度之一，品牌被主動提及的 AI 平台數 ÷ 監測平台總數。[§3.2.4](./ch03-scoring-algorithm.md#324-platform-breadth--平台覆蓋度)。

## Q

- **Query Coverage（查詢覆蓋率）** — 七維度之一，被提及的 intent query 類型多樣性。[§3.2.3](./ch03-scoring-algorithm.md#323-query-coverage--查詢覆蓋率)。
- **QPM（Queries Per Minute）** — 每分鐘查詢次數，API 配額計量單位。GBP API 預設 300 QPM。[§8.5](./ch08-gbp-integration.md#85-同步頻率與配額)。

## R

- **RAG（Retrieval-Augmented Generation）** — 檢索增強生成。本平台採中央共用 RAG SaaS 架構。[§9.4](./ch09-closed-loop.md#94-中央共用-ragsaas-架構的關鍵基礎設施)。
- **RLS（Row-Level Security）** — PostgreSQL 資料列層級存取控制。多租戶隔離的第一層。[§2.5](./ch02-system-overview.md#25-多租戶資料隔離)。

## S

- **sameAs** — Schema.org property，指向其他平台的同一實體（Wikipedia、Wikidata、LinkedIn、GBP）。[§7.3](./ch07-schema-org.md#73-三層-id-互連知識圖)。
- **Schema.org** — 結構化資料詞彙表，Google/Bing/Yahoo/Yandex 聯合推動。[Ch 7](./ch07-schema-org.md)。
- **Sentiment Score（情感分數）** — 七維度之一，提及文本的情感傾向（正／中性／負 聚合）。[§3.2.5](./ch03-scoring-algorithm.md#325-sentiment-score--情感分數)。
- **Sentinel Scan（哨兵掃描）** — 見 Layer 1。
- **SERP（Search Engine Results Page）** — 搜尋引擎結果頁，傳統 SEO 的主戰場。[§1.1](./ch01-geo-era.md#11-生成式搜尋接管使用者習慣)。
- **Sitemap** — XML 格式的 URL 清單，協助爬蟲發現內容。[§6.6](./ch06-axp-shadow-doc.md#66-sitemap-自動產生)。
- **Smoke Test** — 備援路徑啟動時的最小驗證，確保可用性。[§5.6.1](./ch05-multi-provider-routing.md#561-啟動時-smoke-test)。
- **Stale Carry-Forward** — 百原造詞。當平台掃描全失敗時，從歷史帶回最近成功值並標記為「停滯」，維持訊號連續性。[Ch 4](./ch04-stale-carry-forward.md)。

## T

- **Tenant ID（X-Tenant-ID）** — 多租戶 HTTP header，用於中央共用 RAG 的租戶過濾。[§9.4](./ch09-closed-loop.md#94-中央共用-ragsaas-架構的關鍵基礎設施)。

## U

- **UA（User-Agent）** — HTTP 請求標頭，AXP 注入據此判斷是否為 AI Bot。[§6.4](./ch06-axp-shadow-doc.md#64-ai-bot-ua-清單與識別策略)。

## W

- **Webhook** — 伺服器主動推送變動通知。GBP API 目前**不提供** webhook。[§8.6](./ch08-gbp-integration.md#86-無-webhook-的補救方案)。
- **Wikidata** — 開放式知識圖譜，Schema.org `sameAs` 的重要目標。[§7.3](./ch07-schema-org.md#73-三層-id-互連知識圖)。
- **Wizard（設定引導）** — 新品牌的線性七步驟 Schema.org 填寫流程。[§7.6](./ch07-schema-org.md#76-wizard--edit-雙入口設計)。

---

## 相關規範索引

| 規範 | 作用 | 本書章節 |
|------|------|---------|
| [Schema.org](https://schema.org/) | 結構化資料詞彙 | [Ch 6, 7, 9](./ch07-schema-org.md) |
| [JSON-LD 1.1](https://www.w3.org/TR/json-ld11/) | Linked Data 序列化 | [Ch 6, 7](./ch07-schema-org.md) |
| [OpenGraph](https://ogp.me/) | Meta tag for preview | [Ch 6](./ch06-axp-shadow-doc.md) |
| [robots.txt](https://www.rfc-editor.org/rfc/rfc9309.html) | Robots Exclusion Protocol | [Ch 6](./ch06-axp-shadow-doc.md) |
| [sitemaps.xml](https://www.sitemaps.org/protocol.html) | Sitemap Protocol 0.9 | [§6.6](./ch06-axp-shadow-doc.md#66-sitemap-自動產生) |

---

**導覽**：[← Ch 12: 限制與未來工作](./ch12-limitations.md) · [📖 目次](../README.md) · [附錄 B: 公開 API 規格 →](./appendix-b-api.md)
