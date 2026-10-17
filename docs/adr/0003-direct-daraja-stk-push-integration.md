---
blueprint:
  type: adr
  id: art_0003-direct-daraja-stk-push-integration
  version: 1
  status: in_review
  inputs:
    artifacts:
      - { id: art_requirements, version: 1 }
      - { id: art_c4-context, version: 1 }
  evidence:
    - "src/mpesa/mpesa.service.ts: axios calls to MPESA_TOKEN_URL and MPESA_STK_URL; token cached in Redis 'mpesa:access_token' EX 3300; password/timestamp built in-house; normalizePhone; processCallback posts to ledger"
    - "src/mpesa/mpesa.controller.ts: synchronous stk-push endpoint and callback endpoint"
    - "package.json: axios and ioredis dependencies, no M-Pesa SDK"
    - "README.md Day 3: rationale for token caching with a safety-margin TTL, service-owned phone normalisation, testing against the real sandbox"
  guesses:
    - "A third-party aggregator or SDK was rejected to keep full control and learning value; the code does not say so (see Q-04)"
    - "Synchronous processing in the request is a temporary step until BullMQ-based async processing is added, as the README lists it as a goal (see Q-09)"
---

# ADR 0003: Integrate Safaricom Daraja STK Push directly over HTTP, with a Redis-cached OAuth token and synchronous callback processing

Status: accepted (reconstructed from code)

## Context

Customers pay in with M-Pesa. Daraja's Lipa na M-Pesa Online (STK Push) works in two steps: an authenticated request that triggers a PIN prompt, then an asynchronous HTTP callback with the result. Daraja OAuth tokens last about an hour, and token generation is rate-limited. Daraja is strict about formats: the password is `base64(shortcode+passkey+timestamp)`, the timestamp is `YYYYMMDDHHmmss`, and phone numbers must look like `2547XXXXXXXX`.

## Options considered

1. **A payment aggregator** (for example a PSP that wraps M-Pesa). Less integration work, but it adds fees and another dependency.
2. **A community M-Pesa SDK.** Less code, but it hides behaviour and is a supply-chain dependency.
3. **A direct Daraja integration** using `axios`, with request building, token caching and callback parsing written in this codebase.
4. On the callback: process it **synchronously** in the HTTP request, or **acknowledge and enqueue** (BullMQ) for a worker.

## Decision

Option 3 with synchronous callback processing.
- `MpesaService.getAccessToken` caches the token in Redis (`mpesa:access_token`, TTL 3300 s, a safety margin under the 3600 s lifetime).
- `initiateSTKPush` normalises the phone in the service (the DTO checks only the basic shape), builds a `CustomerPayBillOnline` request and returns Daraja's response unchanged. Daraja failures become 503.
- `POST /mpesa/callback` is guarded by the callback idempotency interceptor (ADR 0002). The handler resolves the wallet from the phone number and calls `LedgerService.recordTransaction` (ADR 0001) before responding `{ResultCode:0}`.
- All endpoints, keys and secrets come from environment variables.

## Consequences

- **Positive**
  - Full control over Daraja behaviour, with no extra fees or SDK dependency.
  - Fewer token calls.
  - Phone format rules live in one place.
  - The README says the flow was proven end to end against the real sandbox.
- **Negative / risks**
  - **The callback is unauthenticated, and the body is trusted for amount and phone.** Nothing confirms the callback came from Safaricom or matches an initiated push (Q-01, Q-02).
  - Callback work (a DB lookup and a ledger write) runs inside Safaricom's HTTP request. Slow DB or Redis means slow acknowledgements, and errors return non-2xx codes. Any retry behaviour then depends on Safaricom (Q-10).
  - The timestamp is generated twice: once inside `generatePassword()` and once for the `Timestamp` field. If the two calls fall in different seconds, the password and timestamp disagree and Daraja rejects the request. The timestamp also uses the server's local time zone.
  - `axios` calls have no timeout or retry.
  - Daraja error bodies are echoed to API clients in the 503 message.
  - The token cache uses one fixed key per Redis, which assumes one Daraja app (credential set) per Redis instance.
  - `AccountReference`/`TransactionDesc` are hard-coded (`"fintech-core"`, `"txndesc"`).
  - The DTO `amount` is a JS number and fractional values are allowed (Q-07).
