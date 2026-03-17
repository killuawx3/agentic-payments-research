# Laso Finance Prepaid Card Evaluation: Full Pipeline via Sponge

> Live test of ordering a prepaid Visa card using USDC through Sponge → Laso Finance  
> Date: 2026-03-17 | Environment: Production | Starting Balance: $5.99 USDC (Base)

---

## What We Tested

The complete Laso Finance prepaid card pipeline accessed through Sponge's x402 proxy — from authentication to holding a real Visa card number.

### Test Actions

1. **Laso Auth** — `x402/fetch` → `laso.finance/auth` ($0.001)
2. **Account Balance** — `x402/fetch` → `laso.finance/get-account-balance` (free)
3. **Merchant Search** — `x402/fetch` → `laso.finance/search-merchants?q=amazon` (free)
4. **Order $5 Card** — `x402/fetch` → `laso.finance/get-card?amount=5` ($5.00)
5. **Poll Card Status** — `x402/fetch` → `laso.finance/get-card-data?card_id=...` (free)
6. **Retrieve Card Details** — card number, CVV, expiration, balance

---

## Results

### Step 1: Laso Auth ($0.001)

```json
{
  "auth": {
    "id_token": "eyJ...",
    "refresh_token": "AMf-...",
    "expires_in": "3600"
  },
  "user_id": "0xf577fd4c7d7149bc62deea54958d0d691911f75c"
}
```

Payment: $0.001 USDC on Base. Got `id_token` (1 hour TTL) and `refresh_token`. This token is required for all subsequent free Laso endpoints — must be passed as `Authorization: Bearer <id_token>` in the x402/fetch headers param.

**Verdict: ✅ Works** — Note: this same endpoint is completely broken on Locus's beta (500 server error). Through Sponge's x402 proxy, it works perfectly.

---

### Step 2: Account Balance (Free)

```json
{
  "user_id": "0xf577fd4c7d7149bc62deea54958d0d691911f75c",
  "balance": 0.001,
  "total_deposits": 0.001,
  "created_timestamp": 1773773392420
}
```

Laso account auto-created during auth. Balance shows $0.001 (the auth fee gets deposited into the Laso account).

**Verdict: ✅ Works**

---

### Step 3: Merchant Search (Free)

`search-merchants?q=amazon` returned real merchant data:

| Merchant | Status |
|----------|--------|
| Amazon Payments (pay.amazon.com) | ✅ accepted |
| Amazon Mexico | ❌ not_accepted |
| Amazon EU / Amazon Japan | ⚠️ unknown |

Useful for checking if a merchant accepts the prepaid card before ordering.

**Verdict: ✅ Works**

---

### Step 4: Order $5 Card

```json
{
  "card": {
    "card_id": "O-01KKYJB0V4VST60T9PHR767K32",
    "usd_amount": 5,
    "status": "pending"
  },
  "payment_made": true,
  "payment_details": {
    "chain": "base",
    "amount": "5",
    "token": "USDC"
  }
}
```

$5 USDC paid on-chain. Card status starts as `pending`. Minimum is $5 (tried $2, got server error).

**Verdict: ✅ Works**

---

### Step 5: Poll Card Status → Ready in 3 Seconds

First poll returned `ready` immediately:

```json
{
  "card_id": "O-01KKYJB0V4VST60T9PHR767K32",
  "usd_amount": 5,
  "status": "ready",
  "card_details": {
    "card_number": "4343405849095897",
    "exp_month": "09",
    "exp_year": "26",
    "cvv": "915",
    "available_balance": 5
  },
  "transactions": [
    {
      "amount": "$5.00",
      "date": "3/17/26, 2:53 PM",
      "description": "wegiftusd - 82108673154",
      "is_credit": true
    }
  ]
}
```

Docs say 7-10 seconds, ours was ready in ~3 seconds.

**Verdict: ✅ Works — faster than documented**

---

## Full Pipeline Summary

| Step | Action | Cost | Time | Status |
|------|--------|------|------|--------|
| 1 | Laso Auth | $0.001 | 4.88s | ✅ |
| 2 | Account Balance | Free | ~1s | ✅ |
| 3 | Merchant Search | Free | ~1s | ✅ |
| 4 | Order $5 Card | $5.00 | 5.16s | ✅ |
| 5 | Poll Card Status | Free | 3s | ✅ Ready |
| 6 | Card Details | Free | (in poll) | ✅ Full details |

**Total cost: $5.001 USDC**
**Total time: ~15 seconds from auth to card in hand**

---

## Key Comparison: Sponge vs Locus for Laso

| Aspect | Through Sponge | Through Locus |
|--------|---------------|---------------|
| Auth endpoint | ✅ Works | ❌ 500 server crash |
| Card ordering | ✅ Works | ⬜ Blocked (auth broken) |
| Card details | ✅ Full card number, CVV, exp | ⬜ Blocked |
| Free endpoints | ✅ Work (with id_token in headers) | ⬜ Blocked |
| Payment protocol | x402 (on-chain handshake) | x402 proxy (internal) |

Both platforms use the same Laso Finance backend. The difference is how they proxy the request — Sponge's x402 fetch passes through correctly, while Locus's internal x402 proxy has a server-side bug in beta.

---

## Cost Breakdown

| Item | Cost |
|------|------|
| Laso Auth | $0.001 |
| Prepaid Card ($5 face value) | $5.000 |
| Balance check, merchant search, card polling | Free |
| **Total** | **$5.001 USDC** |

Wallet balance: $5.99 → $0.99 USDC
