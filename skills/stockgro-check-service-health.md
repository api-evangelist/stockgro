---
name: stockgro-check-service-health
description: >-
  Check whether StockGro's TradeView Intraday Model service is up and which of its
  backing dependencies are healthy, before attempting any other call against it.
api: stockgro-tradeview-intraday-model-api
generated: '2026-08-29'
method: generated
source: openapi/stockgro-tradeview-intraday-model-openapi.json
operations:
  - root__get
  - health_check_health_get
---

# Check StockGro TradeView service health

Use this before anything else. The StockGro TradeView Intraday Model API is an internal
platform service that is publicly reachable but **undocumented by StockGro** — there is no
developer portal, no support channel for it, and no status page. The only reliable
liveness signal is the service's own health endpoint.

Base URL: `https://api.stockgro.club` (from the overlay — the published spec declares no
`servers` block).

## Steps

1. **Confirm the service is answering.** Call `root__get` — `GET /`. A healthy service
   returns 200 with a JSON object; the observed body is
   `{"message":"Trade Processing API is running","environment":"prod"}`. Read
   `environment` to confirm you are talking to production.

2. **Read dependency health.** Call `health_check_health_get` — `GET /health`. The
   observed body is
   `{"status":"healthy","environment":"prod","services":{"kafka":"healthy","clickhouse":"healthy"}}`.
   Treat `status` as the overall gate and `services` as the per-dependency detail. The
   response schema is an open object (`additionalProperties: true`), so do not assume the
   `services` keys are fixed — enumerate whatever is present rather than indexing named
   keys.

3. **Stop if anything is degraded.** If `status` is not `healthy`, or any entry in
   `services` is not `healthy`, do not proceed to
   `process_trade_process_trade__post_id__post`. That operation has no documented undo and
   no idempotency protection, so a call made against a degraded backend cannot be safely
   retried or reversed.

## Conventions that apply

- **Auth:** none. Neither operation declares or requires a credential, and both answered
  anonymously on a live probe. See `authentication/stockgro-authentication.yml`.
- **Errors:** the envelope is `{"detail": ...}` with content-type `application/json`, not
  RFC 9457 `application/problem+json`. An unrouted path returns
  `{"detail":"Not Found"}`. See `errors/stockgro-problem-types.yml`.
- **Rate limits:** none published and none signaled — no `RateLimit-*` or `Retry-After`
  header appears on responses. Back off conservatively on your own schedule rather than
  reading one from the response. See `rate-limits/stockgro-rate-limits.yml`.
- **No 5xx is declared** anywhere in the contract. Handle transport and server failures
  defensively; the spec will not tell you their shape.
