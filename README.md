# AUCTA — Indonesian collectibles auction marketplace

A full auction marketplace for watches, cameras, trading cards, sneakers,
gaming, design, electronics and art. It combines a **proxy auction engine**
(anti-snipe, hidden maximums, reserves, immutable bid events), a **seller
desk** with a listing wizard and verification, a **buyer checkout → payment →
shipping → completion** loop, and an **admin operations console** — bilingual
in English and Bahasa Indonesia.

> This is a payment **marketplace**, not escrow. Funds settle to the seller
> through the payment provider on the provider's payout schedule; AUCTA
> verifies settlement server-to-server, snapshots the order immutably and
> mediates disputes. Nothing on the site holds funds in a legal escrow
> arrangement.

## Stack

- **Next.js 16** (App Router, Turbopack, Server Actions-style route handlers)
- **PostgreSQL** via **Drizzle ORM** (row-level locking, JSONB snapshots)
- Tailwind CSS v4, custom warm "paper/ink/bronze" design system
- **Vitest** for unit + real-Postgres concurrency tests
- **Midtrans** adapter (QRIS, virtual account, e-wallet, card) with a
  verified fallback provider for local development

## Quick start

```bash
cp .env.example .env        # fill DATABASE_URL + AUCTA_SECRET
npm install
npx drizzle-kit push        # create tables (or apply drizzle/ migrations)
npm run dev
```

The app seeds a demo catalogue, two sellers (one verified, one pending), an
admin, sample orders/disputes/reports and bid history automatically on boot
(`src/instrumentation.ts`). Sign in with any email — a 6-digit code is issued
(shown on screen in the fallback and queued in the dev inbox at `/dev-mail`).

### Demo accounts

| Role        | Email                              | Notes                          |
| ----------- | ---------------------------------- | ------------------------------ |
| Admin desk  | `admin@aucta.preview`              | Operations console at `/admin` |
| Seller      | `dewa.wardana@aucta.local`         | Verified, active seller desk    |
| Seller      | `laras.putri@aucta.local`          | Pending verification            |

## Scripts

| Command                | Purpose                                      |
| ---------------------- | -------------------------------------------- |
| `npm run dev`          | Dev server (auto-seeds, runs scheduler)      |
| `npm run build/start`  | Production build / server                    |
| `npm test`             | Vitest: pure unit tests + Postgres concurrency |
| `npm run typecheck`    | `tsc --noEmit`                               |
| `npm run db:push`      | Push schema to the configured database       |
| `npm run db:generate` | Generate a Drizzle migration                 |
| `npm run db:studio`    | Drizzle Studio                               |

Integration/concurrency tests in `tests/integration/` need a reachable
`DATABASE_URL`; they create and clean up their own rows. Pure unit tests run
without a database.

## Product surfaces

- `/auctions`, `/sold` — live catalogue + **realized-price archive** with
  median/average/total analytics and saved-search alerts.
- `/auctions/[slug]` — proxy bidding, anti-snipe, condition reports,
  authenticity evidence, locked image hashes, seller trust panel, reporting.
- `/sell` — seller dashboard (KPIs, pipeline funnel, hammer volume,
  transactions), 10-step listing wizard with drafts, photo ordering and
  review states; `/sell/verification` for identity verification.
- `/checkout/[id]` — address, courier/express/collection options, payment.
- `/account?tab=orders` — buyer **and** seller order actions.
- `/admin` — verification & approval queues, live auctions, bidding-safety
  signals (last-second bids, concentration), payment failures, disputes,
  reports, sellers, refunds and an immutable audit log.
- `/notifications`, `/alerts`, `/watchlist`, `/seller/[alias]`.

## Order lifecycle

```
awaiting_payment → paid → preparing → shipped → delivered → completed
        ├ payment_failed → cancelled (lot reverts to unsold)
        ├ disputed ←────────── any post-payment state
        └ refunded / cancelled (terminal)
```

At the hammer the close job writes an **immutable order snapshot** (parties,
lot, hammer price, buyer fee, seller commission, quoted shipping, amount due,
24h payment deadline, 48h dispatch deadline). Shipping adds carrier, tracking
number, carrier status and a proof-of-shipment reference.

### Payments

`src/lib/payments/`

- **Midtrans**: Snap transaction creation and an HTTP-notification webhook at
  `POST /api/payments/midtrans/webhook`. Signatures are verified with
  `sha512(order_id + status_code + gross_amount + SERVER_KEY)`; amounts are
  compared server-side; settlements are idempotent (`SELECT … FOR UPDATE`).
- **Fallback (`manual`)**: generates BCA/BRI/BNI/Mandiri virtual-account or
  QRIS instructions; settlement is confirmed through a verified endpoint
  (`/api/payments/manual/confirm`) which runs the same idempotent,
  amount-checked path. Automatically used when no gateway keys exist.
- A reconciliation poll recovers missed webhooks; an expiry job cancels
  unpaid orders at the deadline.
- Financial endpoints are rate-limited and support idempotency keys.

Configure in the Midtrans dashboard:
`Payment Notification URL = $NEXT_PUBLIC_BASE_URL/api/payments/midtrans/webhook`.

## Auth & security

- Magic-link **and** 6-digit OTP (`/api/auth/request`, `/api/auth/verify`,
  `/api/auth/callback`), one-time hashed credentials, 15-minute expiry,
  attempt limits. Optional Google OAuth with CSRF `state`.
- Session cookies are HMAC-signed (constant-time verification), `httpOnly`,
  `sameSite=lax`, and `secure` in production.
- Open-redirect guard (`safeRedirect`) on every `next` parameter.
- Sellers **cannot bid on their own lots**; proxy/reserve math lives in
  pure, tested modules (`src/lib/auction-math.ts`, `src/lib/order-math.ts`).
- All bids, proxy extensions, stage changes, wins, payments, shipments and
  disputes append immutable `bid_events`; every admin action writes an
  `admin_events` audit record.

## Background jobs

`src/instrumentation.ts` starts a 20s scheduler (`DISABLE_SCHEDULER=true`
turns it off): auction activation, close/winner/unsold processing,
ending-soon reminders, payment expiry + reconciliation, seller dispatch
reminders, delivered-order auto-completion.

## Deployment

- **Database**: any Postgres 15+ (Supabase/Neon/RDS). Apply migrations with
  `npm run db:push` (or the generated SQL in `drizzle/`).
- **App**: Vercel/Node — set the environment variables from `.env.example`.
  Run one process with the scheduler enabled; set `DISABLE_SCHEDULER=true`
  on additional replicas, or move jobs to a dedicated worker.
- Set `NEXT_PUBLIC_BASE_URL` to the public origin before going live (magic
  links and payment callbacks depend on it).
- Provide a strong `AUCTA_SECRET` (`openssl rand -base64 48`).
- Configure Midtrans production keys and SMTP for real settlement + email.

## Testing

```bash
npm test
```

Covers proxy/reserve/increment math, order fee math & state machine, HMAC
session tokens, redirect/rate-limit/idempotency helpers, Midtrans webhook
signatures, and a real-Postgres test firing simultaneous bids to prove
serialization and price convergence.
