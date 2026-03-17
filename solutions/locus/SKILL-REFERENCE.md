# Locus SKILL.md Reference

> This document explains what the Locus **SKILL.md** file is, what it contains, and how it serves as the canonical API reference for AI agents operating on the Locus platform. It also covers the companion skill files (ONBOARDING.md, CHECKOUT.md, LASO.md, HEARTBEAT.md).

---

## What is SKILL.md?

SKILL.md is the **canonical reference document** that AI agents read to learn how to interact with the Locus platform. It's a structured markdown file (version 1.3.0) hosted at `https://paywithlocus.com/skill.md` that contains every API endpoint, authentication method, request/response format, and behavioral instruction an agent needs.

When a human wants to connect an AI agent to Locus, they simply tell the agent:

> "Read https://paywithlocus.com/SKILL.md and follow the instructions."

The agent reads the file, learns the full API surface, and can immediately start making payments, calling APIs, and managing services.

---

## Skill File Ecosystem

SKILL.md is part of a broader set of skill files, each covering a specific domain:

```
+-------------------------------------------------------------------+
|                     Locus Skill File System                       |
|                                                                   |
|  Static Files (hosted at paywithlocus.com):                       |
|  +-------------+  +----------------+  +-----------+               |
|  | SKILL.md    |  | ONBOARDING.md  |  | LASO.md   |              |
|  | (master ref)|  | (setup guide)  |  | (cards &  |              |
|  |             |  |                |  |  payments) |              |
|  +-------------+  +----------------+  +-----------+               |
|  +-------------+  +------------------+  +-------------------+     |
|  | CHECKOUT.md |  | HEARTBEAT.md     |  | REQUEST_CREDITS.md|     |
|  | (merchant   |  | (health checks)  |  | (free USDC)       |    |
|  |  payments)  |  |                  |  |                   |     |
|  +-------------+  +------------------+  +-------------------+     |
|                                                                   |
|  Dynamic Files (fetched via API):                                 |
|  +------------------+  +--------------------+  +---------------+  |
|  | X402ENDPOINTS.md |  | Wrapped API index  |  | APPS.md       | |
|  | GET /x402/       |  | GET /wrapped/md    |  | GET /apps/md  | |
|  | endpoints/md     |  |                    |  |               | |
|  +------------------+  +--------------------+  +---------------+  |
+-------------------------------------------------------------------+
```

| File | URL | Purpose |
|------|-----|---------|
| **SKILL.md** | `paywithlocus.com/skill.md` | Master reference — all core endpoints and behaviors |
| **ONBOARDING.md** | `paywithlocus.com/onboarding.md` | 7-step setup guide for new agents |
| **CHECKOUT.md** | `paywithlocus.com/checkout.md` | Merchant checkout session payments |
| **LASO.md** | `paywithlocus.com/laso.md` | Prepaid cards, Venmo/PayPal, free endpoints |
| **HEARTBEAT.md** | `paywithlocus.com/heartbeat.md` | Periodic health-check protocol |
| **REQUEST_CREDITS.md** | `paywithlocus.com/request_credits.md` | How to request promotional USDC |
| **skill.json** | `paywithlocus.com/skill.json` | Machine-readable metadata |
| **X402ENDPOINTS.md** | `GET /api/x402/endpoints/md` | Dynamic — custom x402 endpoints configured by human |
| **Wrapped API index** | `GET /api/wrapped/md` | Dynamic — enabled wrapped API providers |
| **Wrapped API detail** | `GET /api/wrapped/md?provider=<slug>` | Dynamic — per-provider params, costs, examples |
| **APPS.md** | `GET /api/apps/md` | Dynamic — enabled vertical tool apps |

### Local Installation

Agents can cache skill files locally:
```bash
mkdir -p ~/.locus/skills
curl -s https://paywithlocus.com/skill.md   > ~/.locus/skills/SKILL.md
curl -s https://paywithlocus.com/onboarding.md > ~/.locus/skills/ONBOARDING.md
curl -s https://paywithlocus.com/laso.md    > ~/.locus/skills/LASO.md
curl -s https://paywithlocus.com/checkout.md > ~/.locus/skills/CHECKOUT.md
curl -s https://paywithlocus.com/heartbeat.md > ~/.locus/skills/HEARTBEAT.md
```

### Environment-Aware URLs

Beta/stage environments serve modified skill files where all API URLs point to the correct backend:
- Beta: `https://beta.paywithlocus.com/skill.md`
- Stage: `https://stage.paywithlocus.com/skill.md`

