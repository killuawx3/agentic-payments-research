# Rye vs Purch: Architectural & Feature Comparison

> Two approaches to agentic shopping — fiat-native universal checkout vs crypto-native x402 micropayments.

---

## At a Glance

| | **Rye** | **Purch** |
|---|---|---|
| **Tagline** | Universal agentic checkout that actually works | Shopping agent for humans and AI alike |
| **Payment Rails** | Credit cards (Stripe/BasisTheory tokens) | USDC on Solana (x402 protocol) |
| **Auth Model** | API key (Basic auth) | No API keys — payment proof IS auth |
| **Pricing Model** | $149/month + 3% Amazon fee | Pay-per-call micropayments ($0.01–$0.10 + product cost) |
| **Merchants** | Amazon + Shopify + Best Buy + thousands more | Amazon + Shopify |
| **Product Discovery** | None (URL lookup only) | Built-in AI assistant + structured search |
| **Checkout Flow** | Two-step (create → confirm) or single-step | Single-step (pay + buy in one call) |
| **SDKs** | TypeScript, Python, Ruby, Java | None (REST + x402 only) |
| **Unique Feature** | RyeBot (web agent for any merchant) | Vault (AI agent skill marketplace) |
| **Target User** | Web developers building commerce into AI apps | Crypto-native agents and users |

---

## 1. Payment Architecture

This is the most fundamental difference. Rye uses traditional payment rails (credit cards); Purch uses crypto-native x402 micropayments. This choice shapes everything else — auth, pricing, checkout flow, and who can use the platform.

### Rye: Credit Card Tokenization

```
Buyer/Agent                 Token Vault              Rye API            Merchant
    |                           |                       |                   |
    |-- Card details --------->|                       |                   |
    |   (Stripe.js or          |                       |                   |
    |    BasisTheory iframe)   |                       |                   |
    |                           |                       |                   |
    |<-- Opaque token ---------|                       |                   |
    |   (tok_... or UUID)      |                       |                   |
    |                           |                       |                   |
    |-- Token in confirm call --|-------------------->>|                   |
    |                           |                       |                   |
    |                           |<-- Detokenize --------|                   |
    |                           |                       |-- Pay merchant -->|
    |                           |                       |                   |
    |<-- Order confirmation ----|--------------------<<|                   |
```

**Key properties:**
- Card data never touches Rye servers — tokenized by Stripe or BasisTheory
- Must use **Rye's Stripe publishable keys** (not your own) — tokens are bound to Rye's Stripe account
- BasisTheory tokens are reusable but **CVC expires after 24 hours** (refresh flow available)
- Payment is a **separate step** — create intent first, then confirm with token
- Supports Stripe, BasisTheory, Nekuda, Prava, and pre-funded drawdown balance

### Purch: x402 Protocol (USDC on Solana)

```
Agent/User                      Purch API                     Solana
    |                               |                            |
    |-- GET /x402/search ---------->|                            |
    |   (no payment header)         |                            |
    |                               |                            |
    |<-- 402 Payment Required ------|                            |
    |   { accepts: [{              |                            |
    |       network: "solana",     |                            |
    |       asset: "USDC",         |                            |
    |       maxAmountRequired:     |                            |
    |         "0.01",              |                            |
    |       payTo: "wallet_addr"   |                            |
    |     }]                       |                            |
    |   }                          |                            |
    |                               |                            |
    |-- Send USDC on Solana -------|-------------------------->|
    |                               |                            |
    |-- Retry with                  |                            |
    |   PAYMENT-SIGNATURE header ->|                            |
    |                               |-- Verify on-chain ------->|
    |                               |                            |
    |<-- 200 OK + data ------------|<-- Confirmed --------------|
```

**Key properties:**
- **No API keys at all** — the USDC payment IS the authentication
- Every API call requires an on-chain USDC transfer on Solana
- The 402 response tells the caller exactly how much to pay and where
- For purchases, the x402 amount equals the **product total** (including tax + shipping)
- Wallets powered by Crossmint (smart wallets on Solana)

### Comparison

| Dimension | Rye | Purch |
|-----------|-----|-------|
| **Payment currency** | USD (credit cards) | USDC on Solana |
| **Token format** | Stripe `tok_...` or BasisTheory UUID | Solana tx signature |
| **Auth mechanism** | API key (Basic auth header) | PAYMENT-SIGNATURE header (x402) |
| **API key management** | Yes (separate staging/prod keys) | None |
| **Card data handling** | Tokenized via Stripe/BT (PCI-compliant) | No cards — USDC only |
| **Payment timing** | After offer retrieval (confirm step) | Every API call (per-request) |
| **Reusable credentials** | Yes (BT tokens persist, CVC refreshable) | No (new payment per call) |
| **Pre-funded balance** | Yes (drawdown account) | No (wallet balance on Solana) |
| **Settlement finality** | Card network settlement (days) | On-chain (seconds) |
| **Chargeback risk** | Yes (credit card chargebacks) | No (crypto is final) |
| **Who can pay** | Anyone with a credit card | Anyone with USDC on Solana |

