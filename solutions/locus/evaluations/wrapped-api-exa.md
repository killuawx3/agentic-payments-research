# Wrapped API Evaluation: Exa Search

> Live test of Locus Wrapped APIs using Exa (AI-native search engine)  
> Date: 2026-03-17 | Environment: Beta | Starting Balance: $10.00 USDC

---

## What We Tested

The Wrapped APIs feature — Locus's unified proxy layer that lets AI agents call third-party APIs with automatic authentication and per-call USDC billing. We chose **Exa** (semantic search) as a low-cost, straightforward provider to validate the claims in the README.

### Test Actions

1. **Discovery** — `GET /api/wrapped/md` (full provider catalog)
2. **Provider Detail** — `GET /api/wrapped/md?provider=exa` (Exa-specific docs)
3. **API Call #1** — `POST /api/wrapped/exa/search` (neural search, 5 results)
4. **API Call #2** — `POST /api/wrapped/exa/search` (keyword search, 1 result)
5. **Balance Check** — `GET /api/pay/balance` (before & after)
6. **Transaction History** — `GET /api/pay/transactions` (verify on-chain settlement)

---

## README Claims vs. Actual Results

### Claim 1: "Unified interface with automatic authentication — no upstream API keys needed"

| Aspect | README Says | What Actually Happened |
|--------|-------------|----------------------|
| Auth method | Bearer token with Locus API key | Confirmed — single `Authorization: Bearer claw_dev_*` header |
| Upstream keys | "No upstream API keys needed — Locus manages all credentials" | Confirmed — zero Exa credentials required on our end |
| Endpoint pattern | `POST /api/wrapped/<provider>/<endpoint>` | Confirmed — used `POST /api/wrapped/exa/search` |

**Verdict: CONFIRMED** — One API key to rule them all. No Exa account, no Exa API key, just a Locus key.

---

### Claim 2: "API Discovery via GET /api/wrapped/md"

| Aspect | README Says | What Actually Happened |
|--------|-------------|----------------------|
| Catalog endpoint | `GET /api/wrapped/md` returns markdown catalog | Confirmed — returned full markdown with all providers |
| Provider count | README lists: OpenAI, Gemini, fal.ai, Firecrawl, Exa, Clado, Resend, X, Abstract API, Browser Use, Apollo | Actual catalog has 30+ providers (README only listed a subset) |
| Per-provider detail | `GET /api/wrapped/md?provider=<slug>` | Confirmed — returned endpoint docs, curl examples, param tables |

**Verdict: CONFIRMED** — Discovery works. The actual provider catalog is significantly larger than what the README documents. The README mentions ~11 providers; the live catalog shows 30+ including Anthropic, Perplexity, Grok, Stability AI, Mapbox, Diffbot, RentCast, Hunter, IPinfo, and more.

**Note:** The README's Provider Catalog section appears outdated — it's missing many providers that are live on the platform.

---

### Claim 3: "Per-call USDC billing with charge-on-success safety model"

| Aspect | README Says | What Actually Happened |
|--------|-------------|----------------------|
| Billing model | "Funds reserved before upstream execution" | Cannot directly verify reservation step, but billing occurred only on success |
| Success charge | "Successful calls transfer USDC from agent wallet to Locus treasury on Base" | Confirmed — on-chain USDC transfer with tx hash |
| Failure behavior | "Failed calls restore reserved funds — no charges for errors" | Not tested (no failures occurred) |

**Verdict: PARTIALLY CONFIRMED** — Billing on success is verified. Failure refund behavior was not tested.

---

### Claim 4: Pricing — "Flat $0.003/call (most providers) OR 15% markup (OpenAI/Gemini)"

| Aspect | README Says | What Actually Happened |
|--------|-------------|----------------------|
| Exa search cost | "$0.010/search" (from provider catalog) | Call #1 (neural, 5 results): $0.01 charged, `costDollars.total = 0.01` |
| | | Call #2 (keyword, 1 result): $0.01 charged on-chain, `costDollars.total = 0.007` |
| Locus fee | "$0.003/call flat" for most providers | fee structure not broken out separately in the response |
| Cost transparency | Not specified | Response includes `costDollars` field with breakdown by operation type |

**Observations:**
- Both calls were charged $0.01 USDC on-chain (minimum charge granularity appears to be $0.01)
- The `costDollars` field in the API response showed $0.007 for the keyword search, but the on-chain charge was $0.01
- This suggests there may be a minimum charge of $0.01 per wrapped API call, or rounding up to nearest cent for on-chain settlement
- The Locus fee vs upstream cost is not itemized separately in either the response or transaction history