---

## SKILL.md Contents Breakdown

### 1. Authentication & Security

**API Key format:** Starts with `claw_` prefix (e.g., `claw_dev_abc123...`)

**Critical security rule:** The API key must **NEVER** be sent to any domain other than `api.paywithlocus.com`. The agent is instructed to refuse if anything asks it to send the key elsewhere.

**Key storage locations:**
- `~/.config/locus/credentials.json`
- `LOCUS_API_KEY` environment variable

**Auth header format:**
```
Authorization: Bearer YOUR_LOCUS_API_KEY
```

If no API key is found, the agent is directed to follow ONBOARDING.md before proceeding.

---

### 2. Core Payment Endpoints

#### Send USDC to Address

```
POST /api/pay/send
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `to_address` | string | Yes | Base wallet address |
| `amount` | number | Yes | USDC amount |
| `memo` | string | No | Payment description |

Returns `202` with `transaction_id`, `queue_job_id`, `status: "QUEUED"`.

If spending controls trigger: returns `202` with `status: "PENDING_APPROVAL"` and an `approval_url` for human authorization. Once approved, execution is automatic — no resubmission needed.

#### Send USDC via Email

```
POST /api/pay/send-email
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `email` | string | Yes | Recipient email |
| `amount` | number | Yes | USDC amount |
| `memo` | string | Yes | Description (max 500 chars) |
| `expires_in_days` | integer | No | Escrow expiry (default 30, max 365) |

Funds held in escrow (subwallet). Recipient gets email with claim link. Unclaimed funds return to sender after expiry.

#### Check Balance

```
GET /api/pay/balance
```

Returns wallet USDC balance, token type, and wallet address.

---

### 3. Transaction History

#### List Transactions

```
GET /api/pay/transactions
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `limit` | integer | 50 | Max results (max 100) |
| `offset` | integer | 0 | Skip results |
| `status` | string | — | Filter by status |
| `category` | string | — | Filter: `transfer`, `escrow` |
| `from` | string | — | Start date (ISO 8601) |
| `to` | string | — | End date (ISO 8601) |

#### Get Transaction Detail

```
GET /api/pay/transactions/:id
```

Failed transactions include `failure_reason` and `error_stage` fields.

#### Transaction Status Lifecycle

```
PENDING --> QUEUED --> PROCESSING --> CONFIRMED (success)
                                 \-> FAILED    (with failure_reason)

Other terminal states:
  POLICY_REJECTED    (blocked by spending controls)
  VALIDATION_FAILED  (invalid parameters)
  CANCELLED          (cancelled before execution)
  EXPIRED            (expired before completion)
```

---

### 4. Checkout SDK

For paying merchant checkout sessions. Quick flow:

```
1. Preflight  -->  GET  /api/checkout/agent/preflight/:sessionId
2. Pay        -->  POST /api/checkout/agent/pay/:sessionId
3. Poll       -->  GET  /api/checkout/agent/payments/:txId
```

Detailed reference in CHECKOUT.md (see below).

---

### 5. x402 Endpoints (Pay-per-call APIs)

Custom paid API endpoints configured by the human operator.

#### Discover x402 Catalog

```
GET /api/x402/endpoints/md
```

Returns a markdown document (X402ENDPOINTS.md) listing every x402 service the human has configured — with endpoint URLs, descriptions, curl examples, and input schemas.

#### Call an x402 Endpoint

```
POST /api/x402/<slug>
```

Body contains parameters per the endpoint's schema from X402ENDPOINTS.md.

#### Ad-hoc x402 Call

```
POST /api/x402/call
```

| Field | Required | Description |
|-------|----------|-------------|
| `url` | Yes | Full HTTPS URL |
| `method` | No | `GET` or `POST` (default) |
| `body` | No | JSON body for POST |

#### x402 Transaction History

```
GET /api/x402/transactions          # Paginated list
GET /api/x402/transactions/:id      # Single transaction with full response
```

---

### 6. Wrapped APIs (Pay-per-use Proxy)

Third-party APIs (OpenAI, Gemini, Firecrawl, Exa, etc.) called through Locus with automatic auth and USDC billing.

#### Discovery Flow

```
Step 1: GET /api/wrapped/md                      # Index of all providers
Step 2: GET /api/wrapped/md?provider=<slug>       # Full detail for one provider
Step 3: POST /api/wrapped/<provider>/<endpoint>   # Make the call
```

**Key instruction to agents:** "Only fetch the providers you actually need to keep your context lean."

#### When to Use Which

```
+-------------------+     +------------------+
| Wrapped APIs      |     | x402 Endpoints   |
| /api/wrapped/...  |     | /api/x402/...    |
|                   |     |                  |
| Curated catalog   |     | Custom endpoints |
| (OpenAI, Gemini,  |     | registered by    |
|  Firecrawl, Exa,  |     | human + built-in |
|  Resend, X, etc.) |     | services (Laso)  |
|                   |     |                  |
| Known pricing     |     | Variable pricing |
+-------------------+     +------------------+
         |                         |
         +-------------------------+
                    |
         Same policy guardrails
         (allowance, max tx, approval)
