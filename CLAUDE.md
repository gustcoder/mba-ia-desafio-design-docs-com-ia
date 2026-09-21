# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository actually is

This is **not** a normal feature-development repo. It's the workspace for a documentation exercise
(course challenge, full spec in `README.md`): given a working Order Management System (`src/`, `prisma/`)
and a meeting transcript (`TRANSCRICAO.md`), produce a full design-doc package for a *proposed but
unimplemented* feature — a Webhook Notification System for order status changes.

**The deliverable is documentation only, in `docs/`.** `src/`, `prisma/`, `tests/`, and config files are
context/reference and **must not be modified**. The webhook feature does not exist in the code — do not
write it, scaffold it, or add stubs for it. Everything in the docs must trace back to either
`TRANSCRICAO.md` (a decision/requirement discussed live) or an actual path/symbol in the existing code.

### The document set and its "altitude"

Each doc operates at a different altitude — don't repeat content across them:

| Document | File | Altitude | Answers |
|---|---|---|---|
| PRD | `docs/PRD.md` | Product/business | Why and what? |
| RFC | `docs/RFC.md` | Architecture (concise, 2–4 pages) | How do we propose to solve it, what's still open? |
| ADRs | `docs/adrs/ADR-NNN-*.md` | Single decision | Why exactly this, vs. alternatives? |
| FDD | `docs/FDD.md` | Implementation | How to build it, in detail? |
| Tracker | `docs/TRACKER.md` | Cross-cutting | Where did this claim come from? |

The RFC proposes and opens for review; ADRs close individual decisions; the FDD is the deep implementation
spec. `docs/README.md`-style overlap between RFC and FDD detail level is treated as a defect.

Full mandatory section lists, acceptance checklist, and the ADR naming convention
(`ADR-NNN-titulo-em-kebab-case.md`, 5–8 files) are in the root `README.md` — read it before producing or
editing any doc. Note `docs/adrs/README.md` describes an older `NNNN-titulo` naming convention; the
governing spec for this challenge is the root `README.md`'s `ADR-NNN-titulo-em-kebab-case.md` format.

The six decisions the ADR set must cover (≥5 of 6): Outbox pattern on MySQL, retry/backoff/DLQ policy,
HMAC-SHA256 auth with per-endpoint secret, at-least-once delivery via `X-Event-Id`, separate-process
polling worker, and reuse of existing project patterns.

`docs/TRACKER.md` is the anti-hallucination mechanism: every requirement/decision/constraint in the other
docs should have a row with `Fonte = TRANSCRICAO` (`[hh:mm] Nome` timestamp) or `Fonte = CODIGO` (real file
path). If a claim can't be traced to either, it's probably invented — cut or fix it, don't force a row.

As of now, `docs/PRD.md`, `docs/RFC.md`, `docs/FDD.md`, `docs/TRACKER.md` are empty placeholders and
`docs/adrs/` has no ADRs yet — the package has not been produced.

### Producing the docs: use the project skills, not ad-hoc prompts

The build workflow is defined by the intent brief at `context/intent/docs-creator-v1.md`, which delegates
each document to a dedicated project skill under `.claude/skills/` (invoke with `/<skill-name>`). Only the
README is written manually — the other five documents each have a skill:

| Skill | Produces | Use it to... |
|---|---|---|
| `prd-creator` | `docs/PRD.md` | Cover the Webhook Notification System feature end-to-end (problem, público-alvo, objetivos/métricas, escopo, requisitos funcionais/não funcionais, decisões e trade-offs, dependências, riscos, critérios de aceitação, estratégia de testes). Requires an explicit "fora de escopo" listing ≥2 items discarded/deferred in the meeting. |
| `rfc-creator` | `docs/RFC.md` | Write the concise (2–4 page) architecture proposal: metadados, TL;DR, contexto, proposta técnica de alto nível, ≥2 alternativas reais discutidas e descartadas (com trade-off), ≥2 questões em aberto, impacto/riscos, links para os ADRs. Must not repeat FDD-level implementation detail. |
| `fdd-creator` | `docs/FDD.md` | Write the deep implementation spec: fluxos detalhados (outbox, worker, retry, DLQ), contratos HTTP públicos, matriz de erros `WEBHOOK_*`, estratégias de resiliência, observabilidade, dependências, critérios de aceite técnicos, riscos — plus the challenge-specific "Integração com o sistema existente" section naming ≥4 real code paths. |
| `adr-creator` | `docs/adrs/ADR-NNN-titulo-em-kebab-case.md` | Produce one ADR per architectural decision (Status, Contexto, Decisão, Alternativas Consideradas, Consequências). The intent brief asks for exactly 6 ADRs, one per decision below; the root `README.md` tolerates 5–8 — default to 6 unless told otherwise. |
| `tracker-creator` | `docs/TRACKER.md` | Build the traceability table (`ID \| Documento \| Tipo \| Conteúdo (resumo) \| Fonte \| Localização`) — the anti-hallucination check. Every row's `Fonte` must be `TRANSCRICAO` (with a `[hh:mm] Nome` timestamp) or `CODIGO` (with a real file path); ≥80% coverage of identifiable items across the other docs. |

