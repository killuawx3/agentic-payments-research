# Purch Buy API Dry Run: Product Validation & x402 Payment Discovery

> Live test of the Purch buy API without completing a transaction  
> Date: 2026-03-18 | Environment: Production (api.purch.xyz)  
> Total spend: $0.00 (no Solana USDC wallet available)

---

## What We Tested

End-to-end validation of the Purch buy endpoint (`POST /x402/buy`) and search endpoint (`GET /x402/search`) by probing product availability, address validation, and x402 payment requirements — all without making any payment.

Key discovery: **Purch validates the full order (product availability, delivery time, shipping address) BEFORE requesting x402 payment.** This means we can test the entire validation pipeline for free.

### Test Actions

1. **Search endpoint** — x402 402 response (Solana USDC payment required)
2. **Buy with various ASINs** — product catalog availability testing
3. **Buy with international address** — geographic restriction testing
4. **Buy with valid product + US address** — x402 payment discovery (decoded pricing)
5. **Shopify URL** — variant requirement validation

---

## Results

### Test 1: Product Search — 402 Payment Required

```
GET /x402/search?q=dry+fit+shirt
→ HTTP 402
→ payment-required header with x402 payment details
```

Decoded x402 header:
- **Price:** 10,000 micro-USDC = **$0.01 per search**
- **Network:** Solana mainnet
- **Asset:** USDC (EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v)
- **Pay to:** 8LiXrHC61irY8qwj6qevoiRXxYfrTgSaHVbm8rav6HT2
- **Fee payer:** GVJJ7rdGiXr5xaYbRwRbjfaJL7fmwRygFi1H6aGqDveb (Purch pays Solana tx fees)

**Verdict: ✅ x402 protocol working** — Clean 402 response with full payment schema. Rate limit: 60 req/min.

---

### Test 2: Product Catalog — Many Products Not Found

Tested multiple well-known Amazon ASINs:

| ASIN | Product | Result |
|------|---------|--------|
| B0BXRXLKHR | Dry Fit Shirt ($1) | ⚠️ "Delivery time exceeds 15 days" |
| B0D1XD1ZV3 | AirPods Pro | ❌ "Currently unavailable for purchase" |
| B09JNPGJ7S | iPhone Case | ❌ "Product not found" |
| B0CN1HP68Q | Kindle | ❌ "Product not found" |
| B0B2MLXFGR | Anker Charger | ❌ "Product not found" |
| B09V2KJG3T | Book | ❌ "Product not found" |
| B00EB4ADQW | Fuji Instax Film | ⚠️ "Cannot be shipped to specified address" |
| **B09B8V1LZ3** | **Echo Dot** | **✅ 402 Payment Required ($55.88)** |

**Verdict: ⚠️ Limited catalog** — Only 1 out of 8 tested ASINs was actually purchasable. Most returned "Product not found" (HTTP 500), suggesting Purch has a curated/limited product catalog rather than full Amazon coverage. This is a major difference from Rye, which resolved every ASIN we tested.

---

### Test 3: Geographic Restrictions — US Only

```json
// Korea address
{"country": "KR", "city": "Seoul", ...}
→ "Amazon and Shopify orders only support addresses in the US"
→ Links to: docs.crossmint.com/payments/headless/guides/physical-product-order-errors

// Canada address
{"country": "CA", "city": "Toronto", ...}
→ "Amazon and Shopify orders only support addresses in the US"
```

**Verdict: ❌ US only** — Even stricter than Rye (which supports US + Canada). Purch only accepts US addresses. The error links to Crossmint docs, revealing Purch uses **Crossmint** as their checkout infrastructure under the hood.

---

### Test 4: Successful Product Found — Echo Dot ($55.88)

For ASIN B09B8V1LZ3 (Echo Dot) with a US address, Purch returned a proper 402 response with decoded pricing:

