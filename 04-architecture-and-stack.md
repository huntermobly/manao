# 04. Architecture and stack

## The pieces

| Piece | Runs where | Job |
|---|---|---|
| Tracker | Shopper's browser, on the store | Records classified actions, asks for consent, shows the capture prompt and on-site messages, subscribes to web push, sends events to the backend |
| Console | Merchant's browser, served by the backend at `/app` | Connect, scan, review tags, connect channels, manage templates and coupons, approve messages, answer conversations, make calls, read analytics |
| Backend API | Server | Accounts, sites, event maps, ingest, opt-ins, messages, analytics, links, webhooks |
| Coordinator and agents | Server, inside the backend | Turn events into decisions and messages |
| Queue workers | Server, inside the backend | Delayed work: ending sessions, deferred sends, second messages, nightly Learning |
| Crawler | Server, own process | A headless browser that samples a store's public pages during a scan and checks the tag during verification |
| Channel adapters | Server, inside the backend | WhatsApp (Meta Cloud API), email (SMTP), web push, on-site message, call-back task |
| Demo store | Local, for tests | A small store with product, cart, three checkout steps and an order page |
| Test doubles | Local, for tests | Fake Meta server, mock email provider, mock push service, fake model, controllable clock |

The crawler runs in its own process because a headless browser is heavy and can crash. Everything else runs in one backend process in v1.

## How data moves

```
Store page (tracker) ── events, opt-ins ───────────▶ Backend API ──▶ PostgreSQL
Console (web app)    ── merchant actions ──────────▶ Backend API
Crawler              ── page snapshots ────────────▶ Backend API

Backend:  session ends ▶ coordinator ▶ agents (A2A) ▶ channel adapter
                                                        │
              Meta Cloud API / SMTP server / push service / tracker / call list

Meta webhooks, push receipts, link clicks, unsubscribes ──▶ Backend API ▶ Reply, Learning
```

The console and the store never talk to each other directly. Both talk to the backend and are joined by the site ID. The only code of ours that runs on the store's pages is the tracker, including its setup mode.

## Stack

These are defaults. Change one only with a reason, and say so in your report.

| Area | Choice |
|---|---|
| Language | TypeScript throughout, Node.js LTS, pnpm workspaces |
| Backend | Fastify, PostgreSQL, a Postgres-backed job queue for delayed work (pg-boss or similar), migrations checked in |
| Console | React, Vite, React Router; a single-page app built to static files and served by the backend at `/app` |
| Tracker | Plain TypeScript, no runtime dependencies, bundled per file (`05-tracker.md`); the core file is under 15 KB gzipped |
| Crawler | Playwright with Chromium |
| Email | Nodemailer over SMTP |
| Web push | The `web-push` package with VAPID keys |
| Passwords | Argon2id |
| Validation | JSON Schema for every event, payload and model output, shared between packages |
| Tests | Vitest for units, Playwright for end-to-end runs against the demo store |
| Language model | One wrapper module. Provider, model and key come from environment variables. A fake implementation serves the tests |
| A2A | The official JavaScript SDK if it fits; otherwise the smallest subset of the protocol that gives Agent Cards, message send and task states |
| Local run | Docker Compose for PostgreSQL and a local mail catcher, and one command that starts everything |

## Repository

```
apps/
  backend/        API, coordinator, agents, channel adapters, queue workers; serves the console
  console/        merchant web app
  crawler/        headless browser worker
  demo-store/     test store, on its own origin
  fake-meta/      stand-in for Meta's Graph API, for tests and local runs
packages/
  tracker/        rc.js, rc-ui.js, rc-setup.js, service worker
  snapshot/       page snapshot code, shared by the crawler and the setup module
  shared/         types and JSON Schemas: events, event map, agent payloads, message formats
tests/
  e2e/            the eight acceptance tests
docs/             these files
```

## Environment variables

Platform-wide values come from the environment. Per-store secrets do not: a store's Meta token and app secret, SMTP password and push signing key are entered in the console and stored per site, encrypted (`17-privacy-and-security.md`).

| Variable | Meaning |
|---|---|
| `APP_BASE_URL` | The backend's public address. This is `OUR_DOMAIN` |
| `DATABASE_URL` | PostgreSQL connection |
| `ENCRYPTION_KEY` | Key for encrypting personal data and per-store secrets at rest |
| `SESSION_SECRET` | Signs the console's session cookie |
| `LLM_PROVIDER`, `LLM_MODEL`, `LLM_API_KEY` | Language model settings. `LLM_PROVIDER=fake` selects the scripted fake |
| `META_GRAPH_BASE_URL` | Base address of Meta's Graph API. Defaults to Meta's own; tests and local runs point it at `fake-meta` |
| `META_GRAPH_VERSION` | The Graph API version to call. Set it after reading Meta's current documentation; do not hard-code one |
| `PLATFORM_SMTP_URL` | SMTP server for account emails such as password reset |
| `CRAWLER_ALLOW_PRIVATE_HOSTS` | Development only. Lets the crawler load `localhost` so it can reach the demo store |
| `LOG_LEVEL` | Log verbosity |

## Running it

- `docker compose up` starts PostgreSQL and the mail catcher.
- One command (for example `pnpm dev`) runs migrations and starts the backend, the crawler, the console in watch mode, the demo store and `fake-meta`.
- One command (for example `pnpm test:e2e`) runs the eight acceptance tests.

Locally the pieces sit on different origins so that cross-origin behaviour is real: the backend and console on one port, the demo store on another, `fake-meta` on a third.

## Test doubles

| Double | Stands in for | What it can do |
|---|---|---|
| `fake-meta` | Meta's Graph API and webhooks | Accepts the calls listed in `14-delivery-channels.md`, records them, and on command posts signed webhooks, fails the next send with a chosen error, or rejects the token |
| `MockEmailProvider` | The SMTP server | Records every email with headers and both body parts |
| `MockPushService` | Browser push services | Records every push, and on command reports a subscription as gone |
| Fake model | The language model | Returns scripted outputs per agent, so tests are repeatable |
| Controllable clock | System time | One clock used everywhere, including the job queue, that tests can set and advance |
