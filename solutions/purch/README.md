# Purch: AI Shopping Concierge for Agentic Commerce

> **Describe what you need, and Purch hunts it down. Pay with USDC on Solana.**

Purch is an AI-powered shopping concierge that enables both humans and AI agents to discover and purchase physical products from **Amazon and Shopify** through natural language, pay with **USDC on Solana** via the **x402 micropayment protocol**, and access a marketplace of AI agent skills, knowledge bases, and personas (the "Vault"). It combines conversational product discovery, URL-based instant checkout, social-media-driven gift recommendations, and a digital asset marketplace — all settled in crypto with no traditional payment rails.

---

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Core Features](#core-features)
  - [Chat (Conversational Shopping)](#1-chat-conversational-shopping)
  - [Quick Buy (URL-Based Instant Checkout)](#2-quick-buy-url-based-instant-checkout)
  - [Gift Hunter (Social Media Analysis)](#3-gift-hunter-social-media-analysis)
  - [Vault (AI Agent Marketplace)](#4-vault-ai-agent-marketplace)
- [API Reference](#api-reference)
  - [Product Search](#1-product-search)
  - [AI Shopping Assistant](#2-ai-shopping-assistant)
  - [Buy Product](#3-buy-product)
  - [Vault Search](#4-vault-search)
  - [Vault Buy](#5-vault-buy)
  - [Vault Download](#6-vault-download)
- [Payment Architecture (x402 on Solana)](#payment-architecture-x402-on-solana)
- [Merchant Coverage](#merchant-coverage)
- [Technology Stack](#technology-stack)
- [Pricing](#pricing)
- [Key Diagrams](#key-diagrams)

---

## Architecture Overview

```
+------------------------------------------------------------------+
|                     USER / AI AGENT                               |
|  Natural language (Chat) | Product URL (Quick Buy) | Social URL  |
|  API calls (x402/search, x402/shop, x402/buy)                    |
+----------------------------------+-------------------------------+
                                   |
                    x402 Payment (USDC on Solana)
                    PAYMENT-SIGNATURE header
                                   |
                                   v
+------------------------------------------------------------------+
|                        PURCH PLATFORM                             |
|                   https://api.purch.xyz                           |
|                                                                   |
|  +-------------+  +-------------+  +-------------+               |
|  | AI Shopping |  | Product     |  | Gift        |               |
|  | Assistant   |  | Search      |  | Hunter      |               |
|  | (NLP →      |  | (structured |  | (social     |               |
|  |  product    |  |  filters,   |  |  profile    |               |
|  |  recs)      |  |  paginated) |  |  analysis)  |               |
|  +------+------+  +------+------+  +------+------+              |
|         |                |                |                       |
|         +----------------+----------------+                       |
|                          |                                        |
|                    +-----+------+                                 |
|                    | Purchase   |  POST /x402/buy                 |
|                    | Engine     |  ASIN or productUrl +           |
|                    |            |  shippingAddress + email         |
|                    +-----+------+                                 |
|                          |                                        |
|  +-------------+         |         +-------------------+          |
|  | Vault       |         |         | Crossmint         |          |
|  | Marketplace |         |         | (Smart Wallets)   |          |
|  | (skills,    |         |         |                   |          |
|  |  knowledge, |         |         |                   |          |
|  |  personas)  |         |         |                   |          |
|  +-------------+         |         +-------------------+          |
+---------------------------+----------------------------------+---+
                            |
               +------------+------------+
               |                         |
               v                         v
         +-----------+            +-----------+
         |  Amazon   |            |  Shopify  |
         |  (ASIN)   |            |  (URL +   |
         |           |            |  variant) |
         +-----------+            +-----------+
               |                         |
               +------------+------------+
                            |
                    Order Fulfillment
              (merchant ships to buyer directly)
```

### How It All Connects

1. **User/Agent** interacts via Chat (natural language), Quick Buy (paste URL), Gift Hunter (social profile URL), or direct API calls
2. **Every API call** is paid via x402 protocol — USDC on Solana, sent as `PAYMENT-SIGNATURE` header
3. **Purch AI** searches across 1B+ product items from Amazon and Shopify
4. **Purchase** is executed via ASIN (Amazon) or product URL + variantId (Shopify)
5. **Merchant** fulfills order and ships directly to buyer's address
6. **Vault** operates as a separate marketplace for AI agent digital assets (skills, knowledge, personas)

---

## Core Features

### 1. Chat (Conversational Shopping)

An AI shopping assistant that understands natural language queries and returns curated product recommendations.

```
User                           Purch Chat
  |                                |
  |-- "I need wireless earbuds    |
  |    under $50 with good        |
  |    noise cancellation" ------>|
  |                                |
  |                                |-- Search 1B+ products
  |                                |-- Cross-reference reviews
  |                                |-- Filter by specs/budget
  |                                |
  |<-- Curated picks (3-5 items) -|
  |    with prices, ratings,      |
  |    product links              |
  |                                |
  |-- "Show me the second one" -->|
  |                                |
  |<-- Product details + "Buy?" --|
  |                                |
  |-- "Yes, buy it" ------------->|
  |                                |
  |<-- Order confirmed, paid -----|
  |    in USDC                    |
```

**Capabilities:**
- Natural language product discovery
- Follow-up questions and context tracking
- Budget-aware recommendations
- Feature-based filtering (specs, ratings, Prime eligibility)
- Gift recommendations for various occasions
- Price comparisons and alternatives

**Privacy:** Conversations encrypted, payment info not retained in chat, history deletable.

**Access:** Web app at `app.purch.xyz`

---

### 2. Quick Buy (URL-Based Instant Checkout)

Paste an Amazon product URL and purchase instantly with USDC — no searching, no Amazon account needed.

```
User                           Purch Quick Buy
  |                                |
  |-- Paste Amazon URL ---------->|
  |   amazon.com/dp/B0DJLKV4N9   |
  |                                |
  |<-- Product details shown -----|
  |    (title, price, image)      |
  |                                |
  |-- Confirm + pay USDC -------->|
  |                                |
  |<-- Order confirmed, ----------|
  |    shipping to your address   |
```

**Flow:**
1. Find item on Amazon
2. Paste product URL into Quick Buy
3. Review order details
4. Pay with USDC, receive confirmation

**Privacy:** No Amazon account credentials needed. Sign up with email only. Purchase history private.

---

### 3. Gift Hunter (Social Media Analysis)

Analyzes public social media profiles to generate personalized gift recommendations.

```
User                           Gift Hunter
  |                                |
  |-- Submit social profile URL -->|
  |   (Twitter, Instagram,       |
  |    or TikTok)                 |
  |                                |
  |                                |-- Analyze public profile:
  |                                |   bio, posts, hashtags,
  |                                |   content themes
  |                                |
  |                                |-- Detect: hobbies, brands,
  |                                |   lifestyle, trends
  |                                |
  |<-- 3 personalized gift --------|
  |    suggestions on Amazon      |
  |                                |
  |-- Select + buy with USDC ---->|
```

**Supported platforms:** Twitter/X, Instagram, TikTok (public profiles only)

**Privacy:**
- Only analyzes publicly available information
- Profile data processed and immediately discarded
- No storing of personal information

**Access tiers:**

| Tier | Searches | Requirement |
|------|----------|-------------|
| **Free** | 1 search | New users |
| **Premium** | 5/day | Hold $PURCH tokens |
| **Pay Per Use** | Credits (never expire) | Purchase credits |

---

### 4. Vault (AI Agent Marketplace)

A marketplace for AI agent digital assets — skills, knowledge bases, and personas — purchasable by both humans and AI agents via x402 micropayments.

```
+---------------------------------------------------------------+
|                        PURCH VAULT                             |
|                    vault.purch.xyz                              |
|                                                                |
|  +------------------+ +------------------+ +-----------------+ |
|  |     Skills       | |   Knowledge      | |   Personas      | |
|  |                  | |                  | |                 | |
|  | Ready-to-use     | | Curated domain   | | Customizable    | |
|  | agent workflows  | | expertise and    | | agent comms     | |
|  |                  | | reference data   | | styles and      | |
|  | Examples:        | |                  | | personalities   | |
|  | - Marketing      | | Examples:        | |                 | |
|  |   automation     | | - Industry data  | | Examples:       | |
|  | - Code review    | | - Best practices | | - Professional  | |
|  | - Data analysis  | | - Domain guides  | | - Casual        | |
|  |                  | |                  | | - Technical     | |
|  +------------------+ +------------------+ +-----------------+ |
|                                                                |
|  Categories: Marketing | Development | Automation |            |
|              Career | iOS | Productivity                       |
|                                                                |
|  Pricing: Per-item in USDC (set by creator)                    |
|  Creators: Human or Agent (both can publish)                   |
|  Delivery: ZIP download after purchase                         |
+----------------------------------------------------------------+
```

#### Vault Item Properties

| Field | Type | Description |
|-------|------|-------------|
| `id` | UUID | Unique identifier |
| `productType` | enum | `skill`, `knowledge`, or `persona` |
| `slug` | string | URL-friendly identifier |
| `title` | string | Display name |
| `cardDescription` | string | Short description |
| `price` | number | Price in USDC (whole units) |
| `category` | string | marketing, development, automation, career, ios, productivity |
| `creator` | object | `{ name, type: "human"|"agent", avatarUrl }` |
| `downloads` | number | Total downloads |
| `featured` | boolean | Featured item flag |
| `boost` | object | `{ active, expiresAt }` — promotional boost |

#### Vault Workflow (API)

```
Agent                        Purch API
  |                              |
  |-- GET /x402/vault/search -->|  ($0.01 per search)
  |   ?q=marketing&category=... |
  |                              |
  |<-- VaultSearchResponse -----|
  |   [{ slug, title, price,   |
  |      productType, creator }]|
  |                              |
  |-- POST /x402/vault/buy ---->|  ($ = item price in USDC)
  |   { slug, email,           |
  |     walletAddress }        |
  |                              |
  |<-- { purchaseId,           |
  |      downloadToken } ------|
  |                              |
  |-- GET /x402/vault/download/|  ($0.01 per download)
  |   {purchaseId}?            |
  |   downloadToken=xxx ------>|
  |                              |
  |<-- ZIP file download -------|
```

---

## API Reference

**Base URL:** `https://api.purch.xyz`

**Authentication:** x402 protocol — every request requires a `PAYMENT-SIGNATURE` header containing a USDC payment proof on Solana. No API keys — payment IS authentication.

**OpenAPI spec:** `https://api.purch.xyz/openapi.json`

### 1. Product Search

```
GET /x402/search
Cost: $0.01 per call
```

Search for products with structured filters across Amazon and Shopify.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `q` | string | Yes | Search query (min 1 char) |
| `priceMin` | number | No | Minimum price filter |
| `priceMax` | number | No | Maximum price filter |
| `brand` | string | No | Filter by brand |
| `page` | integer | No | Page number (default: 1) |

**Response:** `SearchResponse`
```json
{
  "products": [
    {
      "asin": "B0DJLKV4N9",
      "title": "Wireless Earbuds Pro",
      "price": 49.99,
      "currency": "USD",
      "rating": 4.5,
      "reviewCount": 1234,
      "imageUrl": "https://...",
      "productUrl": "https://amazon.com/dp/B0DJLKV4N9",
      "source": "amazon"
    }
  ],
  "totalResults": 150,
  "page": 1,
  "hasMore": true
}
```

### 2. AI Shopping Assistant

```
POST /x402/shop
Cost: $0.10 per call
```

Natural language shopping — describe what you want, get AI-curated recommendations.

**Request:**
```json
{
  "message": "I need a birthday gift for my dad who loves grilling, budget around $75",
  "context": {
    "priceRange": { "min": 50, "max": 100 },
    "preferences": ["outdoor", "cooking", "premium quality"]
  }
}
```

**Response:** `ShopResponse`
```json
{
  "reply": "Here are some great grilling gifts for your dad...",
  "products": [
    {
      "asin": "B09XYZ...",
      "title": "Weber Premium Grill Set",
      "price": 72.99,
      "currency": "USD",
      "rating": 4.7,
      "reviewCount": 5678,
      "imageUrl": "https://...",
      "productUrl": "https://amazon.com/dp/...",
      "source": "amazon"
    }
  ]
}
```

### 3. Buy Product

```
POST /x402/buy
Cost: Dynamic (product total including tax + shipping)
```

Purchase a product. The x402 payment amount equals the product's total price.

**Request:**
```json
{
  "asin": "B0DJLKV4N9",
  "shippingAddress": {
    "name": "John Doe",
    "line1": "123 Main St",
    "line2": "Apt 4",
    "city": "San Francisco",
    "state": "CA",
    "postalCode": "94102",
    "country": "US",
    "phone": "+14155551234"
  },
  "email": "john@example.com"
}
```

**Alternative:** Use `productUrl` instead of `asin`:
- Amazon: `https://amazon.com/dp/ASIN`
- Shopify: product URL + `variantId` (required)

**Response:** `X402BuyResponse`
```json
{
  "orderId": "550e8400-e29b-41d4-a716-446655440000",
  "status": "processing",
  "product": {
    "title": "Wireless Earbuds Pro",
    "imageUrl": "https://...",
    "price": { "amount": "49.99", "currency": "USD" }
  },
  "totalPrice": {
    "amount": "54.23",
    "currency": "USD"
  }
}
```

### 4. Vault Search

```
GET /x402/vault/search
Cost: $0.01 per call
```

Search AI agent skills, knowledge bases, and personas.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `q` | string | No | Search query |
| `category` | enum | No | marketing, development, automation, career, ios, productivity |
| `productType` | enum | No | skill, knowledge, persona |
| `minPrice` | integer | No | Min price in USDC |
| `maxPrice` | integer | No | Max price in USDC |
| `cursor` | UUID | No | Pagination cursor |
| `limit` | integer | No | Items per page (1-100, default 30) |

**Response:** `VaultSearchResponse`
```json
{
  "items": [
    {
      "id": "uuid",
      "productType": "skill",
      "slug": "marketing-automation-pro",
      "title": "Marketing Automation Pro",
      "cardDescription": "Automate your marketing workflows...",
      "price": 5,
      "category": "marketing",
      "creator": { "name": "AgentLab", "type": "agent", "avatarUrl": null },
      "downloads": 342,
      "featured": true,
      "boost": { "active": true, "expiresAt": "2026-04-01T00:00:00Z" }
    }
  ],
  "nextCursor": "uuid-or-null"
}
```

### 5. Vault Buy

```
POST /x402/vault/buy
Cost: Dynamic (item price in USDC)
```

**Request:**
```json
{
  "slug": "marketing-automation-pro",
  "email": "user@example.com",
  "walletAddress": "7nYBbH...xyz"
}
```

**Response:** `X402VaultBuyResponse`
```json
{
  "purchaseId": "uuid",
  "downloadToken": "64-hex-char-secret-token",
  "item": {
    "slug": "marketing-automation-pro",
    "title": "Marketing Automation Pro",
    "productType": "skill",
    "price": 5,
    "coverImageUrl": "https://..."
  }
}
```

### 6. Vault Download

```
GET /x402/vault/download/{purchaseId}
Cost: $0.01 per download (including re-downloads)
```

| Parameter | Type | Location | Required | Description |
|-----------|------|----------|----------|-------------|
| `purchaseId` | UUID | path | Yes | Purchase ID |
| `downloadToken` | string | query | Yes | 64-hex-char secret |
| `txSignature` | string | query | No | On-chain tx signature |

**Response:** ZIP file download (`application/octet-stream`)

---

## Payment Architecture (x402 on Solana)

Purch uses the **x402 payment protocol** — every API call is paid for via USDC on Solana. There are no API keys. Payment proof IS the authentication.

### How x402 Works on Purch

```
Agent/User                      Purch API                     Solana
    |                               |                            |
    |-- Request to /x402/search --->|                            |
    |   (no payment header)         |                            |
    |                               |                            |
    |<-- 402 Payment Required ------|                            |
    |   { x402Version: 1,          |                            |
    |     accepts: [{              |                            |
    |       scheme: "exact",       |                            |
    |       network: "solana",     |                            |
    |       asset: "USDC",         |                            |
    |       maxAmountRequired: "0.01",                           |
    |       payTo: "Purch_wallet", |                            |
    |       maxTimeoutSeconds: 60  |                            |
    |     }]                       |                            |
    |   }                          |                            |
    |                               |                            |
    |-- USDC transfer on Solana ---|-------------------------->|
    |                               |                            |
    |-- Retry request with -------->|                            |
    |   PAYMENT-SIGNATURE header   |                            |
    |   (tx proof)                 |                            |
    |                               |-- Verify payment -------->|
    |                               |                            |
    |<-- 200 OK + response ---------|<-- Confirmed -------------|
```

### x402 Response Schema (402 Payment Required)

When a request is made without payment, the API returns:

```json
{
  "x402Version": 1,
  "error": "Payment required",
  "accepts": [
    {
      "scheme": "exact",
      "network": "solana",
      "maxAmountRequired": "0.01",
      "resource": "/x402/search",
      "description": "Search products",
      "mimeType": "application/json",
      "maxTimeoutSeconds": 60,
      "asset": "USDC",
      "payTo": "..."
    }
  ]
}
```

### Key Properties

- **No API keys** — payment signature is the only authentication
- **Per-call pricing** — each endpoint has a fixed or dynamic cost
- **Dynamic pricing** for purchases — the x402 amount equals the product total (including tax + shipping)
- **Settlement chain:** Solana (USDC SPL token)
- **Wallets:** Powered by Crossmint (smart wallets)

---

## Merchant Coverage

| Merchant | Product ID | Variant Selection | Notes |
|----------|-----------|-------------------|-------|
| **Amazon** | ASIN (10-char code) or `amazon.com/dp/ASIN` URL | Via ASIN deep link | Primary catalog (1B+ products) |
| **Shopify** | Product URL | `variantId` required | Any Shopify store |

### Product Schema

```json
{
  "asin": "B0DJLKV4N9",
  "title": "Product Name",
  "price": 49.99,
  "currency": "USD",
  "rating": 4.5,
  "reviewCount": 1234,
  "imageUrl": "https://...",
  "productUrl": "https://amazon.com/dp/...",
  "source": "amazon"
}
```

**Source field:** `amazon` or `shopify`

---

## Internal Architecture (From Source Code)

The `x-purch` repo ([github.com/purch-xyz/x-purch](https://github.com/purch-xyz/x-purch)) reveals how Purch works under the hood. It's a surprisingly thin layer — Purch is essentially **x402 middleware + Crossmint order API + Supabase persistence**.

### System Architecture

```
Client (wallet + x402 signer)
        |
        v
POST /orders/solana
        |
        v
+-------+------------------------------------------+
|              Hono HTTP Router                     |
|                                                   |
|  1. Proxy Fix Middleware                          |
|     (HTTPS detection behind load balancer)        |
|                                                   |
|  2. Payer Logging Middleware                      |
|     (logs wallet address + payment header)        |
|                                                   |
|  3. x402 Payment Middleware (@x402/hono)          |
|     - Coinbase CDP facilitator verifies payment   |
|     - ExactSvmScheme for Solana USDC              |
|     - Price: $0.01 per order creation             |
|     - payTo: env.X402_SOLANA_WALLET_ADDRESS       |
|     - Returns 402 if no valid payment             |
|                                                   |
|  4. Zod Validation Middleware                     |
|     (validates email, payerAddress, productUrl,   |
|      physicalAddress)                             |
|                                                   |
|  5. Order Handler                                 |
|     - Resolves product URL to locator             |
|     - Calls Crossmint Orders API                  |
|     - Saves order + user to Supabase/Postgres     |
|     - Returns orderId + clientSecret +            |
|       serializedTransaction                       |
+---------------------------------------------------+
        |
        v
+-------+------------------------------------------+
|           Crossmint Orders API                    |
|   POST /2022-06-09/orders                         |
|                                                   |
|   Body:                                           |
|   { recipient: { email, physicalAddress },        |
|     locale,                                       |
|     payment: { method: "solana",                  |
|       currency: "usdc", payerAddress },           |
|     lineItems: [{ productLocator }] }             |
|                                                   |
|   Returns:                                        |
|   { clientSecret, order: { orderId,               |
|     payment: { status, preparation: {             |
|       serializedTransaction } },                  |
|     quote: { totalPrice }, lineItems } }          |
+---------------------------------------------------+
        |
        v
+-------+------------------------------------------+
|           Supabase / PostgreSQL                   |
|                                                   |
|   users_x402:                                     |
|   { id, walletAddress, network, email }           |
|   (unique: walletAddress + network)               |
|                                                   |
|   orders_x402:                                    |
|   { id, userId, network, clientSecret(bcrypt),    |
|     createdAt, updatedAt }                        |
+---------------------------------------------------+
```

### The Key Insight: Crossmint Is the Actual Commerce Engine

Purch does NOT handle product resolution, pricing, tax, shipping, or order placement itself. **Crossmint does all of that.** Purch is a thin wrapper that:

1. **Accepts x402 payments** (via Coinbase CDP + `@x402/hono` middleware)
2. **Resolves product URLs** to Crossmint-compatible locators
3. **Forwards to Crossmint's Orders API** (`POST /2022-06-09/orders`)
4. **Persists order metadata** in Supabase for status tracking
5. **Returns a serialized Solana transaction** for the buyer to sign

Crossmint handles the actual commerce: product lookup, pricing, tax calculation, payment processing, and order fulfillment.

### Product URL Resolution

The source code reveals exactly how Purch routes product URLs to different platforms:

```typescript
// From src/orders/createCrossmintOrder.ts

PLATFORM_HOSTNAMES = {
  amazon:            ["amazon."],              // → "amazon:{url}"
  shopify:           ["myshopify.com"],         // → "shopify:{url}:{variantId}"
  browserAutomation: [                          // → "url:{url}"
    "nike.com", "adidas.com", "crocs.com",
    "gymshark.com", "on.com"
  ]
}
```

**Locator format by platform:**

| Platform | Detection | Locator Format | Notes |
|----------|-----------|---------------|-------|
| **Amazon** | Hostname contains `amazon.` | `amazon:{productUrl}` | Includes `a.co`, `amzn.to` short URL expansion |
| **Shopify** | Hostname contains `myshopify.com` OR path contains `/products/` | `shopify:{baseUrl}:{variantId}` | `variantId` extracted from `?variant=` query param; **required** |
| **Browser Automation** | Hostname matches Nike, Adidas, Crocs, Gymshark, On | `url:{productUrl}` | Crossmint uses browser automation for these merchants |
| **Everything else** | Fallback | `url:{productUrl}` | Also uses browser automation |

**Important discoveries from the code:**
- **Short URL expansion:** `a.co`, `amzn.to`, `amzn.com`, `bit.ly`, `t.co` are automatically followed (HEAD request with 5s timeout) before locator resolution
- **URL validation:** Must be HTTPS, no IP addresses, no localhost/local/internal domains, no auth credentials in URL
- **Shopify variant is mandatory:** If a Shopify URL lacks `?variant=`, the API throws a 400 error
- **Browser automation sites** (Nike, Adidas, Crocs, Gymshark, On) are hardcoded — these go through Crossmint's browser automation checkout, not a direct API integration

### x402 Payment Mechanism (Source Code Detail)

```typescript
// From src/index.ts

// 1. Coinbase CDP facilitator validates x402 payments
const facilitatorConfig = createFacilitatorConfig(
  env.X402_CDP_API_KEY_ID,
  env.X402_CDP_API_KEY_SECRET,
);

// 2. Register Solana mainnet with ExactSvmScheme
const facilitatorClient = new HTTPFacilitatorClient(facilitatorConfig);
const resourceServer = new x402ResourceServer(facilitatorClient)
  .register(SOLANA_MAINNET_CAIP2, new ExactSvmScheme());

// 3. Payment middleware on /orders/solana
paymentMiddleware({
  "POST /orders/solana": {
    accepts: [{
      scheme: "exact",
      price: "$0.01",               // Fixed price for API call
      network: SOLANA_MAINNET_CAIP2, // solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp
      payTo: env.X402_SOLANA_WALLET_ADDRESS,
    }],
    maxTimeoutSeconds: 300,
  }
}, resourceServer);
```

**Key technical details:**
- Uses **Coinbase CDP** (Coinbase Developer Platform) as the x402 facilitator — not a custom implementation
- Payment scheme is `exact` (exact amount required, not a range)
- Network identifier follows **CAIP-2** standard: `solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp` (Solana mainnet)
- The x402 price for order creation is **$0.01** — this is just the API fee. The actual product cost is handled separately by Crossmint via a **serialized Solana transaction** returned to the buyer for signing
- `maxTimeoutSeconds: 300` (5 minutes) for payment settlement

### Two-Phase Payment Model

This is crucial to understanding Purch's actual payment flow. There are **two separate payments**:

```
Phase 1: x402 API Fee ($0.01 USDC)
    Agent pays $0.01 to Purch's wallet via x402 protocol
    This authenticates the API call and covers Purch's service fee
        |
        v
Phase 2: Product Payment (dynamic, USDC)
    Crossmint returns a serializedTransaction
    Agent signs and submits this Solana transaction
    This pays the actual product cost (price + tax + shipping)
    Payment goes through Crossmint's payment rails
```

So the agent pays **twice**: once for API access ($0.01 via x402), and once for the product (via Crossmint's serialized transaction). The product payment is NOT part of the x402 flow — it's a separate Solana transaction that Crossmint constructs.

### Order Status Security

Order status lookups use **bcrypt-hashed client secrets**:

```
POST /orders/solana → returns { clientSecret: "raw-secret" }
                       (stored as: SHA-256 → bcrypt hash in Postgres)

GET /orders/{id}    → requires Authorization: <raw-secret>
                       (verified: SHA-256(secret) → bcrypt.compare(hash))
```

The client secret is returned once at order creation and cannot be recovered. Status lookups are authenticated against the bcrypt hash — no API key needed.

### Bazaar Discovery (x402 Ecosystem)

The source code includes **Bazaar metadata** in the x402 payment middleware:

```typescript
extensions: {
  bazaar: {
    discoverable: true,
    category: "ecommerce",
    tags: ["orders", "shopping", "amazon", "crypto"],
    inputSchema: { ... },
    outputSchema: { ... },
  }
}
```

This means Purch is **registered in the x402 Bazaar discovery layer** — Coinbase's CDP catalog where agents can browse and discover x402-enabled services. Any agent browsing the Bazaar can find Purch and call it without prior knowledge of the API.

### Supported Merchants (Expanded)

The source code reveals broader merchant support than the public docs suggest:

| Category | Merchants | Locator Prefix |
|----------|-----------|---------------|
| **Amazon** | All Amazon domains + short URLs (a.co, amzn.to) | `amazon:` |
| **Shopify** | Any .myshopify.com or store with `/products/` path | `shopify:` |
| **Browser Automation** | Nike, Adidas, Crocs, Gymshark, On Running | `url:` |
| **Fallback** | Any other HTTPS URL | `url:` |

The `url:` prefix means Crossmint will attempt browser automation checkout — so Purch theoretically supports **any merchant** where Crossmint's browser automation works, not just Amazon and Shopify.

## Technology Stack

| Component | Technology |
|-----------|-----------|
| **Runtime** | Bun v1.1+ |
| **HTTP Framework** | Hono |
| **Payment protocol** | x402 via Coinbase CDP (`@coinbase/x402`, `@x402/hono`, `@x402/svm`) |
| **x402 Facilitator** | Coinbase Developer Platform (CDP) |
| **Commerce engine** | Crossmint Orders API (`/2022-06-09/orders`) |
| **Settlement chain** | Solana mainnet (CAIP-2: `solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp`) |
| **Currency** | USDC (SPL token) |
| **Database** | PostgreSQL via Supabase |
| **ORM** | Drizzle ORM |
| **Validation** | Zod |
| **Env config** | @t3-oss/env-core |
| **Client SDK** | `x402-fetch` (wraps fetch with automatic x402 payment handling) |
| **Token** | $PURCH (utility token for premium features) |
| **API spec** | OpenAPI 3.0.0 (generated from handler definitions) |
| **Docs** | Docusaurus |
| **Web app** | app.purch.xyz |
| **Vault marketplace** | vault.purch.xyz |
| **Deployment** | Google Cloud Run (Docker) |
| **Live API** | `https://x402.purch.xyz` |

---

## Pricing

### API Endpoint Costs

| Endpoint | Cost | Pricing Type |
|----------|------|-------------|
| `GET /x402/search` | $0.01 | Fixed |
| `POST /x402/shop` | $0.10 | Fixed |
| `POST /x402/buy` | Product total (tax + shipping included) | Dynamic |
| `GET /x402/vault/search` | $0.01 | Fixed |
| `POST /x402/vault/buy` | Item price (USDC) | Dynamic |
| `GET /x402/vault/download/{id}` | $0.01 | Fixed (per download) |

### Cost Model

- **No subscription fees** — pure pay-per-call via x402
- **No API keys to manage** — payment IS authentication
- **Dynamic pricing for purchases** — the x402 payment amount equals the actual product cost
- **Re-downloads cost $0.01** — vault items are not free to re-download

---

## Key Diagrams

### Complete Platform Architecture

```
                    +------------------------------------------+
                    |         User / AI Agent                    |
                    |                                            |
                    |  Chat | Quick Buy | Gift Hunter | API     |
                    +-------------------+----------------------+
                                        |
                              x402 (USDC on Solana)
                             PAYMENT-SIGNATURE header
                                        |
                                        v
+-----------------------------------------------------------------------+
|                          PURCH API GATEWAY                             |
|                       https://api.purch.xyz                            |
|                                                                        |
|  +----------------+  +----------------+  +----------------+            |
|  | /x402/search   |  | /x402/shop     |  | /x402/buy      |           |
|  | $0.01/call     |  | $0.10/call     |  | $=product total|           |
|  |                |  |                |  |                |            |
|  | Structured     |  | NL → AI recs   |  | ASIN/URL →     |           |
|  | filters,       |  | + products     |  | order placed   |           |
|  | paginated      |  |                |  | on merchant    |           |
|  +----------------+  +----------------+  +----------------+            |
|                                                                        |
|  +------------------+  +------------------+  +------------------+      |
|  | /x402/vault/     |  | /x402/vault/buy  |  | /x402/vault/     |     |
|  | search           |  | $=item price     |  | download/{id}    |     |
|  | $0.01/call       |  |                  |  | $0.01/call       |     |
|  |                  |  | slug → purchase  |  |                  |     |
|  | Skills,          |  | + download token |  | ZIP file         |     |
|  | knowledge,       |  |                  |  | delivery         |     |
|  | personas         |  |                  |  |                  |     |
|  +------------------+  +------------------+  +------------------+      |
|                                                                        |
+-----------------------------------------------------------------------+
                |                                      |
                v                                      v
       +----------------+                    +------------------+
       | Product Catalog |                    | Vault Catalog    |
       | Amazon (1B+    |                    | Skills           |
       | items) +       |                    | Knowledge        |
       | Shopify stores |                    | Personas         |
       +-------+--------+                    +--------+---------+
               |                                      |
               v                                      v
     Merchant fulfillment                    ZIP file download
     (ships to buyer)                        (digital delivery)
```

### x402 Payment Flow

```
1. Agent calls endpoint (no payment)
          |
          v
2. API returns 402 Payment Required
   { accepts: [{ network: "solana", asset: "USDC",
     maxAmountRequired: "0.01", payTo: "..." }] }
          |
          v
3. Agent sends USDC on Solana to payTo address
          |
          v
4. Agent retries request with PAYMENT-SIGNATURE header
          |
          v
5. API verifies on-chain payment
          |
          v
6. API returns 200 OK with response data
```

### Shopping Flow (End-to-End)

```
User/Agent                    Purch                      Amazon/Shopify
    |                            |                            |
    |-- "Find wireless earbuds   |                            |
    |    under $50" ------------>|                            |
    |   (POST /x402/shop)       |                            |
    |   ($0.10 USDC paid)       |                            |
    |                            |-- Search 1B+ products ---->|
    |                            |                            |
    |<-- AI picks + products ----|<-- Product data -----------|
    |                            |                            |
    |-- "Buy the first one" ---->|                            |
    |   (POST /x402/buy)        |                            |
    |   { asin, shippingAddr,   |                            |
    |     email }               |                            |
    |   ($54.23 USDC paid =     |                            |
    |    product total)         |                            |
    |                            |-- Place order ------------>|
    |                            |                            |
    |<-- { orderId, status:     |<-- Order confirmation -----|
    |      "processing" } ------|                            |
    |                            |                            |
    |                            |              Merchant ships
    |                            |              to buyer address
```

### Gift Hunter Flow

```
User                         Purch Gift Hunter           Social Platform
  |                              |                           |
  |-- Submit profile URL ------->|                           |
  |   (twitter.com/username)    |                           |
  |                              |-- Scrape public profile ->|
  |                              |   bio, posts, hashtags,  |
  |                              |   content themes         |
  |                              |                           |
  |                              |<-- Profile data ----------|
  |                              |   (immediately discarded  |
  |                              |    after processing)     |
  |                              |                           |
  |                              |-- Analyze: hobbies,      |
  |                              |   brands, lifestyle,     |
  |                              |   trends                 |
  |                              |                           |
  |<-- 3 personalized gift ------|                           |
  |    suggestions with         |                           |
  |    Amazon products          |                           |
  |                              |                           |
  |-- Select + buy with USDC -->|                           |
```

---

## Purch vs Rye: Key Differences

| Dimension | Purch | Rye |
|-----------|-------|-----|
| **Payment** | USDC on Solana (x402 protocol) | Credit card (Stripe/BasisTheory tokens) |
| **Auth model** | No API keys — payment proof IS auth | API key (Basic auth) |
| **API pricing** | Per-call x402 micropayments ($0.01-$0.10 + product cost) | $149/month subscription + 3% Amazon fee |
| **Merchants** | Amazon + Shopify | Amazon + Shopify + Best Buy + thousands more |
| **Checkout model** | Single-step (pay + buy in one call) | Two-step (create intent → confirm) or single-step |
| **Product discovery** | Built-in AI shopping assistant (NLP) + structured search | Product lookup by URL only (no search/discovery) |
| **Unique features** | Gift Hunter (social media analysis), Vault (AI agent marketplace) | Promo code auto-discovery, variant selection, shipment tracking |
| **Settlement** | Crypto only (USDC/Solana) | Fiat only (credit cards) |
| **Target user** | Crypto-native agents and users | Traditional web developers and AI apps |
| **Webhooks** | Not documented | Not available (polling) |
| **SDKs** | None (REST + x402) | TypeScript, Python, Ruby, Java |
| **Rate limits** | Not documented | 5 req/sec, 50 req/day |
| **Geography** | Not documented (likely US for Amazon) | US only |

---

## Summary

Purch enables agentic commerce by providing:

1. **AI Shopping Assistant** — natural language product discovery across 1B+ items from Amazon and Shopify ($0.10/query)
2. **Structured Product Search** — filtered, paginated search with price/brand/rating filters ($0.01/query)
3. **One-Call Checkout** — ASIN or URL → paid + ordered in a single API call (cost = product total in USDC)
4. **Gift Hunter** — social media profile analysis (Twitter, Instagram, TikTok) for personalized gift recommendations
5. **Vault Marketplace** — buy and sell AI agent skills, knowledge bases, and personas, purchasable by both humans and agents
6. **x402 Native** — no API keys, no subscriptions, pure pay-per-call via USDC on Solana
7. **Crypto-Only Settlement** — all payments in USDC on Solana via Crossmint smart wallets

The core architecture is a **thin x402 wrapper around Crossmint's commerce engine**. Purch handles payment authentication via Coinbase CDP's x402 facilitator ($0.01/call), resolves product URLs to platform-specific locators (Amazon, Shopify, browser automation for Nike/Adidas/etc.), and forwards everything to Crossmint's Orders API which handles the actual product lookup, pricing, tax, shipping, and fulfillment. Product payment is a separate Solana transaction (serialized by Crossmint, signed by the buyer). The Vault marketplace adds a unique dimension where agents can buy capabilities (skills, knowledge, personas) from other agents or humans. Purch is registered in the **x402 Bazaar discovery layer**, making it discoverable to any agent browsing Coinbase's CDP catalog.
