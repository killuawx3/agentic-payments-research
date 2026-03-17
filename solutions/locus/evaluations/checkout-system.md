# Checkout System Evaluation: Pure API Test

> Live test of Locus Checkout — session creation, agent payment, and confirmation  
> Date: 2026-03-17 | Environment: Beta | Starting Balance: $9.98 USDC

---

## What We Tested

The Checkout System's API-only flow — creating a merchant session and paying it programmatically as an agent. No frontend/SDK involved, purely backend validation.

### Test Actions

1. **Create Session** — `POST /api/checkout/sessions` (merchant side)
2. **Preflight Check** — `GET /api/checkout/agent/preflight/:sessionId`
3. **Pay Session** — `POST /api/checkout/agent/pay/:sessionId`
4. **Poll Payment** — `GET /api/checkout/agent/payments/:txId`
5. **Check Session Status** — `GET /api/checkout/sessions/:sessionId`
6. **Payment History** — `GET /api/checkout/agent/payments`

---

## README Claims vs. Actual Results

### Claim 1: Session Creation

| Aspect | README Says | What Actually Happened |
|--------|-------------|----------------------|
| Create endpoint | Not explicitly documented as agent-accessible | `POST /api/checkout/sessions` works with agent API key |
| Response | Returns `sessionId` + checkout URL | Confirmed — got session ID, checkout URL, status, expiry |
| Session status | Starts as `PENDING` | Confirmed |
| Expiry | Default 30 min | Confirmed — `expiresAt` was ~30 min from creation |

**Response received:**
```json
{
  "id": "d7433f0f-6e24-456a-bf15-3d0e33df6904",
  "checkoutUrl": "https://beta-checkout.paywithlocus.com/d7433f0f-...",
  "amount": "0.01",
  "currency": "USDC",
  "status": "PENDING",
  "expiresAt": "2026-03-17T16:30:54.792Z"
}
```

**Verdict: CONFIRMED** — Session creation works, and is accessible to agents (not just merchant servers).

---

### Claim 2: Preflight Check — "Returns canPay: true/false with blockers"

| Aspect | README Says | What Actually Happened |
|--------|-------------|----------------------|
| Endpoint | `GET /checkout/agent/preflight/:sessionId` | Confirmed |
| canPay field | Returns `canPay: true/false` | Confirmed — `canPay: true` |
| Blockers | Array of reasons if false | Not triggered (session was payable) |
| Agent info | Not documented | Response included `agent.walletAddress` and `agent.availableBalance` |
| Session info | Not documented in detail | Returned full session details including `sellerWalletAddress` |

**Latency:** 0.54s

**Interesting finding:** `availableBalance` showed `"999999"` which doesn't match our actual $9.98 USDC balance. This appears to be a default/maximum value rather than the real wallet balance, possibly because no allowance limit was set (README says "Blank = full balance access").

**Verdict: CONFIRMED** — Preflight works. Extra agent/session context is a useful bonus not mentioned in the README.

---

### Claim 3: Payment — "Pay a pending session"

| Aspect | README Says | What Actually Happened |
|--------|-------------|----------------------|
| Endpoint | `POST /checkout/agent/pay/:sessionId` | Confirmed |
| Request body | Optional `payerEmail` | Confirmed — accepted email |
| Response | Returns `transactionId` | Confirmed — plus `queueJobId`, `status`, `statusEndpoint` |
| Initial status | Implied `PENDING` or `QUEUED` | Got `"status": "queued"` |
| Status endpoint | Not mentioned in pay response | Response helpfully includes `statusEndpoint` path |

**Latency:** 0.55s

**Response received:**
```json
{
  "transactionId": "307b5aa4-...",
  "queueJobId": "1b989a33-...",
  "status": "queued",
  "sessionId": "d7433f0f-...",
  "amount": "0.01",
  "currency": "USDC",
  "statusEndpoint": "/api/checkout/agent/payments/307b5aa4-...",
  "message": "Payment initiated. Poll statusEndpoint or wait for webhook."
}
```

**Verdict: CONFIRMED** — Payment initiation works. Response is richer than documented (includes queue job ID, status endpoint, helpful message).

---

### Claim 4: Polling — "GET /checkout/agent/payments/:txId"

| Aspect | README Says | What Actually Happened |
|--------|-------------|----------------------|
| Endpoint | `GET /checkout/agent/payments/:transactionId` | Endpoint exists but... |
| Expected statuses | `PENDING -> QUEUED -> PROCESSING -> CONFIRMED` | ❌ Got HTTP 403 |
| Error | Not documented for this scenario | `"Transaction does not belong to this agent"` |

**This is a bug or scoping issue.** The agent that initiated the payment cannot poll its own transaction status. The 403 says "does not belong to this agent" even though we literally just created and paid this session with the same API key.

**Workaround found:** Checking the session directly via `GET /checkout/sessions/:sessionId` DID work and showed the payment was confirmed:

```json
{
  "status": "PAID",
  "paymentTxHash": "0x719d4ce8fc29b07586136a60d97f074a675f01b85d8ce565a155aca80a169b50",
  "payerAddress": "0x5724b4dacf0edbbc507875c87424029aaf4c9d6f",
  "paidAt": "2026-03-17T16:01:45.633Z"
}
```

