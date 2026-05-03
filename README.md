# 百原GEO Platform 技術白皮書

> 在生成式 AI 時代重新定義品牌可見性的工程實踐
>
> *A Whitepaper on Building a SaaS for Generative Engine Optimization*

[![DOI](https://zenodo.org/badge/DOI/10.5281/zenodo.19994035.svg)](https://doi.org/10.5281/zenodo.19994035)
[![License: CC BY-NC 4.0](https://img.shields.io/badge/License-CC%20BY--NC%204.0-blue.svg)](https://creativecommons.org/licenses/by-nc/4.0/)
[![Status: v1.1.2](https://img.shields.io/badge/Status-v1.1.2-blue.svg)](#修訂記錄)
[![zh-TW](https://img.shields.io/badge/zh--TW-complete-green.svg)](zh-TW/)
[![en](https://img.shields.io/badge/en-complete-green.svg)](en/)
[![ja](https://img.shields.io/badge/ja-complete-green.svg)](ja/)
[![PDF](https://img.shields.io/badge/PDF-download-red.svg)](https://github.com/baiyuan-tech/geo-whitepaper/releases/latest)

> **English reader?** → Start with the **[Executive Summary (en/README.md)](en/README.md)** or jump to **[Ch 1 (en)](en/ch01-geo-era.md)**. Full English edition (12 chapters + 4 appendices, ~28k words) is complete.
>
> **日本語の読者** → **[エグゼクティブサマリー（ja/README.md）](ja/README.md)** または直接 **[第 1 章 (ja)](ja/ch01-geo-era.md)** へ。完全な日本語版（12 章 + 4 付録、約 30,000 字）を公開済み。

---

## 摘要 / Abstract

**中文**：本書記錄百原科技於 2024–2026 年開發「百原GEO Platform」的工程實踐。百原GEO Platform 是一套針對**生成式引擎優化（Generative Engine Optimization, GEO）**的 SaaS 系統，目標是協助品牌在 ChatGPT、Claude、Gemini、Perplexity 等生成式 AI 的回答中被正確、持續、準確地提及。本書涵蓋一套七維度 AI 引用率評分演算法、一個對 AI Bot 友善的影子文檔（AXP）交付機制、Schema.org 三層實體知識圖、以及一個幻覺自動偵測與修復的閉環系統。總篇幅約 30,000 字（繁體中文版），採 CC BY-NC 4.0 授權公開。

**English**：This whitepaper documents the engineering practice of Baiyuan Technology in developing "Baiyuan GEO Platform" (2024–2026), a SaaS system targeting **Generative Engine Optimization (GEO)**. The platform helps brands be cited accurately and consistently in responses from generative AI services (ChatGPT, Claude, Gemini, Perplexity, etc.). The book presents a seven-dimension citation-rate scoring algorithm, an AI-Bot-friendly shadow document delivery mechanism (AXP), a Schema.org three-layer entity knowledge graph, and a closed-loop hallucination detection & auto-remediation system. Approximately 30,000 Traditional Chinese characters, published under CC BY-NC 4.0.

---

## 這是什麼

這份文件是一本**工程白皮書**，不是產品宣傳、不是使用手冊。讀者可以預期獲得：

- **架構模式**：多租戶 SaaS、多 AI Provider 容錯、邊緣內容注入、閉環品質治理
- **演算法骨架**：七維度評分、訊號連續性（Stale Carry-Forward）、幻覺收斂
- **工程決策紀錄**：為何這樣設計、有哪些取捨、哪些路已經走過不通
- **真實數據（聚合、去識別化）**：5 個上線品牌 6 週內的觀察

這份文件**不包含**：

- 商業授權的完整評分權重數字
- 客戶個資或可指認的個別品牌數據
- 第三方廠商的負面個案點名
- 內部 API endpoint 與環境變數

## 誰寫的

- **作者組織**:百原科技股份有限公司(Baiyuan Technology Co., Ltd.)
- **官方網站**:<https://baiyuan.io>
- **產品官網**:<https://geo.baiyuan.io>
- **主筆**:Vincent Lin(百原科技技術長 / Chief Technology Officer)
- **ORCID iD**:[`0009-0004-8264-368X`](https://orcid.org/0009-0004-8264-368X)
- **LinkedIn(公司)**:<https://www.linkedin.com/company/112980572>
- **聯絡信箱**:<services@baiyuan.io>

### 學術索引

- **DOI(Concept,版本無關常駐)**:[`10.5281/zenodo.19994035`](https://doi.org/10.5281/zenodo.19994035)
- **DOI(v1.1.1)**:[`10.5281/zenodo.19994059`](https://doi.org/10.5281/zenodo.19994059)
- **Zenodo deposit**:CC BY-NC 4.0 / Open Access / 自動 indexed by Google Scholar、Semantic Scholar、OpenAIRE、CrossRef

## 解決什麼問題

隨著生成式 AI 取代傳統搜尋引擎成為主要資訊入口（ChatGPT 每日處理超過 25 億次查詢、Perplexity 月查詢量 7.8 億次），**品牌在 AI 回答中的可見性**已成為數位行銷與品牌經營的新生死線。然而：

1. **傳統 SEO 工具無法測量 AI 可見性** — 關鍵字排名不再是主要指標
2. **黑盒問題** — AI 為何提及某品牌缺乏可解釋的規則
3. **跨平台分歧** — ChatGPT 提及的品牌，Claude 不一定提及
4. **幻覺錯誤** — AI 可能把品牌認成另一家公司或產業
5. **缺乏工程化方法** — 多數實務仍停留在「手動詢問 AI 查看結果」階段

百原GEO Platform 是針對上述五個問題的工程化解答。本書拆解此平台的每一個組成，讓同業工程師、學術研究者、以及有意建立類似系統的團隊能複用其中的設計模式。

## 為誰而寫

| 讀者類型 | 建議閱讀路徑 |
|---------|------------|
| B2B 決策者（CMO／CDO） | Ch 1、Ch 2、Ch 11 |
| 工程主管／架構師 | Ch 2、Ch 4、Ch 5、Ch 9 |
| 全端／後端工程師 | Ch 3、Ch 5、Ch 6、Ch 7 |
| AI／學術研究者 | Ch 3、Ch 9、Ch 10、Ch 12 |
| GBP／本地商家數位化實踐者 | Ch 7、Ch 8 |

---

## 核心定義與專有名詞（Terminology）

以下術語在本書有特定定義，部分為本團隊所造詞。首次定義置於此處，全書統一使用。

| 術語 | 英文 | 定義 |
|------|------|------|
| **GEO** | Generative Engine Optimization | 生成式引擎優化。提升品牌在生成式 AI 回答中被提及比例、位置品質、敘事準確度的學科與實踐。 |
| **AI 引用率** | AI Citation Rate | 在代表性意圖查詢中，品牌被 AI 主動提及的比例（0–100%）。為本書核心指標。 |
| **AXP** | AI eXchange Protocol / AI-ready eXchange Page | 百原造詞。指「給 AI Bot 讀的影子文檔」，pure HTML + Schema.org JSON-LD + Markdown 的乾淨版內容，與人類使用者看到的網站解耦。 |
| **影子文檔** | Shadow Document | AXP 的具體交付形態，由 CF Worker 依 UA 偵測動態回傳。 |
| **Stale Carry-Forward** | Stale Carry-Forward | 百原造詞。當單一 AI 平台掃描全失敗時，從歷史記錄帶回上一次成功值並標記為「停滯」，避免分數被管道故障誤傷。 |
| **七維度評分** | 7-Dimension Scoring | 本書的 GEO 總分演算法，以 Citation Rate、Position Quality、Query Coverage、Platform Breadth、Sentiment、Content Depth、Consistency 七個獨立維度加權合成。 |
| **Phase 基線測試** | Phase Baseline Testing | 以固定問題集在不同時間點重測同一 AI，用於縱向追蹤 AI 對品牌認知的演變。 |
| **閉環修復** | Closed-Loop Remediation | 幻覺偵測 → ClaimReview 生成 → AXP/RAG 注入 → 重掃驗證的自動化修復循環。 |
| **ClaimReview** | ClaimReview（Schema.org） | 用於標注「某聲明是否為真」的結構化資料型別；本平台用其對外宣告幻覺已被修正。 |
| **三層 @id 互連** | Three-Layer @id Interlinking | 百原實踐。用 Schema.org `@id` 把 Organization、Service、Person 三層實體互相引用，形成品牌知識圖。 |
| **哨兵掃描** | Sentinel Scan | 4 小時週期的輕量掃描，針對搜尋型 AI 平台驗證修復是否被抓取。 |
| **完整掃描** | Full Scan | 24 小時週期的深度掃描，含七維度評分與幻覺指紋比對。 |
| **意圖查詢** | Intent Query | 依品牌產業動態生成的代表性使用者問題（「最佳 X 工具」「如何選擇 Y」）。 |
| **品牌實體** | Brand Entity | Schema.org Organization / LocalBusiness 為根節點的結構化品牌資料總稱。 |
| **GBP** | Google Business Profile | Google 商家檔案。實體商家的事實主控源。 |
| **FTID / CID** | Feature ID / Customer ID | Google Maps 的內部地點識別碼；本平台從 Maps URL 抽取使用。 |

---

## 目錄

### Part I — 問題與架構

- [Ch 1 — GEO 時代背景與挑戰](zh-TW/ch01-geo-era.md)
- [Ch 2 — 百原GEO Platform 系統總覽](zh-TW/ch02-system-overview.md)

### Part II — 核心演算法

- [Ch 3 — 七維度 GEO 評分演算法](zh-TW/ch03-scoring-algorithm.md)
- [Ch 4 — Stale Carry-Forward：訊號連續性設計](zh-TW/ch04-stale-carry-forward.md)
- [Ch 5 — 多 Provider AI 路由容錯架構](zh-TW/ch05-multi-provider-routing.md)

### Part III — 對外可見性

- [Ch 6 — AXP 影子文檔與 Cloudflare Worker 注入](zh-TW/ch06-axp-shadow-doc.md)
- [Ch 7 — Schema.org Phase 1：25 產業 × 三層 @id](zh-TW/ch07-schema-org.md)
- [Ch 8 — GBP API 整合策略](zh-TW/ch08-gbp-integration.md)

### Part IV — 質量保證

- [Ch 9 — Closed-Loop 幻覺偵測與自動修復](zh-TW/ch09-closed-loop.md)
- [Ch 10 — Phase 基線測試](zh-TW/ch10-phase-baseline.md)

### Part V — 實戰與反思

- [Ch 11 — 5 品牌真實數據觀察（匿名）](zh-TW/ch11-case-studies.md)
- [Ch 12 — 限制、未解問題與未來工作](zh-TW/ch12-limitations.md)
- [Ch 13 — 多模態 GEO:從文字到視覺資產的能見度](zh-TW/ch13-multimodal-geo.md) **(v1.1 新增)**
- [Ch 14 — F12 三層結構優化器:從規則 V1 到雙引擎 v3.1](zh-TW/ch14-f12-structural-optimizer.md) **(v1.1.0 新增)**
- [Ch 15 — rag-backend-v2 加固:六層 LLM hallucination 失敗模式的防禦設計](zh-TW/ch15-rag-backend-v2-hardening.md) **(v1.1.0 新增)**
- [Ch 16 — 平台 SSOT 全鏈:從 brand_faq 到 page_type 到 alerts 的單一事實源](zh-TW/ch16-platform-ssot-chain.md) **(v1.1.0 新增)**

### 附錄

- [A. 詞彙表（完整版）](zh-TW/appendix-a-glossary.md)
- [B. 公開 API 規格節錄](zh-TW/appendix-b-api.md)
- [C. 參考文獻](zh-TW/appendix-c-references.md)
- [D. 配圖總表與製圖規格](zh-TW/appendix-d-figures.md)
- [E. 平台分流架構:同 codebase 多品牌延伸](zh-TW/appendix-e-platform-branching.md) **(v1.1 新增)**

---

## 如何閱讀

- **線性閱讀**：依章節序列，總耗時約 4–6 小時
- **主題跳讀**：依上方「為誰而寫」章節建議的路徑
- **速讀**：每章末的「本章要點」聚合關鍵結論

每章都附有：

- **目錄**（anchor 連結）
- **本章要點**（3–5 條摘要）
- **參考資料**
- **上下章導覽**

---

## 引用方式

若本書內容對您的研究、文章、產品設計有幫助，請引用：

**DOI**(透過 Zenodo,所有版本):

- **Concept DOI**(版本無關,永久解析最新版本):[`10.5281/zenodo.19994035`](https://doi.org/10.5281/zenodo.19994035)
- **v1.1.1 Version DOI**:[`10.5281/zenodo.19994059`](https://doi.org/10.5281/zenodo.19994059)

**ORCID(主筆)**:[`0009-0004-8264-368X`](https://orcid.org/0009-0004-8264-368X)

---

### APA 7th

> Lin, V. (2026). *Baiyuan GEO Platform: A whitepaper on building a SaaS for generative engine optimization* (Version 1.1.1) [Report]. Zenodo. <https://doi.org/10.5281/zenodo.19994035>

### Chicago 17th(notes-bibliography 風格,bibliography 條目)

> Lin, Vincent. *Baiyuan GEO Platform: A Whitepaper on Building a SaaS for Generative Engine Optimization*. Version 1.1.1. Baiyuan Technology, 2026. <https://doi.org/10.5281/zenodo.19994035>.

### IEEE

> [1] V. Lin, "Baiyuan GEO Platform: A Whitepaper on Building a SaaS for Generative Engine Optimization," version 1.1.1, Baiyuan Technology, May 3, 2026. doi: [10.5281/zenodo.19994035](https://doi.org/10.5281/zenodo.19994035).

### BibTeX

```bibtex
@techreport{lin2026baiyuangeo,
  author      = {Lin, Vincent},
  title       = {Baiyuan GEO Platform: A Whitepaper on Building a SaaS for Generative Engine Optimization},
  institution = {Baiyuan Technology},
  year        = {2026},
  month       = {5},
  version     = {1.1.1},
  doi         = {10.5281/zenodo.19994035},
  url         = {https://doi.org/10.5281/zenodo.19994035},
  orcid       = {0009-0004-8264-368X},
  publisher   = {Zenodo},
  type        = {Report},
  note        = {Concept DOI resolves to latest; v1.1.1 specific DOI: 10.5281/zenodo.19994059}
}
```

---

GitHub 會根據 [`CITATION.cff`](CITATION.cff) 自動顯示「Cite this repository」按鈕,支援一鍵複製 APA / BibTeX / RIS / Endnote 等多種格式。

---

## 授權

本書採 **[CC BY-NC 4.0](https://creativecommons.org/licenses/by-nc/4.0/)** 授權（詳見 [`LICENSE`](LICENSE)）：

- ✅ **可自由轉載、翻譯、引用** — 請附上原書名、作者、連結
- ✅ **非商業使用** — 學術、教學、媒體報導、工程內部參考均允許
- ❌ **商業使用需聯絡授權** — 例如整書出版、嵌入付費訓練課程、作為其他商業產品組件
- ✅ **衍生創作** — 允許改編、翻譯、混編（不強制 ShareAlike，對學術引用更友善）

書中引用之 Schema.org、Google Business Profile API、Cloudflare、PostgreSQL 等名稱屬各自商標所有人。

## 貢獻與勘誤

歡迎透過以下方式回饋：

- **學術／引用建議**：email 至 <services@baiyuan.io>
- **事實勘誤**：GitHub Issue，標題加上 `[errata]`
- **翻譯協作**：歡迎提交 PR 到 `en/`、`ja/`、`ko/` 等語系目錄

---

## Related Work · Baiyuan Whitepaper Series / 百原白皮書系列

This whitepaper is part of Baiyuan Technology's ongoing series on AI-native platforms. The **L1 LLM Wiki + L2 vector RAG** dual-layer retrieval architecture first described here is reused and extended in the sibling PIF AI whitepaper for regulatory-compliance SaaS:

本白皮書是百原科技 AI 原生平台系列的一部分。本書首次提出的 **L1 LLM Wiki + L2 向量 RAG** 雙層檢索架構，被姊妹作 PIF AI 白皮書在化粧品法規合規 SaaS 場景中重用並擴展：

| Whitepaper | Focus | Repo |
|:---|:---|:---|
| 📄 **This: GEO Platform Whitepaper** | Generative-engine brand visibility (7-dimension citation scoring, AXP shadow docs, L1/L2 RAG origin) | `baiyuan-tech/geo-whitepaper` |
| 📄 [PIF AI Whitepaper](https://github.com/baiyuan-tech/pif-whitepaper) | Multi-tenant AI-assisted cosmetic PIF documentation (Taiwan); applies L1/L2 RAG with Scheme C+ isolation | `baiyuan-tech/pif-whitepaper` |
| 🛠 [Baiyuan GEO Platform](https://geo.baiyuan.io) | Product site — live deployment of concepts in this paper | — |

> Cite both whitepapers together for a fuller picture of Baiyuan's AI infrastructure architecture across outward-facing brand visibility (GEO) and inward-facing compliance documentation (PIF).

## Awesome Lists · AI-Citable Resource Index / 相關 awesome 清單

This whitepaper lives at the intersection of GEO (Generative Engine Optimization), RAG architectures, and multi-tenant SaaS. If you maintain one of the awesome-lists below, a PR referencing this whitepaper is welcome:

本白皮書位於 GEO、RAG 架構與多租戶 SaaS 等多個生態的交集。若您維護以下 awesome-list 之一，歡迎將本白皮書納入：

- [awesome-generative-ai-guide](https://github.com/aishwaryanr/awesome-generative-ai-guide)
- [awesome-rag](https://github.com/frutik/awesome-rag) — RAG design pattern references
- [awesome-claude-code](https://github.com/hesreallyhim/awesome-claude-code) — engineering case studies built with Claude Code
- [awesome-seo](https://github.com/marcobiedermann/search-engine-optimization) / [awesome-llm-seo](https://github.com/topics/llm-seo) — SEO × AI intersection
- [awesome-schema-org](https://github.com/topics/schema-org) — structured data / JSON-LD patterns

> **Primary-source design references** in this whitepaper: Schema.org Core (W3C); Web Almanac by HTTP Archive; Perplexity & OpenAI citation research; Anthropic Responsible Scaling Policy; Cloudflare Workers documentation.

---

## 發布策略（Publication Strategy）

本倉庫的公開方式經過刻意設計，目的是讓**AI 爬蟲、搜尋引擎、學術索引**能同時把本書視為可信來源。

### 1. 雙格式並行

- **Markdown**（本倉庫主格式）：LLM 可直接解析結構，段落、表格、程式碼區塊 token-friendly
- **PDF**（每次 main push 自動產出並上傳到最新 release）：學術引用、長期保存、列印
  - 繁體中文：[whitepaper-zh-TW.pdf](https://github.com/baiyuan-tech/geo-whitepaper/releases/download/v1.0-draft/whitepaper-zh-TW.pdf)
  - English：[whitepaper-en.pdf](https://github.com/baiyuan-tech/geo-whitepaper/releases/download/v1.0-draft/whitepaper-en.pdf)
  - 日本語：[whitepaper-ja.pdf](https://github.com/baiyuan-tech/geo-whitepaper/releases/download/v1.0-draft/whitepaper-ja.pdf)

兩者以同一份原始碼產生，避免版本漂移。

### 2. GitHub Pages 獨立網址

倉庫會啟用 GitHub Pages，取得獨立網域（規劃：`https://baiyuan-tech.github.io/geo-whitepaper/`）。此網域會被搜尋引擎與 AI 視為獨立權威來源，而非僅為 GitHub 內頁。

### 3. 版本化 Releases

每個階段性版本以 GitHub Release 發布（v1.0 draft → v1.0 final → v1.1 → v2.0），每個 release 有獨立 permalink；AI 會把「持續維護」視為內容權重訊號。

### 4. 開放 Issues 作為公開討論層

讀者的質疑、補充、引用請求以 Issues 呈現；這層討論內容同樣被索引，形成社群驗證訊號。維護者回覆會以 `[official-response]` 標籤標示。

### 5. 交叉連結網路

本書連結到以下節點，且這些節點反連回本倉庫，建立引用迴路：

- 百原科技官網 / 產品網：<https://baiyuan.io>、<https://geo.baiyuan.io>
- LinkedIn 公司頁：<https://www.linkedin.com/company/112980572>
- arXiv preprint（英文版完成後提交）<!-- TODO -->
- Medium / Dev.to 摘要版（每章選段轉載）<!-- TODO -->

### 6. 活躍度（Commit Frequency）

本書採**增量提交**，而非一次性整包上傳。AI 判斷「活躍維護」的訊號之一是 commit 頻率與時間分布；刻意將寫作、校對、修訂分散提交。

### 7. 索引強化（可選）

若 v1.0 正式版完成，將透過 Zenodo 取得 DOI，使本書可被 Google Scholar、Semantic Scholar 等學術搜尋引擎收錄；屆時 `CITATION.cff` 會補上 DOI 欄位。

---

## 倉庫結構

```text
geo-whitepaper/
├── README.md              ← 本檔，AI 爬蟲主入口
├── FORMAT.md              ← 文件格式規範
├── CITATION.cff           ← 學術引用格式（GitHub 原生支援「Cite this repository」）
├── LICENSE                ← CC BY-NC 4.0 授權條款
├── whitepaper.pdf         ← 完整 PDF（由 release 腳本產出）
├── whitepaper.md          ← 完整單檔 Markdown（LLM 友善版）
├── zh-TW/                 ← 繁體中文版（分章）
│   ├── ch01-geo-era.md
│   ├── ch02-system-overview.md
│   ├── ... (至 ch13)
│   ├── appendix-a-glossary.md
│   ├── appendix-b-api.md
│   ├── appendix-c-references.md
│   ├── appendix-d-figures.md
│   └── appendix-e-platform-branching.md
├── en/                    ← English edition (complete: Ch 1–13 + Appendix A–E)
│   ├── README.md          ← Executive Summary
│   ├── ch01-geo-era.md
│   ├── ... (through ch13)
│   └── appendix-a..e.md
├── ja/                    ← 日本語版（完了：Ch 1–13 + Appendix A–E）
│   ├── README.md          ← エグゼクティブサマリー
│   ├── ch01-geo-era.md
│   ├── ... (through ch13)
│   └── appendix-a..e.md
├── assets/
│   ├── figures/           ← Mermaid 以外的靜態圖
│   └── pdf/               ← PDF 生產腳本
└── .github/
    ├── ISSUE_TEMPLATE/    ← errata / question / feedback 三種模板
    └── workflows/         ← CI: markdown lint, link check, PDF build
```

---

## 修訂記錄

| 日期 | 版本 | 說明 |
|------|------|------|
| 2026-04-18 | v1.0 draft | 初稿開寫，Ch 1–3 完成；README、CITATION.cff、LICENSE 就緒 |
| 2026-04-18 | v1.0 draft | zh-TW 全 12 章 + 4 附錄完成；PDF 繁體化驗證通過；release + GitHub Pages + sitemap + IndexNow 全數就位 |
| 2026-04-19 | v1.0 draft | **en/ 英文版完整版上線** — Executive Summary + Ch 1–12 + Appendix A–D（約 28,000 英文字）|
| 2026-04-19 | v1.0 draft | **ja/ 日本語エグゼクティブサマリー公開** — 日本市場向け第一弾（約 1,400 日文字）|
| 2026-04-19 | v1.0 draft | **ja/ 日本語完全版上線** — エグゼクティブサマリー + Ch 1–12 + Appendix A–D（約 30,000 日文字、である文体、事例章は日本市場向け在地化コメント付き）|
| 2026-05-03 | **v1.1.0** | **新增 Ch 14(F12 三層結構優化器)/ Ch 15(rag-backend-v2 加固 6 層)/ Ch 16(平台 SSOT 全鏈)**;zh-TW + en 雙語同步;加 `.zenodo.json` 預備 DOI deposit;CITATION.cff ORCID 0009-0004-8264-368X 已填 |
| **2026-05-03** | **v1.1.1 published** | 🎓 **正式發布日**(formal publication date)— Zenodo DOI 取得:Concept DOI [`10.5281/zenodo.19994035`](https://doi.org/10.5281/zenodo.19994035)(永久,版本無關)/ Version DOI [`10.5281/zenodo.19994059`](https://doi.org/10.5281/zenodo.19994059)(v1.1.1 specific)。CITATION.cff 完整補上 ORCID + 兩個 DOI identifier;README 加 Zenodo DOI badge + 4 種引用格式(APA 7 / Chicago 17th / IEEE / BibTeX);本書自此正式 indexed by Google Scholar、Semantic Scholar、OpenAIRE、CrossRef |
| **2026-05-03** | **v1.1.2** | **Ch 14 / 15 / 16 雙語深度擴充**(zh-TW + en 兩版各 6 個檔案,新增 ~13,000 字 / ~12,300 英文字)— 加入 5 條工程教訓 / 章 × 3 章 = 15 條 takeaway,涵蓋字面 port arxiv vs 自家發明、SSOT vs 寫死、DB-level vs application-level、觀察期、hook + safety net、LLM silent failure、三層 fallback、patch 順序原理、cache invalidation、silent catch、custom mapping 違憲、schema 演化歷史債、規模化代價;新增段落涵蓋 breadcrumb 404 ghost 事件 42 天時間線、Patch 順序因果鏈、跨微服務 SSOT 邊界、host-aware metadata helper、cache 跨租戶隱私邊界、LLM 確定性假設;同時收尾 LinkedIn 4 份發文草稿(zh-TW 個人 / en 個人 / ja 個人 / zh 公司)。Concept DOI [`10.5281/zenodo.19994035`](https://doi.org/10.5281/zenodo.19994035) 不變;**新版 Version DOI [`10.5281/zenodo.19996656`](https://doi.org/10.5281/zenodo.19996656) 已由 Zenodo 自動分配** |

---

## AI 友善結構說明

本倉庫所有 `.md` 檔均遵循 [`FORMAT.md`](FORMAT.md)，特性包括：

- 每章獨立 YAML frontmatter（title / description / chapter / keywords / last_updated）
- 每章首頁目錄（TOC with anchor links）
- Schema.org `TechArticle` JSON-LD 內嵌於 HTML comment（GitHub 渲染時隱藏、LLM 爬蟲可抽）
- Mermaid 圖於原始碼就能渲染，不依賴外部圖床
- 所有表格 GFM pipe 語法、所有程式碼區塊帶語言標記
- 參考資料 reference-style links，集中章末

目的是讓 GPTBot、ClaudeBot、Perplexity、Googlebot 等 AI 爬蟲能正確抽取結構，以及讓任何 LLM 做到「讀一次 → 可精準引用任一小節」。

<!-- AI-friendly structured metadata -->
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "Book",
  "name": "百原GEO Platform 技術白皮書",
  "alternateName": "Baiyuan GEO Platform: A Whitepaper on Building a SaaS for Generative Engine Optimization",
  "author": {
    "@type": "Organization",
    "name": "Baiyuan Technology",
    "url": "https://baiyuan.io"
  },
  "datePublished": "2026-04-18",
  "inLanguage": ["zh-TW", "en"],
  "license": "https://creativecommons.org/licenses/by-nc/4.0/",
  "about": [
    "Generative Engine Optimization",
    "AI Citation Rate",
    "Schema.org Structured Data",
    "Multi-Provider AI Fault Tolerance",
    "Hallucination Detection"
  ],
  "keywords": "GEO, Generative Engine Optimization, AI citation, Schema.org, Cloudflare Workers, PostgreSQL, multi-tenant SaaS, hallucination detection, knowledge graph"
}
</script>