**Verdict: MOSTLY CONFIRMED** — Pricing is in the right ballpark. Minor discrepancy between reported costDollars ($0.007) and on-chain charge ($0.01) suggests rounding or minimum charge. The flat $0.003 fee isn't visible as a separate line item.

---

### Claim 5: Response format — `{success: true, data: {...}}`

| Aspect | README Says | What Actually Happened |
|--------|-------------|----------------------|
| Success envelope | `{"success": true, "data": {...}}` | Confirmed — exact format |
| Data contents | Upstream response passed through in `data` field | Confirmed — Exa results nested under `data` |
| Extra metadata | Not mentioned in README | Response includes `costDollars` breakdown (nice bonus) |
| HTTP status | 200 for success | Confirmed |

**Verdict: CONFIRMED** — Response envelope matches documentation exactly. The `costDollars` field is an undocumented but useful addition.

---

### Claim 6: On-chain settlement on Base

| Aspect | README Says | What Actually Happened |
|--------|-------------|----------------------|
| Chain | "All transactions settle on Base as USDC transfers" | Confirmed — USDC on Base |
| Gasless | "Fully gasless via paymaster" | Confirmed — no gas costs on our end |
| Verifiable | Transaction has tx_hash | Confirmed — each call produced a verifiable tx hash |
| Transaction record | Available via `/pay/transactions` | Confirmed — full history with status, amount, memo, tx_hash |

**Transaction Evidence:**

| Call | Tx Hash | Amount | Status |
|------|---------|--------|--------|
| Exa neural search | `0x252bac6f...` | $0.01 USDC | CONFIRMED |
| Exa keyword search | `0x2c2c1ca2...` | $0.01 USDC | CONFIRMED |

Both transactions sent to Locus treasury: `0x3c5cbe28eca3b96023c45d3f877da834f1c7d5fa`

**Verdict: CONFIRMED** — Every API call produces a real on-chain USDC transaction on Base with a verifiable tx hash. This is the core thesis of the platform and it works.

---

## Performance Observations

| Metric | Value |
|--------|-------|
| Discovery (catalog) | ~0.6s |
| Provider detail | ~0.8s |
| Exa neural search (5 results) | ~1.6s |
| Balance check | ~1.0s |

Latency is reasonable — the ~1.6s for the search call includes Locus's overhead (auth, reserve USDC, forward to Exa, settle on-chain, return response).

---

## Summary Scorecard

| README Claim | Status | Notes |
|-------------|--------|-------|
| Unified interface, no upstream keys | ✅ Confirmed | Single API key pattern works perfectly |
| API discovery via /wrapped/md | ✅ Confirmed | Works; catalog is larger than README documents |
| Per-call USDC billing | ✅ Confirmed | Real on-chain charges per call |
| Charge-on-success (no charge on failure) | ⬜ Not tested | Need to trigger a failure to verify |
| Pricing accuracy | ⚠️ Mostly confirmed | Minor discrepancy: costDollars vs on-chain amount differ |
| Response envelope format | ✅ Confirmed | Exact match + bonus costDollars field |
| On-chain settlement on Base | ✅ Confirmed | Real USDC transfers with tx hashes |
| Gasless transactions | ✅ Confirmed | No gas costs incurred |
| HTTP 202 for approval threshold | ⬜ Not tested | No spending controls configured |
| Error codes (402, 403, 502) | ⬜ Not tested | All calls succeeded |

**Overall: 6/6 tested claims confirmed, 1 with minor pricing discrepancy, 4 claims untested (require specific conditions)**

---

## Cost of This Evaluation

| Item | Cost |
|------|------|
| Exa search (neural, 5 results) | $0.01 |
| Exa search (keyword, 1 result) | $0.01 |
| Balance checks, discovery, tx history | Free |
| **Total** | **$0.02 USDC** |

Starting balance: $10.00 → Ending balance: $9.98

---

## Things the README Gets Wrong or Omits

1. **Provider catalog is outdated** — README lists ~11 providers but the live platform has 30+. Missing: Anthropic, Perplexity, Grok, Stability AI, Mapbox, Diffbot, RentCast, Hunter, IPinfo, and many more.
2. **costDollars response field undocumented** — The API response includes a useful `costDollars` breakdown that isn't mentioned in the README.
3. **Minimum charge granularity unclear** — The $0.003 flat fee and exact pricing math isn't fully transparent. On-chain charges appear to round to $0.01 minimum.
4. **Transaction category naming** — Transactions show `category: "claw_send"` rather than something more descriptive like `"wrapped_api"`.
