---
title: "LinkedIn Carousel (zh-TW) — 10-page launch content"
description: "百原GEO Platform 白皮書發布用 LinkedIn Document Post 輪播文案規格，10 頁內容 + 完整設計系統。"
lang: zh-TW
type: social-material
license: CC-BY-NC-4.0
last_updated: 2026-04-18
---

# LinkedIn Carousel (zh-TW) — 10-page launch content

配合 v1.0-draft 白皮書發布的 LinkedIn Document Post 輪播文案。Canva 範本：**LinkedIn Carousel 1080×1350**。匯出 PDF 後於 LinkedIn 貼文點「Add a document」上傳。

---

## 設計系統

| 元素 | 規格 |
|------|------|
| 尺寸 | 1080 × 1350 px（LinkedIn Document 最佳比例） |
| 主色 | 深藍 `#1e3a8a`（封面 / 標題區） |
| 強調色 | 亮橘 `#f97316`（數字 / 關鍵字） |
| 背景 | 米白 `#fafaf9`（內頁）/ 深藍（封面、結尾頁） |
| 標題字型 | Noto Serif TC Bold |
| 內文字型 | Noto Sans TC Regular |
| 數字字型 | Inter / Roboto Mono（tabular figures） |
| 頁腳固定 | `baiyuan GEO Platform Whitepaper · v1.0-draft · CC BY-NC 4.0 · page X/10` |

**設計原則**：每頁至多 25 個中文字 + 3 個數字。大量留白。讀者應在 3 秒看懂一頁。

---

## Page 1 — 封面

**Title（大字、置中）**

```text
當使用者
不再「搜尋」答案
```

**Subtitle（中字、下方）**

```text
而是「詢問」答案，
品牌的可見性規則正在重寫。
```

**Bottom（小字、右下）**

```text
百原GEO Platform
技術白皮書 · v1.0-draft
Vincent Lin · 2026
```

**Visual notes**

- 深藍背景 + 亮橘「詢問」二字高亮
- 左側加一條細橘色垂直線作視覺錨點
- 無其他裝飾

---

## Page 2 — Problem：三個數字

**Title（上方）**

```text
搜尋正在消失。
```

**Body（三個大數字，垂直排列）**

```text
25 億次／日
ChatGPT 使用者查詢量

7.8 億次／月
Perplexity 查詢量

-34~60%
Google SERP 點擊率跌幅
（AI Overview 啟用後）
```

**Bottom caption**

```text
傳統搜尋引擎的流量入口正在被 AI 回答取代。
```

**Visual notes**

- 白底、橘色數字、灰色單位
- 數字左對齊，單位上方更小字
- 右下角加一個小小的向下箭頭 icon 暗示下滑趨勢

---

## Page 3 — Paradigm Shift

**Title**

```text
SEO 已死嗎？不是。
但它不再是戰場。
```

**Body（左右兩欄對比）**

| SEO 時代 | GEO 時代 |
|---------|----------|
| 關鍵字排名 | AI 引用率 |
| 藍色連結 | 自然語言敘述 |
| H1 + 外鏈 | Schema.org + 結構化實體 |
| Google 規則可拆解 | AI 黑盒無公開規則 |
| 每日微調 | 模型重訓週期 |

**Bottom**

```text
GEO（Generative Engine Optimization）是一門平行新學科。
不是 SEO 的延伸。
```

**Visual notes**

- 兩欄垂直對比，SEO 灰色、GEO 深藍+亮橘
- 中間一條細豎線區隔
- 標題橘色「已死嗎？」問號凸顯

---

## Page 4 — 核心指標

**Title**

```text
AI 引用率：
一個決定生死的新指標。
```

**Body（置中定義框）**

```text
當使用者問 AI：
「最好的 B2B CRM 有哪些？」

AI 生成一段包含具體品牌名的回答。
被提到 → 進入候選清單。
沒被提到 → 根本不存在於那次對話。

Google Analytics 測不到。
Search Console 看不到。
只能專門去測。
```

**Bottom**

```text
我們稱這個指標：AI Citation Rate
```

**Visual notes**

