# gaddaa-api

> The backend engine powering Gaddaa — a financial operating system for the working class, built on Stellar.

---

## What This Is

`gaddaa-api` is the Node.js/Express REST API that sits between the Gaddaa frontend and the Stellar blockchain. It handles everything the user never sees — fiat onramp via Paystack, wallet management, savings vault logic, Ajo group coordination, and smart contract invocations on Soroban.

The goal is simple: a market woman in Kaduna transfers ₦5,000 from her GTBank app to her Gaddaa wallet. By the time it hits her dashboard it is USDC on Stellar, earning yield inside a smart contract. She sees naira. She does not care about the rest. This API is the rest.

---

## The Three Things This API Powers

### 1. Gaddaa Wallet
Every user gets a virtual NGN account number (via Paystack Dedicated Virtual Accounts). Fund it like a normal bank transfer. Under the hood, NGN converts to USDC on Stellar via an Anchor. Send to anyone by phone number or Stellar address. Withdraw back to any Nigerian bank account.

### 2. Gaddaa Save
Time-locked savings goals. A user sets a goal name, target amount, and unlock date. Funds lock into a Soroban smart contract. Break it early and pay a penalty. Wait until the date and collect principal plus yield earned from Stellar AMM liquidity pools.

### 3. Gaddaa Ajo
Group savings circles with zero default risk. No rotation. No "collect and ghost." Every member saves until the maturity date. The Soroban contract holds all funds, sweeps them into yield-bearing tokens (YLDS/USDY on Stellar), and on maturity day distributes principal to everyone and the interest jackpot only to perfect payers. The contract is the treasurer.

---

## Tech Stack

| Layer | Technology |
|---|---|
| Runtime | Node.js 20 LTS |
| Framework | Express.js |
| Language | TypeScript |
| Database | PostgreSQL 16 |
| Cache / Idempotency | Redis |
| ORM | Prisma |
| Auth | JWT (access + refresh token rotation) |
| Payments | Paystack (onramp + DVA) · Flutterwave (offramp) |
| Blockchain | Stellar SDK · Soroban Client |
| Validation | Zod |
| Testing | Jest + Supertest |
| Process Manager | PM2 |
| Containerisation | Docker + Docker Compose |

---

## Project Structure

```
gaddaa-api/
├── src/
│   ├── config/          # env, database, redis, stellar config
│   ├── modules/
│   │   ├── auth/        # registration, login, JWT, refresh
│   │   ├── wallet/      # balance, send, receive, DVA management
│   │   ├── save/        # savings goals, vault interactions
│   │   ├── ajo/         # group creation, contributions, payouts
│   │   └── webhooks/    # Paystack + Flutterwave webhook handlers
│   ├── services/
│   │   ├── stellar/     # Horizon API, account management, transactions
│   │   ├── soroban/     # smart contract invocations
│   │   ├── paystack/    # DVA generation, payment collection
│   │   └── anchor/      # USDC issuance and redemption
│   ├── middleware/      # auth guard, error handler, rate limiter
│   ├── utils/           # idempotency, crypto helpers, formatters
│   └── app.ts           # Express app entry point
├── prisma/
│   └── schema.prisma    # database schema
├── tests/
├── docker-compose.yml
├── .env.example
└── package.json
```

---

## Getting Started