Payment was confirmed on-chain ~51 seconds after initiation.

**Verdict: ❌ FAILED** — The documented polling flow (`GET /payments/:txId`) returned 403. This breaks the 3-step flow described in CHECKOUT.md. The session status endpoint works as a fallback but is not the documented pattern. Could be a beta bug, or a scoping issue where "merchant" and "payer" are the same account.

---

### Claim 5: Payment History

| Aspect | README Says | What Actually Happened |
|--------|-------------|----------------------|
| Endpoint | `GET /checkout/agent/payments` with pagination | Returns empty array |
| Expected | Should list our checkout payment | `"payments": []` |

**Verdict: ❌ FAILED** — Payment history returns empty despite a confirmed checkout payment. Same scoping issue as the polling endpoint — the transaction may not be associated with the paying agent's history.

---

### Claim 6: On-chain Settlement

| Aspect | README Says | What Actually Happened |
|--------|-------------|----------------------|
| Payment settles on Base | USDC transfer on-chain | Confirmed — tx hash generated |
| Balance deduction | Should deduct from payer | Balance unchanged ($9.98) |

**Note on balance:** This is expected — we were both the merchant (seller) and the agent (payer). The `sellerWalletAddress` in preflight was our own wallet (`0x5724b...`). So USDC moved from our wallet to our wallet, netting zero. The tx hash `0x719d4ce8...` is real and verifiable on Base.

**Verdict: CONFIRMED** — On-chain settlement occurs. The zero net balance change is expected for self-payment.

---

### Claim 7: Session Status Lifecycle

| README Says | What Actually Happened |
|-------------|----------------------|
| `PENDING -> PAID` (on-chain confirmation) | Confirmed |
| `PENDING -> EXPIRED` (no payment before expiresAt) | Not tested |
| `PENDING -> CANCELLED` (seller-initiated) | Not tested |
| "Once a session exits PENDING, it cannot be modified" | Not tested |

**Verdict: CONFIRMED** for the happy path.

---

## Performance

| Step | Latency |
|------|---------|
| Session creation | ~0.8s |
| Preflight check | 0.54s |
| Payment initiation | 0.55s |
| On-chain confirmation | ~51s (from pay to paidAt) |

The README says "typical confirmation: 10-30 seconds." Our test took ~51 seconds, which exceeds the documented range. Could be beta environment variance or network conditions.

---

## Summary Scorecard

| README Claim | Status | Notes |
|-------------|--------|-------|
| Session creation | ✅ Confirmed | Works, returns checkout URL + session ID |
| Preflight — canPay check | ✅ Confirmed | Extra agent/session context beyond docs |
| Agent payment initiation | ✅ Confirmed | Response richer than documented |
| Payment polling (GET /payments/:txId) | ❌ Failed | 403 — "does not belong to this agent" |
| Payment history (GET /payments) | ❌ Failed | Returns empty despite confirmed payment |
| Session status (GET /sessions/:id) | ✅ Confirmed | Shows PAID with tx hash (usable workaround) |
| On-chain USDC settlement | ✅ Confirmed | Real tx hash on Base |
| Confirmation time 10-30s | ⚠️ Slower | Took ~51s in our test |
| Session lifecycle (PENDING → PAID) | ✅ Confirmed | |
| Error codes (400, 404, 410) | ⬜ Not tested | |
| Policy guardrails on checkout | ⬜ Not tested | |

**Overall: 5/7 tested claims confirmed, 2 failed (polling and payment history), 1 performance deviation**

---

## Issues Found

### 1. CRITICAL — Payment Polling Returns 403
The documented 3-step flow (preflight → pay → poll) breaks at step 3. The agent that initiated the payment gets `403 Forbidden: Transaction does not belong to this agent`. This may be a beta-only bug, or a scoping issue when merchant and payer are the same account.

### 2. MODERATE — Payment History Empty
`GET /checkout/agent/payments` returns no results despite a successful checkout payment. Related to the same ownership/scoping issue.

### 3. MINOR — Confirmation Slower Than Documented
README says 10-30 seconds, actual was ~51 seconds. Could be beta environment, but worth noting.

### 4. OBSERVATION — availableBalance in Preflight
Preflight showed `availableBalance: "999999"` instead of the actual wallet balance ($9.98). This appears to reflect "no limit set" rather than actual funds, which could mislead agents about their true spending capacity.

---

## Cost of This Evaluation

| Item | Cost |
|------|------|
| Session creation | Free |
| Preflight check | Free |
| Checkout payment | $0.01 (self-payment, net zero) |
| Polling, history, balance checks | Free |
| **Total actual cost** | **$0.00 USDC** (self-payment) |

Balance unchanged: $9.98 → $9.98

---

## Possible Caveat

We tested with a single account acting as both merchant and payer. The polling/history issues _may_ be specific to this self-payment scenario. A proper test would involve two separate Locus accounts — one creating the session, one paying it. The core payment mechanics work regardless.