- 整頁用 Q & A 對話框設計（左側灰色 query 氣泡，右側藍色 AI 回答氣泡）
- 「被提到 vs 沒被提到」用對比色（綠 vs 紅，但飽和度壓低）

---

## Page 5 — 七維度評分

**Title**

```text
單一「引用率」會騙了你。
```

**Body（七個維度圓形圖或列表）**

```text
我們把 GEO 健康度拆成七個維度：

1. Citation Rate      被提及率
2. Position Quality   位置品質
3. Query Coverage     查詢覆蓋率
4. Platform Breadth   平台覆蓋度
5. Sentiment          情感分數
6. Content Depth      內容深度
7. Consistency        跨平台一致性
```

**Bottom**

```text
權重刻意不公開——
防止客戶優化指標而非優化實質。
（跟 PageRank 的哲學一樣）
```

**Visual notes**

- 可用七角形 radar chart 簡圖（空白雷達，不放真實分數）
- 七個維度名稱以 circular layout 繞著 radar
- 或直接條列，每行左側一個圓點 icon

---

## Page 6 — 工程洞察 #1

**Title**

```text
當 AI 供應商限流時，
分數不該暴跌。
```

**Body**

```text
任何依賴外部 AI 服務的系統都會遇到限流。
直覺做法（失敗即計 0%）會把管道故障
誤判成品牌失寵——使用者看到分數暴跌
開始恐慌，但品牌其實沒變化。

我們的解法：Stale Carry-Forward

偵測平台全失敗 → 從歷史帶回上次成功值
→ 標記 isStale → 前端紅色徽章明示
→ 使用者不被誤導
```

**Bottom**

```text
GEO 分數反映「品牌狀態」，
不該反映「資料管道健康度」。
```

**Visual notes**

- 折線圖示意：兩條線，失敗即 0 vs carry-forward
- 失敗即 0 那條用紅色 + 鋸齒（斷崖式）
- carry-forward 那條用綠色 + 平穩

---

## Page 7 — 工程洞察 #2

**Title**

```text
給人類看的網站，
不該給 AI 爬蟲讀。
```

**Body**

```text
現代網站充滿 JavaScript、Cookie banner、
廣告追蹤、動態 UI——對人類是體驗，
對 AI 爬蟲是噪音。

我們的解法：AXP（AI-ready eXchange Page）

同一個 URL 透過 Cloudflare Worker
邊緣偵測 User-Agent：

• 人類使用者 → 原站
• GPTBot / ClaudeBot / PerplexityBot
  等 25 種 AI 爬蟲 → 乾淨的影子文檔
  （Pure HTML + Schema.org + Markdown）

實測：AI Bot 流量 2 週內上升 3–5 倍。
```

**Visual notes**

- 左右分屏視覺化：左邊「人類版 React 網站」彩色混亂、右邊「AI 版 pure HTML」極簡黑白
- 中間一個 Cloudflare 雲 icon 做路由分流示意

---

## Page 8 — 工程洞察 #3

**Title**

```text
「AI 說你壞話」不可怕。
可怕的是 AI 根本不提你。
```

**Body**

```text
負面提及 → AI 知道你存在 → 可用 ClaimReview
Schema.org 發起修復。

完全沒被提及 → 你不在候選池 →
沒有修復的著力點。

這是為什麼我們的閉環幻覺修復有一條
核心原則：

  「neutral ≠ 幻覺」

知識來源沒提到 ≠ 錯的。
誤把 neutral 當錯誤 → 觸發錯誤修復動作
→ 污染整個閉環。
```

**Bottom**

```text
偵測採 NLI 三分類 + ChainPoll 三重投票。
不靠窮舉 Ground Truth 也能高精度運作。
```

**Visual notes**

- 三個分類框：entailment（綠）/ contradiction（紅）/ **neutral（灰、最大）**
- 箭頭只從 contradiction 指向「進入修復流程」
- neutral 明顯標「跳過、不判定」

---

## Page 9 — 商業驗證

**Title**

```text
上線一個月，
簽下三組客戶。
```

**Body**