The 6 decisions `adr-creator` must cover: Outbox pattern no MySQL, política de retry com backoff e DLQ,
autenticação HMAC-SHA256 com secret por endpoint, garantia at-least-once via `X-Event-Id`, worker em
processo separado em polling, reuso dos padrões existentes do projeto.

Cross-cutting rules from the intent brief that apply no matter which skill is running:
- Never treat ideas explicitly discarded in `TRANSCRICAO.md` as requirements.
- Never modify `src/`, `prisma/`, `tests/` — this is a documentation-only workflow.
- Avoid vague or filler content; back every claim with a concrete example.
- Read the transcript for underlying intent, not just its literal wording — don't copy-paste it into the docs.

## Commands

```bash
npm run dev          # tsx watch, loads .env, starts API on $PORT (default 3000)
npm run build         # tsc -p tsconfig.build.json -> dist/
npm start             # run built dist/server.js

npm run db:migrate    # prisma migrate dev (requires MySQL up)
npm run db:reset       # prisma migrate reset --force (drops + reseeds)
npm run db:seed        # tsx prisma/seed.ts

npm test               # vitest run (single pass)
npm run test:watch     # vitest watch mode
npx vitest run tests/orders.test.ts   # single test file
npx vitest run -t "test name substring"  # single test by name

npm run lint            # eslint . --ext .ts
npm run format           # prettier --write .

docker compose up -d    # start MySQL (see docker-compose.yml); copy .env.example to .env first
```

Tests hit a real MySQL database via Prisma (no mocking layer) — `tests/setup.ts` truncates the relevant
tables `beforeEach`. Vitest is configured with `singleFork: true` / `fileParallelism: false` because tests
share one database, so they must not run concurrently.

## Architecture of the existing application (context for the docs)

Express 4 + TypeScript (ESM, `NODE16`-style `.js` import extensions throughout) + Prisma/MySQL. No
framework beyond Express; dependency injection is manual and explicit.

**Layering per module** (`src/modules/{auth,users,customers,products,orders}/`): each module is
`*.routes.ts` → `*.controller.ts` → `*.service.ts` → `*.repository.ts`, plus `*.schemas.ts` (Zod). Routes
wire `validate(...)` and `authenticate`/`requireRole` middleware before the controller method; controllers
are thin (parse req → call service → send response); services hold business rules; repositories are the
only layer that touches `PrismaClient` directly (except `OrderService`, see below).

**Composition root**: `src/app.ts`'s `buildControllers(prisma)` manually `new`s every
repository/service/controller and wires them together; `buildApiRouter` (`src/routes/index.ts`) mounts each
module's router under `/api/v1/<module>`. `src/server.ts` just calls `buildApp` and handles
listen/shutdown. There is no IoC container — if the webhook feature needs a new module, it plugs in at
exactly these two spots.

**Order state machine** (`src/modules/orders/order.status.ts`): a transition table keyed by `OrderStatus`
enum (`PENDING → PAID → PROCESSING → SHIPPED → DELIVERED`, with `CANCELLED` reachable from the first three).
`canTransition`, `shouldDebitStock`, `shouldReplenishStock` are pure functions consulted by
`OrderService.changeStatus`.

**`OrderService.changeStatus`** (`src/modules/orders/order.service.ts`) is the load-bearing method for the
webhook feature (per the transcript, this is where the outbound event would be raised): it runs inside a
single `prisma.$transaction`, validates the transition, conditionally debits/replenishes stock, updates
`order.status`, and inserts an `OrderStatusHistory` audit row — all atomically. `OrderService` is also the
one service that takes `PrismaClient` directly (for `$transaction`) instead of going purely through its
repository, unlike the other modules.

**Errors**: `AppError` (`src/shared/errors/app-error.ts`) is the base — `statusCode` + `errorCode` string +
optional `details`. Concrete errors live in `src/shared/errors/http-errors.ts`
(`ValidationError`, `NotFoundError`, `ConflictError`, `UnprocessableEntityError`,
`InvalidStatusTransitionError`, `InsufficientStockError`, ...). The centralized
`src/middlewares/error.middleware.ts` catches `AppError` (uses its status/code/details), `ZodError`
(→ 400 `VALIDATION_ERROR`), known Prisma errors (`P2002` → 409, `P2025` → 404), and falls back to a logged
500. Any new `WEBHOOK_*` error codes should follow this same `AppError` subclass pattern.

**Auth**: `src/middlewares/auth.middleware.ts` — JWT bearer (`authenticate`) populates `req.user =
{id, email, role}`; `requireRole(...roles)` gates by `ADMIN | OPERATOR`. `src/config/env.ts` validates
process env with Zod at boot (fails fast on missing/invalid vars).

**Logging**: Pino (`src/shared/logger/index.ts`), structured JSON, wired into Express via
`src/middlewares/request-logger.middleware.ts` (assigns `req.id`, used in error logs).

**Data model** (`prisma/schema.prisma`): `User`, `Customer`, `Product`, `Order`, `OrderItem`,
`OrderStatusHistory`, `OrderNumberSequence` (integer sequence table for human-readable `ORD-NNNNNN`
numbers, reserved transactionally in `OrderService.reserveOrderNumber`). No outbox/event/webhook tables
exist yet — that's exactly the gap the docs need to design (Outbox pattern per the transcript).

There is intentionally no notification/event/queue/webhook mechanism anywhere in the codebase today.
