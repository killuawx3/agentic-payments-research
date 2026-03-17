# Laso Finance Evaluation: Fiat Off-Ramp Test

> Attempted test of Laso Finance — prepaid Visa cards and Venmo/PayPal via USDC  
> Date: 2026-03-17 | Environment: Beta | Starting Balance: $9.98 USDC

---

## What We Tested

Laso Finance is Locus's fiat off-ramp — it bridges USDC to traditional finance by offering prepaid Visa cards and Venmo/PayPal payments. The system uses a hybrid architecture: paid x402 proxy endpoints (through Locus) and free direct endpoints (on laso.finance).

### Test Actions Attempted

1. **Laso Auth** — `POST /api/x402/laso-auth` ($0.001 — generates session tokens)
2. **Free endpoints** — `GET laso.finance/search-merchants`, `GET laso.finance/get-account-balance`
3. **Ad-hoc x402 call** — `POST /api/x402/call` (alternative auth path)
4. **Balance verification** — confirm no charges on failed calls

---

## README Claims vs. Actual Results

### Claim 1: Authentication — "POST /api/x402/laso-auth, $0.001, no parameters required"

| Aspect | README / LASO.md Says | What Actually Happened |
|--------|----------------------|----------------------|
| Endpoint | `POST /api/x402/laso-auth` | Endpoint exists |
| Parameters | "No parameters required" / `{}` | Confirmed — docs say send `{}` |
| Cost | $0.001 USDC | Never charged (call failed) |
| Response | Returns `id_token` and `refresh_token` | ❌ **HTTP 500 — server crash** |

**Error response:**
```json
{"success": false, "error": "Cannot read properties of undefined (reading 'name')"}
```

We tried multiple request formats:
- Empty body `{}` → 500
- With `name` field → 500
- With `name` + `email` → 500
- With `email` only → 500
- Without body → 400 ("expected object, received undefined")
- Via `/x402/call` alternative → 500

**All attempts returned 500.** The server-side code has a null reference bug — it tries to read `.name` from an undefined object in its internal logic.

**Verdict: ❌ BROKEN** — Laso auth is non-functional in beta. This blocks the entire Laso Finance feature since all subsequent calls require the `id_token` from this endpoint.

---

### Claim 2: Free Endpoints — "Called directly at laso.finance using id_token"

| Endpoint | README Says | What Actually Happened |
|----------|-------------|----------------------|
| `/search-merchants?q=amazon` | Free, returns merchant compatibility | HTTP 401 — "Missing or invalid Authorization header" |
| `/get-account-balance` | Free, returns balance | HTTP 401 — "Missing Authorization header" |

**Verdict: ⬜ BLOCKED** — Free endpoints correctly require authentication. Since the auth endpoint is broken, we can't obtain tokens to test these. The 401 responses are expected behavior (not a bug), but we can't get past the auth gate.

---

### Claim 3: Charge-on-Success Model

| Aspect | README Says | What Actually Happened |
|--------|-------------|----------------------|
| Failed calls | "No charges for errors" | ✅ **Confirmed** — balance stayed at $9.98 despite multiple failed auth attempts |
| Transaction history | Should show no new entries | ✅ Confirmed — still only 2 transactions (both Exa searches) |

**Verdict: ✅ CONFIRMED** — The charge-on-success safety model works. Zero charges for the 500 errors, which is the correct behavior.

---

### Claim 4: Dual Base URL Architecture

| Aspect | README Says | What Actually Happened |
|--------|-------------|----------------------|
| Paid endpoints | Via `api.paywithlocus.com/api/x402/laso-*` | Confirmed — endpoints exist at these paths |
| Free endpoints | Via `laso.finance/*` | Confirmed — laso.finance is up (HTTP 200), endpoints return proper 401 |
| Token flow | Paid auth → id_token → use on free endpoints | Architecture is correct, but auth step is broken |

**Verdict: ✅ ARCHITECTURE CONFIRMED** — The dual-URL design is real and functional. The routing works, auth gating works, the issue is purely a bug in the auth handler.

---

### Claim 5: Pricing

| Action | README Says | Testable? |
|--------|-------------|-----------|
| Authenticate | $0.001 USDC | ❌ Auth broken |
| Order Card | $5–$1000 USDC | ❌ Requires auth + US IP |
| Send Payment | $5–$1000 USDC | ❌ Requires auth |
| Token refresh | Free | ❌ Requires initial auth |
| Card data, balance, merchant search | Free | ❌ Requires auth |

**Verdict: ⬜ UNTESTABLE** — Cannot verify any pricing claims since the auth gateway is broken.

---

## What We Can Confirm (Without Auth)

Despite not being able to complete the auth flow, we verified:

1. **laso.finance is live** — homepage returns HTTP 200
2. **Endpoint routing works** — x402 proxy endpoints exist and route correctly
3. **Free endpoints are properly secured** — return 401 without token (not 500)
4. **No charges on failure** — charge-on-success model holds
5. **Beta environment** — production API correctly rejects beta keys

---

## Summary Scorecard

| README Claim | Status | Notes |
|-------------|--------|-------|
| Auth endpoint ($0.001) | ❌ Broken | Server 500 — null reference in handler |
| Prepaid card ordering | ⬜ Blocked | Requires auth + US IP |
| Venmo/PayPal payments | ⬜ Blocked | Requires auth |
| Free endpoints (merchant search, balance) | ⬜ Blocked | Properly gated, but auth is broken |
| Dual base URL architecture | ✅ Confirmed | Routing works correctly |
| Charge-on-success (no charge on failure) | ✅ Confirmed | Zero charges despite multiple 500s |
| Token lifecycle (id_token + refresh_token) | ⬜ Untestable | Can't get tokens |

**Overall: 2/7 confirmed, 1 broken (blocking), 4 untestable due to auth failure**

---

## Issues Found

### 1. CRITICAL — Laso Auth Endpoint Returns 500
`POST /api/x402/laso-auth` crashes with:
```json
{"success": false, "error": "Cannot read properties of undefined (reading 'name')"}
```
This is a null reference error in the server-side code. It occurs regardless of request body content. This blocks ALL Laso Finance functionality since every other endpoint requires the `id_token` from auth.

**Impact:** Laso Finance is completely unusable in the beta environment.

### 2. DOCUMENTATION NOTE — x402 Endpoint Docs Say "No Parameters Required"
The x402 endpoints listing for laso-auth states `_No parameters required._` and shows `-d '{}'`. The server accepts the request but crashes internally, suggesting the endpoint isn't properly wired up in beta.

---

## Cost of This Evaluation

| Item | Cost |
|------|------|
| Auth attempts (all failed) | $0.00 |
| Free endpoint attempts (all 401) | $0.00 |
| **Total** | **$0.00 USDC** |

Balance unchanged: $9.98 → $9.98

---

## Pricing Table Update

| Provider / Feature | Endpoint | Est. Cost | Status |
|-------------------|----------|-----------|--------|
| Laso Auth | `/x402/laso-auth` | $0.001 | ❌ Broken (500) |
| Laso Prepaid Card | `/x402/laso-get-card` | $5–$1000 | ⬜ Untestable |
| Laso Send Payment | `/x402/laso-send-payment` | $5–$1000 | ⬜ Untestable |
| Laso Free endpoints | `laso.finance/*` | Free | ⬜ Blocked by auth |
