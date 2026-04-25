---
title: "Appendix B — 公開 API 規格節錄"
description: "百原GEO Platform 對外開放的核心 REST endpoint 規格與呼叫範例；內部 admin endpoint 不收錄。"
appendix: B
part: 6
lang: zh-TW
authors:
  - name: Vincent Lin
    affiliation: Baiyuan Technology
license: CC-BY-NC-4.0
last_updated: 2026-04-18
last_modified_at: '2026-04-25T16:16:59Z'
---
















# Appendix B — 公開 API 規格節錄

本附錄節錄百原GEO Platform **對外公開**的 REST endpoint 規格，供第三方整合、研究與審計參考。內部管理性質 endpoint（admin、監控、billing）不收錄。

## 共通規範

### Base URL

```text
https://api.geo.baiyuan.io/v1
```

### Authentication

所有 endpoint 需帶 `Authorization: Bearer <token>` 標頭。取得 token 請見客戶面板「API 金鑰管理」。

### Response Envelope

所有回應為：

```json
{
  "status": "success" | "error",
  "data":   { ... },           // status=success 時
  "error":  "description"      // status=error 時
}
```

### Rate Limit

- 預設 60 req/min；企業方案可提升
- 超限回 HTTP 429，`Retry-After` 標頭告知等待秒數

---

## 1. Brands

### GET `/brands`

列出當前帳戶可見的品牌。

**Query params**：

