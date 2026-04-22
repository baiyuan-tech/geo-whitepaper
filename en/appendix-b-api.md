---
title: "Appendix B — Public API Specification (Excerpt)"
description: "Public REST endpoints of Baiyuan GEO Platform for third-party integration, research, and audit reference. Internal admin endpoints not included."
appendix: B
part: 6
lang: en
authors:
  - name: Vincent Lin
    affiliation: Baiyuan Technology
license: CC-BY-NC-4.0
last_updated: 2026-04-18
last_modified_at: '2026-04-21T06:57:15Z'
---













# Appendix B — Public API Specification (Excerpt)

This appendix excerpts the **public-facing** REST endpoints of Baiyuan GEO Platform, for third-party integration, research, and audit reference. Internal admin endpoints (admin, monitoring, billing) are not included.

## Common conventions

### Base URL

```text
https://api.geo.baiyuan.io/v1
```

### Authentication

All endpoints require an `Authorization: Bearer <token>` header. Tokens are obtained from the customer panel's *"API key management"* page.

### Response envelope

Every response has the form:

```json
{
  "status": "success" | "error",
  "data":   { ... },           // when status=success
  "error":  "description"      // when status=error
}
```

### Rate limit

- Default 60 req/min; raisable on enterprise plans
- Exceeded → HTTP 429 with `Retry-After` header indicating wait seconds

---

## 1. Brands

### GET `/brands`

List brands visible to the current account.

**Query params**:

- `limit` (1–100, default 20)
- `offset` (default 0)
- `industry_code` (optional; see the 25-category enum in [§7.2](./ch07-schema-org.md#72-industry-specialized-type-across-25-categories))

**Response**:

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

Get a single brand's details.

### POST `/brands`

Create a new brand. **Body**: `name` (required), `url`, `industry_code`, `description`, etc.

### PUT `/brands/:id`

Update brand. Body semantics depend on `Content-Type`; merge-patch supported.

---

## 2. Scans

### POST `/scans`

Trigger a scan immediately (not scheduled).

**Body**:

```json
{
  "brand_id": "brd_01H...",
  "platforms": ["chatgpt", "claude", "perplexity"],
  "intent_count": 20,
  "scan_layer": "full"
}
```

**Response** (202 Accepted):

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

Query scan status.

**Response**:

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

Get scan results.

**Response** (abbreviated):

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

## 3. Entity (Schema.org brand entity)

### GET `/brands/:id/entity`

Get a brand's complete Schema.org data (Organization, Services, Employees, Hours, Location, Social Links).

### GET `/brands/:id/entity/schema.json`

Get the generated Schema.org JSON-LD (embeddable directly into the customer's website).

**Response**: `Content-Type: application/ld+json`; body is `{"@context": "https://schema.org", "@graph": [...]}`.

### GET `/brands/:id/entity/completion`

Get completeness score.

**Response**:

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

## 4. AXP (shadow document)

### GET `/axp?host=<domain>&path=<path>`

This endpoint is **called by the Cloudflare Worker**; integrators do not need to call it directly. Returns the shadow document HTML for the given brand and path.

---

## 5. Hallucinations

### GET `/brands/:id/hallucinations`

List detected hallucinations.

**Query params**: `severity` (critical/major/minor/info), `status` (open/resolving/resolved).

### POST `/brands/:id/hallucinations/:hid/repair`

Trigger remediation flow.

---

## 6. Baseline (Phase baseline testing)

### POST `/brands/:id/baseline`

Create a new Phase baseline cohort.

### POST `/brands/:id/baseline/:cohortId/run`

Re-run an existing baseline.

### GET `/brands/:id/baseline/:cohortId/compare`

Get Phase 1 vs Phase 2 / 3 comparison.

---

## 7. Webhooks (customer-receivable)

Baiyuan platform **pushes** the following events to customer-configured webhook URLs (set in the panel):

| Event | Trigger |
|-------|---------|
| `scan.completed` | A scan finished |
| `hallucination.detected` | A new hallucination detected (severity >= major) |
| `hallucination.resolved` | A hallucination converged |
| `completion.threshold_crossed` | Completeness crossed a 60% / 80% / 100% threshold |
| `gbp.sync_failure` | GBP sync failed 3 times consecutively |

Payload shape:

```json
{
  "event": "scan.completed",
  "brand_id": "brd_01H...",
  "scan_id": "scn_01H...",
  "occurred_at": "2026-04-18T09:30:00Z",
  "data": { ... }
}
```

Every webhook request carries an `X-Baiyuan-Signature` header (HMAC-SHA256); customers should verify before processing.

---

## Error codes

| HTTP | Code | Description |
|-----:|------|-------------|
| 400 | `invalid_parameter` | Parameter format or value invalid |
| 401 | `unauthenticated` | No token or token invalid |
| 403 | `forbidden` | Token valid but no resource access |
| 404 | `not_found` | Resource does not exist |
| 409 | `conflict` | State conflict (e.g., scan in progress, cannot re-trigger) |
| 422 | `unprocessable_entity` | Data validation failed |
| 429 | `rate_limited` | Quota exceeded |
| 500 | `internal_error` | Internal error — retry or contact support |

---

**Navigation**: [← Appendix A: Glossary](./appendix-a-glossary.md) · [📖 Index](../README.md) · [Appendix C: References →](./appendix-c-references.md)
