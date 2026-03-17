# x402 Services Evaluation: Exa Search via Sponge

> Live test of Sponge's x402 paid API service using Exa (AI-native search)  
> Date: 2026-03-17 | Environment: Production | Starting Balance: 3.0003 USDC (Base)

---

## What We Tested

Sponge's x402 paid API proxy — the equivalent of Locus's Wrapped APIs. We tested the full 3-step discovery-and-call flow using Exa search as the target service.

### Test Actions

1. **Discover services** — `GET /api/discover` (full catalog)
2. **Service detail** — `GET /api/discover/:serviceId` (Exa endpoints, pricing, baseUrl)
3. **x402 fetch** — `POST /api/x402/fetch` (Exa neural search)
4. **Balance check** — `GET /api/balances` (before & after)
5. **Transaction history** — `GET /api/transactions/history`

---

## SKILL.md Claims vs. Actual Results

### Claim 1: 3-Step Discovery Flow

| Step | SKILL.md Says | What Actually Happened |
|------|--------------|----------------------|
| 1. Discover | `GET /api/discover?query=...` returns services | ✅ Returned 19 services with names, slugs, categories, endpoint counts |
| 2. Details | `GET /api/discover/:serviceId` returns endpoints + pricing + baseUrl | ✅ Full endpoint docs with parameters, instructions, price per call |
| 3. Fetch | `POST /api/x402/fetch` with url, method, body, preferred_chain | ✅ Successfully called Exa search |

**Verdict: ✅ CONFIRMED** — The 3-step flow works exactly as documented.

---

### Claim 2: Service Catalog

