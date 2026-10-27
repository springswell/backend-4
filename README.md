# StellarPOS Backend

The API behind StellarPOS: catalog and stock, sales, customers, staff,
reporting, and **non-custodial Stellar checkout**. Customers pay from their
own wallet by scanning a QR code, and the sale completes on its own once the
payment lands on-chain.

[![CI](https://github.com/StellarPOS-App/backend/actions/workflows/ci.yml/badge.svg)](https://github.com/StellarPOS-App/backend/actions/workflows/ci.yml)
![Node](https://img.shields.io/badge/node-%E2%89%A522.13-339933)
![License](https://img.shields.io/badge/license-MIT-green)

| Repository | What it is |
|---|---|
| [`surface`](https://github.com/StellarPOS-App/surface) | The point-of-sale web app cashiers and managers use |
| [`backend`](https://github.com/StellarPOS-App/backend) (this repo) | REST + WebSocket API |
| [`contracts`](https://github.com/StellarPOS-App/contracts) | Soroban contracts: settlement, loyalty, treasury, escrow |

## Highlights

- **Pay by QR, settled by the chain.** `POST /payments/requests` returns a
  [SEP-0007](https://github.com/stellar/stellar-protocol/blob/master/ecosystem/sep-0007.md)
  `web+stellar:pay` URI with a unique memo. A background watcher follows the
  merchant account on Horizon, matches the memo, checks the asset and amount,
  and completes the sale. Every terminal hears about it over Socket.IO.
- **The backend never holds keys.** It only watches a public key. There is no
  secret key in configuration, the database, or logs.
- **Nothing is taken on trust.** A Stellar payment counts only when Horizon
  shows a successful transaction paying the right account, the right asset
  (code *and* issuer, so look-alike assets are rejected) and at least the
  right amount.
- **Stock you can trust.** A sale reserves stock in a single database
  transaction: if any line is short, nothing changes. Voids and expired QR
  checkouts release the stock again.
- **No float money.** Prices are integer cents. Stellar amounts are handled
  as integer stroops (`bigint`) and always rounded in the merchant's favor.
- **Zero infrastructure to start.** Embedded SQLite through Node's built-in
  `node:sqlite`, plus in-process stand-ins for Redis and queues. `npm run dev`
  is all you need.

## Quick start

Requires **Node.js 22.13+** (for `node:sqlite`).

```bash
git clone https://github.com/StellarPOS-App/backend.git
cd backend
npm install
cp .env.example .env          # then set JWT_SECRET (openssl rand -hex 32)
npm run db:seed               # demo store: products, customers, 2 weeks of sales
npm run dev                   # http://localhost:3000
```

The seed creates three accounts, all with password `stellarpos-demo`:

| Email | Role |
|---|---|
| `admin@stellarpos.dev` | admin: everything, including staff management |
| `manager@stellarpos.dev` | manager: catalog, stock, refunds, wallet balance |
| `cashier@stellarpos.dev` | cashier: sales, payments, customers |

Without the seed, the first account registered becomes the admin.

### Turn on Stellar checkout

1. Create a testnet account in any wallet, or run
   `stellar keys generate shop --network testnet --fund`.
2. If you take USDC, add a trustline to it from that account.
3. Set `STELLAR_MERCHANT_PUBLIC_KEY` in `.env` (public key only) and restart.

The watcher starts automatically. `GET /api/v1/store` now reports Stellar as
enabled, and the surface app shows "Pay with Stellar" at checkout.

### Docker

```bash
docker compose up --build    # API on :3000, data in the stellarpos-data volume
```

## Scripts

| Command | What it does |
|---|---|
| `npm run dev` | Watch mode with `tsx` |
| `npm run build` / `npm start` | Compile to `dist/` (aliases rewritten) and run |
| `npm test` | 56 Jest + Supertest tests against an in-memory database |
| `npm run check` | Lint, type-check and test: what CI runs |
| `npm run db:seed` | Seed the demo store (refuses to touch a non-empty database) |
| `npm run db:backup` | Snapshot the database now; set `BACKUP_INTERVAL_HOURS` for automatic backups |

## API at a glance

All routes are under `/api/v1`. Responses use one envelope:
`{ "success": true, "data": ... }` or
`{ "success": false, "error": "...", "details"?: [...] }`.
Lists add `pagination: { page, limit, total, pages }`.

| Area | Endpoints |
|---|---|
| Auth | `POST /auth/register`, `POST /auth/login`, `GET/PATCH /auth/me` |
| Staff (admin) | `GET/POST /users`, `PATCH /users/:id/role` |
| Store | `GET /store` (public settings), `GET /store/wallet` (live balances) |
| Products | `GET /products`, `GET /products/categories`, `GET/PUT/DELETE /products/:id`, `POST /products/:id/stock` |
| Customers | `GET/POST /customers`, `GET/PUT/DELETE /customers/:id` |
| Sales | `GET/POST /transactions`, `GET /transactions/:id`, `POST /transactions/:id/void`, `POST /transactions/:id/refund` |
| Payments | `POST /payments` (cash, card, or Stellar by hash), `GET /payments/transaction/:id` |
| QR checkout | `POST /payments/requests`, `GET /payments/requests/:id`, `POST /payments/requests/:id/verify`, `POST /payments/requests/:id/cancel` |
| Reports | `GET /analytics/summary`, `/revenue-by-day`, `/hourly`, `/top-products` |

Full request and response details: [docs/api.md](docs/api.md).
Real-time events: `payment:confirmed`, `payment:expired`,
`transaction:completed`, `inventory:low`, `analytics:updated` (Socket.IO, with the JWT in
`auth.token`).

## How a Stellar checkout works

```mermaid
sequenceDiagram
    participant T as Terminal (surface)
    participant A as API
    participant W as Customer wallet
    participant H as Horizon
    T->>A: POST /transactions (cart)
    A-->>T: sale S-000123, pending, stock reserved
    T->>A: POST /payments/requests
    A-->>T: web+stellar:pay?...&memo=SPK7Q2M9XAB4
    T->>W: QR code
    W->>H: payment to merchant, memo SPK7Q2M9XAB4
    loop every 5s
      A->>H: payments for merchant since cursor
    end
    H-->>A: matching payment
    A->>A: check asset, issuer, amount, memo; complete sale
    A-->>T: socket payment:confirmed
```

Unpaid checkouts expire after `STELLAR_PAYMENT_TTL_MINUTES`, and their stock
is released. The design, edge cases and threat model are in
[docs/stellar-payments.md](docs/stellar-payments.md).

## Project layout

```
src/
  app.ts                  Express app (exported for tests)
  server.ts               HTTP + Socket.IO bootstrap, watcher, graceful shutdown
  config/config.ts        Typed environment config; fails fast on bad values
  middleware/             auth (JWT + roles), Joi validation, errors, 404
  models/                 SQLite data access, one file per table
  services/
    checkout.service.ts   Payment rules: cash, card, Stellar, QR, expiry
    stellar.service.ts    SEP-7 URIs, on-chain verification
    horizon.ts            The only module that talks to Horizon
    paymentWatcher.ts     Cursor-based polling of the merchant account
    realtime.ts           Socket.IO events to terminals
  routes/                 One router per resource
  validation/             Joi schemas
  utils/                  database, money math, logger, helpers, stand-ins
  scripts/seed.ts         Demo data
tests/                    Jest + Supertest, Horizon mocked at horizon.ts
```

## Configuration

Everything is in [`.env.example`](.env.example), with comments. The app
refuses to start without `JWT_SECRET`, and in production requires it to be at
least 32 characters.

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md). Security reports go through
[SECURITY.md](SECURITY.md), not public issues.

## License

[MIT](LICENSE)

## Mock Stellar Horizon in Tests\n\nIntegration tests must use a mock Stellar Horizon server to avoid rate limits, network flakiness, and test pollution from real testnet state. Use stellar/js-stellar-sdk test helpers or a custom nock interceptor.