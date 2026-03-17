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

### Claim 4: Pricing — Estimated cost discoverable via API

You can fetch estimated costs per endpoint via `GET /api/wrapped/md?provider=<slug>`. Each endpoint lists an **"Estimated cost"** — a single combined price per call (Locus fee + upstream cost bundled together).

**Exa pricing from discovery endpoint:**

| Endpoint | Estimated Cost |
|----------|---------------|
| Search | $0.010 |
| Contents | $0.004/page |
| Find Similar | $0.010 |
| Answer | $0.008 |
| Research Status | Free |
| Research List | Free |

**What we actually paid:**

| Call | Estimated | costDollars (response) | On-chain charge |
|------|-----------|----------------------|----------------|
| Neural search (5 results) | $0.010 | not captured | $0.01 |
| Keyword search (1 result) | $0.010 | $0.007 | $0.01 |

Estimates are close to actual. On-chain charges appear to round up to $0.01 minimum.

**Sample pricing across providers (from discovery endpoints + testing):**

| Provider / Feature | Endpoint | Est. Cost | Billing Source |
|-------------------|----------|-----------|----------------|
| Exa | Search | $0.010 | USDC wallet |
| Exa | Contents | $0.004/page | USDC wallet |
| OpenAI | Chat | ~$0.001–$0.20 (model-dependent) | USDC wallet |
| Anthropic | Messages | ~$0.001–$0.20 (model-dependent) | USDC wallet |
| Firecrawl | Scrape | $0.003+ | USDC wallet |
| Resend | Send Email | $0.004 | USDC wallet |
| X (Twitter) | All endpoints | $0.016 flat | USDC wallet |
| Abstract (email, IP, etc.) | All endpoints | $0.006 | USDC wallet |
| Checkout | Per transaction | Transaction amount | USDC wallet |
| **Build (PaaS)** | **Per service** | **$0.25/month** | **Separate Build credits** |
| **Build Addons** | **Postgres/Redis** | **$0.25/month each** | **Separate Build credits** |

**Verdict: CONFIRMED** — Cost-per-call is discoverable and roughly accurate. Token-based providers (OpenAI, Anthropic, Gemini) show a range instead of a fixed price, which makes sense since cost depends on usage.

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
| Pricing discoverable & accurate | ✅ Confirmed | Estimated costs match actual charges, discoverable via API |
| Response envelope format | ✅ Confirmed | Exact match + bonus costDollars field |
| On-chain settlement on Base | ✅ Confirmed | Real USDC transfers with tx hashes |
| Gasless transactions | ✅ Confirmed | No gas costs incurred |
| HTTP 202 for approval threshold | ⬜ Not tested | No spending controls configured |
| Error codes (402, 403, 502) | ⬜ Not tested | All calls succeeded |

**Overall: 7/7 tested claims confirmed, 4 claims untested (require specific conditions)**

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
3. **Transaction category naming** — Transactions show `category: "claw_send"` rather than something more descriptive like `"wrapped_api"`.