- `limit`（1–100，預設 20）
- `offset`（預設 0）
- `industry_code`（可選，見 [§7.2](./ch07-schema-org.md#72-25-類產業特化-type-的設計) 的 25 類 enum）

**Response**：

```json
{
  "status": "success",
  "data": {
    "brands": [
      {
        "id": "brd_01H...",
        "name": "Example Brand",
        "industry_code": "saas_application",
        "url": "https://example.com",
        "completion": 78,
        "current_geo_score": 62
      }
    ],
    "total": 42,
    "limit": 20,
    "offset": 0
  }
}
```

### GET `/brands/:id`

取得單一品牌詳情。

### POST `/brands`

建立新品牌。**body**：`name`（必填）、`url`、`industry_code`、`description` 等。

### PUT `/brands/:id`

更新品牌。body 內容依 `Content-Type` 而定；支援 merge-patch 語義。

---

## 2. Scans

### POST `/scans`

觸發一次掃描（立刻執行、非排程）。

**body**：

```json
{
  "brand_id": "brd_01H...",
  "platforms": ["chatgpt", "claude", "perplexity"],  // optional; 預設全部
  "intent_count": 20,
  "scan_layer": "full" | "sentinel"                   // 預設 "full"
}
```

**Response**（202 Accepted）：

```json
{
  "status": "success",
  "data": {
    "scan_id": "scn_01H...",
    "estimated_duration_seconds": 180,
    "queue_position": 3
  }
}
```

### GET `/scans/:id`

查詢掃描狀態。

**Response**：

```json
{
  "status": "success",
  "data": {
    "scan_id": "scn_01H...",
    "state": "queued" | "running" | "completed" | "failed",
    "progress": 0.75,
    "started_at": "2026-04-18T09:15:00Z",
    "completed_at": null,
    "results_url": "/scans/scn_01H.../results"
  }
}
```

### GET `/scans/:id/results`

取得掃描結果。

**Response**（節錄）：

```json
{
  "status": "success",
  "data": {
    "scan_id": "scn_01H...",
    "scan_layer": "full",
    "geo_score": 62,
    "dimensions": {
      "citation":    { "value": 55, "isStale": false },
      "position":    { "value": 70, "isStale": false },
      "coverage":    { "value": 60, "isStale": false },
      "breadth":     { "value": 58, "isStale": false },
      "sentiment":   { "value": 72, "isStale": false },
      "depth":       { "value": 50, "isStale": false },
      "consistency": { "value": 65, "isStale": false }
    },
    "platforms": {
      "chatgpt":    { "sov": 60, "position": "top-third" },
      "claude":     { "sov": 58, "position": "top-third" },
      "perplexity": { "sov": 48, "position": "middle" }
    }
  }
}
```

---

## 3. Entity（Schema.org 品牌實體）

### GET `/brands/:id/entity`

取得品牌的 Schema.org 完整資料（含 Organization、Services、Employees、Hours、Location、Social Links）。

### GET `/brands/:id/entity/schema.json`

取得生成後的 Schema.org JSON-LD（可直接嵌入客戶網站）。

**Response**：`Content-Type: application/ld+json`；body 為 `{"@context": "https://schema.org", "@graph": [...]}`。

### GET `/brands/:id/entity/completion`

取得完整度分數。

**Response**：

```json
{
  "status": "success",
  "data": {
    "total_percent": 78,
    "breakdown": {
      "basic_info":  { "percent": 100, "weight_pct": 25 },
      "location":    { "percent":  80, "weight_pct": 15 },
      "services":    { "percent":  60, "weight_pct": 15 },
      "employees":   { "percent":  40, "weight_pct": 10 }
    },
    "missing_high_priority": ["description", "logo_url"]
  }
}
```

---

## 4. AXP（影子文檔）

### GET `/axp?host=<domain>&path=<path>`

此 endpoint **由 Cloudflare Worker 呼叫**，一般整合開發者無需直接使用。返回的是該品牌該路徑的影子文檔 HTML。

---

## 5. Hallucinations

### GET `/brands/:id/hallucinations`

列出已偵測到的幻覺。

**Query params**：`severity`（critical/major/minor/info）、`status`（open/resolving/resolved）。

### POST `/brands/:id/hallucinations/:hid/repair`

觸發修復流程。

---

## 6. Baseline（Phase 基線測試）

### POST `/brands/:id/baseline`

建立新的 Phase 基線 cohort。

### POST `/brands/:id/baseline/:cohortId/run`

對既有基線重測。

### GET `/brands/:id/baseline/:cohortId/compare`

取得 Phase 1 vs Phase 2 / 3 的對比。

---

## 7. Webhooks（客戶可接收）

百原平台**主動推送**以下事件到客戶自設的 webhook URL（客戶於面板設定）：

| 事件 | Trigger |
|------|---------|
| `scan.completed` | 一次掃描完成 |
| `hallucination.detected` | 偵測到新幻覺（severity >= major） |
| `hallucination.resolved` | 幻覺收斂 |
| `completion.threshold_crossed` | 完整度跨過 60% / 80% / 100% 關鍵門檻 |
| `gbp.sync_failure` | GBP 同步連續失敗 3 次 |

Payload 格式：

```json
{
  "event": "scan.completed",
  "brand_id": "brd_01H...",
  "scan_id": "scn_01H...",
  "occurred_at": "2026-04-18T09:30:00Z",
  "data": { ... }
}
```

每個 webhook 帶有 `X-Baiyuan-Signature` 標頭（HMAC-SHA256），客戶應驗證後才處理。

---

## 錯誤代碼

| HTTP | Code | 說明 |
|-----:|------|------|
| 400 | `invalid_parameter` | 參數格式／值不合法 |
| 401 | `unauthenticated` | 未帶 token 或 token 失效 |
| 403 | `forbidden` | 帶 token 但無此資源權限 |
| 404 | `not_found` | 資源不存在 |
| 409 | `conflict` | 狀態衝突（如 scan 進行中不能重複觸發） |
| 422 | `unprocessable_entity` | 資料驗證失敗 |
| 429 | `rate_limited` | 超過配額 |
| 500 | `internal_error` | 內部錯誤，請重試或聯絡支援 |

---

**導覽**：[← 附錄 A: 詞彙表](./appendix-a-glossary.md) · [📖 目次](../README.md) · [附錄 C: 參考文獻 →](./appendix-c-references.md)
