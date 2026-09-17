# Evolution Go — Codebase Overview

Senior review of `evolution-go` as it exists today. Written for engineers who need to work in this repo, not for marketing.

---

## Verdict

This is a **WhatsApp Multi-Device API gateway**. It wraps [whatsmeow](https://github.com/tulir/whatsmeow), exposes a Gin REST API, keeps multiple named WhatsApp sessions in one process, and fans events out to webhook, WebSocket, RabbitMQ, and NATS.

The product shape is sound. The domain split is readable. Ops basics (graceful shutdown, DB pool, Docker, Swagger) are in place.

The system is **not yet a multi-node service**. Instance clients live in process memory. Two files carry most of the risk: `pkg/whatsmeow/service/whatsmeow.go` (~2.9k LOC) and `pkg/sendMessage/service/send_service.go` (~3.4k LOC). Tests cover almost none of the real surface.

Treat it as a **mature MVP / early production gateway**, not as a horizontally scaled platform.

---

## What it is

| Item | Detail |
|------|--------|
| Product | REST + event gateway for WhatsApp Web Multi-Device |
| Org | Evolution Foundation; sister of the Node Evolution API |
| Consumers | Evo CRM and other ecosystem products |
| License | Apache 2.0 plus brand conditions; runtime license gate in `pkg/core` |
| Language | Go 1.25.0 |
| HTTP | Gin |
| WhatsApp | `go.mau.fi/whatsmeow` |
| App DB | PostgreSQL via GORM |
| Session store | whatsmeow `sqlstore` on PostgreSQL or SQLite |
| Media | MinIO (optional) |
| Events | Webhook, WebSocket, RabbitMQ, NATS |
| UI | Prebuilt React manager at `/manager` |

Size: **56 Go files**, about **21k LOC**. Four test files.

---

## How the system is shaped

Layered handler → service → repository. Not hexagonal. No DI container. `cmd/evolution-go/main.go` is the composition root.

```
HTTP client
    │
    ▼
Gin + CORS + core.GateMiddleware (license)
    │
    ├── /license/*          public (activation)
    ├── /passkey-ceremony/* public (ephemeral token)
    ├── /manager, /swagger  public
    ├── /ws                 GLOBAL_API_KEY query param
    └── feature routes      apikey header
            │
            ▼
      handler → service → (repository | whatsmeow client)
            │
            ├── in-memory clientPointer[instanceId]
            ├── GORM users DB (instances, messages, labels)
            ├── whatsmeow sqlstore (keys, devices, sessions)
            └── Producer (webhook / AMQP / NATS / WS)
```

Request flow is consistent: bind JSON, resolve instance from `apikey`, look up `*whatsmeow.Client` from a process map, call WhatsApp, optionally persist, optionally emit events.

---

## Boot sequence

`cmd/evolution-go/main.go`:

1. `-dev` loads `.env`.
2. `config.Load()` — fail-fast on missing DB / `GLOBAL_API_KEY` / `DATABASE_SAVE_MESSAGES`.
3. Users DB via GORM (`CreateUsersDB`).
4. Auth DB: `POSTGRES_AUTH_DB` or SQLite `{exe}/dbdata/users.db`.
5. AutoMigrate `Instance`, `Message`, `Label`.
6. `core.InitializeRuntime` — license context.
7. Optional RabbitMQ dial (30s heartbeat).
8. `setupRouter` wires maps, producers, services, Gin, routes.
9. Optional `ConnectOnStartup` reconnects instances for `CLIENT_NAME`.
10. Heartbeat + `http.Server` + 10s graceful shutdown.

Runtime state created in `setupRouter` and passed everywhere:

- `clientPointer map[string]*whatsmeow.Client`
- `killChannel map[string]chan bool`

That is the instance registry. One process, one registry. No shared cache, no sticky-session contract, no replica story that actually works.

---

## Package map

| Area | Role |
|------|------|
| `pkg/instance` | CRUD, QR, pair, disconnect, proxy, advanced settings |
| `pkg/sendMessage` | Outbound: text, media, poll, sticker, location, contact, buttons, list, carousel, status |
| `pkg/message` | React, presence, read/played, download, delete, edit, optional persist |
| `pkg/chat` | Pin / archive / mute / history-sync (several routes marked broken) |
| `pkg/group` `community` `newsletter` `label` `poll` `call` `user` | Thin wrappers over a connected client |
| `pkg/whatsmeow` | Client lifecycle, event switch, QR, passkey, persist, dispatch |
| `pkg/events` | `Producer` interface + four implementations |
| `pkg/passkey` | Public WebAuthn ceremony + in-memory store |
| `pkg/storage` | `MediaStorage` + MinIO |
| `pkg/middleware` | Instance token, admin key, JID normalize |
| `pkg/routes` | Single route table |
| `pkg/config` | Env + DB factories |
| `pkg/logger` | Per-instance file logs |
| `pkg/telemetry` | HTTP telemetry middleware — **not registered** |
| `pkg/server` | `GET /server/ok` |
| `pkg/core` | License register / activate / heartbeat / gate |
| `pkg/internal/event_types` | Canonical event names |
| `pkg/utils` | JID, proxy, shared helpers |

Repositories exist only for **instance**, **message**, and **label**. Everything else is handler + service talking to the live client.

---

## WhatsApp integration

`pkg/whatsmeow/service/whatsmeow.go` owns the protocol boundary.

1. `StartInstance` / `StartClient` opens `sqlstore` (`POSTGRES_AUTH_DB` or `dbdata/main.db`).
2. Load or create `store.Device`, set platform / OS / WA version.
3. `whatsmeow.NewClient` stored in `clientPointer`.
4. Wrapped as `MyClient` (producers, repos, poll, passkey).
5. `AddEventHandler(myEventHandler)` — large type switch on incoming events.
6. `CallWebhook` filters by instance `Events`, then `sendToQueueOrWebhook`.
7. Optional `persistMessageAsync` when `DATABASE_SAVE_MESSAGES=true`.
8. `ConnectOnStartup` restores sessions for the current `CLIENT_NAME`.

This file is the integration hub. Almost every feature eventually lands here. That is why it is both the most important and the most dangerous file in the repo.

---

## Events

Contract in `pkg/events/interfaces/producer.go`:

```go
Produce(queueName string, payload []byte, webhookUrl string, userID string) error
CreateGlobalQueues() error
```

| Channel | File | Auth / config |
|---------|------|----------------|
| Webhook | `pkg/events/webhook` | Global `WEBHOOK_URL` + per-instance URL; 5 retries, 30s |
| RabbitMQ | `pkg/events/rabbitmq` | Global + per-instance flags; reconnect |
| NATS | `pkg/events/nats` | Same pattern |
| WebSocket | `pkg/events/websocket` | `GET /ws?token=&instanceId=` — token is **`GLOBAL_API_KEY`**, not instance token |

Event names live in `pkg/internal/event_types`. Subscription is a string on the instance row. Filter + fan-out lives inside whatsmeow, not in a dedicated dispatcher.

One `Producer` interface is the right idea. The interface is too thin for the real behavior (reconnect, global queues, instance flags). Implementations compensate with constructor flags.

---

## Persistence

Two databases, by design:

1. **Auth / session** — whatsmeow `sqlstore`. Keys, devices, sessions. Not GORM models.
2. **Users / app** — GORM:
   - `pkg/instance/model` — instances, tokens, webhooks, flags
   - `pkg/message/model` — optional message rows
   - `pkg/label/model` — labels

Migrations are GORM `AutoMigrate` plus `core.MigrateDB()` for license tables. There is no versioned SQL migration set. `make migrate-up` points at a `migrations/` folder that is not in this tree.

`Config.CreateAuthDB()` exists and is unused. If called, it would use the users DSN. Dead API, leftover smell.

SQLite fallback: `{exe}/dbdata/users.db` (aux) and `{exe}/dbdata/main.db` (whatsmeow). Fine for local. Do not run production on it.

---

## Auth and license

| Mechanism | Where | Rule |
|-----------|--------|------|
| Instance API | `pkg/middleware/auth_middleware.go` | Header `apikey` → `GetInstanceByToken` → `ctx.Set("instance")` |
| Admin API | same | `apikey == GLOBAL_API_KEY` |
| WebSocket | `main.go` | Query `token` must equal `GLOBAL_API_KEY` |
| License | `pkg/core/c0.go` `GateMiddleware` | Most routes `503 LICENSE_REQUIRED` until activated |
| Passkey | `pkg/passkey` | Public, gated by short-lived ceremony token |

Bypass list includes manager, swagger, passkey, `/ws`, `/server/ok`.

`pkg/core` is obfuscated license client (heartbeat to `license.evolutionfoundation.com.br`). Product decision, not architecture. Operators will hit 503 on first boot until `/manager/login` completes activation. That must stay in the runbook.

---

## HTTP surface

Single table: `pkg/routes/routes.go`.

| Prefix | Auth | Purpose |
|--------|------|---------|
| `/instance` (create/all/delete/…) | Admin key | Lifecycle |
| `/instance` (connect/qr/pair/…) | Instance token | Session |
| `/send` `/user` `/message` `/chat` `/group` `/call` `/community` `/label` `/unlabel` `/newsletter` `/polls` | Instance token | Features |
| `/server/ok` | None | Liveness |
| `/swagger/*` `/manager` | None | Docs + UI |
| `/license/*` | License flow | Activation |
| `/passkey-ceremony/*` | Ceremony token | Pairing |
| `/ws` | Global key | Event stream |

JID middleware rewrites phone/JID fields on JSON and multipart before handlers run. That is the right place for it.

---

## Deploy

- Multi-stage `Dockerfile`: `golang:1.25.0-alpine` → `alpine:3.19.1` with `ffmpeg` and `poppler-utils` (media/PDF thumbnails).
- Compose examples under `docker/examples/`.
- CI publishes `evoapicloud/evolution-go` from `VERSION`.
- Makefile covers `dev`, `build`, `swagger`, `docker-*`.

`make status` curls `/health`. The live liveness route is `/server/ok`. Same class of drift as `WADEBUG` vs `DEBUG_ENABLED`.

---

## What is done well

1. **Domain folders match the API.** A new engineer can go from route → handler → service without hunting.
2. **Event channels share one interface.** Adding a producer is mechanical.
3. **Session store is separated from app data.** Correct for whatsmeow; keys do not belong in the CRM schema.
4. **Composition is explicit.** `setupRouter` is long but honest. You can see every dependency.
5. **Ops hygiene exists.** Pool limits, AMQP heartbeat, reconnect, lumberjack per instance, 10s shutdown.
6. **JID normalization is middleware**, not copy-pasted in every handler.
7. **Wiki + Swagger** are unusually complete for a repo this size.
8. **Passkey pairing** is a real product feature, not a stub, and it is documented as public by design.

---

## What will hurt you

### 1. Two god files

`send_service.go` and `whatsmeow.go` together are ~6.3k lines. Event handling, client lifecycle, persist, media, QR, passkey, and dispatch sit in one package. Send paths for every message type sit in one service.

This is the main maintainability problem. A bug in poll persist and a bug in QR timeout should not share a 3k-line file.

**Practice:** split by concern (`client`, `events`, `persist`, `pairing`) behind the existing `WhatsmeowService` interface. Split send by message kind. Do not rewrite the protocol layer.

### 2. Almost no tests

Four files: utils, message repo, thumbnail, referral. No handler tests, no event-pipeline tests, no producer tests, no auth tests.

For a gateway that talks to a third-party protocol, this is under-insured. The Makefile already has `test-race` and coverage targets. The suite does not justify them.

**Practice:** start with middleware, JID validation, event-type filter, and repository. Then fake `Producer` + fake client for send/instance. Do not try to mock all of whatsmeow first.

### 3. In-memory registry — no horizontal scale

`clientPointer` is a Go map. Wiki text about load-balanced replicas is aspirational. A second process does not see the first process's sessions. Sticky routing or "one instance = one node" is required until the registry is externalized.

**Practice:** document the constraint as a hard ops rule. Do not sell multi-replica until session ownership is designed.

### 4. Maps are not synchronized at the type level

The maps are shared across goroutines (HTTP, event handler, startup reconnect, kill). If locking is only local inside whatsmeow, you have a race surface. `make test-race` is the right tool; there is almost nothing to run it on.

### 5. Auth model is inconsistent

REST instance routes use instance token. Admin routes use `GLOBAL_API_KEY`. WebSocket uses the global key in a query string. Query-string secrets land in logs and proxies.

CORS is registered twice (`main.go` and `routes.go`): `Allow-Origin: *` plus `Allow-Credentials: true`. That combination is invalid per spec and is copy-paste.

WebSocket `CheckOrigin` is wide open.

**Practice:** one CORS middleware. Instance-scoped WS tokens. Origin allowlist in production.

### 6. Config and docs drift

| Docs / Makefile | Code |
|-----------------|------|
| `WADEBUG` | `DEBUG_ENABLED` (`pkg/config/env/env.go`) |
| `LOGTYPE` | `LOG_TYPE` |
| `/health` in `make status` | `/server/ok` |
| README "Go 1.24+" | `go 1.25.0` |
| Wiki "horizontal scale" | Single-process maps |

Operators will set the documented names and get silent defaults.

### 7. Dead or unused paths

- `pkg/telemetry` is never `Use`d. License heartbeat in `core` is a second telemetry path.
- `CreateAuthDB()` unused.
- Chat pin/mute/archive and `GET /group/myall` are registered with `TODO: not working`.

Shipping known-broken routes is worse than omitting them. Clients will integrate against Swagger and fail in production.

### 8. License and core package

`pkg/core/c0.go` (~978 LOC) is obfuscated. Fine as a product control. Poor as an engineering boundary: hard to test, hard to reason about failure, hard to run air-gapped. Gate-before-ready is an operational footgun if the runbook is skipped.

### 9. Manual wiring and naming

`main.go` imports every handler/service. Package dirs (`sendMessage`) and package names (`send_service`) ignore Go conventions (`sendmessage`, package `service` or `send`). Not a runtime bug. It slows every review and every import.

### 10. Sleeps and TODOs as control flow

`time.Sleep` on connect paths and leftover `// NOVO` / `TODO` comments are signs the lifecycle is timed, not signaled. That will flake under load and on slow networks.

### 11. Makefile vs reality

`migrate-up`, `/health`, `/debug/pprof` targets assume infrastructure this binary does not expose. Either add the endpoints or delete the targets. Stale Make is how on-call wastes the first hour.

---

## Industry-practice gaps (Go / API)

These are the deltas versus how a long-lived Go service is usually built:

| Practice | This repo |
|----------|-----------|
| Small packages, one job each | Two 3k-line services |
| Interfaces at process boundaries | Concrete maps passed through constructors |
| Versioned migrations | GORM AutoMigrate only |
| Table-driven handler tests | Four unit files |
| Config from a single source of truth | README and env constants disagree |
| Structured errors (`errors.Is`, problem codes) | Many `gin.H{"error": "..."}` strings, some in Portuguese |
| Context deadlines on outbound I/O | Mixed; not a project-wide rule |
| pprof / metrics | Makefile mentions pprof; no registration in `main` |
| Race-safe shared state | Maps + goroutines, little test evidence |
| Idiomatic names | `sendMessage`, `instance_service` |

None of these block a single-node deploy. All of them raise the cost of the next ten features.

---

## How to navigate

1. Read `README.md` then this file. Wiki architecture doc is a teaching analogy, not the source of truth.
2. Start at `cmd/evolution-go/main.go` → `setupRouter`.
3. Routes: `pkg/routes/routes.go`. Swagger: `/swagger/index.html`.
4. Instance life: `pkg/instance/service` → `pkg/whatsmeow/service` (`StartClient`, `myEventHandler`).
5. Send: `pkg/sendMessage/handler` → `send_service.go` (search the method).
6. Events: `myEventHandler` → `CallWebhook` / `SendToGlobalQueues` → `pkg/events/*`.
7. Config: `pkg/config` + `pkg/config/env` + `.env.example` (trust the constants, not the README table).
8. Persistence: three GORM models. Session crypto is inside whatsmeow, not here.
9. License / manager: `pkg/core`, `/manager`.
10. Local run: `make setup`, copy `.env.example`, `make dev`.

---

## If you change one thing first

Do not start with a rewrite.

1. Align env names and health route with the code.
2. Remove or hide routes marked not working.
3. Deduplicate CORS; stop advertising credentials with `*`.
4. Add tests for auth middleware, JID middleware, and event-type filter.
5. Keep new WhatsApp behavior behind `WhatsmeowService`; stop growing the god file when you can extract a type.

After that, the codebase is honest about what it is: a solid single-process WhatsApp gateway with a clean HTTP skin and a heavy integration core.

---

## File index (largest risk first)

| File | Why it matters |
|------|----------------|
| `pkg/whatsmeow/service/whatsmeow.go` | Client + events + persist + pairing |
| `pkg/sendMessage/service/send_service.go` | Entire outbound message surface |
| `pkg/core/c0.go` | License gate; API is dark until activated |
| `cmd/evolution-go/main.go` | Wiring, DBs, shutdown |
| `pkg/routes/routes.go` | Contract with API clients |
| `pkg/middleware/auth_middleware.go` | Who can call what |
| `pkg/config/env/env.go` | Actual env names |
| `pkg/events/interfaces/producer.go` | Event bus contract |
| `pkg/instance/model/instance_model.go` | Instance as the tenant |

Reviewed against the tree as of this document. Features not in the code (Redis cache, real horizontal session sharing, `/health`) are not part of this system.
