---
title: "付録B — 公開API仕様（抜粋）"
description: "百元 GEO Platform の公開 REST エンドポイント、第三者統合・研究・監査参照用。内部管理者エンドポイントは含まない。"
appendix: B
part: 6
lang: ja
authors:
  - name: Vincent Lin
    affiliation: 百元科技 (Baiyuan Technology)
license: CC-BY-NC-4.0
last_updated: 2026-04-18
---

# 付録B — 公開API仕様（抜粋）

本付録は百元 GEO Platform の**公開向け** REST エンドポイントを抜粋したものであり、第三者統合・研究・監査参照を目的とする。内部管理者エンドポイント（admin、monitoring、billing）は含まない。

## 共通規約

### ベースURL

```text
https://api.geo.baiyuan.io/v1
```

### 認証

全エンドポイントは `Authorization: Bearer <token>` ヘッダを要求する。トークンは顧客パネルの *「APIキー管理」* ページから取得する。

### レスポンスエンベロープ

全レスポンスは以下の形式を取る：

```json
{
  "status": "success" | "error",
  "data":   { ... },           // status=success のとき
  "error":  "description"      // status=error のとき
}
```

### レートリミット

- デフォルト 60 req/min、エンタープライズプランで引き上げ可能
- 超過 → HTTP 429 と `Retry-After` ヘッダ（待機秒数）

---

## 1. Brands

### GET `/brands`

現アカウントに見えるブランドを列挙。

**クエリパラメータ**：

- `limit`（1〜100、デフォルト 20）
- `offset`（デフォルト 0）
- `industry_code`（オプション、25分類 enum は [§7.2](./ch07-schema-org.md#72-industry-specialized-type-across-25-categories) 参照）

**レスポンス**：

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

単一ブランドの詳細を取得。

### POST `/brands`

新規ブランド作成。**Body**：`name`（必須）、`url`、`industry_code`、`description` 等。

### PUT `/brands/:id`

ブランド更新。Body 意味論は `Content-Type` に依存、merge-patch 対応。

---

## 2. Scans

### POST `/scans`

スキャンを即時トリガ（スケジュール無し）。

**Body**：

```json
{
  "brand_id": "brd_01H...",
  "platforms": ["chatgpt", "claude", "perplexity"],
  "intent_count": 20,
  "scan_layer": "full"
}
```

**レスポンス**（202 Accepted）：

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

スキャンステータス問い合わせ。

**レスポンス**：

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

スキャン結果取得。

**レスポンス**（抜粋）：

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

## 3. Entity（Schema.org ブランドエンティティ）

### GET `/brands/:id/entity`

ブランドの完全な Schema.org データ取得（Organization、Services、Employees、Hours、Location、Social Links）。

### GET `/brands/:id/entity/schema.json`

生成済み Schema.org JSON-LD の取得（顧客サイトに直接埋込可能）。

**レスポンス**：`Content-Type: application/ld+json`、body は `{"@context": "https://schema.org", "@graph": [...]}`。

### GET `/brands/:id/entity/completion`

完成度スコア取得。

**レスポンス**：

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

## 4. AXP（シャドウドキュメント）

### GET `/axp?host=<domain>&path=<path>`

本エンドポイントは **Cloudflare Worker から呼び出される**、統合者が直接呼ぶ必要は無い。指定ブランド・パスのシャドウドキュメントHTMLを返す。

---

## 5. Hallucinations

### GET `/brands/:id/hallucinations`

検出済ハルシネーション列挙。

**クエリパラメータ**：`severity`（critical/major/minor/info）、`status`（open/resolving/resolved）。

### POST `/brands/:id/hallucinations/:hid/repair`

修復フロー発動。

---

## 6. Baseline（フェーズ・ベースライン試験）

### POST `/brands/:id/baseline`

新規フェーズ・ベースラインコホート作成。

### POST `/brands/:id/baseline/:cohortId/run`

既存ベースラインの再実行。

### GET `/brands/:id/baseline/:cohortId/compare`

Phase 1 vs Phase 2 / 3 の比較取得。

---

## 7. Webhooks（顧客受信可能）

百元プラットフォームは顧客設定済 Webhook URL（パネルで設定）へ以下イベントを**プッシュ**する：

| イベント | トリガ |
|-------|---------|
| `scan.completed` | スキャン完了 |
| `hallucination.detected` | 新規ハルシネーション検出（severity >= major） |
| `hallucination.resolved` | ハルシネーション収束 |
| `completion.threshold_crossed` | 完成度が 60% / 80% / 100% のしきい値を跨ぐ |
| `gbp.sync_failure` | GBP 同期が連続3回失敗 |

ペイロード形式：

```json
{
  "event": "scan.completed",
  "brand_id": "brd_01H...",
  "scan_id": "scn_01H...",
  "occurred_at": "2026-04-18T09:30:00Z",
  "data": { ... }
}
```

全 Webhook リクエストは `X-Baiyuan-Signature` ヘッダ（HMAC-SHA256）を持つ、顧客は処理前に検証すべきである。

---

## エラーコード

| HTTP | Code | 説明 |
|-----:|------|-------------|
| 400 | `invalid_parameter` | パラメータ形式または値が不正 |
| 401 | `unauthenticated` | トークン無し、または無効 |
| 403 | `forbidden` | トークン有効だがリソースアクセス権無し |
| 404 | `not_found` | リソース不存在 |
| 409 | `conflict` | 状態衝突（例：スキャン進行中、再トリガ不可） |
| 422 | `unprocessable_entity` | データ検証失敗 |
| 429 | `rate_limited` | クォータ超過 |
| 500 | `internal_error` | 内部エラー — 再試行、またはサポート連絡 |

---

**ナビゲーション**：[← 付録A：用語集](./appendix-a-glossary.md) · [📖 目次](../README.md) · [付録C：参考文献 →](./appendix-c-references.md)