```

---

### 7. Laso Finance

Prepaid Visa cards and Venmo/PayPal payments. Referenced in SKILL.md, detailed in LASO.md.

Quick reference from SKILL.md:

| Action | Endpoint | Cost |
|--------|----------|------|
| Authenticate | `POST /api/x402/laso-auth` | $0.001 |
| Order card | `POST /api/x402/laso-get-card` | Dynamic ($5-$1000) |
| Send payment | `POST /api/x402/laso-send-payment` | Dynamic ($5-$1000) |
| Card data | `GET laso.finance/get-card-data` | Free |
| Payment status | `GET laso.finance/get-payment-status` | Free |
| Balance | `GET laso.finance/get-account-balance` | Free |
| Withdraw | `POST laso.finance/withdraw` | Free |
| Search merchants | `GET laso.finance/search-merchants` | Free |

---

### 8. Apps (Vertical Tools)

Dashboard-enabled tools fetched dynamically:

```
GET /api/apps/md    # Returns APPS.md with all enabled app documentation
```

Example: "Hire with Locus" for tasking human workers.

---

### 9. Policy Guardrails

Three spending controls configured by the human operator:

| Control | Effect | HTTP Response |
|---------|--------|---------------|
| **Allowance** | Max total USDC the agent can spend | 403 if exceeded |
| **Max transaction size** | Cap per single transaction | 403 if exceeded |
| **Approval threshold** | Txs above this need human approval | 202 with `approval_url` |

Agent behavior on 403: inform the human that a policy limit was reached.
Agent behavior on 202 (PENDING_APPROVAL): share the `approval_url` with the human. Transaction auto-executes after approval.

---

### 10. Feedback System

```
POST /api/feedback
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `category` | string | Yes | `error`, `general`, `endpoint`, `suggestion` |
| `endpoint` | string | No | The specific API endpoint |
| `message` | string | Yes | Description (max 5000 chars) |
| `context` | object | No | Error codes, request/response snippets |
| `source` | string | No | `error`, `heartbeat`, `manual` |

**When to submit:**
- On **every error response** (4xx/5xx) — source: `error`
- During **daily heartbeat** — source: `heartbeat`
- Anytime with suggestions — source: `manual`

---

### 11. Response Envelope Format

All Locus API responses follow a consistent format:

**Success:**
```json
{"success": true, "data": {...}}
```

**Error:**
```json
{"success": false, "error": "Short error code", "message": "Human-readable description"}
```

**HTTP Status Codes:**

| Code | Meaning |
|------|---------|
| 200 | OK |
| 202 | Accepted / async (check status) |
| 400 | Bad request |
| 401 | Invalid API key |
| 403 | Policy rejected |
| 429 | Rate limited |
| 500 | Server error |

---

## ONBOARDING.md — Setup Guide

A 7-step process split between human and agent responsibilities:

```
Human Steps (1-5):                          Agent Steps (6-7):
+----------------------------------+        +----------------------------------+
| 1. Create account at             |        | 6. Save credentials:             |
|    app.paywithlocus.com          |        |    ~/.config/locus/              |
|                                  |        |    credentials.json              |
| 2. Create wallet                 |        |    OR LOCUS_API_KEY env var      |
|    (save private key!)           |        |                                  |
|    ~30s deployment on Base       |        | 7. Verify with:                  |
|                                  |        |    GET /api/pay/balance           |
| 3. Generate API key              |        |    200 = success                 |
|    (claw_* prefix, shown once)   |        |    401 = invalid key             |
|                                  |        +----------------------------------+
| 4. Fund wallet                   |
|    (USDC on Base)                |
|                                  |
| 5. Optional: set spending limits |
|    (allowance, max tx, approval) |
+----------------------------------+
```

---

## CHECKOUT.md — Merchant Payments

Enables agents to pay merchant checkout sessions programmatically.

### Environment-Aware API Bases