### Prerequisites
- Node.js 20+
- PostgreSQL 16
- Redis
- A Paystack test account
- Stellar testnet account (get one at [Stellar Laboratory](https://laboratory.stellar.org))

### Installation

```bash
# Clone the repo
git clone https://github.com/your-username/gaddaa-api.git
cd gaddaa-api

# Install dependencies
npm install

# Copy environment variables
cp .env.example .env
# Fill in your values — see Environment Variables section below

# Run database migrations
npx prisma migrate dev

# Start development server
npm run dev
```

### With Docker

```bash
docker-compose up --build
```

This spins up the API, PostgreSQL, and Redis together.

---

## Environment Variables

```env
# App
NODE_ENV=development
PORT=3000
JWT_SECRET=your_jwt_secret
JWT_REFRESH_SECRET=your_refresh_secret

# Database
DATABASE_URL=postgresql://user:password@localhost:5432/gaddaa

# Redis
REDIS_URL=redis://localhost:6379

# Paystack
PAYSTACK_SECRET_KEY=sk_test_xxxx
PAYSTACK_WEBHOOK_SECRET=your_webhook_secret

# Flutterwave
FLW_SECRET_KEY=FLWSECK_TEST-xxxx
FLW_WEBHOOK_SECRET=your_flw_webhook_secret

# Stellar
STELLAR_NETWORK=testnet
STELLAR_HORIZON_URL=https://horizon-testnet.stellar.org
STELLAR_MASTER_SECRET=your_stellar_secret_key

# Anchor (USDC issuance)
ANCHOR_BASE_URL=https://testanchor.stellar.org
ANCHOR_JWT_SECRET=your_anchor_jwt
```

---

## API Overview

### Auth
```
POST   /api/v1/auth/register         # create account with phone number
POST   /api/v1/auth/login            # login, returns access + refresh token
POST   /api/v1/auth/refresh          # rotate refresh token
POST   /api/v1/auth/logout           # revoke refresh token
```

### Wallet
```
GET    /api/v1/wallet                # get balance and virtual account number
POST   /api/v1/wallet/send           # send to phone number or stellar address
GET    /api/v1/wallet/transactions   # transaction history
POST   /api/v1/wallet/withdraw       # offramp to Nigerian bank account
```

### Save
```
GET    /api/v1/save                  # list all savings goals
POST   /api/v1/save                  # create a new savings goal
GET    /api/v1/save/:id              # get goal details + yield earned
POST   /api/v1/save/:id/break        # early withdrawal with penalty
```

### Ajo
```
GET    /api/v1/ajo                   # list groups user belongs to
POST   /api/v1/ajo                   # create a new Ajo group
POST   /api/v1/ajo/join              # join a group via invite code
GET    /api/v1/ajo/:id               # group details, members, contribution status
POST   /api/v1/ajo/:id/contribute    # make a contribution for current cycle
GET    /api/v1/ajo/:id/history       # contribution history for a group
```

### Webhooks
```
POST   /api/v1/webhooks/paystack     # Paystack payment events (HMAC-SHA512 verified)
POST   /api/v1/webhooks/flutterwave  # Flutterwave events (HMAC-SHA256 verified)
```

---

## Key Engineering Decisions

**Monetary values stored as integers (kobo)**
All amounts in the database are stored as `BIGINT` in kobo (1 NGN = 100 kobo) to avoid floating-point errors. Conversion to naira happens only at the presentation layer.

**Idempotency on all payment operations**
Every payment endpoint accepts an `Idempotency-Key` header. Redis stores processed keys for 24 hours. Duplicate requests return the cached response without re-processing. This protects against double-charges on retries.

**Webhook verification before processing**
All Paystack webhooks are verified via HMAC-SHA512 signature before any business logic runs. All Flutterwave webhooks verified via HMAC-SHA256. Invalid signatures return 401 and are logged.

**Stellar transactions are fire-and-verify**
The API submits transactions to Stellar Horizon and stores the transaction hash immediately. A background worker polls Horizon to confirm finality and updates internal state. This prevents blocking API responses on blockchain confirmation times.

**Virtual account per user, not per transaction**
Each user gets one dedicated Paystack virtual account number on registration. All deposits to that account number are attributed to that user automatically via webhook. No payment link generation needed per deposit.

---

## Database Schema (Key Tables)

```
users           — identity, phone, KYC status
wallets         — stellar address, internal USDC balance, naira display balance
savings_goals   — vault contract ID, target, current, unlock date, status
ajo_groups      — contract ID, member count, contribution amount, maturity date
ajo_members     — group membership, rotation position, contribution status
transactions    — full ledger, stellar tx hash, idempotency key, type, status
webhook_logs    — provider, event type, payload hash, processed timestamp
```

---

## Contributing

This project is part of the Stellar Drips wave. Contributors are welcome across:

- **Backend logic** — Express routes, service layer, database queries
- **Stellar integration** — Horizon API, Soroban contract calls, Anchor flows
- **Testing** — unit tests for services, integration tests for API routes
- **DevOps** — CI/CD pipeline, Docker optimisation, Leapcell deployment

Please read through this README fully before picking up an issue. Understand the three modules (Wallet, Save, Ajo) and the Stellar data flow before touching payment or blockchain code.

Open an issue before submitting a large PR. Discuss first, code second.

---

## Related Repos

| Repo | Description |
|---|---|
| [gaddaa-web](https://github.com/your-username/gaddaa-web) | Next.js frontend — the user-facing product |
| [gaddaa-contracts](https://github.com/your-username/gaddaa-contracts) | Soroban smart contracts — Savings Vault and Ajo Escrow |

---

## License

MIT — build freely, give credit.

---

*Built in Nigeria. For the working class. On Stellar.*