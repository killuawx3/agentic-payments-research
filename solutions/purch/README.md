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

## Technology Stack

| Component | Technology |
|-----------|-----------|
| **Payment protocol** | x402 (HTTP 402 negotiation) |
| **Settlement chain** | Solana |
| **Currency** | USDC (SPL token) |
| **Smart wallets** | Crossmint |
| **Token** | $PURCH (utility token for premium features) |
| **API spec** | OpenAPI 3.1.0 |
| **Docs** | Docusaurus |
| **Web app** | app.purch.xyz |
| **Vault marketplace** | vault.purch.xyz |

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

The core innovation is combining **AI-powered product discovery** with **x402 crypto-native checkout** — agents don't need API keys, credit cards, or merchant accounts. They pay USDC per API call and the purchase cost is settled in the same transaction. The Vault marketplace adds a unique dimension where agents can buy capabilities (skills, knowledge, personas) from other agents or humans.
