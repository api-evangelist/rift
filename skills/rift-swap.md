---
name: Swap assets with Rift
description: Get a best-price quote and execute a cross-venue swap order through Rift's Router API, using an idempotency key so retries never duplicate an order.
api: openapi/rift-openapi.yml
operations: [getMarketQuote, createMarketOrder, checkRiftStatus, checkProviderStatus]
---

# Swap assets with Rift

Rift is a DEX aggregator that routes a swap across 20+ venues for best-price execution. Base URL: `https://router-gateway-v3-production.up.railway.app`. The API is in beta and documents no auth header on these endpoints.

## Steps

1. **(Optional) Confirm the service is up.** Call `checkRiftStatus` (`GET /health`) and `checkProviderStatus` (`GET /providers`). Proceed only when `status` is `ok` and the venues you need appear `reachable`.

2. **Get a quote.** Call `getMarketQuote` (`POST /quote`) with `from`, `to` (Chain.Symbol identifiers, e.g. `Ethereum.USDC` → `Bitcoin.BTC`), and `fromAmount`. Keep `amountFormat: readable` unless you are passing raw base units. Read back `quoteId`, `estimatedOut`, and `expiry`.

3. **Create the order before the quote expires.** Call `createMarketOrder` (`POST /order/market`) with the `quoteId`, `fromAddress`, `toAddress`, `refundAuthorizer`, and a **required** `idempotencyKey` (16-128 chars, pattern `^[A-Za-z0-9._:-]+$`). Generate one stable key per logical order and reuse it on every retry of that same order — this is how Rift dedupes.

4. **Handle failures.** A `409` on order creation usually means the quote expired or the idempotencyKey was reused for different parameters — re-quote (step 2) and retry with a fresh key. On `503`/`504`, back off and re-check health (step 1) before retrying. See `errors/rift-problem-types.yml`.

## Rules
- Never create an order without an `idempotencyKey`; retry the same order with the same key.
- Quotes expire — always check `expiry` and re-quote rather than reusing a stale `quoteId`.
- Errors are signaled by HTTP status only (no problem+json envelope).
