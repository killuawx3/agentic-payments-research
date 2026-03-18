# Rye: Universal Checkout API for Agentic Commerce

> **Turn any product URL into a completed checkout — no merchant integration required.**

Rye is a Universal Checkout API that enables AI agents and applications to purchase physical products from **any merchant** (Amazon, Shopify, Best Buy, and thousands more) by providing just a product URL and buyer information. Rye handles the entire checkout pipeline — product validation, pricing, tax calculation, shipping, payment processing, and order placement — with **90%+ reliability** and **<35s latency**, requiring zero merchant-side integration.

---

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [How Universal Checkout Works](#how-universal-checkout-works)
  - [The Problem](#the-problem)
  - [The Rye Solution](#the-rye-solution)
  - [Checkout Intent Lifecycle](#checkout-intent-lifecycle)
- [Core API Endpoints](#core-api-endpoints)
  - [Checkout Intents](#1-checkout-intents)
  - [Product Lookup](#2-product-lookup)
  - [Shipments](#3-shipments)
  - [Billing & Drawdown](#4-billing--drawdown)
- [Payment Methods](#payment-methods)
- [Merchant Coverage & Routing](#merchant-coverage--routing)
- [Promo Codes](#promo-codes)
- [Variant Selection](#variant-selection)
- [Agent Integration (Clawdbot Skill)](#agent-integration-clawdbot-skill)
- [SDKs](#sdks)
- [Environments](#environments)
- [Limitations & Constraints](#limitations--constraints)
- [Pricing](#pricing)
- [API Comparison: Universal Checkout vs Sync](#api-comparison-universal-checkout-vs-sync)
- [Key Diagrams](#key-diagrams)

---

## Architecture Overview

```
+------------------------------------------------------------------+
|                      DEVELOPER / AI AGENT                         |
|  Provides: product URL + buyer info + payment token               |
|  Receives: order confirmation, tracking, shipment details         |
+----------------------------------+-------------------------------+
                                   |
                            REST API (v1)
                   Authorization: Basic <API_KEY>
                                   |
                                   v
+------------------------------------------------------------------+
|                         RYE PLATFORM                              |
|                                                                   |
|  +-------------+  +-----------+  +------------+  +-------------+ |
|  | Product     |  | Pricing & |  | Payment    |  | Order       | |
|  | Validation  |  | Tax Calc  |  | Processing |  | Placement   | |
|  | & Lookup    |  | & Shipping|  | (Stripe/BT)|  | (RyeBot)    | |
|  +------+------+  +-----+-----+  +-----+------+  +------+------+ |
|         |               |              |                |         |
|         v               v              v                v         |
|  +------+---------------+--------------+----------------+------+ |
|  |                    RyeBot Web Agent                          | |
|  |  Performs checkout flows on merchant sites on behalf of      | |
|  |  shoppers. Handles cart, shipping, tax, payment submission.  | |
|  +-------------------------------------------------------------+ |
+----------------------------------+-------------------------------+
                                   |
                    +--------------+--------------+
                    |              |              |
                    v              v              v
              +---------+   +---------+   +-----------+
              | Amazon  |   | Shopify |   | Best Buy  |
              | (via Rye|   | (direct |   | (approved |
              | account)|   | store)  |   | partners) |
              +---------+   +---------+   +-----------+
                    |              |              |
                    +--------------+--------------+
                                   |
                           Order Fulfillment
                    (merchant ships to buyer directly)
```

### How It All Connects

1. **Developer/Agent** sends a product URL + buyer details + payment method to Rye
2. **Rye** validates the product, fetches live pricing/availability/shipping/tax from the merchant
3. **Rye** returns an offer for confirmation (or auto-confirms in single-step mode)
4. **RyeBot** (Rye's web agent) performs the actual checkout on the merchant's site
5. **Merchant** fulfills the order and ships directly to the buyer
6. **Developer/Agent** polls for status and receives order confirmation

---

## How Universal Checkout Works

### The Problem

Building commerce into AI apps requires custom integrations for each merchant. Every merchant has different systems for:

- Resolving the correct product URL and variants
- Retrieving accurate pricing and availability
- Selecting shipping methods
- Calculating taxes
- Processing checkout securely
- Handling order tracking and latency

Without a universal solution, every one of these steps requires custom code and brittle integrations that break when merchants change their sites.

### The Rye Solution

Rye abstracts the entire checkout pipeline into a single API. The developer provides:
1. A **product URL** (from any supported merchant)
2. **Buyer information** (name, address, email, phone)
3. A **payment method** (Stripe token, BasisTheory token, or drawdown balance)

Rye handles everything else — product validation, pricing, tax, shipping, order placement, and tracking.

**Key metrics (from whitepaper):**
- **90%+ reliability** across merchants
- **<35 seconds** median latency (product URL to order placed)
- **No merchant integration** required — RyeBot handles checkout flows on arbitrary merchant sites

### Checkout Intent Lifecycle

A Checkout Intent represents the end-to-end process of purchasing a product. Each intent progresses through a series of states:

```
                    Create Checkout Intent
                    (POST /checkout-intents)
                            |
                            v
                  +-------------------+
                  | retrieving_offer  |  Rye fetches live pricing,
                  | (async, poll      |  availability, taxes, and
                  |  every 10 sec)    |  shipping from merchant
                  +--------+----------+
                           |
              +------------+------------+
              |                         |
              v                         v
    +-------------------+         +---------+
    | awaiting_          |         | failed  |
    | confirmation       |         | (e.g.   |
    |                    |         | invalid |
    | Offer ready:       |         | URL,    |
    | - product details  |         | out of  |
    | - subtotal         |         | stock)  |
    | - tax              |         +---------+
    | - shipping         |
    | - total            |
    +--------+-----------+
             |
             | Confirm Checkout Intent
             | (POST /checkout-intents/{id}/confirm)
             | + payment method
             v
    +-------------------+
    | placing_order     |  RyeBot executes checkout
    | (async, poll      |  on merchant site
    |  every 5-10 sec)  |
    +--------+----------+
             |
    +--------+--------+
    |                  |
    v                  v
+-----------+    +---------+
| completed |    | failed  |
| (order    |    | (declined
| confirmed)|    | payment,
|           |    | OOS,
| - orderId |    | etc.)
| - total   |    +---------+
| - product |
+-----------+
```

**Timing expectations:**
- `retrieving_offer` → `awaiting_confirmation`: seconds to minutes (poll every 10s, may take up to 45 min for some merchants)
- `placing_order` → `completed`/`failed`: seconds to minutes (poll every 5-10s)

---

## Core API Endpoints

**Base URLs:**

| Environment | URL |
|-------------|-----|
| **Staging** | `https://staging.api.rye.com/api/v1/` |
| **Production** | `https://api.rye.com/api/v1/` |

**Authentication:** `Authorization: Basic <RYE_API_KEY>`

### 1. Checkout Intents

The primary workflow for purchasing products.

#### Create Checkout Intent

```
POST /api/v1/checkout-intents
```

Creates a new intent and begins fetching the offer from the merchant.

**Request:**
```json
{
  "buyer": {
    "firstName": "John",
    "lastName": "Doe",
    "email": "john.doe@example.com",
    "phone": "212-333-2121",
    "address1": "123 Main St",
    "address2": "Apt 1",
    "city": "New York",
    "province": "NY",
    "country": "US",
    "postalCode": "10001"
  },
  "productUrl": "https://flybyjing.com/collections/shop/products/sichuan-chili-crisp",
  "quantity": 1,
  "promoCodes": ["SAVE10"],
  "discoverPromoCodes": true,
  "constraints": {
    "maxTotalPrice": 50000
  }
}
```

**Response:**
```json
{
  "id": "ci_12345abcde",
  "buyer": { ... },
  "quantity": 1,
  "productUrl": "https://...",
  "status": "retrieving_offer",
  "createdAt": "2023-10-27T10:00:00Z",
  "updatedAt": "2023-10-27T10:00:00Z"
}
```

#### Retrieve Checkout Intent (Poll for Status)

```
GET /api/v1/checkout-intents/{id}
```

Poll this endpoint to check the current state. When `awaiting_confirmation`, the response includes the full offer with pricing, tax, shipping, and product details.

#### Confirm Checkout Intent

```
POST /api/v1/checkout-intents/{id}/confirm
```

Submit payment to complete the order.

**Request:**
```json
{
  "paymentMethod": {
    "type": "stripe_token",
    "stripeToken": "tok_1RkrWWHGDlstla3f1Fc7ZrhH"
  }
}
```

#### Single-Step Purchase (Best Buy / Partner Endpoints)

```
POST /api/v1/checkout-intents/purchase
```

Combines create + confirm into a single call. Used for Best Buy (requires approval) and partner integrations.

```json
{
  "buyer": { ... },
  "productUrl": "https://www.bestbuy.com/product/apple-airtag-silver/JJGCQ8XFQY",
  "quantity": 1,
  "paymentMethod": {
    "type": "stripe_token",
    "stripeToken": "tok_visa"
  }
}
```

#### Partner Purchase (Clawdbot)

```
POST /api/v1/partners/clawdbot/purchase
GET  /api/v1/partners/clawdbot/purchase/{id}
```

No API key needed — authenticated by partner path. Uses BasisTheory tokens.

### 2. Product Lookup

```
GET /api/v1/products/lookup?url={productUrl}
```

Retrieve product data (title, price, availability, images) by URL without creating a checkout intent.

### 3. Shipments

```
GET /api/v1/shipments                    # List shipments (paginated)
GET /api/v1/shipments/{id}               # Get specific shipment
GET /api/v1/checkout-intents/{id}/shipments  # Shipments for an intent
```

### 4. Billing & Drawdown

```
GET /api/v1/billing/balance              # Current drawdown balance
GET /api/v1/billing/transactions         # Balance transaction history
```

Drawdown is a pre-funded account balance that can be used as a payment method instead of card tokens.

### 5. Brand Lookup

```
GET /api/v1/brands?domain={domain}       # Get brand info by domain
```

---

## Payment Methods

Rye supports multiple payment tokenization providers. Card data never touches Rye's servers directly — it's tokenized through a third-party vault.

```
Buyer/Agent                 Token Vault              Rye API            Merchant
    |                           |                       |                   |
    |-- Card details --------->|                       |                   |
    |   (secure iframe/SDK)    |                       |                   |
    |                           |                       |                   |
    |<-- Token (opaque ID) ----|                       |                   |
    |                           |                       |                   |
    |-- Token + checkout ID -------------------------------->|             |
    |                           |                       |                   |
    |                           |<-- Detokenize --------|                   |
    |                           |                       |                   |
    |                           |                       |-- Pay merchant -->|
    |                           |                       |                   |
    |<-- Order confirmation --------------------------------|              |
```

### Supported Providers

| Provider | Type String | Token Format | Notes |
|----------|-------------|-------------|-------|
| **Stripe** | `stripe_token` | `tok_...` | Most common. Must use **Rye's Stripe publishable key**, not your own |
| **BasisTheory** | `basis_theory_token` | UUID (`d1ff0c32-...`) | PCI-compliant vault. Used by Clawdbot skill. CVC expires after 24 hours |
| **Nekuda** | `nekuda_token` | User ID + mandate data | Alternative provider |
| **Prava** | — | — | Referenced in docs |
| **Drawdown** | — | Pre-funded balance | No card needed — charges against account balance |

### Stripe Key Requirement

**Critical:** You must use Rye's Stripe publishable keys, not your own. Using your own key links the token to your Stripe account, causing checkout intent confirmation to fail.

| Environment | Stripe Publishable Key |
|-------------|----------------------|
| **Staging** | `pk_test_51LgDhrHGDlstla3fdqlULAne0rAf4Ho6aBV2cobkYQ4m863Sy0W8DNu2HOnUeYTQzQnE4DZGyzvCB8Yzl1r38isl00H9sVKEMu` |
| **Production** | `pk_live_51LgDhrHGDlstla3fOYU3AUV6QpuOgVEUa1E1VxFnejJ7mWB4vwU7gzSulOsWQ3Q90VVSk1WWBzYBo0RBKY3qxIjV00LHualegh` |

### BasisTheory Card Capture Flow (Agent Use)

For AI agents (e.g., Clawdbot), BasisTheory provides a hosted card capture page:

```
Agent opens: https://mcp.rye.com/bt-card-capture
    |
    v
User enters card in BasisTheory secure iframes
(card goes directly to BT vault, never touches agent)
    |
    v
User gets token UUID, pastes back into chat
    |
    v
Agent uses token in purchase API call
```

If CVC expires (after 24 hours), a refresh page is available:
```
https://mcp.rye.com/bt-cvc-refresh?token_id=SAVED_TOKEN_ID
```

---

## Merchant Coverage & Routing

Rye supports **thousands of merchants** without requiring any merchant-side integration. RyeBot (Rye's web agent) performs checkout flows on merchant sites directly.

### How Merchant Routing Works

```
Product URL submitted
        |
        v
+-------+--------+
| URL Resolution  |  Identify merchant platform
| & Validation    |  (Amazon, Shopify, Best Buy, etc.)
+-------+--------+
        |
        v
+-------+--------+
| RyeBot         |  Rye's web agent navigates the
| Checkout Agent |  merchant's checkout flow:
|                |  - Add to cart
|                |  - Enter shipping info
|                |  - Apply payment
|                |  - Submit order
+-------+--------+
        |
        v
Order placed on merchant's site
```

### Merchant-Specific Behavior

| Merchant | Payment Routing | Shipping | Notes |
|----------|----------------|----------|-------|
| **Amazon** | Processed through Rye's Amazon account | Free 2-day Prime for orders >$15; $6.99 for <$15 | 3% fee. Cannot connect to personal Amazon account |
| **Shopify** | Card forwarded directly to store's payment processor | Store's standard shipping | No markup from Rye |
| **Best Buy** | Via Rye API | Standard | Requires prior approval for production |
| **Other merchants** | Via RyeBot checkout automation | Varies | Depends on merchant support |

### RyeBot Allowlisting

RyeBot is Rye's web agent that performs checkout flows on behalf of shoppers. Merchants can allowlist RyeBot to ensure it can access their site without being blocked by anti-bot security measures.

**Merchants with advanced anti-bot protections** (like REI) may not be supported if they actively block automated checkout.

---

## Promo Codes

Rye supports two promo code strategies that can be used independently or combined.

### Manual Promo Codes

Pass an array of codes in the `promoCodes` field when creating a checkout intent. Rye shuffles the provided codes and applies the first one that works.

```json
{
  "productUrl": "https://...",
  "buyer": { ... },
  "promoCodes": ["SAVE10", "WELCOME20", "FREESHIP"]
}
```

**Constraints:**
- Alphanumeric format
- Maximum 32 characters per code
- Up to 16 codes per order

### Automatic Discovery

Set `discoverPromoCodes: true` to have Rye search its aggregated code database and automatically apply the highest-discount option.

```json
{
  "productUrl": "https://...",
  "buyer": { ... },
  "discoverPromoCodes": true
}
```

### Combined

Both methods can work together — Rye merges discovered codes with manually provided codes and applies the best one available.

**Response fields:**
- `appliedPromoCodes` — which codes were applied
- `discount.amountSubunits` — discount amount in cents

Invalid codes are silently ignored — no error-catching needed.

---

## Variant Selection

How to specify product variants depends on the merchant platform.

### Amazon & Shopify

Use a **deep link** to the specific variant in the `productUrl` field. The `variantSelections` field is **not supported** for these merchants.

```json
{
  "productUrl": "https://amazon.com/dp/B0DJLKV4N9"
}
```

### Other Merchants

Use the `variantSelections` array with label/value pairs that exactly match the product page:

```json
{
  "productUrl": "https://example.com/product/123",
  "variantSelections": [
    { "label": "Size", "value": "8.5" },
    { "label": "Color", "value": "Blue" }
  ]
}
```

**Note:** Variant information is not available via the Universal Checkout API. Use a third-party API (e.g., Rainforest) for variant discovery.

---

## Agent Integration (Clawdbot Skill)

Rye provides an official Clawdbot (Claude Code) skill for agent-driven shopping: [`rye-com/clawdbot-buy-anything`](https://github.com/rye-com/clawdbot-buy-anything).

### Installation

```bash
clawdhub install buy-anything
# or
npx clawdhub install buy-anything
```

### How the Agent Skill Works

The skill is a **doc-only skill** (SKILL.md + README.md, no code). It teaches the agent to call Rye's REST API via `curl`.

```
User shares product URL (Amazon or Shopify)
        |
        v
Agent collects: shipping address + spending limit
        |
        v
Agent opens BasisTheory card capture page in browser
  https://mcp.rye.com/bt-card-capture
  (PCI-compliant — card never touches the agent)
        |
        v
User pastes back opaque BasisTheory token (UUID)
        |
        v
Agent calls: POST /api/v1/partners/clawdbot/purchase
  { productUrl, buyer, paymentMethod: { basisTheoryToken },
    constraints: { maxTotalPrice } }
        |
        v
Agent polls: GET /api/v1/partners/clawdbot/purchase/{id}
  every 5 seconds until completed | failed
        |
        v
Show confirmation (product name, total, order ID)
Save token + address to memory for future 1-click purchases
```

### Agent Skill Properties

| Property | Details |
|----------|---------|
| **Auth** | No API key needed — `/partners/clawdbot/` path handles auth |
| **Payment** | BasisTheory token (card capture via hosted page) |
| **Merchants** | Amazon + Shopify only |
| **Spending limits** | `constraints.maxTotalPrice` in cents |
| **Memory** | Saves BT token + address + spending limit locally |
| **CVC refresh** | BasisTheory CVC expires after 24 hours; refresh URL provided |
| **Pricing** | Shopify: no markup. Amazon: 3% fee |

### Example Conversation Flow

```
User: Buy this for me https://amazon.com/dp/B0DJLKV4N9

Agent: I'll help you buy that! Where should I ship it?

User: John Doe, 123 Main St, San Francisco CA 94102, john@example.com

Agent: What's your max purchase price?

User: $500

Agent: Opening a secure card entry page in your browser...
       [Opens https://mcp.rye.com/bt-card-capture]

User: d1ff0c32-a1b2-4c3d-8e4f-567890abcdef

Agent: Order submitted! Waiting for confirmation...
       [Polls GET /purchase/{id} every 5 seconds]

Agent: Order confirmed!
       Product: Wireless Earbuds Pro
       Total: $358.44 (includes 3% service fee)
       Order ID: RYE-ABC123
```

After saving credentials, future purchases are one-step:

```
User: Buy this https://cool-store.com/products/gadget
Agent: Order placed! Total: $29.99
```

---

## SDKs

Official SDKs are available for multiple languages:

| Language | Package | Installation |
|----------|---------|-------------|
| **TypeScript** | `checkout-intents` | `npm install checkout-intents` |
| **Python** | `checkout_intents` | `pip install checkout-intents` |
| **Ruby** | — | Available via docs |
| **Java** | — | Available via docs |

### TypeScript Example

```typescript
import CheckoutIntents from 'checkout-intents';

const client = new CheckoutIntents({
  apiKey: process.env['RYE_API_KEY'],
});

// Create checkout intent
const intent = await client.checkoutIntents.create({
  buyer: {
    firstName: 'John', lastName: 'Doe',
    email: 'john@example.com', phone: '212-333-2121',
    address1: '123 Main St', city: 'New York',
    province: 'NY', country: 'US', postalCode: '10001',
  },
  productUrl: 'https://flybyjing.com/collections/shop/products/sichuan-chili-crisp',
  quantity: 1,
});

// Poll for status
const status = await client.checkoutIntents.retrieve(intent.id);

// Confirm with payment
const confirmed = await client.checkoutIntents.confirm(intent.id, {
  paymentMethod: { type: 'stripe_token', stripeToken: 'tok_visa' }
});
```

### Python Example

```python
from checkout_intents import CheckoutIntents

client = CheckoutIntents(api_key=os.environ["RYE_API_KEY"])

intent = client.checkout_intents.create(
    buyer={
        "first_name": "John", "last_name": "Doe",
        "email": "john@example.com", "phone": "212-333-2121",
        "address1": "123 Main St", "city": "New York",
        "province": "NY", "country": "US", "postal_code": "10001",
    },
    product_url="https://flybyjing.com/collections/shop/products/sichuan-chili-crisp",
    quantity=1,
)
```

### Postman Collection

A Postman collection is available for API testing via the Rye documentation.

---

## Environments

Rye provides completely isolated staging and production environments.

| | Staging | Production |
|---|---|---|
| **API Base** | `https://staging.api.rye.com/api/v1/` | `https://api.rye.com/api/v1/` |
| **Console** | `https://staging.console.rye.com` | `https://console.rye.com` |
| **API Keys** | Separate (staging-only) | Separate (production-only) |
| **Orders** | Not executed (safe testing) | Real orders placed |
| **Payments** | Stripe test cards | Real charges |
| **Rate Limits** | 5 req/sec, 50 req/day | 5 req/sec, 50 req/day (default) |

**Key rules:**
- Staging keys do NOT work in production, and vice versa
- Staging `/confirm` endpoint does not actually place orders
- Use Stripe test card numbers in staging (e.g., `4242 4242 4242 4242`)

### Testing Workflow

1. Set up a Shopify development store (via Shopify Partners)
2. Enable test mode for Shopify Payments
3. Use the staging API with a product URL from your test store
4. Follow the quickstart guide to run a test checkout

---

## Limitations & Constraints

| Limitation | Details |
|------------|---------|
| **Geography** | US market only |
| **Product types** | Physical products only (no digital goods, subscriptions, or services) |
| **Quantity** | One product per checkout intent |
| **Webhooks** | Not available — must use polling |
| **Anti-bot merchants** | Merchants with advanced anti-bot protections (e.g., REI) are unsupported |
| **Rate limits** | Default: 5 requests/second, 50 requests/day |
| **Variants** | Not discoverable via Rye API — use third-party APIs |
| **Amazon routing** | Orders go through Rye's account — cannot connect to buyer's personal Amazon account |
| **Offer retrieval** | May take up to 45 minutes for some merchants (poll every 10 seconds) |
| **Best Buy** | Requires prior approval for production use |

### Error Handling

Rye uses standard HTTP status codes:

| Code | Meaning |
|------|---------|
| 200 | Success |
| 400 | Bad Request — invalid input parameters |
| 401 | Unauthorized — missing or invalid API key |
| 403 | Forbidden — feature not approved for your account |
| 404 | Not Found |
| 429 | Rate Limited |
| 500 | Internal Server Error (retry) |

Error responses include `error` code and `message` fields. Integrations should implement retry logic where appropriate.

---

## Pricing

| Plan | Cost | Notes |
|------|------|-------|
| **Universal Checkout** | $149/month | First month free |
| **Amazon orders** | +3% fee | Covers transaction costs |
| **Shopify orders** | No markup | Standard store pricing |

The Sync API (legacy/alternative) uses GMV-based fees with volume discounts and deposit requirements.

---

## API Comparison: Universal Checkout vs Sync

Rye offers two APIs that serve different use cases:

| Dimension | Universal Checkout | Sync API |
|-----------|-------------------|----------|
| **Merchant support** | Any product URL (Amazon, Shopify, Best Buy, etc.) | Shopify and Amazon only |
| **Architecture** | REST API | GraphQL |
| **Cart support** | No (one product per intent) | Yes (full cart functionality) |
| **Product data** | Limited (via offer response) | Comprehensive (variants, images, etc.) |
| **Payment** | Stripe/BasisTheory token generated by buyer | Developer charges via own payment provider; Rye pays merchant from deposit |
| **Amazon routing** | Via Rye's account (auto-Prime >$15) | Via developer's Amazon Business account |
| **Pricing** | $149/month, first month free | GMV-based fees with volume discounts |
| **Webhooks** | No | Yes |
| **Commissions** | No | Yes (affiliate commissions) |
| **Surcharges** | No | Customizable |

**Hybrid approach:** Use Sync API for product data retrieval and Universal Checkout for order submission.

---

## Key Diagrams

### Complete Checkout Flow (Two-Step)

```
Developer/Agent                  Rye API                     Merchant
      |                              |                          |
      |-- 1. POST /checkout-intents->|                          |
      |   { productUrl, buyer,      |                          |
      |     quantity }              |                          |
      |                              |-- Fetch offer ---------->|
      |                              |   (price, tax, shipping) |
      |                              |                          |
      |<-- 2. { id, status:         |<-- Offer data -----------|
      |    "retrieving_offer" } ----|                          |
      |                              |                          |
      |-- 3. GET /checkout-intents/ |                          |
      |   {id} (poll every 10s) --->|                          |
      |                              |                          |
      |<-- 4. { status:             |                          |
      |   "awaiting_confirmation",  |                          |
      |    offer: { product,        |                          |
      |    subtotal, tax,           |                          |
      |    shipping, total } } -----|                          |
      |                              |                          |
      |-- 5. POST /checkout-intents/|                          |
      |   {id}/confirm              |                          |
      |   { paymentMethod:          |                          |
      |     { stripeToken } } ----->|                          |
      |                              |-- Place order ---------->|
      |                              |   (RyeBot checkout)     |
      |                              |                          |
      |<-- 6. { status:             |<-- Order confirmation ---|
      |   "completed",              |                          |
      |    orderId } --------------|                          |
      |                              |                          |
      |                              |              Merchant ships
      |                              |              to buyer directly
```

### Agent Shopping Flow (Clawdbot)

```
User                    Agent (Claude)            BasisTheory          Rye API
  |                         |                         |                   |
  |-- "Buy this [URL]" --->|                         |                   |
  |                         |                         |                   |
  |<-- "Where to ship?" ---|                         |                   |
  |-- Address + email ----->|                         |                   |
  |                         |                         |                   |
  |<-- "Max price?" --------|                         |                   |
  |-- "$500" -------------->|                         |                   |
  |                         |                         |                   |
  |<-- Opens card capture   |                         |                   |
  |    page in browser ---->|                         |                   |
  |                         |                         |                   |
  |-- Enter card details --|------------------------>|                   |
  |   (secure iframe)      |                         |                   |
  |                         |                         |                   |
  |<-- Token UUID ----------|<-----------------------|                   |
  |-- Paste token --------->|                         |                   |
  |                         |                         |                   |
  |                         |-- POST /purchase -------|----------------->|
  |                         |   { productUrl, buyer, |                   |
  |                         |     basisTheoryToken,  |                   |
  |                         |     maxTotalPrice }    |                   |
  |                         |                         |                   |
  |                         |<-- { id: "ci_..." } ---|                   |
  |                         |                         |                   |
  |                         |-- GET /purchase/{id} --|-- Poll every 5s ->|
  |                         |                         |                   |
  |                         |<-- { state: "completed",|                  |
  |                         |     offer, orderId } ---|                   |
  |                         |                         |                   |
  |<-- "Order confirmed!    |                         |                   |
  |    Total: $358.44" -----|                         |                   |
  |                         |                         |                   |
  |<-- "Save card for       |                         |                   |
  |    next time?" ---------|                         |                   |
```

### Payment Token Flow

```
+-------------------+     +-------------------+     +-------------------+
|   Card Capture    |     |   Token Vault     |     |   Rye API         |
|   (Browser)       |     |   (Stripe / BT)   |     |                   |
|                   |     |                   |     |                   |
|  Secure iframe    |     |  Card → Token     |     |  Token → Payment  |
|  collects card    |---->|  (PCI-compliant   |---->|  (detokenize +    |
|  details          |     |   vault)          |     |   pay merchant)   |
|                   |     |                   |     |                   |
|  Card NEVER       |     |  Token is opaque  |     |  Rye never sees   |
|  reaches agent    |     |  (cannot reverse) |     |  raw card data    |
|  or Rye server    |     |                   |     |                   |
+-------------------+     +-------------------+     +-------------------+
```

---

## Summary

Rye enables agentic commerce by providing:

1. **Universal Checkout API** — turn any product URL into a completed purchase with a single API call, no merchant integration required
2. **Broad merchant coverage** — Amazon, Shopify, Best Buy, and thousands more via RyeBot web agent automation
3. **PCI-compliant payment flow** — card data tokenized via Stripe or BasisTheory; never touches the agent or Rye's servers
4. **Agent-native skill** — official Clawdbot (Claude Code) skill with conversational checkout, spending limits, and saved credentials
5. **Automatic promo code discovery** — Rye finds and applies the best available discount at checkout
6. **Simple pricing** — $149/month flat + 3% on Amazon orders; no markup on Shopify
7. **Multi-language SDKs** — TypeScript, Python, Ruby, Java

The core innovation is **abstracting the entire merchant checkout pipeline** into a single API call. Instead of building custom integrations for each merchant's cart, shipping, tax, and payment systems, developers provide a product URL and Rye's web agent handles the rest — achieving 90%+ reliability with <35s latency across thousands of merchants.
