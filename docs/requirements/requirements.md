---
blueprint:
  type: requirements
  id: art_requirements
  version: 1
  status: in_review
  inputs:
    artifacts: []
  evidence:
    - "src/main.ts: global prefix api/v1, global ValidationPipe (whitelist, forbidNonWhitelisted, transform), PORT"
    - "src/mpesa/mpesa.controller.ts: POST /mpesa/stk-push and POST /mpesa/callback (with MpesaCallbackIdempotencyInterceptor)"
    - "src/mpesa/mpesa.service.ts: OAuth token caching, password/timestamp, phone normalisation, STK Push, callback processing, wallet resolution"
    - "src/mpesa/dto/initiateSTK.dto.ts: phoneNumber string, amount number >= 1"
    - "src/ledger/ledger.service.ts: recordTransaction balance check and atomic write"
    - "src/idempotency/idempotency.service.ts and src/idempotency/interceptor/*.ts: claim/complete with Redis TTLs"
    - "prisma/schema.prisma and prisma/migrations/*: User, Account, Transaction, LedgerEntry, enums, unique constraints"
    - "src/*-test/*.controller.ts: developer test endpoints wired into AppModule"
    - "docker-compose.yml: Postgres 16 and Redis 7 with healthchecks"
    - "README.md: stated goals (idempotency, double-entry ledger, M-Pesa, async processing, observability) and Day 0-4 progress log"
  guesses:
    - "Callers of POST /mpesa/stk-push are a trusted client (app or internal service); no authentication exists in code (see Q-04)"
    - "Users and their WALLET accounts are created out of band (e.g. Prisma Studio/seed), as no registration endpoint exists (see Q-08)"
    - "The *-test controllers are development scaffolding, not intended product features (see Q-05)"
    - "KES is the only currency and amounts are whole shillings at the M-Pesa boundary (see Q-07)"
---

# Requirements: fintech-core (as built)

fintech-core is a NestJS backend that takes customer deposits through Safaricom M-Pesa (Daraja STK Push) and records them in a double-entry ledger in PostgreSQL. Redis provides an idempotency guard and caches the Daraja access token. The README describes the project as an incremental "production-grade" portfolio backend. Day 4 (M-Pesa callback into the ledger) is the latest completed step. Reliable async processing (BullMQ), reconciliation and observability are listed in the README but not built yet.

All HTTP paths below sit under the global prefix `/api/v1`.

## Functional requirements

| ID | Requirement | Where in code |
|----|-------------|---------------|
| R-01 | `GET /` returns a fixed greeting string, which works as a basic liveness check. | `src/app.controller.ts:8`, `src/app.service.ts:5` |
| R-02 | Every route is served under `/api/v1`, and request bodies backed by DTO classes are validated: unknown fields are rejected and values are converted to the declared types. | `src/main.ts:8-16` |
| R-03 | A client can start an M-Pesa STK Push with `POST /mpesa/stk-push` by sending `phoneNumber` (a non-empty string) and `amount` (a number ≥ 1). | `src/mpesa/mpesa.controller.ts:11-14`, `src/mpesa/dto/initiateSTK.dto.ts` |
| R-04 | The phone number is normalised to `254[17]XXXXXXXX`: non-digits are removed and a leading `0` becomes `254`. Any other result is rejected with 400 before Daraja is called. | `src/mpesa/mpesa.service.ts:130-142` |
| R-05 | The system gets a Daraja OAuth token with HTTP Basic auth (consumer key and secret) and caches it in Redis under `mpesa:access_token` for 3300 s, so a cached token is reused instead of calling Daraja again. | `src/mpesa/mpesa.service.ts:57-101` |
| R-06 | The system builds the STK Push request: `Password = base64(shortcode + passkey + timestamp)`, `Timestamp = YYYYMMDDHHmmss`, `TransactionType = CustomerPayBillOnline`, `PartyA/PhoneNumber` = the normalised phone, `PartyB` = the shortcode, the configured `CallBackURL`, and `AccountReference = "fintech-core"`. | `src/mpesa/mpesa.service.ts:103-128,175-204` |
| R-07 | Daraja's synchronous STK response (MerchantRequestID, CheckoutRequestID, ResponseCode, …) is returned to the caller unchanged. A Daraja or network failure becomes a 503 that includes Daraja's error body. | `src/mpesa/mpesa.service.ts:206-225` |
| R-08 | The system receives Safaricom's asynchronous STK result on `POST /mpesa/callback` and acknowledges it with `{ResultCode: 0, ResultDesc: "Callback processed successfully"}`. | `src/mpesa/mpesa.controller.ts:16-21` |
| R-09 | Callbacks are de-duplicated on `Body.stkCallback.CheckoutRequestID`. If the ID is missing the callback gets 400. A duplicate that arrives while the first is still processing gets 409. A duplicate that arrives after the first completed gets the first response replayed. | `src/idempotency/interceptor/mpesaCallback.idempotency.interceptor.ts` |
| R-10 | A callback with `ResultCode != 0` (failed or cancelled payment) is logged and acknowledged. Nothing is written to the ledger and no Transaction row is created. | `src/mpesa/mpesa.service.ts:231-237` |
| R-11 | For a successful callback, `Amount`, `MpesaReceiptNumber` and `PhoneNumber` are read from `CallbackMetadata.Item`. If amount or phone is missing the callback gets 400. | `src/mpesa/mpesa.service.ts:239-249` |
| R-12 | The paying phone number is matched to a `User.phone_number` and then to that user's `WALLET` account. A 404 is returned if there is no such user or the user has no wallet. | `src/mpesa/mpesa.service.ts:152-173` |
| R-13 | A successful payment is recorded as a balanced Transaction: CREDIT the user's wallet and DEBIT the M-Pesa settlement account (`MPESA_SETTLEMENT_ACCOUNT_ID`) for the same amount. `checkoutRequestId`, `mpesaReceiptNumber` and `phoneNumber` are stored on the Transaction. | `src/mpesa/mpesa.service.ts:251-283` |
| R-14 | The ledger accepts only complete transactions: at least 2 entries, with sum(DEBIT) = sum(CREDIT) computed in `Prisma.Decimal`. Anything else is rejected with 400 before the database is touched. | `src/ledger/ledger.service.ts:22-49` |
| R-15 | A Transaction (always status `COMPLETED`) and all its LedgerEntry rows are written in one database transaction, so either all of them are saved or none are. | `src/ledger/ledger.service.ts:52-78` |
| R-16 | Account balances are never stored; they are derived from ledger entries. No balance query or endpoint exists yet. | `prisma/schema.prisma:46-58` (no balance column), `README.md:70-88` |
| R-17 | Any endpoint can opt into header-based idempotency: the `Idempotency-Key` header is required (400 if missing), with the same in-flight (409) and replay rules as R-09. | `src/idempotency/interceptor/idempotency.interceptor.ts` |
| R-18 | The idempotency store has two operations. `claim` does `SET idempotencyKey:<k> {processing} NX EX 30`. `complete` does `SET ... {completed, response} XX EX 86400`. If `complete` finds no existing key it fails with 404. | `src/idempotency/idempotency.service.ts:20-61` |
| R-19 | Domain data: a User has a unique phone number, a unique email and a password. An Account has a type (`WALLET`, `PLATFORM_FEE`, `MPESA_SETTLEMENT`) and an optional owning user. LedgerEntry amounts are `Decimal(12,2)` with a direction of `CREDIT` or `DEBIT`. | `prisma/schema.prisma:16-88` |
| R-20 | Developer test endpoints are part of the running app: `POST /idempotency-test/callback`, `POST /ledger-test/wanjiru-payment`, `POST /ledger-test/broken-payment`, `POST /mpesa-test/stk`, and `GET /mpesa-test/token` (which returns the Daraja access token). | `src/app.module.ts:19-24`, `src/*-test/*.controller.ts` |