| Environment | API Base |
|-------------|----------|
| Production | `api.paywithlocus.com/api` |
| Beta | `beta-api.paywithlocus.com/api` |

Beta keys only work with beta API; production keys only with production.

### Agent Checkout Flow

```
Agent                           Locus API                    Merchant
  |                                 |                            |
  |-- 1. Preflight check --------->|                            |
  |   GET /checkout/agent/         |                            |
  |   preflight/:sessionId        |                            |
  |                                 |                            |
  |<-- canPay: true/false ---------|                            |
  |   (+ blockers if false)        |                            |
  |                                 |                            |
  |-- 2. Pay ---------------------->|                            |
  |   POST /checkout/agent/        |                            |
  |   pay/:sessionId              |                            |
  |   { payerEmail (optional) }    |                            |
  |                                 |                            |
  |<-- transactionId --------------|                            |
  |                                 |                            |
  |-- 3. Poll status -------------->|                            |
  |   GET /checkout/agent/         |                            |
  |   payments/:txId              |                            |
  |   (every 2 seconds)            |                            |
  |                                 |-- Webhook: PAID ---------->|
  |<-- CONFIRMED ------------------|                            |
```

**Status progression:** `PENDING` -> `QUEUED` -> `PROCESSING` -> `CONFIRMED` or `FAILED`

### Additional Checkout Endpoints

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/checkout/agent/payments` | GET | List payment history (paginated, filterable) |
| `/checkout/sessions/:id` | GET | Get session details and state |

### Checkout Features

- **Policy constraints** apply (allowance, max tx, approval threshold)
- **Webhooks:** `checkout.session.paid` and `checkout.session.expired` events, verified via HMAC-SHA256
- **Receipt customization:** Line items, tax, company branding via `receiptConfig` in session creation
- **External wallets:** MetaMask/Coinbase via Payment Router contract
- **Rate limit:** 100 requests/minute, poll every 2 seconds, confirmation typically in 10-30 seconds

---

## LASO.md — Prepaid Cards & Payments

Complete API reference for Laso Finance, covering both paid (x402) and free endpoints.

### Dual Base URL Architecture

```
+--------------------------------------------+
|  Paid Endpoints (x402 proxy)               |
|  Base: api.paywithlocus.com/api            |
|  Auth: Bearer YOUR_LOCUS_API_KEY           |
|  Deducts USDC from wallet                  |
|                                            |
|  POST /api/x402/laso-auth      ($0.001)   |
|  POST /api/x402/laso-get-card  (dynamic)  |
|  POST /api/x402/laso-send-payment (dynamic)|
+--------------------------------------------+
              |
              | Returns id_token + refresh_token
              v
+--------------------------------------------+
|  Free Endpoints (direct to Laso)           |
|  Base: laso.finance                        |
|  Auth: Bearer <id_token>                   |
|  No USDC cost                              |
|                                            |
|  GET  /get-card-data                       |
|  GET  /get-payment-status                  |
|  GET  /get-account-balance                 |
|  POST /withdraw                            |
|  GET  /get-withdrawal-status               |
|  POST /refresh-card-data                   |
|  GET  /search-merchants                    |
|  POST /refresh (token refresh)             |
+--------------------------------------------+
```

### Token Lifecycle

Every paid endpoint returns `id_token` (expires ~1 hour) and `refresh_token`. The `id_token` is required for all free endpoints. When it expires, call `POST laso.finance/refresh` (free) to get a new one.

### Prepaid Card Workflow

```
1. Order card:    POST /api/x402/laso-get-card  { amount: 50 }
                  Returns: card_id, status: "pending"
                  (Does NOT return card numbers yet)
                        |
2. Poll status:   GET laso.finance/get-card-data?card_id=xxx
                  Every 2-3 seconds until status = "ready" (~7-10s)
                        |
3. Use card:      Response includes card_number, cvv, exp_month, exp_year
                  Use at online checkout like any Visa prepaid
                        |
4. Monitor:       POST laso.finance/refresh-card-data  (re-scrape balance)
                  GET  laso.finance/get-card-data       (check transactions)
```

**Constraints:**
- Amount: $5-$1,000
- Non-reloadable (leftover balance is inaccessible)
- US-only (IP-locked)
- Check merchant compatibility first: `GET laso.finance/search-merchants?q=<name>`

### Payment Workflow (Venmo/PayPal)

```
1. Send:   POST /api/x402/laso-send-payment
           { amount: 25, platform: "venmo", recipient_id: "5551234567" }

