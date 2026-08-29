---
name: stockgro-process-trade-post
description: >-
  Trigger scoring of a single StockGro trade post by id through the TradeView Intraday
  Model API, with the safety checks its contract does not enforce.
api: stockgro-tradeview-intraday-model-api
generated: '2026-08-29'
method: generated
source: openapi/stockgro-tradeview-intraday-model-openapi.json
operations:
  - health_check_health_get
  - process_trade_process_trade__post_id__post
---

# Process a StockGro trade post

`process_trade_process_trade__post_id__post` — `POST /process/trade/{post_id}` — manually
triggers processing of a trade post. Per the operation's own description it dispatches
FastAPI background tasks and returns a processing status.

**Read this before calling it.** This is the only write-shaped operation in the contract
and it is the least safe one to automate:

- It has **no idempotency mechanism**. There is no `Idempotency-Key` header or equivalent,
  so a retry re-triggers processing rather than returning the first result.
- It has **no documented reversal** and **no stated window**. No cancel, undo, or revert
  counterpart exists in the contract, and StockGro publishes nothing saying whether
  processing can be taken back. Treat the call as irreversible.
- It is **undocumented by the provider**. StockGro publishes no developer documentation
  for this service, so the downstream effect of "processing" is not stated anywhere
  public.

Do not call it speculatively, in a retry loop, or across a range of ids.

## Steps

1. **Pre-check health.** Run the `stockgro-check-service-health` skill first. If `status`
   or any dependency is not `healthy`, stop — an unreversible call against a degraded
   ClickHouse or Kafka is exactly the failure you cannot undo.

2. **Have a real `post_id`.** The path parameter is a required `string` and is the
   operation's only input. The contract does **not** describe the trade-post entity, its
   id format, or any prefix, and there is no list or lookup operation to discover ids
   from — see `data-model/stockgro-data-model.yml`. Obtain the id from the caller; never
   generate, guess, or enumerate one.

3. **Call it once.** `POST /process/trade/{post_id}` with no body and no credential. On
   success expect 200 with an open JSON object (`additionalProperties: true`) carrying the
   processing status; read the returned keys rather than assuming a fixed shape.

4. **Handle 422 without retrying blindly.** A validation failure returns the
   `HTTPValidationError` envelope: `detail` is an array whose entries carry `loc` (the
   path to the offending input), `msg`, and `type`. Read `loc` to see which parameter was
   rejected, fix it, and only then call again. A 422 means the request never took effect,
   so a corrected retry is safe.

5. **Do not auto-retry any other failure.** Because the operation is neither idempotent nor
   reversible, a timeout or 5xx leaves you unable to tell whether processing started.
   Surface the ambiguity to the caller rather than retrying.

## Conventions that apply

- **Auth:** none declared or required — see `authentication/stockgro-authentication.yml`.
- **Errors:** `{"detail": ...}`, `application/json`, not RFC 9457 — see
  `errors/stockgro-problem-types.yml`.
- **Reversibility and idempotency:** both recorded as absent in
  `conventions/stockgro-conventions.yml`.
