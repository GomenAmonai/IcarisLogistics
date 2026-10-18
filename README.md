# IcarisLogistics

IcarisLogistics is a logistics CRM/MVP for **China → Russia freight forwarding**.

It combines:
- a public landing page with lead capture,
- a manager workflow for shipment operations,
- a Telegram bot + Telegram Mini App for customer access,
- and a Go backend with PostgreSQL.

> This repository is intentionally honest about its state: production is **not currently live** (details in [Deployment status](#deployment-status)).

## What the platform does today (MVP)

- **Lead intake:** landing page calculator + lead form (`Lead` persisted in PostgreSQL).
- **Manager panel:** email/password login, lead/shipment operations, shipment status updates, messaging, and shipment-level payment records.
- **Telegram bot:** customer onboarding (`/start`), quick status checks (`/status`), and operational notifications.
- **Telegram Mini App:** shipment list/details, status timeline, chat with manager, and payment history.
- **Notifications:** Telegram notifications are queued via an **outbox** (`notifications` table + retry dispatcher) for at-least-once delivery.

## Tech stack

- **Backend:** Go 1.26, Gin, GORM, PostgreSQL, SQL migrations (`golang-migrate`), JWT auth.
- **Frontend:** React 19 + TypeScript + Vite + Tailwind v4.
- **Telegram:** `go-telegram-bot-api` (long polling or webhook mode), Mini App auth via signed `initData` (HMAC validation).
- **Local infra:** Docker Compose (Postgres).

## Architecture

Layering in backend:

`HTTP (Gin) → service (business logic) → repository (GORM) → PostgreSQL`

```text
backend/
  cmd/{api,migrate,createmanager}   # entrypoints / CLI
  internal/
    domain/        # core entities
    repository/    # data access
    service/       # business logic
    http/          # routes + handlers
    middleware/    # auth/body/rate-limit middleware
    token/         # JWT issuance/parsing
    telegram/      # Telegram initData (HMAC) validation
    bot/           # bot commands + outbox dispatcher
  migrations/      # SQL migrations
frontend/
  index.html  + src/          # landing page
  manager.html + src/manager/ # manager panel
  webapp.html + src/webapp/   # Telegram Mini App
```

See also: [`docs/architecture.md`](./docs/architecture.md).

## Security and authentication model

- **Manager auth:** `POST /api/auth/login` returns JWT (`role=manager`).
- **Client auth:** `POST /api/app/auth/telegram` validates Telegram `initData` signature and returns JWT (`role=client`).
- **Role-based guards:** manager routes use `RequireAuth`, Mini App routes use `RequireClientAuth`.
- **Operational safety checks:** request body size limits, route-level rate limiting, CORS allowlist support, trusted proxy handling.

## Core user flow

1. Visitor submits a lead from the landing page.
2. Manager receives Telegram notification and qualifies the lead.
3. Customer is onboarded through Telegram (`Client` linked by `telegram_id`).
4. Manager creates/updates shipment, statuses, messages, and payment records.
5. Customer tracks shipments and chat in the Mini App, with `/status` in bot as a quick command.

## Local setup

### 1) Start Postgres

```bash
docker compose up -d postgres
```

### 2) Run backend

```bash
cd backend
go run ./cmd/migrate up
go run ./cmd/createmanager -email=manager@example.com -name="Manager" -password='<your-password>'
go run ./cmd/api
```

Backend runs on `http://localhost:8080` by default.

### 3) Run frontend

```bash
cd frontend
npm install
npm run dev
```

Frontend dev server runs on `:5173`:
- landing page: `/`
- Telegram Mini App: `/webapp.html`
- manager panel: `/manager.html`

Without `TELEGRAM_BOT_TOKEN`, bot behavior is disabled and Mini App Telegram-signature auth is unavailable.
For local UI debugging without Telegram, dev mode accepts `/webapp.html?token=<client-jwt>`.

## Configuration

Backend env template: [`backend/.env.example`](./backend/.env.example)

| Variable | Purpose | Default |
|---|---|---|
| `DATABASE_URL` | PostgreSQL DSN | local DB on `:5433` |
| `HTTP_ADDR` | API bind address (overrides `PORT`) | `:8080` |
| `PORT` | platform-provided port | — |
| `APP_ENV` | app environment | `development` |
| `JWT_SECRET` | JWT signing secret | dev default (must be overridden in non-dev) |
| `JWT_TTL` | JWT lifetime | `4h` |
| `TELEGRAM_BOT_TOKEN` | Telegram bot token | empty (bot disabled) |
| `MANAGER_CHAT_ID` | manager chat for notifications | empty |
| `ALLOWED_ORIGINS` | CORS allowlist (comma-separated) | empty |
| `TRUSTED_PROXIES` | trusted proxy IP/CIDR list | private ranges + CGNAT |
| `TELEGRAM_WEBHOOK_URL` | webhook endpoint URL | empty (long polling mode) |
| `TELEGRAM_WEBHOOK_SECRET` | webhook secret | empty |

Frontend build-time env:

| Variable | Purpose |
|---|---|
| `VITE_API_BASE` | backend base URL without `/api` |
| `VITE_PRIVACY_POLICY_URL` | public privacy policy URL |
| `VITE_PRIVACY_NOTICE_VERSION` | privacy notice version stored with lead consent |

If privacy URL/version is missing, lead form submission is intentionally disabled.

## API surface (high-level)

Public:
- `GET /api/health`
- `POST /api/leads`
- `POST /api/auth/login`
- `POST /api/app/auth/telegram`

Manager JWT routes:
- `/api/leads`, `/api/clients`, `/api/shipments`, shipment status/messages/payments endpoints

Client JWT (Mini App) routes:
- `/api/app/shipments`, shipment details/messages/payments for own shipments

## Testing and quality checks

```bash
docker compose up -d postgres
cd backend && go run ./cmd/migrate up && go test ./... && go vet ./... && go build ./...
cd ../frontend && npm ci && npm run lint && npm run build
```

The repository test suite expects PostgreSQL. Some DB-backed repository tests are skipped if DB is unavailable.

CI workflow: [GitHub Actions / CI](https://github.com/GomenAmonai/IcarisLogistics/actions/workflows/ci.yml)

## Deployment status

Production is **not currently deployed**.

- Previous Railway API/database were removed.
- Existing Vercel deployment may show an outdated build and is **not** an active production service.
- Planned Russian-hosted infrastructure and release phases are documented in [`docs/ROADMAP.md`](./docs/ROADMAP.md).

Migrations should run as a controlled step before rolling out a new API version.
Do not run multiple API instances with the same Telegram token in long polling mode (Telegram returns conflict 409).

## Current limitations (intentional MVP scope)

- JWT-only auth flow (no refresh token / revocation yet).
- Manager creation is CLI-based (`cmd/createmanager`).
- Chat currently has no realtime transport (no SSE/WebSocket yet).
- Outbox dispatcher is single-process oriented.
- Some hardening tasks are tracked as technical debt.

See: [`docs/tech-debt.md`](./docs/tech-debt.md)

## Additional documentation

- [`docs/ROADMAP.md`](./docs/ROADMAP.md) — release roadmap and rollout constraints.
- [`docs/architecture.md`](./docs/architecture.md) — domain and system architecture notes.
- [`docs/backend-onboarding.en.md`](./docs/backend-onboarding.en.md) — deep backend walkthrough for engineers.
- [`docs/backend-onboarding.md`](./docs/backend-onboarding.md) — Russian version of backend onboarding.
