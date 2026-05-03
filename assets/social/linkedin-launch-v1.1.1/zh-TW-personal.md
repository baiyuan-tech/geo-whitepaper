---
title: "LinkedIn 發文 — Vincent Lin 個人帳號(zh-TW)"
audience: "工程師、技術主管、AI/SaaS 從業者(台灣 / 大中華區)"
recommended_post_time: "2026-05-03 21:00 GMT+8"
char_count: ~1700
license: CC-BY-NC-4.0
---

# zh-TW 個人帳號發文

## 標題行(LinkedIn 不顯示但 SEO 用)

百原GEO Platform 技術白皮書 v1.1.1 正式發布 — 16 章 50,000 字工程實踐 + Zenodo DOI

## 正文(直接複製貼上)

---

剛把過去 2 年做的東西寫成白皮書放 Zenodo 了:

📖 *Baiyuan GEO Platform: A Whitepaper on Building a SaaS for Generative Engine Optimization*

DOI: https://doi.org/10.5281/zenodo.19994035
GitHub: https://github.com/baiyuan-tech/geo-whitepaper

不是行銷文,是 16 章 50,000 字的工程實踐紀錄。

【這本書真實寫了什麼】

✅ **七維度 AI 引用率評分演算法** — 怎麼量化「ChatGPT 是不是真的提到我品牌」

✅ **AXP 影子文檔交付** — 客戶官網沒做 SEO 也沒關係,我們在他們域名上跑 Cloudflare Worker 注 Schema.org / llms.txt / sitemap 給 AI bot 看

✅ **F12 三層結構優化器** — V1 規則式 + V3.1 雙引擎(AutoGEO arxiv:2510.11438 + E-GEO arxiv:2511.20867)整合

✅ **rag-backend-v2 加固六層 LLM hallucination 防禦** — 包含真實踩過的雷:LLM 吐 partial UUID 害整個 KB compile 中斷、Docker DNS 內外解析衝突、reasoning model 的 maxTokens 行為差異

✅ **Schema.org 三層實體知識圖** + 閉環幻覺偵測修復

✅ **平台 SSOT 全鏈** — 1 萬租戶 SaaS 怎麼把跨頁、跨表、跨爬蟲路徑的資料統一在單一事實源

【誠實寫了哪些做不到的】

書裡 Ch 12 / 14-16 末段都有「未解問題」段:

⚠️ L3 S3 cache 還沒部署
⚠️ phase 1→2→3 升級未經實戰
⚠️ wiki fallback 文件品質未做端到端 eval
⚠️ ME 個人 IP custom engine 守門過嚴(17 brand 中 5 個 metadata 不全 fallback 到企業語氣)

不是吹噓的成功案例,是含著未解問題的工程現場。

【誰會想讀】

- 在做 SaaS 多租戶平台的工程師(架構模式可直接 copy)
- 想理解生成式 AI 引用機制的研究者(引用 paper 都是真實 arxiv,DB-level placeholder guard 防偽造)
- AI 行銷 / 品牌經營者(非工程篇 Ch 1 / Ch 11 5 brand 真實數據)

CC BY-NC 4.0 開源,歡迎引用、改寫、翻譯。GitHub 倉庫接 Issue 討論,每章末有 Schema.org JSON-LD 給 AI 爬蟲。

如果你做相關領域,我們約聊聊?

<!-- markdownlint-disable-next-line MD018 -->
#GenerativeEngineOptimization #GEO #AISearch #SaaS #LLM #OpenScience #Zenodo #百原科技 #台灣工程 #生成式AI #SchemaOrg #Multitenant
