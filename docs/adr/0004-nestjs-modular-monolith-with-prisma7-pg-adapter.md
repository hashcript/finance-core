---
blueprint:
  type: adr
  id: art_0004-nestjs-modular-monolith-with-prisma7-pg-adapter
  version: 1
  status: in_review
  inputs:
    artifacts:
      - { id: art_c4-container, version: 1 }
  evidence:
    - "src/app.module.ts: one NestJS app composed of feature modules (Idempotency, Ledger, Mpesa, Prisma) and test modules"
    - "src/prisma/prisma.service.ts: PrismaService extends PrismaClient constructed with new PrismaPg({ connectionString: DATABASE_URL })"
    - "prisma/schema.prisma: generator 'prisma-client' output ../generated/prisma, moduleFormat cjs; datasource without url"
    - "prisma.config.ts: CLI datasource url from DATABASE_URL via dotenv"
    - "docker-compose.yml: Postgres 16 + Redis 7 for local dev"
    - "README.md Day 0 and Day 2: Prisma 7 split configuration and driver-adapter rationale"
  guesses:
    - "A single deployable is intended for the foreseeable future; no service boundaries or messaging between modules exist (see Q-09)"
---

# ADR 0004: NestJS modular monolith on PostgreSQL, accessed through Prisma 7 with the `pg` driver adapter

Status: accepted (reconstructed from code)

## Context

The project is built one focused module at a time (idempotency, ledger, M-Pesa, and later async processing and observability). It needs dependency injection, request interceptors for cross-cutting concerns such as idempotency, DTO validation, and a relational store with real transactions for the ledger. Prisma 7 changed its configuration model: the schema no longer carries the connection URL, and the runtime client needs a driver adapter.

## Options considered

1. **Separate services** for each concern (payments, ledger). Gives isolation, but brings distributed-transaction and operational cost far too early.
2. **A NestJS modular monolith**: one process with feature modules wired through DI, global `ConfigModule` and `PrismaModule`, and interceptors and pipes for cross-cutting concerns.
3. Data access choices: **Prisma** (typed client and migrations), a lighter query builder, or raw `pg`.
4. Prisma 7 connection: the **built-in engine** or the **`@prisma/adapter-pg` driver adapter**.

## Decision

Option 2 with Prisma.
- One NestJS 11 application. `AppModule` imports the feature modules; `PrismaModule` and `ConfigModule` are global.
- A global `ValidationPipe` (`whitelist`, `forbidNonWhitelisted`, `transform`) and the `/api/v1` prefix are set in `main.ts`.
- `PrismaService` extends `PrismaClient` and is built with `new PrismaPg({ connectionString: process.env.DATABASE_URL })`. It connects when the module starts and disconnects when it shuts down.
- The client is generated into `generated/prisma` (CJS, git-ignored) and imported as `generated/prisma/client` through `baseUrl`.
- Migrations run through Prisma Migrate, configured in `prisma.config.ts`.
- PostgreSQL 16 and Redis 7 run locally under Docker Compose.

## Consequences

- **Positive**
  - Simple to run and reason about, and DB transactions span the whole ledger write.
  - Cross-cutting concerns (idempotency, validation) are declarative.
  - Typed queries and versioned migrations.
- **Negative / risks**
  - Every module shares one process and one failure domain, and there is no async worker (BullMQ is a dependency but unused).
  - `PrismaService` reads `process.env` directly instead of `ConfigService`, which is inconsistent with the other services and harder to test.
  - Redis clients are created ad hoc inside services, with no shared provider and no shutdown hook.
  - `forbidNonWhitelisted` applies only to class DTOs. The callback body is typed as an interface (`StkCallbackBody`), so it is not validated at all.
  - The test controllers are part of the production module graph (Q-05).
  - The generated client must exist before build or test (`prisma generate`), and absolute `src/...` imports depend on the `baseUrl` setting.
  - The existing Jest specs do not provide the DI dependencies (PrismaService, ConfigService), so the scaffolded unit tests will fail. One spec uses `node:test` inside Jest.
