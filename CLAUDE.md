# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

This is a course assignment ("Da Reunião ao Documento: Design Docs Gerados por IA"). The deliverable is **documentation only** — a package of design docs (PRD, RFC, FDD, ADRs, Tracker) for a "Order Notification Webhooks" feature that does not exist yet in the code. The full assignment brief (objectives, required sections, acceptance criteria) is in `README.md` — read it before producing or editing any doc.

The repo contains a working Node.js/TypeScript Order Management System (OMS) that serves as the **factual source** the docs must stay grounded in, plus `TRANSCRICAO.md`, the literal transcript of the technical meeting where the webhook feature was decided (`[hh:mm] Speaker: ...` format, ~55 min, 5 participants: Larissa/tech lead, Marcos/PM, Bruno/eng pleno, Diego/eng sênior platform, Sofia/security).

**Hard constraint: do not modify `src/`, `prisma/`, `tests/`, or config files (`.env.example`, `tsconfig*.json`, `vitest.config.ts`, `.eslintrc.json`, `docker-compose.yml`, `package.json`, etc.).** The application code is context/reference only. The only files to create or edit are:

- `docs/PRD.md`, `docs/RFC.md`, `docs/FDD.md`, `docs/TRACKER.md`
- `docs/adrs/ADR-NNN-titulo-em-kebab-case.md` (5–8 files)
- `README.md` (root — gets fully replaced with the production-process writeup, per the structure in the current README's "6. README com o processo" section)

All of `docs/PRD.md`, `docs/RFC.md`, `docs/FDD.md`, `docs/TRACKER.md` are currently placeholder stubs (`<!-- documento a ser elaborado -->`); `docs/adrs/` has only a generic `README.md`, no ADRs yet.

## Non-negotiable rule while producing docs

Every requirement, decision, or constraint written into a doc must be traceable to either `TRANSCRICAO.md` (cite as `[hh:mm] Nome`) or an existing file in the codebase (cite as a file path). **Never invent requirements, numbers, or decisions.** If a claim can't be pinned to a transcript timestamp or a source file, it doesn't belong in the docs — this is what `docs/TRACKER.md` exists to enforce (≥80% of identifiable items need a tracker row; ≥70% of rows must cite `TRANSCRICAO` with a valid timestamp; ≥5 rows must cite `CODIGO` with a real path).

The transcript explicitly discards or defers some ideas raised in the meeting — those must **not** appear as requirements in the docs, and should instead show up in "Fora de escopo" (PRD) / "Questões em aberto" (RFC).

Suggested production order (from the README): ADRs first (they're the skeleton) → RFC → FDD → PRD last (highest-level, becomes a consolidation) → Tracker in parallel/at the end → README last.

## Architecture the docs must reference

Read these before writing FDD/ADRs — the feature (outbox pattern, webhook worker, HMAC auth, retry/DLQ) hooks into this existing code, and the FDD's mandatory "Integração com o sistema existente" section must name ≥4 real paths:

- **Order state machine**: `src/modules/orders/order.status.ts` — `canTransition`, `shouldDebitStock`, `shouldReplenishStock` define the `OrderStatus` transition graph (PENDING → PAID → PROCESSING → SHIPPED → DELIVERED, with CANCELLED branches). This is where the meeting's outbox-insert-on-status-change hook would attach.
- **Status change transaction**: `src/modules/orders/order.service.ts` — `changeStatus()` runs a single Prisma `$transaction` that updates `Order.status`, inserts into `OrderStatusHistory`, and debits/replenishes `Product.stockQuantity`. The meeting decided webhook event rows must be inserted in this same transaction (outbox pattern), never as a separate synchronous HTTP call.
- **Error model**: `src/shared/errors/app-error.ts` (base `AppError`: statusCode + errorCode + details) and `src/shared/errors/http-errors.ts` (concrete subclasses like `NotFoundError`, `ConflictError`, `UnprocessableEntityError`, `InvalidStatusTransitionError`, `InsufficientStockError`). New `WEBHOOK_*` error codes in the FDD should follow this same pattern.
- **Central error handling**: `src/middlewares/error.middleware.ts` — catches `AppError`, `ZodError`, and known Prisma errors, formats a uniform `{ error: { code, message, details? } }` JSON response.
- **Auth**: `src/middlewares/auth.middleware.ts` — `authenticate` (JWT bearer) and `requireRole(...roles)` (RBAC over `ADMIN`/`OPERATOR`). Relevant if webhook management endpoints need role gating.
- **Logging**: `src/shared/logger/index.ts` — Pino logger with `redact` paths for secrets (`*.password`, `*.token`, auth headers). Any webhook secret/signature logging must follow this redaction convention.
- **Schema/DB**: `prisma/schema.prisma` — MySQL via Prisma; `Order`, `OrderItem`, `OrderStatusHistory`, `OrderNumberSequence` models show the existing conventions (`@id @default(uuid()) @db.Char(36)`, `@@map` to snake_case tables, indexed status/timestamp columns) that a new `webhook_outbox`/webhook config tables should match.
- **Module layout**: each feature lives under `src/modules/<name>/` with `*.controller.ts`, `*.service.ts`, `*.repository.ts`, `*.routes.ts`, `*.schemas.ts` (Zod). `src/routes/index.ts` wires module routers together. A new webhooks module should follow this shape.
- **Request validation**: `src/middlewares/validate.middleware.ts` + per-module Zod schemas (e.g. `src/modules/orders/order.schemas.ts`).

## Commands

```bash
npm run dev          # tsx watch, loads .env
npm run build         # tsc -p tsconfig.build.json
npm start             # run compiled dist/server.js
npm run db:migrate    # prisma migrate dev
npm run db:seed       # tsx prisma/seed.ts
npm test              # vitest run (single run, not watch)
npm run test:watch    # vitest watch mode
npm run lint           # eslint . --ext .ts
npm run format          # prettier --write .
```

Run a single test file: `npx vitest run tests/orders.test.ts`.

Tests need a live MySQL (via `docker-compose up -d mysql`) — `tests/setup.ts` connects Prisma and truncates all tables `beforeEach`. `vitest.config.ts` forces `fileParallelism: false` / `singleFork: true`, so tests run sequentially against the same DB.

None of this should normally need to run for this assignment, since the deliverable doesn't touch `src/`/`prisma`/`tests/` — but use it to verify a claim in the docs (e.g. confirm a route, schema field, or error code actually exists) rather than trusting memory.