## Non-functional requirements

| ID | Requirement | Where in code |
|----|-------------|---------------|
| R-21 | Money is held as `Decimal(12,2)` in the database and handled with `Prisma.Decimal` inside the ledger, never with JS floats. The edges still use JS `number` (the DTO `amount` and the callback `Amount`). | `prisma/schema.prisma:83`, `src/ledger/ledger.service.ts:34-43`, `src/mpesa/dto/initiateSTK.dto.ts:8-10` |
| R-22 | Unique constraints on `Transaction.checkoutRequestId` and `Transaction.mpesaReceiptNumber` back up the Redis guard at the database level, so the same M-Pesa payment cannot be recorded twice. | `prisma/migrations/20260715075442_add_mpesa_fields_to_transaction/migration.sql` |
| R-23 | Idempotency timing: an in-flight claim lasts 30 s; a completed result is remembered for 24 h. | `src/idempotency/idempotency.service.ts:27,51` |
| R-24 | All configuration comes from environment variables through the global `ConfigModule`: `DATABASE_URL`, `REDIS_URL`, `PORT`, `MPESA_TOKEN_URL`, `MPESA_CONSUMER_KEY`, `MPESA_CONSUMER_SECRET`, `MPESA_SHORTCODE`, `MPESA_PASSKEY`, `MPESA_STK_URL`, `MPESA_CALLBACK_URL`, `MPESA_SETTLEMENT_ACCOUNT_ID`. A missing `REDIS_URL` stops the app at startup; other missing values cause a 500 when first used. | `src/app.module.ts:15-17`, `src/idempotency/idempotency.service.ts:13-16`, `src/mpesa/mpesa.service.ts` |
| R-25 | Errors are mapped to HTTP statuses: upstream Daraja failures become 503, missing configuration becomes 500, bad input or an unbalanced ledger becomes 400, an unknown user or wallet becomes 404, and an in-flight duplicate becomes 409. | `src/mpesa/mpesa.service.ts`, `src/ledger/ledger.service.ts`, interceptors |
| R-26 | PostgreSQL is reached through Prisma 7 with the `@prisma/adapter-pg` driver adapter. The connection opens when the module starts and closes when it shuts down. | `src/prisma/prisma.service.ts`, `prisma.config.ts` |
| R-27 | Local infrastructure is Postgres 16 and Redis 7 under Docker Compose, with healthchecks and a named data volume. The API itself runs on the host. | `docker-compose.yml` |
| R-28 | Engineering tooling: TypeScript (strictNullChecks), ESLint, Prettier, and Jest for unit and e2e tests. The test suites are scaffold-level and out of date (see the NFR document). | `package.json`, `tsconfig.json`, `src/**/*.spec.ts`, `test/` |
| R-29 | Planned but not built: async processing with BullMQ (a dependency, never imported), reconciliation, observability, CI/CD. | `README.md:3-16,181-185`, `package.json:30` |
