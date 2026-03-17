# Locus: Onchain Agentic Payment Infrastructure

> **One USDC balance for wallets, APIs, deployments, checkout, and more.**

Locus is a payment infrastructure platform purpose-built for AI agents. It provides a unified USDC-based system on the **Base blockchain** that enables agents to autonomously manage wallets, process payments, deploy cloud services, access third-party APIs, hire human workers, and interact with traditional finance — all governed by human-configured spending controls.

---

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Core Components](#core-components)
  - [Smart Wallets (ERC-4337)](#1-smart-wallets-erc-4337)
  - [Wrapped APIs](#2-wrapped-apis)
  - [Checkout System](#3-checkout-system)
  - [Build (PaaS)](#4-build-paas)
  - [USDC Transfers](#5-usdc-transfers)
  - [Tasks (Human Labor Marketplace)](#6-tasks-human-labor-marketplace)
  - [Laso Finance (Fiat Off-Ramp)](#7-laso-finance-fiat-off-ramp)
- [Spending Controls & Governance](#spending-controls--governance)
- [Authentication & Agent Onboarding](#authentication--agent-onboarding)
- [Payment Router Smart Contract](#payment-router-smart-contract)
- [Provider Catalog & Pricing](#provider-catalog--pricing)
- [Beta Program](#beta-program)
- [Key Diagrams](#key-diagrams)

---

## Architecture Overview

```
+------------------------------------------------------------------+
|                        HUMAN OPERATOR                            |
|  (Dashboard: app.paywithlocus.com)                               |
|  - Set spending controls (allowance, max tx, approval threshold) |
|  - Approve high-value transactions                               |
|  - View transaction audit log                                    |
|  - Toggle wrapped API providers                                  |
+----------------------------------+-------------------------------+
                                   |
                          Governance & Controls
                                   |
                                   v
+------------------------------------------------------------------+
|                         AI AGENT                                 |
|  Authenticates via claw_dev_* API key -> JWT (15 min TTL)        |
+------+--------+--------+--------+--------+--------+-------------+
       |        |        |        |        |        |
       v        v        v        v        v        v
  +--------+ +------+ +-------+ +-----+ +------+ +------+
  | Wallet | | APIs | |Checkout| |Build| |Tasks | | Laso |
  | (USDC) | |(Wrap)| | (Pay) | |(PaaS)| |(Hire)| |(Fiat)|
  +--------+ +------+ +-------+ +-----+ +------+ +------+
       |        |        |        |        |        |
       +--------+--------+--------+--------+--------+
                         |
                    Base Blockchain
                   (USDC + Gasless)
```

### How It All Connects

1. **Human** creates account, deploys smart wallet, generates API key, sets spending limits
2. **Agent** receives API key, reads SKILL.md (canonical endpoint reference), authenticates
3. **Agent** performs actions (API calls, payments, deploys) — each deducts USDC from wallet
4. **Spending controls** enforce limits; high-value actions require human approval
5. **All transactions** settle on Base as USDC transfers (gasless via paymaster)

---

## Core Components

### 1. Smart Wallets (ERC-4337)

Locus implements a **dual-key non-custodial smart wallet** system on Base using the ERC-4337 (Account Abstraction) standard.

#### Wallet Architecture

```
+---------------------------------------------------------------+
|                     Locus Smart Wallet                        |
|                  (ERC-4337 on Base)                            |
|                                                               |
|  +------------------+     +--------------------+              |
|  |    User Key      |     |  Permissioned Key  |              |
|  | (client-side,    |     |  (AWS KMS,         |              |
|  |  never stored    |     |   revocable by     |              |
|  |  on Locus)       |     |   user at any time)|              |
|  +--------+---------+     +---------+----------+              |
|           |                         |                         |
|           |   Both can sign txs     |                         |
|           +------------+------------+                         |
|                        |                                      |
|                        v                                      |
|              +---------+----------+                           |
|              | Single-Signer      |                           |
|              | Account validates  |                           |
|              | against either key |                           |
|              +--------------------+                           |
|                                                               |
|  Key Security Properties:                                     |
|  - UUPS upgradeability DISABLED (_authorizeUpgrade reverts)   |
|  - Implementation is IMMUTABLE                                |
|  - User key can revoke permissioned key at any time           |
|  - Permissioned key can rotate itself, but cannot revoke user |
+---------------------------------------------------------------+
```

#### Key Management

| Key | Storage | Capabilities | Revocation |
|-----|---------|-------------|------------|
| **User Key** | Client-side only, never on Locus servers | Full wallet control, can revoke permissioned key | N/A (owner) |
| **Permissioned Key** | AWS KMS | Sign transactions for agent operations | User can revoke via `revokePermissionedKey()` |

#### Wallet Deployment

- All wallets deployed as **deterministic ERC1967 proxies** via `CREATE2` through `LocusFactory`
- Addresses are **predictable before deployment** using `LocusFactory.getAddress()`
- Factory extends Solady's ERC4337Factory (52-byte runtime code minimal proxies)
- Permissioned key set **atomically during deployment** to prevent frontrunning

#### Gasless Transactions

All transactions are **fully gasless** — Locus operates a proprietary **paymaster** on Base that sponsors all gas fees. Users never need ETH for transaction fees.

#### Subwallet System (Email Escrow)

Subwallets are isolated escrow contracts that enable sending USDC to email addresses:

```
Sender Wallet                    Subwallet (Escrow)              Recipient
     |                                |                              |
     |-- createAndFundSubwallet() --->|                              |
     |   (UserOp via EntryPoint,      |                              |
     |    signed by permissioned key) |                              |
     |                                |                              |
     |                                |--- Email with OTP ---------> |
     |                                |                              |
     |                                |<-- Claim with OTP + addr --- |
     |                                |                              |
     |                                |--- transferToken() --------> |
     |                                |   (to chosen wallet)         |
     |                                |                              |
     |                                |-- deactivate() -->           |
     |                                |  (returns to reuse pool)     |
```

**Key properties:**
- Each subwallet enforces a `disburseBefore` deadline — funds cannot be disbursed after expiry
- After claiming, subwallets deactivate and return to a **reuse pool** (activated via `activate()` instead of deploying new proxies)
- Hard cap of **100 subwallet proxies** per wallet (`MAX_SUBWALLETS`)

#### ILocusSmartWallet Interface

| Category | Functions |
|----------|-----------|
| **Initialization** | `initialize()` — sets owner, USDC address, subwallet impl, permissioned key |
| **Owner** | `revokePermissionedKey()` |
| **Permissioned Key** | `setPermissionedKey()` — key rotation |
| **Subwallets** | `createAndFundSubwallet()`, `createAndFundSubwalletUSDC()`, `reclaimSubwallet()`, `disburseSubwallet()` |
| **Transfers** | `transferUSDC()` — direct USDC transfers |
| **Views** | `getPermissionedKey()`, `activePermissionedKeyPublic()`, `activeSessionKeyPublic()`, `getDeployedSubwallet()`, `getSubwalletStats()`, `getSubwalletBalance()` |

---

### 2. Wrapped APIs

Wrapped APIs let agents call third-party services through a **unified interface** with automatic authentication and **per-call USDC billing**. No upstream API keys needed — Locus manages all credentials.

#### How Wrapped APIs Work

```
AI Agent                        Locus Platform                   Upstream Provider
   |                                 |                                |
   |-- 1. GET /wrapped/md ---------->|                                |
   |   (Discover available APIs)     |                                |
   |<-- Provider catalog ------------|                                |
   |                                 |                                |
   |-- 2. POST /wrapped/openai/chat->|                                |
   |   + API key + params            |                                |
   |                                 |-- 3. Reserve USDC from wallet  |
   |                                 |     (allowance check)          |
   |                                 |                                |
   |                                 |-- 4. Forward to upstream ----->|
   |                                 |     (Locus credentials)        |
   |                                 |                                |
   |                                 |<-- 5. Response ---------------|
   |                                 |                                |
   |                                 |-- 6. Deduct USDC on Base      |
   |                                 |     (agent wallet -> treasury) |
   |                                 |                                |
   |<-- 7. Return upstream response--|                                |
   |   (data field)                  |                                |
```

#### Charge-on-Success Safety Model

- Funds are **reserved before** upstream execution (prevents overspending)
- **Failed calls restore** reserved funds — no charges for errors
- Successful calls transfer USDC from agent wallet to Locus treasury **on Base**
- High-value calls exceeding approval threshold return **HTTP 202** with an approval URL; human approves once, execution proceeds automatically

#### Dual Cost Structure

| Component | Pricing |
|-----------|---------|
| **Upstream cost** | Varies by provider (passed through) |
| **Locus fee** | Flat **$0.003/call** (most providers) OR **15% markup** on token costs (OpenAI/Gemini) |

#### API Discovery Endpoints

```
GET /api/wrapped/md                         # Lightweight markdown catalog
GET /api/wrapped/md?provider=[name]         # Detailed provider info
GET /api/wrapped                            # Structured JSON listing
```

#### Execution Pattern

```bash
# Call any wrapped API
curl -X POST https://api.paywithlocus.com/api/wrapped/<provider>/<endpoint> \
  -H "Authorization: Bearer YOUR_LOCUS_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{ ...upstream parameters... }'
```

#### Error Codes

| Code | Meaning |
|------|---------|
| 200 | Success — response in `data` field |
| 202 | Awaiting human approval (includes `approval_url`) |
| 400 | Malformed request |
| 402 | Insufficient balance |
| 403 | Policy violation / disabled endpoint / allowance exceeded |
| 404 | Unknown provider/endpoint |
| 502 | Upstream failure (no charge) |

---

### 3. Checkout System

Locus Checkout enables merchants to accept **USDC payments on Base** through a hosted checkout experience with a React SDK.

#### Checkout Flow

```
Merchant Server                Locus API                  Buyer/Agent
      |                            |                          |
      |-- Create session --------->|                          |
      |   (amount, description,    |                          |
      |    webhook URL)            |                          |
      |                            |                          |
      |<-- sessionId + URL --------|                          |
      |                            |                          |
      |-- Render LocusCheckout --->|                          |
      |   (React SDK component)    |                          |
      |                            |--- Checkout page ------->|
      |                            |                          |
      |                            |    Select payment method:|
      |                            |    1. Locus Wallet       |
      |                            |    2. External Wallet    |
      |                            |    3. AI Agent (API)     |
      |                            |                          |
      |                            |<-- Payment submitted ----|
      |                            |                          |
      |                            |-- On-chain confirmation  |
      |                            |                          |
      |<-- Webhook: PAID ----------|                          |
      |   (txHash, payerAddress,   |                          |
      |    paidAt)                 |                          |
```

#### Session Lifecycle

```
PENDING  -->  PAID      (on-chain confirmation)
   |
   +-------->  EXPIRED   (no payment before expiresAt, default 30 min)
   |
   +-------->  CANCELLED (seller-initiated)
```

Once a session exits PENDING, it **cannot be modified**.

#### Three Payment Methods

| Method | How It Works | Gas Fees |
|--------|-------------|----------|
| **Locus Wallet** | Single-click for account holders, no wallet popup | Gasless (paymaster) |
| **External Wallet** | MetaMask / Coinbase / WalletConnect via Payment Router contract | Buyer pays gas |
| **AI Agent** | Programmatic session creation + payment via API | Gasless |

#### React SDK Integration

```bash
npm install @withlocus/checkout-react
```

**Three display modes:**
- **Embedded** — inline iframe (min-height: 700px)
- **Popup** — centered 450x650 window
- **Redirect** — full-page navigation to hosted checkout

```jsx
import { LocusCheckout } from '@withlocus/checkout-react';

<LocusCheckout
  sessionId={sessionId}
  mode="embedded"           // 'embedded' | 'popup' | 'redirect'
  onSuccess={(data) => {
    // data: { sessionId, amount, currency, txHash, payerAddress, paidAt }
  }}
  onCancel={() => {}}
  onError={(err) => {}}
/>
```

#### Webhook Verification

Locus POSTs to merchant endpoints with HMAC-SHA256 signature in `X-Signature-256` header. Merchants verify by computing HMAC-SHA256 of JSON payload using webhook secret, then comparing with timing-safe comparison.

---

### 4. Build (PaaS)

Build with Locus is an **agent-native Platform-as-a-Service** that deploys containerized services to AWS. It eliminates manual AWS console interaction, Dockerfile creation, and DevOps overhead.

#### Deployment Architecture

```
+-------------------------------------------------------------------+
|                        Build with Locus                           |
|                                                                   |
|  +----------+     +-----------+     +-----------+                 |
|  |  Project  |---->|Environment|---->|  Service  |                |
|  | (regional |     |(dev/stage/|     |(container)|                |
|  |  grouping)|     | prod)     |     |           |                |
|  +----------+     +-----------+     +-----+-----+                |
|                                           |                       |
|                    Source Types:           |                       |
|                    +-- GitHub repo         |                       |
|                    +-- Docker image        |                       |
|                    +-- Git push / S3       |                       |
|                                           v                       |
|                                    +------+------+                |
|                                    | Deployment  |                |
|                                    | Lifecycle:  |                |
|                                    |             |                |
|                                    | queued      |                |
|                                    |   |         |                |
|                                    |   v         |                |
|                                    | building    |  <-- skip for  |
|                                    |   |         |     Docker img |
|                                    |   v         |                |
|                                    | deploying   |                |
|                                    |   |         |                |
|                                    |   v         |                |
|                                    | healthy /   |                |
|                                    | failed      |                |
|                                    +-------------+                |
|                                                                   |
|  Addons:  +-- PostgreSQL (auto-injects DATABASE_URL)              |
|           +-- Redis (auto-injects REDIS_URL)                      |
|                                                                   |
|  Domains: +-- Bring your own (CNAME + SSL)                        |
|           +-- Purchase through Locus                              |
+-------------------------------------------------------------------+
```

#### Deployment Timing

| Phase | Duration | Notes |
|-------|----------|-------|
| Queued | 5-30 sec | Normal; 2-5 min possible |
| Building | 2-5 min | Clone + Docker build (skipped for image source) |
| Deploying | 30-90 sec | Container startup + health check |
| **Total (GitHub/git)** | **3-7 min** | |
| **Total (Docker image)** | **1-2 min** | |

#### Health Check Requirement

**Critical:** Services MUST expose a `/health` endpoint returning HTTP 200. Without it, deployments will time out and fail.

#### Environment Variable Priority

```
Service variables  >  Environment variables  >  Addon variables (auto-injected)
```

Variables require a new deployment to take effect.

#### Agent Workflow for Deployments

1. Load SKILL.md (canonical API reference)
2. Exchange `claw_*` API key for JWT (15-min TTL)
3. Create project -> environment -> service -> trigger deployment
4. Poll status every 60 seconds (silent monitoring)
5. Report completion with service URL or failure logs

---

### 5. USDC Transfers

Two methods for sending USDC:

#### Direct to Address

```
Agent/User --> Locus API --> On-chain USDC transfer --> Recipient wallet
```

Transfer executes **immediately on-chain** to any Base wallet address.

#### Direct to Email

```
Sender --> createAndFundSubwallet() --> Subwallet (escrow)
                                            |
                                            v
                                     Email with OTP --> Recipient
                                            |
                                            v
                                     Claim to any wallet
```

- Funds held in secure subwallet until recipient claims
- Recipient receives OTP email and can withdraw to any wallet address
- Time-limited escrow with `disburseBefore` deadline

---

### 6. Tasks (Human Labor Marketplace)

Agents can hire human workers for specialized tasks like graphic design and content creation.

#### Task Flow

```
Agent                           Locus Tasks                    Human Tasker
  |                                  |                              |
  |-- 1. Select category + timeline->|                              |
  |      (1 day / 3 days / 7 days)  |                              |
  |                                  |                              |
  |-- 2. Select price tier -------->|                              |
  |      (1=budget, 2=mid, 3=prem) |                              |
  |                                  |                              |
  |-- 3. Submit work request ------>|                              |
  |      (description + refs)       |                              |
  |                                  |-- 4. Match with tasker ----->|
  |                                  |                              |
  |                                  |<-- 5. Deliver work ---------|
  |                                  |                              |
  |<-- 6. Forward deliverable ------|                              |
```

**Key conditions:**
- 1-day grace period applies to all deadlines
- Refund issued if no suitable tasker found
- If tasker charges less than selected tier, difference is refunded

---

### 7. Laso Finance (Fiat Off-Ramp)

Laso Finance bridges USDC to traditional finance — prepaid Visa cards and Venmo/PayPal payments.

#### Laso Architecture

```
Agent (USDC wallet)
   |
   |-- Authenticate via x402 proxy ($0.001 USDC)
   |
   +-- Prepaid Visa Cards
   |   |-- Order: $5-$1,000
   |   |-- Ready in 7-10 seconds (poll every 2-3s)
   |   |-- Non-reloadable, US-only (IP-locked)
   |   |-- Returns: card number, CVV, expiration
   |   +-- Check merchant compatibility database before ordering
   |
   +-- Venmo / PayPal Payments
       |-- Amount: $5-$1,000 USDC
       |-- Venmo: requires recipient phone number
       |-- PayPal: requires recipient email address
       +-- Must confirm details with human before execution
```

#### Laso Cost Structure

| Operation | Cost |
|-----------|------|
| Authentication | $0.001 USDC |
| Token refresh | Free |
| Card ordering | Card amount ($5-$1,000) |
| Venmo/PayPal payment | Payment amount ($5-$1,000) |
| Status checks, queries, balance, merchant search | Free |

#### Merchant Compatibility Statuses

| Status | Meaning |
|--------|---------|
| `accepted` | Confirmed successful usage |
| `not_accepted` | Documented declines (don't order) |
| `unknown` | Unclear outcomes (proceed with caution) |
| Not listed | No prior attempts (presumed compatible) |

---

## Spending Controls & Governance

Locus implements a **three-tier spending control** system configured by human operators through the dashboard:

```
+-------------------------------------------------------------------+
|                    Spending Controls                               |
|                                                                   |
|  +------------------+  +-------------------+  +-----------------+ |
|  |   Allowance      |  | Max Transaction   |  | Approval        | |
|  |                  |  | Size              |  | Threshold       | |
|  | Total USDC the   |  | Per-transaction   |  | Amount above    | |
|  | agent can spend  |  | ceiling           |  | which human     | |
|  |                  |  |                   |  | must approve    | |
|  | Blank = full     |  | Exceeding returns |  |                 | |
|  | balance access   |  | HTTP 403          |  | Blank = no      | |
|  |                  |  |                   |  | approval needed | |
|  | Decrements with  |  |                   |  | $0 = all need   | |
|  | each spend       |  |                   |  | approval        | |
|  +------------------+  +-------------------+  +-----------------+ |
|                                                                   |
|  Applied universally across: Wrapped APIs, Build, Checkout,       |
|  Tasks, Laso Finance, USDC Transfers                              |
+-------------------------------------------------------------------+
```

**Approval flow for high-value transactions:**

```
Agent attempts action above threshold
        |
        v
   HTTP 202 returned
   { pending_approval_id, approval_url, estimated_cost_usdc }
        |
        v
   Human reviews at approval_url
        |
   +----+----+
   |         |
Approve    Deny
   |         |
   v         v
Action     Agent
executes   notified
auto.      of denial
```

---

## Authentication & Agent Onboarding

### Production Setup

```
1. Register at app.paywithlocus.com
2. Create Wallet (deploys to Base in ~30 seconds)
3. Generate API Key (claw_dev_* prefix, shown once)
4. Fund wallet (USDC on Base, or request credits)
5. Configure spending controls
6. Give agent the API key + SKILL.md URL
7. Verify: GET /api/pay/balance
```

### Beta Environment (Agent Self-Registration)

```
Agent                           Locus Beta API
  |                                  |
  |-- POST /api/register ----------->|
  |   (optional: name, email)       |
  |                                  |
  |<-- apiKey, ownerPrivateKey, -----|
  |    walletId, walletAddress       |
  |    (SAVE IMMEDIATELY -           |
  |     shown only once)             |
  |                                  |
  |-- Poll GET /api/status --------->|
  |   (wallet deployment)            |
  |                                  |
  |-- Share claim URL with human --->|  Human links to dashboard
  |                                  |
```

**Beta defaults:** $10 allowance, $5 max transaction, 5 registrations/IP/hour

### Environment Comparison

| | Production | Beta |
|---|---|---|
| **API Base** | `api.paywithlocus.com/api` | `beta-api.paywithlocus.com/api` |
| **Dashboard** | `app.paywithlocus.com` | `beta.paywithlocus.com` |
| **Key prefix** | `claw_dev_*` | `claw_dev_*` |
| **Chain** | Base (mainnet) | Base (mainnet) — **real USDC** |
| **Agent self-register** | No | Yes |

---

## Payment Router Smart Contract

The Payment Router is an on-chain contract on Base that handles **external wallet checkout payments** (MetaMask, Coinbase Wallet, WalletConnect).

### Contract Addresses

| Network | Address |
|---------|---------|
| Base mainnet (8453) | `0x34184b7bCB4E6519C392467402DB8a853EF57806` |
| USDC (Base mainnet) | `0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913` |
| USDC (Base Sepolia) | `0x036CbD53842c5426634e7929541eC2318f3dCF7e` |

### External Wallet Payment Flow

```
Buyer Wallet                Payment Router              Locus Backend
     |                      (on Base)                        |
     |                           |                           |
     |-- 1. approve(router,  --->|                           |
     |       amount)             |                           |
     |                           |                           |
     |-- 2. pay(sessionId,  ---->|                           |
     |       seller, amount)     |                           |
     |                           |-- emit CheckoutPayment -->|
     |                           |   (sessionId, sender,     |
     |                           |    recipient, amount)     |
     |                           |                           |
     |                           |               Update session: PAID
     |                           |               Notify merchant
```

**Or in a single transaction using EIP-2612:**

```
Buyer -- payWithPermit(sessionId, recipient, amount, deadline, v, r, s) --> Router
```

### Session ID Encoding

UUIDs are converted to `bytes32`:
```
UUID:    a1b2c3d4-e5f6-7890-abcd-ef1234567890
bytes32: 0xa1b2c3d4e5f67890abcdef123456789000000000000000000000000000000000
```

### Contract Interface

| Function | Purpose |
|----------|---------|
| `pay(bytes32 sessionId, address recipient, uint256 amount)` | Standard 2-tx payment |
| `payWithPermit(...)` | Single-tx payment with EIP-2612 permit |
| `sessionPaid(bytes32 sessionId)` | View: check if session is paid |
| `usdc()` | View: USDC token address |

**USDC uses 6 decimals:** $25.00 = `25000000`

---

## Provider Catalog & Pricing

### AI & Language Models

| Provider | Endpoints | Pricing Model |
|----------|-----------|---------------|
| **OpenAI** | Chat, embeddings, image gen, TTS, moderation | 15% markup on token costs |
| **Google Gemini** | Multimodal chat, vision, PDF, embeddings | 15% markup on token costs |
| **fal.ai** | 600+ generative models (image, video, audio) | Free to $0.50 per generation |

### Web & Data

| Provider | Endpoints | Price per Call |
|----------|-----------|---------------|
| **Firecrawl** | Scrape, crawl, extract, batch, sitemap | $0.003+ (credit-based) |
| **Exa** | Semantic search, content retrieval, Q&A | $0.010/search |
| **Browser Use** | AI browser automation | $0.01+ (model-dependent) |

### Sales & Enrichment

| Provider | Endpoints | Price per Call |
|----------|-----------|---------------|
| **Clado** | People search, LinkedIn enrichment | $0.01/result |
| **Apollo** | People/company enrichment (275M+ contacts) | $0.008-$0.043 |

### Communication & Social

| Provider | Endpoints | Price per Call |
|----------|-----------|---------------|
| **Resend** | Transactional email (batch up to 100) | $0.004/email |
| **X (Twitter)** | Tweets, users, search, trends (read-only) | $0.016/call |

### Data Validation

| Provider | Endpoints | Price per Call |
|----------|-----------|---------------|
| **Abstract API** | Email, IP, phone, VAT, timezone (12 services) | $0.006/call |

**Platform fee:** $0.003/call flat (most providers) or 15% markup (OpenAI/Gemini)

---

## Beta Program

- Separate environment with **agent self-registration** capability
- Uses **real USDC on Base mainnet** (not testnet)
- Default limits: $10 allowance, $5 max transaction
- Rate limit: 5 registrations per IP per hour
- Credentials do NOT transfer between beta and production
- APIs may change without notice
- Free credits available for testing ($5-$50, manual review)

---

## Key Diagrams

### Complete Platform Architecture

```
                    +------------------------------------------+
                    |          Human Operator Dashboard         |
                    |  +------+ +--------+ +-------+ +------+  |
                    |  |Wallet| |Spending| |Approve| | Audit|  |
                    |  |Mgmt  | |Controls| |  Txs  | |  Log |  |
                    |  +------+ +--------+ +-------+ +------+  |
                    +-------------------+----------------------+
                                        |
                                        | Governance
                                        v
+--------+    claw_* API Key     +------+-------+
|   AI   |--------------------->|  Locus API   |
| Agent  |<---------------------| Gateway      |
+--------+    JWT (15 min)      +------+-------+
                                       |
                 +----------+----------+----------+----------+
                 |          |          |          |          |
                 v          v          v          v          v
          +------+--+ +----+----+ +---+---+ +---+---+ +----+----+
          | Wrapped  | |Checkout | | Build | | Tasks | |  Laso   |
          |  APIs    | | System  | | (PaaS)| |(Human)| |(Finance)|
          +------+---+ +----+----+ +---+---+ +---+---+ +----+----+
                 |          |          |          |          |
                 v          v          v          v          v
          +------+---+ +----+----+ +---+---+ +---+---+ +----+----+
          | OpenAI,  | |Payment  | | AWS   | |Fiverr,| | Visa    |
          | Gemini,  | |Router   | | ECS   | |Upwork | | Cards,  |
          | Firecrawl| |Contract | |Fargate| | etc.  | | Venmo,  |
          | Exa, ... | |(Base)   | |       | |       | | PayPal  |
          +----------+ +---------+ +-------+ +-------+ +---------+
                            |
                            v
                    +-------+--------+
                    | Base Blockchain |
                    |  USDC + ERC4337 |
                    |  Smart Wallets  |
                    |  Paymaster      |
                    |  (Gasless)      |
                    +----------------+
```

### Agent Transaction Lifecycle

```
Agent initiates action
        |
        v
+-------+--------+
| Check allowance |-----> Exceeds? --> HTTP 403
+-------+--------+
        |
        v
+-------+--------+
| Check max tx   |-----> Exceeds? --> HTTP 403
+-------+--------+
        |
        v
+-------+---------+
| Check approval  |-----> Above threshold? --> HTTP 202
| threshold       |       (human approves via URL)
+-------+---------+
        |
        v
+-------+--------+
| Reserve USDC   |
| from wallet    |
+-------+--------+
        |
        v
+-------+--------+
| Execute action |-----> Fails? --> Restore reserved USDC
| (API/deploy/   |                  (no charge)
|  payment/etc)  |
+-------+--------+
        |
        v (success)
+-------+---------+
| Deduct USDC    |
| on Base chain  |
| (wallet ->     |
|  treasury)     |
+-----------------+
```

### Smart Wallet Contract Hierarchy

```
+------------------+
|  LocusFactory    |  (Deploys wallets via CREATE2)
|  (ERC4337Factory)|
+--------+---------+
         |
         | CREATE2 (deterministic address)
         v
+--------+-----------+
| Locus Smart Wallet |  (ERC1967 Proxy -> Solady ERC4337)
|                    |
|  Owner Key --------+---> revokePermissionedKey()
|  Permissioned Key -+---> setPermissionedKey() (rotate)
|                    |
|  +-- Subwallets ---+---> createAndFundSubwallet()
|  |   (up to 100)  |     reclaimSubwallet()
|  |                 |     disburseSubwallet()
|  +-- USDC Ops ----+---> transferUSDC()
|                    |
|  UUPS Upgrade: DISABLED (immutable)
+--------------------+
         |
         | Creates
         v
+--------+---------+
|    Subwallet     |  (Minimal Proxy, reusable)
|                  |
|  disburseBefore  |  (time-limited escrow)
|  isActive        |  (reuse pool management)
|  transferToken() |  (deadline-enforced)
|  activate()      |  (reuse instead of redeploy)
|  deactivate()    |  (return to pool)
+------------------+
```

---

## Summary

Locus enables onchain agentic payments by providing:

1. **Non-custodial smart wallets** (ERC-4337) with dual-key security and gasless transactions on Base
2. **Unified USDC billing** across all services — APIs, deployments, checkout, transfers, tasks, and fiat off-ramps
3. **Human governance** via three-tier spending controls (allowance, max tx, approval threshold)
4. **Agent autonomy** with self-registration, programmatic API access, and autonomous deployment capabilities
5. **On-chain settlement** — all payments are verifiable USDC transfers on Base blockchain
6. **Fiat bridge** via Laso Finance (Visa cards, Venmo, PayPal) enabling agents to spend in the traditional economy

The core innovation is treating **USDC as a universal API payment layer** — agents don't need credit cards, bank accounts, or upstream API keys. They have one wallet, one balance, and one set of controls that govern all spending across the entire platform.
