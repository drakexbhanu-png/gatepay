# GatePay — Project Write-Up

> Pay any token on any chain, settle SOL on Solana, get into the channel.

**Live bot:** [@GatePayy_bot](https://t.me/GatePayy_bot)
**Repository:** [github.com/Bhanutejabeeram/gatepay](https://github.com/Bhanutejabeeram/gatepay)
**Hackathon track:** Cross-Chain Checkout (Frontier Hackathon, sponsored by KIRAPAY)

---

## 1. What GatePay is

GatePay is a Telegram bot that turns any private channel into a paid subscription. Subscribers pay with any token on any blockchain — channel owners always receive native **SOL on Solana**, settled in seconds. The bot auto-grants Telegram channel access on payment confirmation, sends renewal reminders 3 days before expiry, and removes expired members from the channel.

It's effectively *Stripe for paid Telegram channels*, built natively on cross-chain payment infrastructure. Every creator who signs up becomes a new KIRAPAY merchant; every subscription pulls liquidity across chains into Solana.

## 2. The problem

Telegram is the de-facto home of crypto communities — analyst groups, trading rooms, signal channels, paid newsletters. But monetizing a private channel today runs on screenshots and DMs:

- Subscriber asks how to pay → owner says *"send USDT to this address"* → subscriber holds ETH on Arbitrum, USDC on Base, or SOL on Solana → friction kills the sale.
- If an owner accepts multiple chains, they juggle wallets across networks just to know who paid for what.
- Tracking expiry, renewals, and access removal is manual.

There's no Stripe equivalent for paid Telegram communities. GatePay fills that gap.

## 3. How GatePay integrates KIRAPAY

KIRAPAY is structurally the only thing that makes GatePay possible. Strip KIRAPAY out and the bot has no payment flow at all.

### 3.1 What we use KIRAPAY for

| KIRAPAY surface | Where it's used in GatePay |
|---|---|
| `POST /api/link/generate` | Creating a fresh cross-chain checkout link per subscribe click. We pass `tokenOut: { chainId: "sol", address: "SOL" }` so settlement is always SOL on Solana, regardless of what the subscriber pays with. |
| `customOrderId` field | We encode `"{telegramUserId}:{channelId}:{durationDays}:{nonce}"` here. KIRAPAY ships it back to us on the webhook, letting us grant the exact right access to the exact right subscriber. This is our reconciliation key. |
| `POST /api/webhooks` | Registered once at deploy time. KIRAPAY then pushes us `transaction.created` and `transaction.succeeded` events. |
| Webhook payload | Drives `grantAccess` — creates a one-time Telegram invite link, DMs the subscriber, DMs the owner with metrics, upserts the Subscription row. |
| `GET /api/wallet/transactions/{id}` | Fallback when the webhook payload doesn't include `customOrderId` — we fetch the full transaction record to reconcile. |
| Signature verification | HMAC-SHA256 on the raw body, base64-encoded in `X-Kirapay-Signature: sha256=...`. We verify on every inbound webhook. |

### 3.2 Cross-chain magic in one paragraph

A subscriber clicking *Pay with KIRAPAY* lands on a hosted checkout that detects their wallet and lists supported tokens across 11 chains (Ethereum, Polygon, Base, BNB, Arbitrum, Optimism, Avalanche, HyperEVM, Unichain, Soneium, Solana). They pay in whatever they already hold. KIRAPAY's solver network routes the payment cross-chain and deposits native SOL into the creator's Solana wallet. The whole cross-chain hop completes in 30–60 seconds. We never custody funds.

### 3.3 Why this is "depth", not "an add-on"

- The bot has no other payment path. There's no manual fallback, no Stripe option, no "send USDC to this address" path.
- KIRAPAY's cross-chain routing is what unlocks Telegram's audience — Telegram users hold assets on every chain; restricting checkout to one chain would cut conversion by 10×.
- We chose Solana settlement specifically because creators want native SOL exposure but the audience pays from everywhere. KIRAPAY is the only piece that bridges those two.
- Every creator who onboards adds a new merchant to KIRAPAY's network — GatePay is a distribution layer, not just a consumer.

## 4. How to use GatePay — step by step

### 4.1 Creator flow

1. Open Telegram and DM **[@GatePayy_bot](https://t.me/GatePayy_bot)** → send `/start`. The bot replies with a menu (My channels, My subscriptions, Wallet, Help).
2. Create or open the private Telegram channel you want to monetize.
3. In that channel: tap the channel name → *Administrators* → *Add Admin* → search **@GatePayy_bot** → enable the *Invite Users via Link* permission → confirm.
4. The bot DMs you instantly with **"Connected to *your channel name*"** and a **Set up channel** button.
5. Tap **Set up channel** to start the 4-step wizard:
   - **Step 1** Display title — what subscribers see at checkout (max 80 chars)
   - **Step 2** Subscription price in USD (e.g. `9.99`)
   - **Step 3** Subscription duration in days (e.g. `30`)
   - **Step 4** Solana wallet address — first time only; reused for all your future channels
6. The bot confirms with **"*Your Channel* is live"** and shows your subscribe URL. Share it anywhere — bio, tweets, newsletters.
7. Manage anytime via `/channels` — shows active subscriber count, lifetime revenue, and a Delete button per channel. View or change your settlement wallet via `/wallet`.

### 4.2 Subscriber flow

1. Click a subscribe URL (e.g. `https://gatepay-production.up.railway.app/c/abc123xy`). It deep-links into Telegram and opens GatePay.
2. The bot shows the channel name, price, and duration with a **Pay with KIRAPAY** button.
3. Tap the button — it opens KIRAPAY's hosted checkout.
4. Connect your wallet (MetaMask, Phantom, WalletConnect) and pick a token on any supported chain.
5. Confirm the transaction. KIRAPAY routes it cross-chain to native SOL on the creator's Solana wallet.
6. Within ~30 seconds, the bot DMs you a **Payment confirmed** message with a one-time **Join channel** link.
7. Tap **Join channel** — you're in.
8. Run `/mysubs` anytime to see your active subscriptions and join links. The bot DMs you a one-tap renewal 3 days before expiry, and removes you from the channel at expiry (with a resubscribe link).

## 5. Technical architecture (high level)

- **Bot** — TypeScript + grammY, runs on long-polling, deployed on Railway
- **HTTP server** — Express, hosts `POST /webhooks/kirapay`, `GET /c/:slug` (subscribe redirect), `GET /pay/:paymentId` (server-side double-pay guard)
- **Database** — Prisma + Postgres on Neon; tables for Owner, Channel, PendingChannel, Subscription, Payment
- **Scheduler** — node-cron, runs hourly; sends renewal reminders at T-3d, removes expired members at T+0
- **State machine for Payments** — `pending → succeeded` (or `failed`); idempotent on `kirapayTxId` so webhook replays are safe
- **Single-pane UX** — all navigation buttons edit the existing message in place; no stacked bubbles

## 6. Reliability + scale story

- Webhook handler is idempotent — replaying the same `kirapayTxId` is a no-op.
- Pending setups persist in Postgres, so a bot restart doesn't lose creator state mid-wizard.
- Stale "Pay with KIRAPAY" message bubbles auto-delete from chat after a successful subscription, preventing accidental double-pay.
- `/pay/:paymentId` server-side redirect re-checks subscription state at click time — even a tap on an old chat bubble bounces back to "Already subscribed" rather than reaching KIRAPAY.
- Solana wallet validation (base58, 32–44 chars) on every entry.
- All inbound webhook payloads validated via zod against both KIRAPAY's documented shape *and* the actual shape they ship (the two differ; we handle both).

## 7. Official links

- **Live bot** — [https://t.me/GatePayy_bot](https://t.me/GatePayy_bot)
- **GitHub repository** — [https://github.com/Bhanutejabeeram/gatepay](https://github.com/Bhanutejabeeram/gatepay)
- **KIRAPAY** — [https://www.kira-pay.com](https://www.kira-pay.com)
- **KIRAPAY developer docs** — [https://docs.kira-pay.com](https://docs.kira-pay.com)
- **Frontier Hackathon** — Cross-Chain Checkout track, sponsored by KIRAPAY