**Key insight:** Rye's fiat model is accessible to any buyer with a credit card but adds complexity (tokenization, key management, PCI compliance). Purch's crypto model eliminates auth/key management entirely but requires the caller to hold USDC on Solana — a higher barrier for mainstream users but natural for crypto-native agents.

---

## 2. Checkout Flow Architecture

### Rye: Two-Step (or Single-Step) Intent Model

Rye uses a **Checkout Intent** state machine that separates offer retrieval from payment confirmation.

```
Two-Step Flow:

Agent                          Rye API                      Merchant
  |                               |                            |
  |-- POST /checkout-intents ---->|                            |
  |   { productUrl, buyer,       |                            |
  |     quantity }               |                            |
  |                               |                            |
  |<-- { id, status:             |                            |
  |    "retrieving_offer" } -----|-- Fetch price/tax/ship --->|
  |                               |                            |
  |-- Poll GET /{id}             |                            |
  |   every 10 seconds --------->|                            |
  |                               |                            |
  |<-- { status:                 |<-- Offer data ------------|
  |   "awaiting_confirmation",   |                            |
  |    offer: { subtotal, tax,   |                            |
  |    shipping, total,          |                            |
  |    product } } --------------|                            |
  |                               |                            |
  |   (Agent reviews offer,      |                            |
  |    decides to proceed)       |                            |
  |                               |                            |
  |-- POST /{id}/confirm ------->|                            |
  |   { paymentMethod:           |                            |
  |     { stripeToken } }       |-- Place order via RyeBot ->|
  |                               |                            |
  |-- Poll GET /{id}             |                            |
  |   every 5 seconds ---------->|                            |
  |                               |                            |
  |<-- { status: "completed",   |<-- Order confirmed --------|
  |      orderId } --------------|                            |
```

**State machine:** `retrieving_offer` → `awaiting_confirmation` → `placing_order` → `completed` | `failed`

**Timing:** Offer retrieval can take seconds to 45 minutes. Order placement: seconds to minutes.

**Single-step alternative:** `POST /checkout-intents/purchase` combines create + confirm (used for Best Buy and partner endpoints like Clawdbot).

### Purch: Single-Call Purchase

Purch collapses the entire checkout into **one API call**. There's no intent, no polling, no confirmation step.

```
Agent                          Purch API                    Merchant
  |                               |                            |
  |-- POST /x402/buy ----------->|                            |
  |   { asin, shippingAddress,   |                            |
  |     email }                  |                            |
  |   + PAYMENT-SIGNATURE        |                            |
  |   (USDC = product total)     |                            |
  |                               |-- Resolve product -------->|
  |                               |-- Calculate total -------->|
  |                               |-- Place order ------------>|
  |                               |                            |
  |<-- { orderId, status:        |<-- Order confirmation -----|
  |      "processing",           |                            |
  |      product, totalPrice } --|                            |
```

No state machine. No polling for offer. No separate confirmation. One call, one payment, one response.

### Comparison

| Dimension | Rye | Purch |
|-----------|-----|-------|
| **Checkout steps** | 2 (create → confirm) or 1 (single-step) | Always 1 |
| **Offer preview** | Yes — see price/tax/shipping before paying | No — price is determined at x402 negotiation |
| **State machine** | 5 states with polling | Single `processing` status |
| **Polling required** | Yes (every 5-10 seconds, up to 45 min) | Not documented |
| **Payment timing** | After reviewing offer | Upfront (x402 amount = product total) |
| **Cancel/abort** | Don't confirm the intent | Not possible (payment already made) |
| **Price discovery** | Via offer in `awaiting_confirmation` state | Via `GET /x402/search` or `POST /x402/shop` before buying |

**Key insight:** Rye's two-step model gives the agent a chance to review the exact price (including tax and shipping) before committing payment. Purch requires the agent to know (or trust) the price upfront since x402 payment happens atomically with the request. Rye is safer for price-sensitive purchases; Purch is faster for agents that already know what they want.

---

## 3. Product Discovery

This is where Purch has a significant advantage — Rye has no product search capabilities at all.