19 services available (vs Locus's 30+ wrapped APIs):

| Service | Category | Endpoints |
|---------|----------|-----------|
| AgentMail | email | 59 |
| Alchemy | crypto_data | 15 |
| Allium | crypto_data | 12 |
| Apollo | prospect | 4 |
| Auor | data | 5 |
| Browserbase | browser | 4 |
| CoinGecko | crypto_data | 5 |
| CoinMarketCap | crypto_data | 4 |
| Dome | predict | 16 |
| Exa | search | 7 |
| Gemini | image | 1 |
| Laso Finance | payments | 11 |
| Nansen | crypto_data | 26 |
| Nyne | prospect | 18 |
| OpenRouter | llm | 5 |
| Quick Intel | crypto_data | 1 |
| Quicknode | crypto_data | 2 |
| Reducto | parse | 2 |
| Textbelt | sms | 2 |

**Notable differences from Locus:** Sponge has more crypto-data services (Alchemy, Nansen, CoinGecko, CoinMarketCap, Quicknode) and offers OpenRouter for LLM access. Locus has more general-purpose APIs (Firecrawl, Resend, X/Twitter, Anthropic, Stability AI, Mapbox, etc.).

**Verdict: ✅ CONFIRMED** — Catalog is real, discoverable, and slanted toward crypto/trading use cases.

---

### Claim 3: x402 Payment Protocol

| Aspect | SKILL.md Says | What Actually Happened |
|--------|--------------|----------------------|
| Payment method | "Sponge handles the 402 payment handshake automatically" | ✅ Confirmed — single POST, payment handled transparently |
| Chain selection | `preferred_chain` parameter | Used `"base"`, payment settled on Base |
| Payment proof | Not detailed | Response includes `payment_made: true`, `payment_details` with chain/amount/to, and base64-encoded payment receipt in headers |

**Payment details from our call:**
```json
{
  "payment_made": true,
  "payment_details": {
    "chain": "base",
    "amount": "0.01",
    "token": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
    "to": "0x6302D9e6DBB22fEC3c350551568Bb39B4b35Ad57"
  }
}
```

**Decoded payment-response header:**
```json
{
  "success": true,
  "transaction": "0xf18eb977...48743a02",
  "network": "base",
  "payer": "0xf577fd4c...911f75c"
}
```

**Verdict: ✅ CONFIRMED** — x402 protocol works transparently. On-chain USDC payment with verifiable tx hash, plus rich payment metadata in the response.

---

### Claim 4: Pricing

Exa endpoints from discovery:

| Endpoint | Listed Price | Currency |
|----------|-------------|----------|
| /search | 10000 | USDC |
| /contents | 10000 | USDC |
| /findSimilar | 10000 | USDC |
| /answer | 10000 | USDC |
| /research | 10000 | USDC |
| /researchStatus | 10000 | USDC |
| /researchList | 10000 | USDC |

**Price interpretation:** Listed as `"10000"` in `"USDC"` — this is in USDC micro-units. 10000 = $0.01 USDC (USDC has 6 decimals, so 10000 / 1000000 = $0.01).

**Actual charge:** $0.01 USDC (3.0003 → 2.9903), matching the listed price.

**Verdict: ✅ CONFIRMED** — Prices are accurate. Listed in micro-units, which is a bit unintuitive but consistent.

---

### Claim 5: On-chain Settlement

| Aspect | What Happened |
|--------|--------------|
| Chain | Base |
| Token | USDC (0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913) |
| Amount | 0.01 USDC |
| Tx hash | 0xf18eb9773117107b8114a25d719a6c30030c57325b9898949ba249f148743a02 |
| Recipient | 0x6302D9e6DBB22fEC3c350551568Bb39B4b35Ad57 |
| Gasless? | **No** — EOA wallet, agent pays gas (but gas was pre-funded) |

**Verdict: ✅ CONFIRMED** — Real on-chain USDC settlement with verifiable tx hash. Unlike Locus, Sponge doesn't abstract gas — the tiny ETH balance (0.000000001) was presumably used or the x402 proxy covers gas somehow.

---

## Performance

| Operation | Latency |
|-----------|---------|
| Discover (full catalog) | ~4.2s |
| Service detail | ~0.6s |
| x402 Exa search | ~8.6s |
| Balance check | ~3.2s |

The x402 call was significantly slower than Locus's wrapped API (~8.6s vs ~1.6s). This likely includes the x402 payment negotiation handshake (agent signs tx → submits → waits for confirmation → receives response).

---

## Summary Scorecard

| Claim | Status | Notes |
|-------|--------|-------|
| 3-step discovery flow | ✅ Confirmed | Discover → Details → Fetch |
| 19 x402 services available | ✅ Confirmed | Crypto-heavy catalog |
| Automatic x402 payment | ✅ Confirmed | Transparent payment handshake |
| Pricing per endpoint | ✅ Confirmed | Listed in micro-units, $0.01 for Exa search |
| On-chain settlement | ✅ Confirmed | Real USDC tx on Base |
| Payment proof in response | ✅ Confirmed | payment_details + base64 receipt |

**Overall: 6/6 confirmed**

---

## Cost

| Item | Cost |
|------|------|
| Exa search (neural, 3 results) | $0.01 USDC |
| Discovery, details, balance, transactions | Free |
| **Total** | **$0.01 USDC** |

Balance: $3.0003 → $2.9903

---

## Comparison: Sponge x402 vs Locus Wrapped APIs

| Aspect | Sponge x402 | Locus Wrapped APIs |
|--------|------------|-------------------|
| Exa search cost | $0.01 | $0.01 |
| Service count | 19 | 30+ |
| Latency | ~8.6s | ~1.6s |
| Discovery | 3-step (discover → detail → fetch) | 2-step (GET /wrapped/md → POST /wrapped/provider/endpoint) |
| Payment protocol | x402 (HTTP 402 handshake) | Internal proxy (pre-reserve → execute → settle) |
| Response includes | payment_details + receipt + upstream data | costDollars + upstream data |
| Gasless | No (EOA, agent pays gas) | Yes (paymaster) |
| Chain options | Base or Solana per service | Base only |