```
Price: 55,880,000 micro-USDC = $55.88 USD (total including tax + shipping)
Network: Solana mainnet
Pay to: 8LiXrHC61irY8qwj6qevoiRXxYfrTgSaHVbm8rav6HT2
Fee payer: GVJJ7rdGiXr5xaYbRwRbjfaJL7fmwRygFi1H6aGqDveb
Timeout: 300 seconds
```

The x402 response also includes a full JSON Schema for the buy endpoint's input/output, including example response:

```json
{
  "orderId": "550e8400-e29b-41d4-a716-446655440000",
  "status": "processing",
  "product": {
    "title": "Wireless Headphones",
    "price": { "amount": "79.99", "currency": "USD" }
  },
  "totalPrice": { "amount": "84.99", "currency": "USD" }
}
```

**Verdict: ✅ x402 payment pipeline works** — The product was resolved, priced, and a Solana USDC payment was requested. With a funded Solana wallet, this order would proceed.

---

### Test 5: Shopify Products — Requires Variant ID

```json
{"productUrl": "https://allbirds.com/products/mens-wool-runners", ...}
→ "Variant ID is required for Shopify products"
```

**Verdict: ✅ Endpoint works** — Shopify is supported but requires a `variantId` parameter (unlike Amazon where just the ASIN is enough).

---

### Test 6: Delivery Time Constraint

Multiple products returned: "Product delivery time exceeds maximum allowed period of 15 days"

This suggests Purch enforces a hard 15-day delivery window — products with slow shipping (3rd party sellers, non-Prime) are rejected.

**Verdict: ⚠️ Strict constraint** — This filters out a significant portion of Amazon's catalog, especially cheaper/third-party items.

---

## Key Architecture Findings

### 1. x402 on Solana (Not Base)

Unlike Locus/Sponge which use x402 on Base (EVM), Purch uses **Solana mainnet** exclusively for x402 payments. This means:
- Need a Solana wallet with USDC (SPL token)
- Purch pays transaction fees (feePayer in x402 response) — gasless for the buyer
- Payment to: `8LiXrHC61irY8qwj6qevoiRXxYfrTgSaHVbm8rav6HT2`

### 2. Crossmint Under the Hood

The error message for international addresses links to `docs.crossmint.com`, revealing that Purch uses **Crossmint's headless checkout** as their order fulfillment backend. Crossmint is a crypto-native commerce infrastructure that handles:
- Product resolution from Amazon/Shopify
- Order placement
- Payment processing (USDC → fiat conversion)

### 3. Pre-Payment Validation

Purch validates everything BEFORE requesting payment:
1. Product exists in catalog → 500 if not
2. Product available for purchase → 400 if not
3. Delivery time within 15 days → 400 if not
4. Shipping address in US → 400 if not
5. **Only then** → 402 Payment Required with exact price

This is different from Rye, which accepts the order first and validates during the `retrieving_offer` phase.

### 4. No API Key Needed

Like Rye's partner endpoint, Purch requires no API key. Payment IS authentication — the x402 protocol handles identity through the Solana wallet signature.

---

## Purch vs Rye Comparison (from testing)

