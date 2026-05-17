# VERSIONING — Smart Campus Access Gate API

## Contract Version History

| Version | Date | Author | Summary |
|---|---|---|---|
| 1.0.0 | 2026-05-17 | Đỗ Tuyến | Initial contract — Pair 03 Core Business ↔ Access Gate |

---

## v1.0.0 — 2026-05-17

### Added
- `GET /health` — health check endpoint (no auth required)
- `GET /access/logs/recent` — cursor-based paginated access log list with filters (gateId, cardId, direction, fromTime, toTime)
- `GET /access/logs/{logId}` — get single access log detail by UUID
- `GET /gates/{gateId}/status` — get current gate operational state (OPEN/CLOSED/FAULT)
- `GET /cards/{cardId}` — get RFID card info with polymorphic `cardStatus` (oneOf: ACTIVE/SUSPENDED/BLOCKED)

### Schema Design Decisions
- `cardStatus` uses `oneOf` + `discriminator` (statusCode) to represent polymorphic card states
- `operatorNote`, `faultReason`, `expiresAt` use `type: [string, "null"]` (OpenAPI 3.1 union type)
- All error responses use `Problem Details` (RFC 7807) via `application/problem+json`
- `additionalProperties: false` applied to all major object schemas

### Negotiation Outcomes (v1.0)
- Cursor-based pagination (not offset) agreed for `/access/logs/recent`
- Field named `direction` (not `accessDirection`) per Consumer preference
- `operatorNote` always present in response (even as null) — not absent
- Log retention: 90 days minimum; out-of-range query returns `200 + items: []`
- Polling interval: minimum 60s for scheduled polling of access logs