### Rye: No Discovery

Rye is purely a **checkout API**. It takes a product URL as input and converts it to a purchase. There is no way to search for products through Rye.

```
Agent must find URL externally
(Google, merchant site, affiliate API, etc.)
        |
        v
Rye API: productUrl → checkout → order
```

The `GET /api/v1/products/lookup?url=...` endpoint retrieves product data by URL but cannot search across catalogs.

### Purch: Built-In AI Discovery

Purch provides two discovery mechanisms:

#### Structured Search (`GET /x402/search`)

```json
GET /x402/search?q=wireless+earbuds&priceMax=50&brand=Sony&page=1

Response:
{
  "products": [...],
  "totalResults": 150,
  "page": 1,
  "hasMore": true
}
```

Filters: query, priceMin, priceMax, brand, page. Searches across Amazon + Shopify catalogs (1B+ items claimed).

#### AI Shopping Assistant (`POST /x402/shop`)

```json
POST /x402/shop
{
  "message": "Birthday gift for my dad who loves grilling, budget $75",
  "context": {
    "priceRange": { "min": 50, "max": 100 },
    "preferences": ["outdoor", "cooking", "premium"]
  }
}

Response:
{
  "reply": "Here are some great grilling gifts...",
  "products": [...]
}
```

Natural language → AI-curated product recommendations with contextual understanding.

#### Gift Hunter (Social Analysis)

Submit a Twitter/Instagram/TikTok profile URL → Purch analyzes public posts/bio/hashtags → returns 3 personalized gift suggestions.

### Comparison

| Discovery Feature | Rye | Purch |
|-------------------|-----|-------|
| **Product search** | None | `GET /x402/search` ($0.01/call) |
| **AI recommendations** | None | `POST /x402/shop` ($0.10/call) |
| **Social-based gifting** | None | Gift Hunter (profile → gifts) |
| **Product lookup by URL** | Yes (`GET /products/lookup`) | Implicit in `/x402/buy` |
| **Full-pipeline discovery → purchase** | No — need external search | Yes — search → buy in same API |

---

## 4. Merchant Coverage & Order Routing

### Rye: RyeBot Universal Agent

Rye's core differentiator is **merchant breadth**. RyeBot is a web agent that can perform checkout on **any merchant site** — not just Amazon and Shopify.

```
Product URL from ANY supported merchant
        |
        v
+-------+--------+
| URL Resolution  |  Identify merchant platform
+-------+--------+
        |
        v
+-------+--------+
| RyeBot         |  Web agent navigates merchant's
| (Checkout Bot) |  actual checkout flow:
|                |  - Add to cart
|                |  - Enter shipping
|                |  - Apply payment
|                |  - Submit order
+-------+--------+
        |
        v
Order placed on merchant's native site
```

**Supported merchants:** Amazon, Shopify (any store), Best Buy (with approval), and thousands more.

**RyeBot allowlisting:** Merchants can whitelist RyeBot to prevent anti-bot blocking.

**Limitation:** Merchants with advanced anti-bot protections (e.g., REI) may not work.

### Purch: Amazon + Shopify Only

Purch supports two merchant platforms:

| Merchant | Product ID | Variant Handling |
|----------|-----------|-----------------|
| **Amazon** | ASIN (10-char) or `amazon.com/dp/ASIN` URL | Via ASIN deep link |
| **Shopify** | Product URL | `variantId` required in request body |

No web agent. No arbitrary merchant support.

### Comparison

