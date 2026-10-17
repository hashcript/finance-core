---
blueprint:
  type: sequence
  id: art_seq-mpesa-stk-deposit
  version: 1
  status: in_review
  inputs:
    artifacts:
      - { id: art_c4-container, version: 1 }
  evidence:
    - "src/mpesa/mpesa.controller.ts: initiateStkPush and handleCallback (with MpesaCallbackIdempotencyInterceptor)"
    - "src/mpesa/mpesa.service.ts: getAccessToken (Redis cache 3300s), generatePassword/generateTimestamp, normalizePhone, initiateSTKPush, processCallback, resolveWalletAccountId"
    - "src/idempotency/interceptor/mpesaCallback.idempotency.interceptor.ts and src/idempotency/idempotency.service.ts: claim (NX EX 30) / complete (XX EX 86400)"
    - "src/ledger/ledger.service.ts: balance check then prisma.$transaction creating Transaction + LedgerEntry rows"
    - "README.md Day 4: verified live that a resent callback yields exactly 1 Transaction and 2 LedgerEntry rows"
  guesses:
    - "Safaricom retries a callback when it gets a non-2xx response; the README says so but the code cannot confirm it (see Q-10)"
    - "Nothing links the initiation response to the later callback inside fintech-core; the client is assumed to keep the CheckoutRequestID if it needs to (see Q-02)"
---

# Sequence: M-Pesa STK Push deposit (initiate → callback → ledger)

```mermaid
sequenceDiagram
    autonumber
    actor C as API client
    participant MC as MpesaController
    participant MS as MpesaService
    participant R as Redis
    participant D as Safaricom Daraja
    actor P as Customer phone
    participant IC as MpesaCallbackIdempotencyInterceptor
    participant IS as IdempotencyService
    participant LS as LedgerService
    participant DB as PostgreSQL

    rect rgb(240,245,255)
    note over C,D: Phase 1: initiate (synchronous)
    C->>MC: POST /api/v1/mpesa/stk-push {phoneNumber, amount}
    MC->>MC: ValidationPipe(InitiateSTKDto): string, number ≥ 1
    MC->>MS: initiateSTKPush(dto)
    MS->>R: GET mpesa:access_token
    alt cache miss
        MS->>D: GET MPESA_TOKEN_URL (Basic key:secret)
        D-->>MS: {access_token}
        MS->>R: SET mpesa:access_token EX 3300
    end
    MS->>MS: password = b64(shortcode+passkey+ts), timestamp, normalizePhone → 2547XXXXXXXX (else 400)
    MS->>D: POST MPESA_STK_URL (Bearer) {CustomerPayBillOnline, Amount, PartyA, CallBackURL…}
    alt Daraja error / network failure
        D-->>MS: 4xx/5xx
        MS-->>C: 503 "Failed to initiate STK Push: …"
    else accepted
        D-->>MS: {MerchantRequestID, CheckoutRequestID, ResponseCode…}
        MS-->>C: 201 Daraja response (nothing persisted)
    end
    end

    D-)P: STK PIN prompt
    P-)D: Customer enters PIN (or cancels / times out)

    rect rgb(240,255,240)
    note over D,DB: Phase 2: asynchronous result callback
    D->>MC: POST /api/v1/mpesa/callback {Body.stkCallback…} (unauthenticated)
    MC->>IC: intercept()
    IC->>IC: key = Body.stkCallback.CheckoutRequestID (missing → 400)
    IC->>IS: claim(key)
    IS->>R: SET idempotencyKey:key {processing} NX EX 30
    alt key already exists
        IS->>R: GET idempotencyKey:key
        alt status = processing
            IC-->>D: 409 Conflict
        else status = completed
            IC-->>D: 201 cached response (replay, no side effects)
        end
    else claimed
        IC->>MC: next.handle()
        MC->>MS: processCallback(body)
        alt ResultCode ≠ 0
            MS->>MS: console.log failure, no DB writes
        else ResultCode = 0
            MS->>MS: read Amount, MpesaReceiptNumber, PhoneNumber (missing → 400)
            MS->>DB: findUnique User by phone_number, include accounts
            alt no user / no WALLET
                MS-->>D: 404 (key stays "processing" until 30 s TTL)
            else wallet found
                MS->>LS: recordTransaction([CR wallet amt, DR settlement amt], {checkoutRequestId, receipt, phone})
                LS->>LS: ≥2 entries and Σ DR = Σ CR (Decimal), else 400
                LS->>DB: BEGIN; INSERT Transaction(COMPLETED, refs); INSERT LedgerEntry ×2; COMMIT
                Note right of DB: unique(checkoutRequestId, mpesaReceiptNumber)<br/>is the backstop for duplicates
                LS-->>MS: {transaction, ledgerEntries}
            end
        end
        MC-->>IC: {ResultCode: 0, ResultDesc: "Callback processed successfully"}
        IC--)IS: complete(key, response) (fire-and-forget in tap)
        IS--)R: SET idempotencyKey:key {completed, response} XX EX 86400
        IC-->>D: 201 {ResultCode: 0, …}
    end
    end
```

## Notes

- **Why this flow.** It is the only path that moves money into the ledger, and it exercises all three core modules (M-Pesa, idempotency, ledger) plus every external dependency.
- **Status codes.** NestJS returns 201 for `@Post` handlers by default, so Safaricom receives 201 rather than 200.
- **No correlation between the two phases.** Phase 1 stores nothing. Phase 2 trusts the callback's `Amount` and `PhoneNumber` completely and never checks them against the request that started the push (Q-02). The two phases are linked only by Daraja's CheckoutRequestID.
- **Failure handling.**
  - Exceptions in Phase 2 (400/404/500) leave the Redis key in `processing`. Retries within 30 s get 409; after that, a retry is processed as new (Q-10).
  - A duplicate that reaches the DB after the Redis record has expired (more than 24 h later) hits the unique constraint. That produces an unmapped 500, but no double credit.
  - `complete()` runs after the handler's value is emitted and is not awaited. In the short gap before it finishes, a duplicate gets 409 rather than the replayed response.
- **Atomicity.** The Transaction and its two entries are written in a single Prisma interactive transaction. The Redis claim and the DB write are not atomic with each other; the DB unique constraint covers that gap.
