---
blueprint:
  type: adr
  id: art_0001-double-entry-ledger-with-derived-balances
  version: 1
  status: in_review
  inputs:
    artifacts:
      - { id: art_requirements, version: 1 }
      - { id: art_domain, version: 1 }
  evidence:
    - "prisma/schema.prisma: Account has no balance column; LedgerEntry(amount Decimal(12,2), entryType CREDIT|DEBIT) linked to Account and Transaction"
    - "src/ledger/ledger.service.ts: recordTransaction is the only write path, validates ≥2 entries and Σ DEBIT = Σ CREDIT with Prisma.Decimal, writes in one $transaction"
    - "README.md Day 2: explicit rationale for no mutable balance column, single entry point, Decimal arithmetic, atomic writes"
  guesses:
    - "Balance reads will be computed by aggregating LedgerEntry per account (no implementation exists); snapshotting is deferred until performance requires it (see Q-06)"
---

# ADR 0001: Double-entry ledger with derived balances and a single balanced write path

Status: accepted (reconstructed from code)

## Context

The system holds customer money in wallets that are funded from M-Pesa. Mutable balance columns can drift away from transaction history: a failed partial write, a race between two updates, or a bug that changes one side only. A single payment can also touch several accounts with different amounts, for example a customer wallet, a platform fee account and the M-Pesa settlement account. The README states that preventing "silent data corruption" is a core goal.

## Options considered

1. **A mutable `Account.balance` column**, updated with each payment. It is simple and fast to read, but the balance and the history can disagree, and the column needs careful locking.
2. **Single-entry transaction rows** (one row with an amount and from/to accounts). Easier than option 3, but it cannot naturally express one payment that splits across three or more accounts.
3. **A double-entry ledger** where `Transaction` is a grouping container, `LedgerEntry` holds one row per account touched (amount and DEBIT/CREDIT), and balances are always derived.
4. **Option 3 plus per-entry balance snapshots.** The README discusses this as a future read optimisation.

## Decision

Option 3.
- `Account` stores identity and type only (`WALLET`, `PLATFORM_FEE`, `MPESA_SETTLEMENT`).
- `Transaction` has no amount of its own.
- `LedgerEntry` is the single source of truth.
- `LedgerService.recordTransaction(entries, metadata)` is the only way to write. It takes the complete set of entries and rejects the set unless there are at least 2 entries and the sum of DEBIT equals the sum of CREDIT. The sums are computed in `Prisma.Decimal`, and amounts are passed as strings. It then writes the Transaction and all its entries inside one `prisma.$transaction`.
- Amounts are stored as `Decimal(12,2)`.

## Consequences

- **Positive**
  - The books are self-auditing, and unbalanced transactions cannot be written through the service.
  - Writes are atomic, so a Transaction is never left without its entries.
  - The model supports multi-leg transactions such as fee splits.
- **Negative / open**
  - **No balance read exists yet.** Computing one means aggregating every entry for the account, and the sign convention per account type is not defined (Q-06).
  - The balance check happens before the DB write. An amount with more than 2 decimal places passes the check exactly but is then rounded by `Decimal(12,2)`, which could leave the stored entries unbalanced. Amounts are also not checked to be positive.
  - Nothing at the DB level enforces balance: no constraint or trigger checks that a transaction's entries sum to zero. Writes that bypass `LedgerService` (Prisma Studio, raw SQL) are not protected.
  - `Transaction.status` is always `COMPLETED`, so the `PENDING`/`FAILED` lifecycle the enum suggests is unused (Q-02).
  - There is no currency column, which implies a single currency (Q-07).