```text
不是理論。是實測數據。

我們運營 5 個 pilot 品牌 6 週，
同時在第一個付費月簽下：

• 連鎖醫美
  → GBP 整合 + 醫療類幻覺偵測

• 新興連鎖餐飲
  → Location 級 AXP + Phase 基線測試

• 頂級芳療／瑜珈
  → Content Depth 與 Sentiment 雙維度
    （敘事品質甚於頻次）

三組產業、三種核心需求。
印證七維度設計的必要性。
```

**Visual notes**

- 三個客戶 type icon（醫美、餐飲、瑜伽）橫向排列
- 每個 icon 下方列一行核心 feature
- 底部加 "5 pilot brands × 6 weeks / 3 paying clients / 1 month" 時間軸視覺

---

## Page 10 — CTA

**Title（置中、大字）**

```text
全書 30,000 字
12 章 · 44 張架構圖 · 4 附錄
CC BY-NC 4.0 全文公開
```

**Body（三個大連結區塊）**

```text
📖 閱讀網頁版
baiyuan-tech.github.io/geo-whitepaper

⬇️ 下載 PDF
github.com/baiyuan-tech/geo-whitepaper/releases

🇬🇧 English Executive Summary
github.com/baiyuan-tech/geo-whitepaper/tree/main/en
```

**Bottom（作者署名）**

```text
Author: Vincent Lin · Baiyuan Technology
Commercial licensing: services@baiyuan.io

歡迎 star · fork · translate · cite
```

**Visual notes**

- 深藍底（與封面呼應，視覺閉環）
- 三個 CTA 用橘色 outline button 樣式
- QR code 於右下（直接指向 GitHub repo）
- 最底部放 GitHub logo + star icon 小圖示暗示「歡迎 star」

---

## 執行清單

- [ ] Canva 建新檔：「LinkedIn Carousel Post」1080×1350
- [ ] 10 頁分別填入以上內容
- [ ] 匯出為 PDF（不是個別 PNG——LinkedIn Document 只吃 PDF）
- [ ] LinkedIn 發貼 → 下方「Add」→「Add a document」→ 上傳 PDF
- [ ] 貼文文案用「版本 C（爭議性）」搭配輪播最有力
- [ ] 發文時段：週二或週四早上 8:30 台灣時間
- [ ] 發完 30 分鐘內自己補一則留言：「📖 全書入口：<https://github.com/baiyuan-tech/geo-whitepaper>」

---

## 搭配貼文文案（版本 C，爭議性）

```text
我在剛發布的技術白皮書裡寫了一句會得罪很多人的話：

「SEO 做得再好，在 AI 時代也可能完全沒用。」

原因：主流 LLM 預訓練消化的是 Common Crawl、Wikipedia、
Wikidata、各廠自建的檢索資料集——Google 排名在這個流程中
只佔一小部分訊號，而且是間接的。

換句話說，你 SEO 排第一不代表 AI 提你。AI 要看的是：
結構化實體資料、可信權威連結、跨平台認知一致性。
完全不同的槓桿。

這本 30,000 字的白皮書，就是過去 18 個月我們在真實運營
百原GEO Platform 時寫下的實踐報告。不是理論、不是行銷稿——
是工程紀錄。包含：

・7 維評分演算法（為什麼我們不揭露權重）
・AXP 影子文檔（我們對 25 種 AI 爬蟲做了什麼）
・閉環幻覺修復（為什麼「neutral ≠ 幻覺」是核心原則）
・上線首月簽下 3 組客戶的產業差異觀察

CC BY-NC 4.0 全文公開、可引用、可翻譯。

https://github.com/baiyuan-tech/geo-whitepaper

你覺得 SEO 到底還會存在幾年？歡迎留言。

#SEO #GEO #GenerativeAI #MarketingTechnology #Disruption
```

---

## 版本備註

- 本檔版本對應白皮書 v1.0-draft（2026-04-18 發布）
- 未來白皮書大改版時，Page 9（商業驗證）的客戶數字需同步更新
- 英文版輪播等 `en/` 全書完成後再做獨立設計，不直譯
