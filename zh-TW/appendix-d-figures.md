---
title: "Appendix D — 配圖總表與製圖規格"
description: "百原GEO Platform 白皮書全書 Mermaid 圖／圖表／配圖的編號、主題、出處章節、建議工具、製圖準則。"
appendix: D
part: 6
lang: zh-TW
authors:
  - name: Vincent Lin
    affiliation: Baiyuan Technology
license: CC-BY-NC-4.0
last_updated: 2026-04-18
last_modified_at: '2026-04-19T12:46:05Z'
---








# Appendix D — 配圖總表與製圖規格

本書所有圖表集中索引。Mermaid 圖於 `.md` 原始檔內嵌、GitHub 原生渲染；其他靜態圖（若有）放 `assets/figures/`。

## D.1 總表

| 編號 | 類型 | 主題 | 章節 |
|------|------|------|------|
| Fig 1-1 | xychart-beta | 2023–2025 搜尋管道流量遷徙趨勢 | [Ch 1](./ch01-geo-era.md#fig-1-1搜尋管道流量遷徙示意) |
| Fig 1-2 | 對照表 | SEO vs GEO 核心差異 | [Ch 1 §1.3](./ch01-geo-era.md#13-geo-不是-seo-的延伸是獨立新學科) |
| Fig 2-1 | flowchart | 閉環系統六階段 | [Ch 2 §2.1](./ch02-system-overview.md#21-設計哲學從監測工具到閉環系統) |
| Fig 2-2 | flowchart | 品牌治理完整循環 | [Ch 2 §2.3](./ch02-system-overview.md#23-核心資料流) |
| Fig 2-3 | flowchart | 多租戶 RLS + App-level filter 雙保險 | [Ch 2 §2.5](./ch02-system-overview.md#25-多租戶資料隔離) |
| Fig 3-1 | graph | 七維度雷達關聯示意 | [Ch 3 §3.2](./ch03-scoring-algorithm.md#32-七維度設計) |
| Fig 4-1 | xychart-beta | 三策略訊號對比（直接計 0 / Stale Carry-Forward） | [Ch 4 §4.3](./ch04-stale-carry-forward.md#43-stale-carry-forward-設計) |
| Fig 4-2 | stateDiagram | 平台資料狀態機 (Fresh/Stale/Expired/Reset) | [Ch 4 §4.3](./ch04-stale-carry-forward.md#43-stale-carry-forward-設計) |
| Fig 4-3 | ASCII | 前端 isStale 徽章示意 | [Ch 4 §4.6](./ch04-stale-carry-forward.md#46-函數骨架) |
| Fig 5-1 | flowchart | modelRouter 雙路徑架構 | [Ch 5 §5.2](./ch05-multi-provider-routing.md#52-modelrouter-架構) |
| Fig 5-2 | flowchart | 切換觸發決策樹 | [Ch 5 §5.5](./ch05-multi-provider-routing.md#55-切換觸發與觀察性) |
| Fig 6-1 | flowchart | 同品牌兩種視角（人類版 vs AI 版） | [Ch 6 §6.1](./ch06-axp-shadow-doc.md#61-問題為何同一份-html-服務兩方都吃虧) |
| Fig 6-2 | flowchart | AXP 文件三層結構 | [Ch 6 §6.2](./ch06-axp-shadow-doc.md#62-axp-設計影子文檔的結構) |
| Fig 6-3 | flowchart | Cloudflare Worker 路由決策流程 | [Ch 6 §6.3](./ch06-axp-shadow-doc.md#63-cloudflare-worker-注入機制) |
| Fig 6-4 | flowchart | 25 種 AI Bot UA 四類分群 | [Ch 6 §6.4](./ch06-axp-shadow-doc.md#64-ai-bot-ua-清單與識別策略) |
| Fig 6-5 | flowchart | SaaS 自家品牌路徑分類決策樹 | [Ch 6 §6.5](./ch06-axp-shadow-doc.md#65-saas-自家品牌的路徑衝突) |
| Fig 6-6 | code block | JSON-LD flat vs nested 對比 | [Ch 6 §6.7](./ch06-axp-shadow-doc.md#67-json-ld-扁平化的踩坑紀錄) |
| Fig 7-1 | table | 25 產業分類表（實體 16 + 線上 7 + 保底 2） | [Ch 7 §7.2](./ch07-schema-org.md#72-25-類產業特化-type-的設計) |
| Fig 7-2 | flowchart | 三層 @id 知識圖譜 | [Ch 7 §7.3](./ch07-schema-org.md#73-三層-id-互連知識圖) |
| Fig 7-3 | flowchart | 實體 vs 線上欄位權重分歧 | [Ch 7 §7.4](./ch07-schema-org.md#74-實體商家-vs-線上服務欄位權重的分歧) |
| Fig 7-4 | flowchart | Wizard + Edit 雙入口流程 | [Ch 7 §7.6](./ch07-schema-org.md#76-wizard--edit-雙入口設計) |
| Fig 7-5 | flowchart | GBP URL Parser 四分支決策樹 | [Ch 7 §7.7](./ch07-schema-org.md#77-gbp-url-parser) |
| Fig 8-1 | flowchart | 資料單向流：GBP → Schema → AXP → AI | [Ch 8 §8.1](./ch08-gbp-integration.md#81-為何-gbp-是實體商家的事實主控源) |
| Fig 8-2 | flowchart | 兩種代管模式對比（Manager / OAuth） | [Ch 8 §8.3](./ch08-gbp-integration.md#83-代管模式選擇) |
| Fig 8-3 | table | 同步頻率矩陣 | [Ch 8 §8.5](./ch08-gbp-integration.md#85-同步頻率與配額) |
| Fig 8-4 | flowchart | GBP 整合四階段 Roadmap | [Ch 8 §8.7](./ch08-gbp-integration.md#87-phase-14-roadmap) |
| Fig 9-1 | flowchart | 閉環六階段 | [Ch 9 §9.1](./ch09-closed-loop.md#91-為什麼偵測不足需要閉環) |
| Fig 9-2 | flowchart | 幻覺五類分類樹 | [Ch 9 §9.2](./ch09-closed-loop.md#92-ai-幻覺的五種類型) |
| Fig 9-3 | flowchart | 三層知識來源餵 NLI | [Ch 9 §9.3](./ch09-closed-loop.md#93-偵測主機制nli-分類--chainpoll-投票) |
| Fig 9-4 | flowchart | 中央共用 RAG 架構 | [Ch 9 §9.4](./ch09-closed-loop.md#94-中央共用-ragsaas-架構的關鍵基礎設施) |
| Fig 9-5 | flowchart | LLM Wiki 文件生命週期 | [Ch 9 §9.5](./ch09-closed-loop.md#95-l1-llm-wiki被動檢索之上的主動語意層) |
| Fig 9-6 | flowchart | ClaimReview 三條注入路徑 | [Ch 9 §9.6](./ch09-closed-loop.md#96-修復claimreview-生成與多路徑注入) |
| Fig 9-7 | flowchart | 兩層掃描分工矩陣 | [Ch 9 §9.7](./ch09-closed-loop.md#97-兩層掃描閉環) |
| Fig 9-8 | xychart-beta | 幻覺 prevalence 收斂曲線 | [Ch 9 §9.8](./ch09-closed-loop.md#98-收斂時序與驗收) |
| Fig 10-1 | flowchart | 常規掃描 vs 基線測試 | [Ch 10 §10.1](./ch10-phase-baseline.md#101-常規掃描的縱向追蹤盲點) |
| Fig 10-2 | flowchart | Phase 1/2/3 資料結構 | [Ch 10 §10.2](./ch10-phase-baseline.md#102-phase-基線測試設計) |
| Fig 10-3 | flowchart | 四類變化觀察矩陣 | [Ch 10 §10.4](./ch10-phase-baseline.md#104-四類觀察軸) |
| Fig 11-1 | graph | 5 品牌七維度雷達 (週 1 vs 週 6) | [Ch 11 §11.2](./ch11-case-studies.md#112-geo-分數分布) |
| Fig 11-2 | flowchart | 平台覆蓋相對強度（英語強項 vs 中文在地） | [Ch 11 §11.3](./ch11-case-studies.md#113-平台覆蓋差異) |
| Fig 11-3 | xychart-beta | 完整度 × 引用率聚合趨勢 | [Ch 11 §11.4](./ch11-case-studies.md#114-schemaorg-完整度與引用率的相關性) |
| Fig 11-4 | xychart-beta | AXP 部署前後 AI Bot 流量 | [Ch 11 §11.5](./ch11-case-studies.md#115-axp-部署前後對比) |
| Fig 11-5 | pie | 客戶端踩坑類型分布 | [Ch 11 §11.6](./ch11-case-studies.md#116-客戶端常見踩坑) |
| Fig 12-1 | table | 現況覆蓋矩陣 | [Ch 12 §12.1](./ch12-limitations.md#121-目前做不到的事) |
| Fig 12-2 | flowchart | 未來功能依賴圖（短/中/長期） | [Ch 12 §12.4](./ch12-limitations.md#124-未來工作-roadmap) |

**總計**：13 章共 44 張圖（Mermaid 主導，含部分 ASCII art、table、code block）。

---

## D.2 製圖準則

### 1. 風格統一

全書使用以下調色盤（Mermaid `%%{init: {'theme':'base'}}%%` 後再微調）：

| 用途 | 色碼 |
|------|------|
| 主色（節點邊框、重點） | `#1e40af`（深藍） |
| 輔色（強調、箭頭） | `#ea580c`（亮橘） |
| 警示（錯誤、失敗路徑） | `#dc2626`（紅） |
| 成功（修復、收斂） | `#16a34a`（綠） |
| 中性（背景、輔助） | `#64748b`（灰藍） |

Mermaid 的 `style` 語法可在必要節點套用以上色碼：

```mermaid
flowchart LR
    A[節點]
    style A fill:#1e40af,color:#fff
```

### 2. 圖優先使用 Mermaid

- **技術流程圖**（flowchart / stateDiagram / sequenceDiagram）一律 Mermaid
- **資料關係圖**（graph、ER diagram）Mermaid
- **趨勢圖**（xychart-beta）Mermaid
- **pie chart** Mermaid

理由：

- 可版控、可 diff、可由讀者複製再製
- 不依賴外部圖床（圖床失效會造成白皮書壞圖）
- AI 爬蟲能讀 Mermaid 原始 code，圖本身也是結構化資料

### 3. 靜態圖（SVG/PNG）的使用時機

僅在以下情況使用靜態圖，放 `assets/figures/`：

- Mermaid 無法表達的複雜視覺（多層嵌套、藝術化圖示）
- 產品截圖（UI 畫面、客戶面板示範）
- 實際資料圖表需要 Recharts 互動性（PDF 版降為靜態 PNG）

所有靜態圖必須：

- 同時保留原始檔（`.fig`、`.drawio`、Figma 連結）
- 以 SVG 為首選格式（可縮放、檔案小）
- 檔名採 `fig-<chapter>-<number>-<slug>.svg` 格式，例：`fig-11-03-completion-citation.svg`

### 4. 圖片 alt text 規範

所有靜態圖必須有**描述性 alt text**，格式：

```markdown
![Fig X-Y 主題簡述：具體內容 <br>*Fig X-Y: 詳細 caption*](assets/figures/fig-XX-YY-slug.svg)
```

alt text 須能讓**不看圖的讀者理解圖的主旨**（同時服務視障使用者與 AI 爬蟲）。

### 5. 中英雙版的圖獨立製作

- 中文版的圖**不直譯**成英文版；英文版重新繪製，符合英文語境慣例
- 數據單位、日期格式、文字方向依語系調整
- 兩版共用編號（Fig 1-1 中文版 / Fig 1-1 英文版同對應）

### 6. 產品截圖的去識別化

UI 截圖必須：

- 客戶品牌名打碼（用 `Brand A`、`Example Brand` 替代）
- 真實數字模糊或替換為示例（例：`$12,345` → `$XX,XXX`）
- 內部 URL 路徑打碼（僅顯示結構，不顯示具體域名）
- Email 地址遮蔽（僅顯示 `user@***.com` 形式）

### 7. Mermaid 圖的相容性

Mermaid 於 GitHub Markdown、GitLab、Obsidian、VS Code Preview 均原生支援。但**在 Pandoc PDF 生成**時需要：

- CI 在 build 時用 `mermaid-cli` 將 code block 預轉為 SVG
- 或使用 Pandoc filter `pandoc-mermaid` 自動處理

`build-pdf.yml` 工作流已包含此步驟（見 [`.github/workflows/build-pdf.yml`](../.github/workflows/build-pdf.yml)）。

---

## D.3 製作進度

| 狀態 | 圖數量 | 佔比 |
|------|------:|-----:|
| ✅ 已於 `.md` 內完成 | 44 | 100% |
| 🟡 需產品實際截圖 | 0 | 0% |
| ⚪ 英文版另繪 | 0（pending 英文版） | — |

---

**導覽**：[← 附錄 C: 參考文獻](./appendix-c-references.md) · [📖 目次](../README.md)
