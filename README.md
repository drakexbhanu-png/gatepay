# GatePay

> Pay any token on any chain, settle SOL on Solana, get into the channel.

**Live on Telegram:** [@GatePayy_bot](https://t.me/GatePayy_bot)

GatePay is a Telegram bot that turns any private channel into a paid subscription. Subscribers pay with any token on any blockchain — owners always receive native SOL on Solana, settled in seconds. The bot auto-grants Telegram channel access on payment confirmation, sends renewal reminders, and removes expired members.

Built for the **Frontier Hackathon — Cross-Chain Checkout track**, sponsored by [KIRAPAY](https://www.kira-pay.com).

---

## The problem

Telegram is the de-facto home of crypto communities — analyst groups, trading rooms, signal channels, paid newsletters. But monetizing a private Telegram channel today still runs on screenshots and DMs:

1. Subscriber: "How do I pay?"
2. Owner: "Send me USDT to this address" — but the subscriber holds ETH on Arbitrum, SOL on Solana, or USDC on Base
3. Subscriber gives up, or bridges, or both — conversion rate drops off a cliff
4. Owner manually verifies on-chain, manually adds the subscriber, manually tracks expiry

There's no Stripe for Telegram channels. There's no good way for a creator to say "I'll accept any token from any chain, and I want to be paid in SOL." That's the gap GatePay fills.

## The solution

A two-sided bot:

- **For creators:** add `@GatePayy_bot` as admin in your private channel, run `/setup`, set a price and your Solana wallet, share the subscribe URL anywhere.
- **For subscribers:** click a subscribe URL, pay with whatever you have on whatever chain — KIRAPAY routes it cross-chain to native SOL on the creator's wallet. The bot DMs you a one-time invite link the moment payment confirms.

Subscriptions auto-expire. 3 days before expiry, subscribers get a renewal DM with a fresh checkout button. At expiry, the bot removes the user from the channel and DMs a resubscribe link. The creator's wallet just keeps receiving SOL.

## Why KIRAPAY is the core enabler

GatePay is structurally impossible without intent-based cross-chain payments. Take KIRAPAY out and the bot has no payment flow at all.

- **Customer side:** "any token on any chain" is what unlocks Telegram's chain-fragmented audience. The same checkout works for a subscriber holding USDC on Base, SOL on Solana, USDT on BNB Chain, or ETH on Arbitrum. Forcing one chain at checkout cuts the addressable audience by 10x.
- **Owner side:** uniform SOL settlement means a single Solana wallet, single tax basis, no treasury fragmentation. A creator running 10 channels with 1000 subscribers gets one settlement stream, not ten.
- **Reconciliation:** the `customOrderId` field we ship to KIRAPAY (`{telegramUserId}:{channelId}:{durationDays}:{nonce}`) round-trips back on the webhook, letting us grant access to the exact right subscriber for the exact right channel without holding any custodial state.

GatePay is also a **distribution layer** for KIRAPAY: every Telegram creator who signs up becomes a new KIRAPAY merchant.

## How it works

```
Subscriber clicks subscribe URL
   ↓
Bot generates KIRAPAY payment link + records pending Payment
   customOrderId = "{tgId}:{channelId}:{days}:{nonce}"
   ↓
Subscriber pays on KIRAPAY checkout (any chain, any token)
   ↓
KIRAPAY routes cross-chain → settles native SOL to creator's wallet
   ↓
KIRAPAY fires webhook: { type: "transaction.created", data: { transaction, status: "Success", customOrderId, ... } }
   ↓
Server verifies HMAC signature → parses customOrderId → grantAccess()
   ↓
grantAccess(): create one-time invite link → DM subscriber → upsert Subscription
   → DM owner with metrics (active subscribers, lifetime revenue)
   → delete stale "Pay" message bubbles from chat
```

## Features

**Creator surface**
- 4-step setup wizard: title → price → duration → Solana wallet (first time only; reused on subsequent channels)
- Add bot as admin → bot auto-DMs with a "Set up channel" button to start the wizard
- `/channels` with subscriber counts + lifetime revenue per channel, delete button per channel with confirmation
- `/wallet` to view or update settlement wallet (with format validation)
- Real-time owner DM on every new subscription, including subscriber identity and live channel metrics

**Subscriber surface**
- One-tap `/start sub_<slug>` deep-link from any subscribe URL
- "Already subscribed" view if active — no Pay button shown
- `/mysubs` — single message showing all active subscriptions with Join + Renew buttons per row
- Payment-confirmed DM with visible join link + tappable Join button
- Auto renewal reminder 3 days before expiry with one-tap renew

**Reliability**
- Idempotent webhook handler — safe to replay, dedupes by `kirapayTxId`
- Double-pay prevention: Pay button routes through `/pay/<id>` server-side redirect that re-checks subscription state at click-time
- Stale Pay bubbles auto-deleted after successful subscription
- Pending setups persisted in Postgres — survive bot restarts
- Failed payments marked, not deleted — full audit trail
- Webhook signature verification (HMAC-SHA256, both base64 and hex formats supported)

**Operations**
- Hourly cron: renewal reminders at T-3d, auto-kick at expiry (with resubscribe DM)
- All Telegram messages in single-pane edit mode — menu navigation replaces the current message in place instead of stacking bubbles

## Tech stack

| | |
|---|---|
| Language | TypeScript (strict, ESM) |
| Telegram framework | [grammY](https://grammy.dev) |
| HTTP server | Express 5 |
| Database | Prisma + Neon Postgres |
| Scheduler | node-cron |
| Validation | zod |
| Logger | pino + pino-pretty |
| Dev runner | tsx (watch mode) |
| Tunneling (local) | ngrok |

## Setup

### Prerequisites

- Node.js 20+
- A Telegram bot from [@BotFather](https://t.me/BotFather) — copy the token
- A Postgres connection string ([Neon](https://neon.tech) free tier works)
- A KIRAPAY API key from [dashboard.kira-pay.com](https://dashboard.kira-pay.com)
- A Solana wallet address for receiving subscription revenue
- ngrok (or any HTTPS tunnel) — KIRAPAY needs to reach your webhook

### Install and configure

```bash
git clone https://github.com/Bhanutejabeeram/gatepay.git
cd gatepay
npm install
cp .env.example .env
```

Fill in `.env`:

```bash
TELEGRAM_BOT_TOKEN=                 # from @BotFather
BOT_USERNAME=                       # your bot's username without @
PUBLIC_BASE_URL=http://localhost:3000  # replaced with ngrok URL below

KIRAPAY_API_KEY=                    # from dashboard.kira-pay.com
KIRAPAY_BASE_URL=https://api.kira-pay.com/api
KIRAPAY_WEBHOOK_SECRET=             # any random 32+ char string

DEFAULT_SETTLEMENT_CHAIN_ID=sol
DEFAULT_SETTLEMENT_TOKEN_ADDRESS=SOL
DEFAULT_SETTLEMENT_ADDRESS=         # your fallback Solana wallet (optional)

DATABASE_URL=                       # Neon connection string

PORT=3000
LOG_LEVEL=info
NODE_ENV=development
```

### Push schema and run

```bash
npm run db:push     # applies Prisma schema to Postgres
npm run dev         # starts bot + Express server on PORT
```

### Expose the webhook

In a second terminal:

```bash
ngrok http 3000
```

Copy the `https://*.ngrok-free.app` URL into `.env` as `PUBLIC_BASE_URL`, then restart `npm run dev`. Register the webhook with KIRAPAY (idempotent, safe to re-run):

```bash
npm run kirapay:register-webhook
```

### Bot configuration

In `@BotFather`, for your bot:

- `/setcommands` → paste:
  ```
  start - Welcome menu
  setup - Register a new gated channel
  channels - List your channels
  mysubs - View your subscriptions
  wallet - View or update your Solana wallet
  cancel - Abort the current setup
  help - Show all commands
  ```
- `/setprivacy` → **Disabled** (so the bot can read channel updates)
- `/setjoingroups` → **Enabled**

### First run as a creator

1. Open Telegram, DM `@<your_bot_username>`, send `/start`
2. Create or open a private Telegram channel
3. Channel info → Administrators → Add Admin → search for your bot → enable *Invite Users via Link*
4. Bot DMs you with a *Set up channel* button → tap it
5. Walk through 4 steps: title → price → duration → Solana wallet
6. Bot replies with your subscribe URL — share anywhere

### First run as a subscriber

1. Click any subscribe URL (e.g. `https://your-ngrok-url/c/<slug>`)
2. It deep-links into the bot — tap *Pay with KIRAPAY*
3. Pay on the KIRAPAY checkout page (any chain, any token)
4. Within ~30 seconds, bot DMs you a one-time invite link — tap *Join channel*

## Architecture

```
src/
├── index.ts          entry — boots bot + Express + cron, wires shutdown
├── bot.ts            grammY bot: /start /setup /channels /mysubs /wallet /help /cancel,
│                     callback handlers, setup wizard, view helpers
├── server.ts         Express endpoints:
│                       GET  /c/:slug          subscribe redirect → bot deep-link
│                       GET  /pay/:paymentId   server-side redirect with sub re-check
│                       POST /webhooks/kirapay HMAC-verified KIRAPAY webhook
│                       GET  /healthz
├── payment.ts        startSubscriptionFlow (create pending Payment + KIRAPAY link)
│                     grantAccess (idempotent — creates invite, DMs subscriber, DMs owner,
│                                  upserts Subscription, cleans stale Pay bubbles)
├── kirapay.ts        REST client: createPaymentLink, getTransactionById, configureWebhook
├── cron.ts           hourly: renewal reminders at T-3d, kick expired + resubscribe DM
├── slug.ts           channel slug generator + customOrderId codec
├── format.ts         fmtExpiry, daysUntil, mdEscape, isValidSolanaAddress
├── config.ts         zod-validated env
├── log.ts            pino logger
└── db.ts             Prisma client singleton

prisma/
└── schema.prisma     Owner, Channel, PendingChannel, Subscription, Payment

scripts/
└── register-webhook.ts   one-shot KIRAPAY webhook registration
```

## Data model

- **Owner** — Telegram user who creates channels; stores their Solana settlement wallet (one per owner, reused across all their channels)
- **PendingChannel** — bot promoted to admin but `/setup` not finished; persisted in DB so restarts don't lose state
- **Channel** — configured gated channel with slug, price, duration, settlement wallet
- **Subscription** — one row per `(channel, subscriber)`; `expiresAt` is bumped on each renewal; `lastInviteLink` is the most recent one-time link issued
- **Payment** — one row per subscribe click; advances through `pending → succeeded` (or `failed`); stores `customOrderId`, `kirapayTxId`, source-chain hash, settlement amount, subscriber identity captured at click-time

## Hackathon submission

**Track:** Cross-Chain Checkout (Frontier Hackathon, sponsored by KIRAPAY)

**Judging criteria mapping**

| Criterion | Weight | How GatePay scores |
|---|---|---|
| Depth of KIRAPAY usage | 40% | KIRAPAY is the only payment rails — no fallback, no alternative |
| Problem solving & use case | 25% | Telegram creator monetization is a multi-billion-dollar gap with no Stripe equivalent today |
| Technical integration & scalability | 20% | Webhook idempotency, in-DB pending state, persistent sessions, multi-channel multi-owner support |
| Presentation & communication | 15% | Single-pane in-place UI; clear creator + subscriber surfaces |

## License

MIT