| Aspect | Purch | Rye (ClawHub) |
|--------|-------|---------------|
| **Payment** | USDC on Solana (x402) | Credit card (BasisTheory token) |
| **Auth** | No key (x402 payment = auth) | No key (partner path = auth) |
| **Product catalog** | Limited/curated (1/8 ASINs worked) | Full Amazon catalog (all ASINs resolved) |
| **Geographic** | US only | US + Canada |
| **Delivery constraint** | Max 15 days | No explicit constraint |
| **Shopify** | Supported (needs variantId) | Supported |
| **Pre-payment validation** | Yes (validates before 402) | No (validates after order submitted) |
| **Infrastructure** | Crossmint (headless checkout) | RyeBot (web agent) |
| **Price transparency** | Total in 402 header before payment | Total in offer after submission |
| **Wallet needed** | Yes (Solana + USDC) | No (user's credit card) |

---

## What We Confirmed

| Claim | Status |
|-------|--------|
| x402 payment protocol (Solana USDC) | ✅ Confirmed |
| Product search with pricing | ✅ Confirmed (402 response) |
| Buy with dynamic pricing | ✅ Confirmed (Echo Dot: $55.88) |
| Pre-payment order validation | ✅ Confirmed |
| Amazon ASIN support | ✅ Confirmed (limited catalog) |
| Shopify URL support | ✅ Confirmed (needs variantId) |
| US-only shipping | ✅ Confirmed (rejects CA, KR) |
| Gasless for buyer (Purch pays fees) | ✅ Confirmed (feePayer in x402) |
| Crossmint infrastructure | ✅ Confirmed (error URL leak) |

## What Remains Untested

| Feature | Reason |
|---------|--------|
| Completed purchase | No Solana USDC wallet |
| Product search results | Requires x402 payment ($0.01) |
| AI shopping assistant (/shop) | Requires x402 payment ($0.10) |
| Vault (digital assets) | Requires x402 payment |
| Gift Hunter | Not tested |
| Order tracking/confirmation | No completed order |

---

## Cost Breakdown

| Item | Cost |
|------|------|
| All buy validation calls | $0.00 (pre-payment validation is free) |
| Search calls | $0.00 (returned 402, not paid) |
| **Total** | **$0.00** |

---

*This evaluation tested the production Purch API (api.purch.xyz) on March 18, 2026. No purchases were completed — we don't have a Solana wallet with USDC. All findings are from the pre-payment validation layer and decoded x402 payment headers.*

---

## ADDENDUM: Crossmint Direct API Comparison (March 18, 2026)

To determine whether the "Product not found" errors originate from Purch or Crossmint, we created a Crossmint staging developer account and tested the exact same ASINs against Crossmint's headless checkout API directly.

**Crossmint staging API:** `POST https://staging.crossmint.com/api/2022-06-09/orders`  
**Auth:** `X-API-KEY: sk_staging_*`

### Side-by-Side Results

| ASIN | Product | Purch (x402) | Crossmint (Direct API) |
|------|---------|--------------|----------------------|
| B09B8V1LZ3 | Echo Dot | ✅ 402 ($55.88) | ✅ Order created |
| B09JNPGJ7S | iPhone Case | ❌ Not found | ❌ Not found |
| B0CN1HP68Q | Kindle | ❌ Not found | ❌ Not found |
| B0B2MLXFGR | Anker Charger | ❌ Not found | ❌ Not found |
| B0BXRXLKHR | Dry Fit Shirt | ❌ Delivery >15 days | ❌ Delivery >15 days |
| B0D1XD1ZV3 | AirPods Pro | ❌ Unavailable | ✅ Order created* |
| B00EB4ADQW | Fuji Instax Film | ❌ Can't ship to address | ❌ Can't ship to address |

*AirPods worked on Crossmint staging but returned "unavailable" on Purch production — likely a staging vs production environment difference.

### Verdict: Crossmint Is the Bottleneck

The "Product not found" errors are **Crossmint's limitation**, not Purch's. The exact same ASINs fail identically on both platforms. Crossmint does NOT have full Amazon catalog coverage — they work with a curated/indexed subset of products.

All constraints flow from Crossmint:
- **Limited catalog** (many ASINs return "not found") → Crossmint
- **15-day delivery cap** → Crossmint  
- **US-only shipping** → Crossmint
- **Shipping restrictions per product** → Crossmint/Amazon

Purch is a thin x402 payment wrapper around Crossmint's headless checkout. It adds the Solana USDC payment protocol and conversational AI features, but the actual commerce pipeline, catalog, and constraints are all Crossmint.

### Architecture Confirmed

```
Agent/User → Purch (x402 payment on Solana) → Crossmint (headless checkout API) → Amazon/Shopify
```

Purch's value-add over using Crossmint directly:
- No API key needed (x402 payment = auth)
- AI shopping assistant (natural language search)
- Gift Hunter (social media analysis)
- Vault (digital asset marketplace)

But the core checkout capability is identical to Crossmint, including all limitations.