| Dimension | Rye | Purch |
|-----------|-----|-------|
| **Merchant count** | Thousands (any site RyeBot can navigate) | 2 (Amazon + Shopify) |
| **Amazon** | Yes (via Rye's account, 3% fee, auto-Prime >$15) | Yes (USDC payment) |
| **Shopify** | Yes (card forwarded to store processor) | Yes (needs variantId) |
| **Best Buy** | Yes (requires approval) | No |
| **Arbitrary merchants** | Yes (via RyeBot web agent) | No |
| **Anti-bot handling** | RyeBot allowlisting program | N/A |
| **Merchant integration needed** | None (RyeBot automates checkout) | None (API handles it) |

---

## 5. Promo Codes & Price Optimization

### Rye: Manual + Auto-Discovery

Rye has a two-pronged promo code strategy:

**Manual:** Pass up to 16 codes (max 32 chars each). Rye shuffles and applies the first that works.
```json
{ "promoCodes": ["SAVE10", "WELCOME20", "FREESHIP"] }
```

**Auto-discovery:** Set `discoverPromoCodes: true` and Rye searches its aggregated code database for the best available discount.

**Combined:** Both methods work together — Rye merges discovered + manual codes, applies the highest discount.

**Response includes:** `appliedPromoCodes` and `discount.amountSubunits`.

### Purch: None

No promo code support documented. No price optimization features.

### Comparison

| Feature | Rye | Purch |
|---------|-----|-------|
| **Manual promo codes** | Yes (up to 16 per order) | No |
| **Auto promo discovery** | Yes (`discoverPromoCodes: true`) | No |
| **Price constraints** | Yes (`constraints.maxTotalPrice`) | No |
| **Discount reporting** | Yes (applied codes + amount) | No |

---

## 6. Spending Controls & Guardrails

### Rye: Price Constraints

Rye supports a `constraints` object on checkout intents:

```json
{
  "constraints": {
    "maxTotalPrice": 50000
  }
}
```

`maxTotalPrice` is in cents. The API rejects the order if the total (including tax, shipping, fees) exceeds this limit.

The Clawdbot skill adds a conversational spending limit — the agent asks the user for a max price on first purchase and enforces it on subsequent orders.

### Purch: None (Built Into x402)

No explicit spending controls. However, the x402 protocol provides an implicit guardrail — the agent must have enough USDC in its wallet to cover the purchase. The `maxAmountRequired` field in the 402 response tells the agent the exact cost before it commits payment.

### Comparison

| Control | Rye | Purch |
|---------|-----|-------|
| **Max price constraint** | Yes (`maxTotalPrice` in cents) | No (implicit via wallet balance) |
| **Price preview before payment** | Yes (offer in `awaiting_confirmation`) | Yes (x402 `maxAmountRequired` in 402 response) |
| **Agent spending limits** | Via Clawdbot skill (conversational) | No |
| **Reject over-budget orders** | API-level enforcement | Agent must check before paying |

---

## 7. Agent Integration

### Rye: Clawdbot Skill + SDKs

Rye provides multiple integration paths:

**Clawdbot Skill** (`rye-com/clawdbot-buy-anything`):
- Doc-only skill (SKILL.md teaches agent to call REST API via `curl`)
- No API key needed (partner endpoint: `/partners/clawdbot/purchase`)
- BasisTheory card capture via hosted page
- Memory system (saves token + address for repeat purchases)
- Conversational spending limits

**Official SDKs:**
- TypeScript (`checkout-intents`)
- Python (`checkout_intents`)
- Ruby, Java

**Postman Collection** for API testing.

### Purch: REST API Only

Purch has no SDK, no MCP server, no agent skill, and no tool definitions. Integration is purely REST + x402.

The x402 model means any agent with a Solana wallet and USDC can call the API without any setup — but there's no framework-level integration to make it easier.

### Comparison

| Integration | Rye | Purch |
|-------------|-----|-------|
| **Agent skill** | Yes (Clawdbot) | No |
| **MCP server** | No | No |
| **TypeScript SDK** | Yes | No |
| **Python SDK** | Yes | No |
| **Other SDKs** | Ruby, Java | No |
| **Postman collection** | Yes | No |
| **Tool definitions** | No (Clawdbot uses curl) | No |
| **Zero-setup for agents** | No (needs API key or partner path) | Yes (just needs USDC wallet) |
| **Repeat purchase optimization** | Yes (saved BT tokens) | No (new x402 payment each time) |

---

## 8. The Vault: Purch's Unique Feature

Purch has an entire subsystem that Rye has no equivalent for — the **Vault**, a marketplace for AI agent digital assets.

```
+---------------------------------------------------------------+
|                        PURCH VAULT                             |
|                                                                |
|  +------------------+ +------------------+ +-----------------+ |
|  |     Skills       | |   Knowledge      | |   Personas      | |
|  | Agent workflows  | | Domain expertise | | Communication   | |
|  | (marketing,      | | and reference    | | styles and      | |
|  |  code review,    | | materials        | | personalities   | |
|  |  data analysis)  | |                  | |                 | |
|  +------------------+ +------------------+ +-----------------+ |
|                                                                |
|  Categories: Marketing | Development | Automation |            |
|              Career | iOS | Productivity                       |
|                                                                |
|  Creators: Human or Agent                                      |
|  Pricing: Per-item in USDC (set by creator)                    |
|  Delivery: ZIP download                                        |
|  Payment: x402 micropayments                                   |
+----------------------------------------------------------------+
```

**API flow:** Search ($0.01) → Buy (item price) → Download ($0.01)

This represents a fundamentally different vision — Purch isn't just a shopping API, it's a **marketplace where agents buy capabilities from other agents**. Rye is purely a checkout pipe.

---

## 9. Pricing Model

### Rye: Subscription + Per-Order Fees

| Component | Cost |
|-----------|------|
| **Monthly subscription** | $149/month (first month free) |
| **Amazon orders** | +3% transaction fee |
| **Shopify orders** | No markup |
| **Best Buy orders** | Not documented |

Fixed cost regardless of API call volume. The 3% Amazon fee covers Rye's cost of routing through their Amazon account.

### Purch: Pure Pay-Per-Call

| Endpoint | Cost |
|----------|------|
| `GET /x402/search` | $0.01 |
| `POST /x402/shop` | $0.10 |
| `POST /x402/buy` | Product total (dynamic) |
| `GET /x402/vault/search` | $0.01 |
| `POST /x402/vault/buy` | Item price (dynamic) |
| `GET /x402/vault/download/{id}` | $0.01 |

No subscription. No monthly minimum. Cost scales linearly with usage.

### Break-Even Analysis

At $0.01-$0.10 per call, Purch is cheaper for low-volume agents. Rye's $149/month is cheaper for high-volume usage (>1,490 shop queries or >14,900 searches per month before Purch costs more in API fees alone — excluding product costs).

However, the comparison isn't clean because:
- Rye charges 3% on Amazon orders; Purch's Amazon fee structure is unclear
- Rye includes checkout for thousands of merchants; Purch covers only two
- Purch charges for search/discovery that Rye doesn't even offer

---

## 10. Error Handling & Reliability

### Rye

**Claimed metrics:** 90%+ reliability, <35s median latency.

**Error codes:** Standard HTTP (400, 401, 403, 404, 429, 500).

**Failure modes:**
- `retrieving_offer` can take up to 45 minutes for slow merchants
- `failed` state includes `failureReason.message`
- Anti-bot merchants may reject RyeBot
- Invalid promo codes silently ignored (no error)

**Recovery:** Poll-based. Agent checks status periodically. No webhooks.

### Purch

**Claimed metrics:** Not documented.

**Error codes:** HTTP 400 (validation), 402 (payment required), 403 (unauthorized download), 404 (not found).

**Failure modes:**
- Insufficient USDC → 402 with payment details
- Invalid ASIN/URL → 400 validation error
- Vault item not found → 404

**Recovery:** Not documented. No polling/webhook mechanism described.

---

## 11. Design Philosophy Summary

| Dimension | Rye | Purch |
|-----------|-----|-------|
| **Core metaphor** | "Universal checkout pipe" | "AI shopping concierge" |
| **Payment philosophy** | Fiat-first (cards, tokens, traditional rails) | Crypto-first (USDC, x402, on-chain settlement) |
| **Auth philosophy** | Traditional (API keys, Basic auth) | Payment IS auth (no keys) |
| **Merchant philosophy** | Universal (any merchant via web agent) | Focused (Amazon + Shopify only) |
| **Discovery philosophy** | Not our problem (URL in, order out) | Built-in (NLP search, AI recs, social analysis) |
| **Agent philosophy** | Skill + SDKs (teach the agent) | Raw API (agent figures it out) |
| **Pricing philosophy** | Subscription (predictable monthly cost) | Micropayments (pay exactly what you use) |
| **Unique value** | Merchant breadth + checkout reliability | Discovery + crypto-native + Vault marketplace |

---

## 12. When to Use Which

**Choose Rye when:**
- You need **broad merchant coverage** beyond Amazon/Shopify (Best Buy, DTC brands, etc.)
- You want **fiat payment** (credit cards) — mainstream users, no crypto requirement
- You need **price preview before payment** (two-step offer → confirm flow)
- You want **promo code auto-discovery** to reduce costs
- You need **SDKs** (TypeScript, Python, Ruby, Java) for structured integration
- You want an **agent skill** (Clawdbot) with saved credentials and repeat purchases
- **Checkout reliability** matters (90%+ claimed, RyeBot web agent)
- You're building a product where **monthly subscription** is acceptable ($149/month)

**Choose Purch when:**
- Your agent is **crypto-native** and already holds USDC on Solana
- You want **zero auth setup** — no API keys, no account registration, just pay and use
- You need **built-in product discovery** (AI shopping, structured search, social gifting)
- You want **pay-per-call pricing** with no monthly commitment
- You're interested in the **Vault marketplace** (buying/selling agent skills and knowledge)
- **On-chain settlement finality** matters (no chargebacks, instant confirmation)
- You only need **Amazon + Shopify** merchant coverage
- Your use case involves **micropayment-scale API calls** ($0.01 per search)