2. Check:  GET laso.finance/get-payment-status
```

- Venmo: requires 10-digit phone number (digits only)
- PayPal: requires email address
- Amount: $5-$1,000
- Must confirm details with human before execution

---

## HEARTBEAT.md — Health Monitoring

A periodic protocol that runs every 30 minutes. State tracked in `~/.config/locus/state.json`.

### What Heartbeat Does

```
Every 30 minutes:
  |
  +-- Check for skill file updates (daily via skill.json)
  |   |-- If new version: re-fetch all skill files
  |   +-- Notify user of update
  |
  +-- Execute app-specific heartbeats (from APPS.md)
  |
  +-- Submit daily feedback summary (once per 24 hours)
  |   source: "heartbeat"
  |
  +-- On API errors: submit immediate feedback
      source: "error"
```

### Communication Rules

- **Notify user** about: skill updates, policy violations (403s), app-specific events
- **Do NOT notify** about: routine heartbeats with no changes

### Response Format

| Output | When |
|--------|------|
| `HEARTBEAT_OK — No Locus updates` | Nothing changed |
| `Locus: Skill files updated to v[version]` | New version available |
| `Locus: [situation]` | Human attention needed |

---

## Complete API Endpoint Reference

This is the full table from SKILL.md listing every action an agent can take:

### Payments & Transfers

| Action | Method | Endpoint |
|--------|--------|----------|
| Send USDC | POST | `/api/pay/send` |
| Send USDC via email | POST | `/api/pay/send-email` |
| Check balance | GET | `/api/pay/balance` |
| List transactions | GET | `/api/pay/transactions` |
| Get transaction | GET | `/api/pay/transactions/:id` |

### Checkout

| Action | Method | Endpoint |
|--------|--------|----------|
| Preflight check | GET | `/api/checkout/agent/preflight/:sessionId` |
| Pay session | POST | `/api/checkout/agent/pay/:sessionId` |
| Payment status | GET | `/api/checkout/agent/payments/:txId` |
| Payment history | GET | `/api/checkout/agent/payments` |
| Get session | GET | `/api/checkout/sessions/:id` |

### x402 Endpoints

| Action | Method | Endpoint |
|--------|--------|----------|
| Fetch catalog | GET | `/api/x402/endpoints/md` |
| Call endpoint | POST | `/api/x402/:slug` |
| Call any URL | POST | `/api/x402/call` |
| List transactions | GET | `/api/x402/transactions` |
| Get transaction | GET | `/api/x402/transactions/:id` |

### Wrapped APIs

| Action | Method | Endpoint |
|--------|--------|----------|
| Provider index | GET | `/api/wrapped/md` |
| Provider detail | GET | `/api/wrapped/md?provider=<slug>` |
| Call API | POST | `/api/wrapped/:provider/:endpoint` |

### Laso Finance

| Action | Method | Endpoint | Cost |
|--------|--------|----------|------|
| Authenticate | POST | `/api/x402/laso-auth` | $0.001 |
| Order card | POST | `/api/x402/laso-get-card` | Dynamic |
| Send payment | POST | `/api/x402/laso-send-payment` | Dynamic |
| Card data | GET | `laso.finance/get-card-data` | Free |
| Payment status | GET | `laso.finance/get-payment-status` | Free |
| Balance | GET | `laso.finance/get-account-balance` | Free |
| Withdraw | POST | `laso.finance/withdraw` | Free |
| Withdrawal status | GET | `laso.finance/get-withdrawal-status` | Free |
| Refresh card | POST | `laso.finance/refresh-card-data` | Free |
| Search merchants | GET | `laso.finance/search-merchants` | Free |
| Refresh token | POST | `laso.finance/refresh` | Free |

### System

| Action | Method | Endpoint |
|--------|--------|----------|
| Fetch app docs | GET | `/api/apps/md` |
| Submit feedback | POST | `/api/feedback` |

---

## Summary

The SKILL.md file system is Locus's mechanism for **bootstrapping AI agents**. Instead of requiring complex SDK integrations or manual configuration, the entire platform is documented in markdown files that agents can read and immediately act upon. The key innovation is that:

1. **One file teaches the agent everything** — SKILL.md is the single entry point
2. **Dynamic catalogs** keep the agent up-to-date on available services
3. **Environment-aware URLs** handle beta/staging/production automatically
4. **Heartbeat protocol** ensures the agent stays current with platform changes
5. **Policy guardrails are built into the API responses** (403s and 202s) — the agent doesn't need to implement spending logic, it just reacts to HTTP status codes
