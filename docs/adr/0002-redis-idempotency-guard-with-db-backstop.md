---
blueprint:
  type: adr
  id: art_0002-redis-idempotency-guard-with-db-backstop
  version: 1
  status: in_review
  inputs:
    artifacts:
      - { id: art_requirements, version: 1 }
      - { id: art_c4-container, version: 1 }
  evidence:
    - "src/idempotency/idempotency.service.ts: claim() SET NX EX 30 {processing}; complete() SET XX EX 86400 {completed, response}"
    - "src/idempotency/interceptor/idempotency.interceptor.ts: key from Idempotency-Key header; 409 on processing; replay on completed"
    - "src/idempotency/interceptor/mpesaCallback.idempotency.interceptor.ts: same logic keyed on Body.stkCallback.CheckoutRequestID"
    - "prisma/migrations/20260715075442_add_mpesa_fields_to_transaction/migration.sql: unique indexes on checkoutRequestId and mpesaReceiptNumber"
    - "README.md Day 1 and Day 4: reasoning about TTLs, crash mid-request, two-Redis risk, DB constraint as backstop"
  guesses:
    - "A single Redis instance is the intended deployment, as the README itself names multiple independent Redis boxes as a double-processing risk (see Q-09)"
    - "The 30 s processing TTL is chosen to exceed the worst-case handler duration; no measurement backs it (see Q-10)"
---

# ADR 0002: Redis claim/complete idempotency guard as a NestJS interceptor, backed by DB unique constraints

Status: accepted (reconstructed from code)

## Context

Payment callbacks and retried client requests can arrive more than once, and sometimes at the same moment. Processing a duplicate means crediting or debiting a customer twice. M-Pesa STK callbacks carry no idempotency header, but every callback includes Daraja's `CheckoutRequestID`. The guard should be reusable across endpoints and should not clutter the business handlers.

## Options considered

1. **DB-only de-duplication** with a unique constraint on the external reference. It is strongly consistent, but concurrent duplicates still run the handler and fail late with a constraint error, and no cached response can be replayed.
2. **An in-memory lock or map.** Fails once there is more than one process and is lost on restart.
3. **A Redis claim/complete guard**: an atomic `SET NX` claim with a short TTL, replaced by a completed record with the response and a long TTL. Implemented as a NestJS interceptor, so handlers stay free of it.
4. **A dedicated idempotency table** in Postgres, in the same DB transaction as the ledger write. Exactly-once with respect to the ledger, but more complex.

## Decision

Option 3, with option 1 as a backstop.
- `IdempotencyService.claim(key)` runs `SET idempotencyKey:<key> {"status":"processing"} NX EX 30`.
- `complete(key, response)` runs `SET … {"status":"completed","response":…} XX EX 86400`.
- There are two interceptors with the same state machine and different key sources:
  - `IdempotencyInterceptor` reads the `Idempotency-Key` header and is used by the test endpoint.
  - `MpesaCallbackIdempotencyInterceptor` reads `Body.stkCallback.CheckoutRequestID` and is used on `/mpesa/callback`.
- A missing key gives 400. A key that is still processing gives 409. A completed key replays the stored response.
- `Transaction.checkoutRequestId` and `mpesaReceiptNumber` are `UNIQUE` in Postgres as the final guard.

## Consequences

- **Positive**
  - Concurrent duplicates are rejected cheaply before any business logic runs, and completed duplicates get the same answer.
  - The README says a real duplicate Safaricom callback was tested live and produced exactly 1 Transaction.
  - If Redis is wrong (TTL expired, data lost, split instances), the DB constraint still prevents a double ledger write.
- **Negative / risks**
  - The Redis claim and the DB commit are not atomic, so correctness ultimately depends on the DB constraint. A constraint violation (Prisma `P2002`) is not handled and surfaces as 500.
  - **The claim is not released when the handler fails.** The key stays `processing` for 30 s (retries get 409) and then a retry is treated as new (Q-10).
  - If a handler runs longer than 30 s, the claim expires mid-flight: a duplicate could start, and `complete()` then fails its `XX` condition (logged only).
  - `complete()` is fire-and-forget inside `tap`, so there is a short window where a duplicate arriving just after success gets 409 instead of the replay.
  - The two interceptors duplicate their logic. They differ only in how they extract the key, so a shared base class or key-extractor would be clearer.
  - Redis becomes a hard dependency of the callback path: if Redis is down, callbacks get 500.
  - The replay cache holds whole response bodies for 24 h.
