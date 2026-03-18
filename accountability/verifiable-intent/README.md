# Verifiable Intent (VI): Cryptographic Agent Authorization for Commerce

> **Open specification for creating tamper-evident delegation chains that bind AI agent actions to human-approved scope — using SD-JWT credentials with selective disclosure.**

Verifiable Intent (VI) is a **layered SD-JWT credential format** that creates a cryptographic chain from a credential provider (e.g., Mastercard) → user → AI agent, proving that an agent's purchase actions were within the scope the user authorized. It defines how users express purchase constraints (amount limits, allowed merchants, approved products), how agents prove they acted within those constraints, and how merchants and payment networks verify the full chain — all while ensuring each party sees **only the claims relevant to their role** via selective disclosure.

- **Spec:** [verifiableintent.dev/spec](https://verifiableintent.dev/spec/)
- **Repo:** [github.com/agent-intent/verifiable-intent](https://github.com/agent-intent/verifiable-intent)
- **Status:** Draft v0.1 (February 2026)
- **Maintained by:** Mastercard; open to multi-stakeholder contribution
- **License:** Apache 2.0
- **Standards:** SD-JWT (RFC 9901), JWS (RFC 7515), JWK (RFC 7517), JWT (RFC 7519), RFC 7800 (`cnf` claim), ES256, SHA-256

---

## Table of Contents

- [The Problem](#the-problem)
- [How VI Solves It](#how-vi-solves-it)
- [Architecture Overview](#architecture-overview)
- [The Three Layers](#the-three-layers)
- [Two Execution Modes](#two-execution-modes)
- [Constraint System](#constraint-system)
- [Selective Disclosure](#selective-disclosure)
- [Checkout-Payment Integrity Binding](#checkout-payment-integrity-binding)
- [Verification Model](#verification-model)
- [What VI Can and Cannot Do](#what-vi-can-and-cannot-do)
- [Real-World Use Cases](#real-world-use-cases)
- [Comparison with ACTIS, ERC-8150, and ICME](#comparison-with-actis-erc-8150-and-icme)
- [Key Diagrams](#key-diagrams)

---

## The Problem

When a human delegates a purchase to an AI agent, **no party in the transaction can verify that the agent's actions actually reflect the user's wishes.** Today's payment infrastructure assumes human presence at the point of transaction — AI agents break that assumption.

| Stakeholder | Risk Without VI |
|-------------|----------------|
| **User** | Agent overspends, selects wrong products, or transacts with untrusted merchants |
| **Merchant** | Increased chargebacks from unauthorized agent transactions; no proof agent was authorized |
| **Payment Network** | Dispute liability is ambiguous — who authorized the transaction? |
| **Credential Provider** | Credential misuse by agents operating outside user-granted scope |
| **Agent Platform** | Liability for agent actions without provable authorization chain |

The key gap: existing payment systems prove **identity** ("are you the cardholder?") but not **delegation** ("is this agent authorized by the cardholder, and within what scope?").

---

## How VI Solves It

VI creates a **cryptographic delegation chain** where each layer binds the next through key confirmation claims:

```
Credential Provider (e.g., Mastercard)
  |
  | Signs L1 credential (SD-JWT)
  | Binds user's public key via cnf.jwk
  v
User
  |
  | Signs L2 mandate (KB-SD-JWT)
  | Defines constraints: max $300, only Tennis Warehouse, rackets only
  | Binds agent's public key via cnf.jwk (Autonomous mode)
  v
Agent
  |
  | Signs L3 credentials (KB-SD-JWT)
  | Concrete values: $279.99, Tennis Warehouse, Babolat Pure Aero
  | Must satisfy all L2 constraints
  v
Merchant receives L3b (checkout details)
Payment Network receives L3a (payment details)
  |
  | Each verifies the full chain independently
  | Each sees ONLY the claims relevant to their role
```

**Key properties:**
- **Tamper-evident:** Modifying any layer breaks the signature chain
- **Scope-constrained:** Agent's concrete actions are checked against user's constraints
- **Privacy-preserving:** Merchant sees checkout but not payment; network sees payment but not checkout
- **Non-repudiable:** Each party's signature is cryptographic proof of their authorization

---

## Architecture Overview

```
+------------------------------------------------------------------+
|  LAYER 1 (L1) — SD-JWT                                           |
|  Issuer: Credential Provider (Mastercard, Visa, etc.)             |
|  Lifetime: ~1 year                                                |
|                                                                    |
|  Contains:                                                         |
|  - User identity claims (email, pan_last_four, scheme)            |
|  - cnf.jwk = User's public key (device Secure Enclave)           |
|  - vct: credential type (e.g., Mastercard card profile)           |
|                                                                    |
|  Signed by: Credential Provider private key                        |
|  Verified via: JWKS endpoint                                      |
+----------------------------------+-------------------------------+
                                   |
                    L2 signed by key in L1 cnf.jwk
                                   |
                                   v
+------------------------------------------------------------------+
|  LAYER 2 (L2) — KB-SD-JWT / KB-SD-JWT+KB                         |
|  Issuer: User                                                      |
|  Lifetime: 15 min (Immediate) / 24h–30d (Autonomous)              |
|                                                                    |
|  IMMEDIATE MODE:                  AUTONOMOUS MODE:                 |
|  - Final checkout values          - Checkout CONSTRAINTS           |
|  - Final payment values           - Payment CONSTRAINTS            |
|  - No cnf in mandates             - cnf.jwk = Agent's key         |
|  - typ: "kb-sd-jwt"              - typ: "kb-sd-jwt+kb"            |
|  - No L3 needed                   - Agent creates L3              |
|                                                                    |
|  sd_hash binds L2 to serialized L1                                |
|  Signed by: User's private key (from L1 cnf.jwk)                 |
+----------------------------------+-------------------------------+
                                   |
                    L3 signed by key in L2 mandate cnf.jwk
                    (Autonomous mode only)
                                   |
                                   v
+------------------------------------------------------------------+
|  LAYER 3 (L3a + L3b) — KB-SD-JWT (Autonomous only)               |
|  Issuer: Agent                                                     |
|  Lifetime: ~5 minutes (RECOMMENDED)                                |
|                                                                    |
|  L3a (→ Payment Network):        L3b (→ Merchant):                |
|  - Payment mandate               - Checkout mandate               |
|  - amount, currency, payee       - checkout_jwt, items             |
|  - transaction_id                - checkout_hash                   |
|                                                                    |
|  kid in header matches L2 cnf.kid (agent key resolution)          |
|  No cnf in payload (terminal — no further delegation)             |
|  sd_hash binds to L2 base + relevant disclosures                  |
|  Signed by: Agent's private key (from L2 mandate cnf.jwk)        |
+------------------------------------------------------------------+
```

---

## The Three Layers

### Layer 1: Credential Provider → User

The root of trust. A payment credential (like a digital card) issued by a provider (Mastercard, Visa, bank).

| Field | Value | Purpose |
|-------|-------|---------|
| `typ` | `"sd+jwt"` | Identifies as SD-JWT |
| `vct` | `"https://credentials.mastercard.com/card"` | Credential type profile |
| `cnf.jwk` | User's public key | Binds user — L2 must be signed by this key |
| `pan_last_four` | `"8842"` | Card identifier (always visible) |
| `scheme` | `"mastercard"` | Card network |
| `email` | Selectively disclosable | User identity |
| Lifetime | ~1 year | Long-lived root credential |

### Layer 2: User → Agent/Verifier

The user's purchase authorization. Contains either final values (Immediate) or constraints + agent delegation (Autonomous).

**Immediate mode** — user confirms exact purchase:
```json
{
  "typ": "kb-sd-jwt",
  "sd_hash": "<hash of serialized L1>",
  "checkout_mandate": {
    "vct": "mandate.checkout",
    "checkout_jwt": "<merchant-signed cart JWT>",
    "checkout_hash": "<SHA-256 of checkout_jwt>"
  },
  "payment_mandate": {
    "vct": "mandate.payment",
    "amount": 27999,
    "currency": "USD",
    "payee": { "id": "tennis-warehouse", "name": "Tennis Warehouse" },
    "transaction_id": "<same as checkout_hash>"
  }
}
```

**Autonomous mode** — user sets constraints, agent acts within them:
```json
{
  "typ": "kb-sd-jwt+kb",
  "sd_hash": "<hash of serialized L1>",
  "checkout_mandate": {
    "vct": "mandate.checkout.open",
    "constraints": [
      { "type": "mandate.checkout.allowed_merchant", "allowed_merchants": [...] },
      { "type": "mandate.checkout.line_items", "items": [...] }
    ],
    "cnf": { "kid": "agent-key-1", "jwk": { ... } }
  },
  "payment_mandate": {
    "vct": "mandate.payment.open",
    "constraints": [
      { "type": "payment.amount", "currency": "USD", "min": 0, "max": 30000 },
      { "type": "payment.allowed_payee", "allowed_payees": [...] }
    ],
    "cnf": { "kid": "agent-key-1", "jwk": { ... } }
  }
}
```

### Layer 3: Agent → Merchant/Network (Autonomous only)

The agent's concrete actions, split for privacy:

**L3a** (to Payment Network): payment mandate with actual amount, currency, payee, transaction_id
**L3b** (to Merchant): checkout mandate with checkout_jwt, items, checkout_hash

Each L3 includes:
- `kid` in header matching L2 `cnf.kid` (key resolution)
- `sd_hash` binding to L2 base + relevant disclosures
- No `cnf` in payload (terminal — no further delegation)
- Signed by agent's private key

---

## Two Execution Modes

| | Immediate | Autonomous |
|---|---|---|
| **Layers** | 2 (L1 + L2) | 3 (L1 + L2 + L3) |
| **User role** | Reviews and confirms exact values | Sets constraints; may not be present at transaction time |
| **Agent role** | Forwarding only (no delegation) | Selects products, creates checkout, builds L3 |
| **L2 typ** | `kb-sd-jwt` | `kb-sd-jwt+kb` |
| **Mandate vct** | `mandate.checkout` / `mandate.payment` | `mandate.checkout.open` / `mandate.payment.open` |
| **Constraints?** | No — final values | Yes — constraint arrays |
| **Agent key bound?** | No `cnf` in mandates | `cnf.jwk` + `cnf.kid` in mandates |
| **L3 created?** | No | Yes (L3a + L3b) |
| **L2 lifetime** | ~15 minutes | 24 hours – 30 days |
| **L3 lifetime** | N/A | ~5 minutes |
| **Use case** | User-confirmed purchases, re-orders, one-click buy | Delegated shopping, automated replenishment, price-watching |
| **Example** | "Buy these 3 tennis balls for $5.99" | "Buy me a racket under $300 from Tennis Warehouse" |

---

## Constraint System

Constraints only exist in **Autonomous mode** L2 mandates. They define the boundaries within which the agent may act.

### Checkout Constraints

| Type | Purpose | Example |
|------|---------|---------|
| `mandate.checkout.line_items` | Restrict what products the agent may select | Allowed SKUs, acceptable brands, quantity limits |
| `mandate.checkout.allowed_merchant` | Restrict which merchants the agent may shop at | Merchant allowlist by ID/name/website |

### Payment Constraints

| Type | Purpose | Example |
|------|---------|---------|
| `payment.amount` | Min/max transaction amount | `{ "currency": "USD", "min": 0, "max": 30000 }` |
| `payment.allowed_payee` | Restrict who receives payment | Payee allowlist by ID/name |
| `payment.instrument` | Restrict payment method | Specific card, specific instrument type |
| `payment.budget` | Cumulative spending limit across transactions | `{ "currency": "USD", "total": 100000 }` |
| `payment.recurrence` | Subscription terms | Frequency, date range, occurrence cap |
| `payment.agent_recurrence` | Multi-transaction within single mandate pair | Occurrence cap + cumulative budget |
| `payment.reference` | Binds payment to specific checkout | `conditional_transaction_id` linking payment to checkout hash |

### How Constraints Are Verified

```
L2 Constraints (user-defined)          L3 Values (agent-created)
────────────────────────────           ─────────────────────────

payment.amount:                        L3a payment mandate:
  currency: USD                          amount: 27999
  min: 0                                 currency: USD
  max: 30000                             payee: Tennis Warehouse

Verifier checks:                       Result:
  27999 >= 0? ✓                          PASS
  27999 <= 30000? ✓
  currency == "USD"? ✓

payment.allowed_payee:
  allowed: [Tennis Warehouse,
            Wilson Sporting]

Verifier checks:
  "Tennis Warehouse" in allowed? ✓       PASS
```

**Machine-enforceable** constraints (amount, payee, merchant) are checked algorithmically. **Descriptive** fields (product name, brand, color, size) are informational only — not subject to automated verification.

---

## Selective Disclosure

Each party in the transaction sees **only what they need**. This is the core privacy property of SD-JWT.

```
                    L1 + L2 + L3a + L3b
                    (full credential chain)
                            |
              +-------------+-------------+
              |                           |
              v                           v
        MERCHANT                    PAYMENT NETWORK
              |                           |
    Disclosed:                   Disclosed:
    - L3b checkout mandate       - L3a payment mandate
      (items, merchant,            (amount, currency,
       checkout_jwt)                payee, transaction_id)
    - prompt_summary             - Selected merchant
      (optional)                   (for constraint check)

    Hidden:                      Hidden:
    - Payment mandate            - Checkout mandate
    - Amount, payee              - Items, product details
    - Card details               - Full merchant details


              DISPUTE RESOLUTION
              (sees everything)
              - Both mandates disclosed
              - Full chain verified
              - checkout_hash == transaction_id confirmed
```

---

## Checkout-Payment Integrity Binding

A critical problem: checkout details go to the merchant, payment details go to the network. How do you ensure the $279.99 payment is for the specific checkout containing the tennis racket, not some other checkout?

### The Binding Mechanism

```
checkout_jwt (merchant-signed cart)
        |
        v
checkout_hash = Base64url(SHA-256(ASCII(checkout_jwt)))
        |
        +──────────────────+──────────────────+
        |                                     |
        v                                     v
  Checkout Mandate                      Payment Mandate
  (in L3b → Merchant)                  (in L3a → Network)

  checkout_hash: "abc123..."           transaction_id: "abc123..."

  MUST be equal ←─────────────────────→ MUST be equal
```

**Verification:** When both mandates are disclosed (e.g., dispute resolution), the verifier:
1. Recomputes `SHA-256(checkout_jwt)` from the disclosed checkout JWT
2. Compares to `checkout_hash` in checkout mandate
3. Compares to `transaction_id` in payment mandate
4. All three must match — otherwise the payment doesn't correspond to the checkout

---

## Verification Model

### What Each Party Verifies

**Merchant verifies:**
1. L1 signature against Credential Provider JWKS
2. L2 signature against key in L1 `cnf.jwk`
3. `sd_hash` bindings between layers
4. Checkout mandate matches submitted checkout
5. (Autonomous) L3b signature against agent key from L2 `cnf.jwk`
6. `typ` headers at each layer

**Payment Network verifies:**
1. Full L1 → L2 → L3a signature chain
2. `sd_hash` bindings
3. Payment mandate values
4. (Autonomous) L3a values satisfy L2 payment constraints
5. `checkout_hash` == `transaction_id` cross-reference
6. Nonce freshness, expiration at all layers
7. Cumulative spend tracking for budget/recurrence constraints

### Verification Flow (Autonomous Mode)

```
Payment Network receives: L1 + L2 (payment disclosed) + L3a

Step 1: Verify L1 signature
  → Fetch Credential Provider JWKS
  → ES256 verify L1 against provider's public key
  → Check L1 typ == "sd+jwt"

Step 2: Verify L2 signature
  → Extract user public key from L1 cnf.jwk
  → ES256 verify L2 against user's key
  → Check L2 typ == "kb-sd-jwt+kb" (Autonomous)
  → Verify sd_hash binds L2 to L1

Step 3: Resolve agent key
  → Extract cnf.jwk from L2 payment mandate
  → Match L3a header kid against L2 cnf.kid
  → Confirm keys match

Step 4: Verify L3a signature
  → ES256 verify L3a against agent's key
  → Check L3a typ == "kb-sd-jwt"
  → Verify sd_hash binds L3a to L2 + relevant disclosures
  → Confirm NO cnf in L3a payload (terminal)

Step 5: Check constraints
  → L3a amount within L2 payment.amount range?
  → L3a payee in L2 payment.allowed_payee list?
  → Cumulative spend within L2 payment.budget?
  → transaction_id matches expected binding?

Step 6: Check freshness
  → All layers not expired?
  → Nonce not replayed?

ALL PASS → Authorize payment
ANY FAIL → Reject
```

---

## What VI Can and Cannot Do

### What VI Guarantees

| Guarantee | Mechanism |
|-----------|-----------|
| **Agent acted within user-approved scope** | L3 values checked against L2 constraints; cryptographic chain proves delegation |
| **Delegation chain is tamper-evident** | ES256 signatures at each layer; modification breaks signature |
| **Each party sees only relevant claims** | SD-JWT selective disclosure; merchant sees checkout, network sees payment |
| **Payment corresponds to specific checkout** | `checkout_hash` == `transaction_id` binding |
| **Agent key is user-authorized** | L2 `cnf.jwk` binds specific agent key; L3 signed by that key |
| **Credentials expire** | L1 ~1yr, L2 Autonomous ~30d, L3 ~5min |
| **No further delegation** | L3 has no `cnf` — agent can't sub-delegate |
| **Cross-merchant replay blocked** | `nonce` + `aud` + per-mandate-pair tracking |

### What VI Does NOT Guarantee

| Non-guarantee | Why | What Would Be Needed |
|---------------|-----|---------------------|
| **Agent selects the BEST product** | VI verifies the selection is WITHIN constraints, not that it's optimal | Agent quality evaluation, price comparison |
| **Agent isn't compromised** | A compromised agent with valid key can act within constraints | Agent platform security, key rotation, short L3 lifetimes |
| **User wrote good constraints** | "Buy anything under $300" is a valid but weak constraint | Better constraint UX, template libraries |
| **Descriptive fields are accurate** | Product name, brand, color are informational — not machine-verified | Application-layer product validation |
| **Post-transaction delivery** | VI covers authorization, not fulfillment | Order tracking, delivery confirmation |
| **Agent identity (who the agent IS)** | VI binds a key, not a real-world identity | Agent attestation (optional extension in spec) |
| **Dispute resolution** | VI provides evidence for disputes, not resolution | External dispute process |
| **Non-payment actions** | VI is designed for purchase transactions | Broader agent authorization frameworks |

### Plain-Language Scenarios

**Where VI saves you:**

```
1. Agent overspends
   User constraint: max $300.
   Agent tries to buy $500 item.
   → Network checks L3a amount against L2 constraint → REJECTED.

2. Agent shops at wrong merchant
   User constraint: only Tennis Warehouse and Wilson.
   Agent tries to checkout at Random Store.
   → Merchant not in allowed_merchants → REJECTED.

3. Chargeback dispute
   User claims they didn't authorize the purchase.
   → Merchant shows L1→L2→L3b chain with user's ES256 signature
     on the constraints that permitted this exact purchase.
   → Cryptographic proof of authorization. Chargeback denied.

4. Agent tries to buy unauthorized product category
   User constraint: only tennis rackets (specific SKUs).
   Agent adds tennis balls to cart.
   → L3b line items don't match L2 line_item constraints → REJECTED.
```

**Where VI doesn't save you:**

```
1. User approves broad constraints
   "Buy anything under $1000 from any merchant."
   Agent buys something useless for $999.
   → VI: constraints satisfied. Authorized.
   → VI enforces scope, not judgment.

2. Agent gets the worst deal
   User says "buy a racket under $300."
   Same racket is $200 on Amazon, agent pays $299 at Tennis Warehouse.
   → VI: $299 <= $300 AND Tennis Warehouse is allowed. Authorized.
   → VI doesn't optimize — it constrains.

3. Credential Provider compromised
   If the L1 issuer's key is compromised, fake L1s can be created.
   → Mitigated by key rotation, JWKS endpoint, short L2/L3 lifetimes.
   → But a compromised root is a compromised root.
```

---

## Real-World Use Cases

### Use Case 1: Autonomous Shopping Agent

```
User: "Buy me a tennis racket and some balls. Budget $300.
       Only from Tennis Warehouse."

L2 (Autonomous):
  Checkout constraints:
    - allowed_merchant: [Tennis Warehouse]
    - line_items: [racket (qty 1), balls (qty 1-3)]
  Payment constraints:
    - amount: { min: 0, max: 30000, currency: USD }
    - allowed_payee: [Tennis Warehouse]
  Agent key bound via cnf.jwk

Agent browses Tennis Warehouse:
  - Selects Babolat Pure Aero ($279.99)
  - Selects can of Penn balls ($5.99)
  - Gets merchant-signed checkout_jwt

Agent creates L3a (→ network) and L3b (→ merchant):
  - L3a: amount=28598, payee=Tennis Warehouse, transaction_id=<hash>
  - L3b: checkout_jwt, checkout_hash=<hash>

Network verifies: 28598 <= 30000 ✓, payee allowed ✓, chain valid ✓
Merchant verifies: checkout matches catalog ✓, chain valid ✓
→ Payment authorized.
```

### Use Case 2: Automated Subscription Renewal

```
User: "Renew my SaaS subscriptions monthly. Max $50/each, $200/month total."

L2 (Autonomous):
  Payment constraints:
    - payment.amount: { max: 5000, currency: USD }
    - payment.budget: { total: 20000, currency: USD }
    - payment.recurrence: { frequency: monthly, occurrences: 12 }
  Checkout constraints:
    - allowed_merchant: [Notion, GitHub, Vercel]

Agent renews each month:
  Month 1: Notion $10, GitHub $20, Vercel $49 = $79 total
  → Each L3 checked: amount ≤ $50 ✓, cumulative $79 ≤ $200 ✓

  Month 2: Same renewals
  → Network tracks cumulative across L3s per L2
```

### Use Case 3: Immediate Mode — One-Click Reorder

```
User: "Reorder my usual tennis balls."

Agent builds checkout (Tennis Warehouse, 3x Penn balls, $17.97).
User reviews exact cart and price. Confirms.

L2 (Immediate):
  checkout_mandate: { checkout_jwt: "...", checkout_hash: "abc..." }
  payment_mandate: { amount: 1797, currency: USD, transaction_id: "abc..." }
  No constraints (final values). No agent delegation.

Agent forwards L1+L2 to merchant and network.
→ No L3 needed. User signed the exact values.
```

---

## Comparison with ACTIS, ERC-8150, and ICME

| Dimension | ACTIS | ERC-8150 | ICME | VI |
|-----------|-------|----------|------|-----|
| **Layer** | Evidence (post-execution) | Enforcement (pre-execution, on-chain) | Verification (pre-execution, API) | Authorization (pre-execution, credential chain) |
| **Core question** | "Was this record tampered with?" | "Do txs match signed intent?" | "Does action comply with policy?" | "Is this agent authorized by the user within this scope?" |
| **Mechanism** | Hash chains + Ed25519 | ZK-SNARKs + smart contract | SMT solver + zkML | SD-JWT delegation chain + selective disclosure |
| **Blockchain?** | No | Yes (Ethereum) | No | No |
| **Privacy** | Full transparency | Intent hidden (ZK) | Policy hidden (ZK) | Role-based selective disclosure |
| **Payment integration** | None | On-chain ERC-20 only | Any (advisory API) | Native to payment networks (Mastercard profile) |
| **Scope definition** | N/A (records only) | Exact actions (amounts, recipients) | Natural language → formal logic | Structured constraints (amount, merchant, items) |
| **Who verifies** | Any party (offline) | Smart contract (on-chain) | ICME API (off-chain) | Merchant + payment network (inline) |
| **Industry backing** | Open source | ERC draft | Startup (ICME Labs) | Mastercard |
| **Standards base** | Custom (SHA-256, Ed25519) | EIP-712, Circom | SMT-LIB, Jolt | SD-JWT (RFC 9901), JWS, JWK, JWT, RFC 7800 |
| **Agent delegation** | N/A | No (user signs exact intent) | N/A (checks actions) | Yes (cryptographic key binding L2→L3) |
| **Constraint types** | N/A | Per-action amounts | Any formalizable rule | 8 registered types (amount, merchant, payee, items, budget, recurrence, instrument, reference) |
| **Multi-party verification** | Any party can verify | Smart contract only | API caller | Merchant AND network verify independently with different disclosures |

### How They Fit Together

```
+------------------------------------------------------------------------+
|                                                                         |
|  AUTHORIZATION (Verifiable Intent):                                     |
|  "Is this agent authorized by the user within this scope?"              |
|  → SD-JWT delegation chain (Credential Provider → User → Agent)         |
|  → Constraints checked by merchant + payment network                    |
|  → Selective disclosure per role                                        |
|                                                                         |
|  POLICY COMPLIANCE (ICME):                                              |
|  "Does this action comply with organizational policy?"                  |
|  → Natural language → formal logic → SAT/UNSAT                          |
|  → ZK proof of compliance (optional)                                    |
|                                                                         |
|  ON-CHAIN ENFORCEMENT (ERC-8150):                                       |
|  "Do these on-chain transactions match the signed intent?"              |
|  → ZK-SNARK proof of calldata-intent alignment                         |
|  → Smart contract reverts if proof fails                                |
|                                                                         |
|  ─────────────── Transaction executes ───────────────                   |
|                                                                         |
|  EVIDENCE (ACTIS):                                                      |
|  "Here's a tamper-evident record of the full negotiation"               |
|  → Hash-linked, signed transcript                                       |
|  → Offline verifiable                                                   |
|                                                                         |
+------------------------------------------------------------------------+
```

---

## Key Diagrams

### Autonomous Mode — Full Flow

```
Credential Provider            User                Agent              Merchant         Network
      |                          |                    |                   |                |
      |  1. Issue L1 (SD-JWT)    |                    |                   |                |
      |  cnf.jwk = user key      |                    |                   |                |
      |------------------------->|                    |                   |                |
      |                          |                    |                   |                |
      |                          |  2. "Buy racket    |                   |                |
      |                          |   under $300 from  |                   |                |
      |                          |   Tennis Warehouse" |                   |                |
      |                          |                    |                   |                |
      |                          |  3. Create L2      |                   |                |
      |                          |  (KB-SD-JWT+KB)    |                   |                |
      |                          |  constraints +     |                   |                |
      |                          |  agent cnf.jwk     |                   |                |
      |                          |  signed by user key |                   |                |
      |                          |------------------->|                   |                |
      |                          |                    |                   |                |
      |                          |                    |  4. Browse,       |                |
      |                          |                    |  select products  |                |
      |                          |                    |  within L2        |                |
      |                          |                    |  constraints      |                |
      |                          |                    |                   |                |
      |                          |                    |  5. Get checkout  |                |
      |                          |                    |  JWT from merchant|                |
      |                          |                    |<-----------------|                |
      |                          |                    |                   |                |
      |                          |                    |  6. Create L3a    |                |
      |                          |                    |  (payment →       |                |
      |                          |                    |   network)        |                |
      |                          |                    |  Create L3b       |                |
      |                          |                    |  (checkout →      |                |
      |                          |                    |   merchant)       |                |
      |                          |                    |                   |                |
      |                          |                    |  7. Present       |                |
      |                          |                    |  L3b + chain ---->|                |
      |                          |                    |  L3a + chain -----|--------------->|
      |                          |                    |                   |                |
      |                          |                    |                   |  8. Verify     |
      |                          |                    |                   |  L1 sig ✓      |
      |                          |                    |  8. Verify        |  L2 sig ✓      |
      |                          |                    |  L1 sig ✓         |  L3a sig ✓     |
      |                          |                    |  L2 sig ✓         |  constraints ✓ |
      |                          |                    |  L3b sig ✓        |  sd_hash ✓     |
      |                          |                    |  checkout ✓       |  binding ✓     |
      |                          |                    |  sd_hash ✓        |                |
      |                          |                    |                   |  9. Authorize  |
      |                          |                    |                   |  payment       |
      |                          |                    |                   |<---------------|
      |                          |                    |<-----------------|                |
      |                          |                    |  "Order confirmed"|                |
```

### Selective Disclosure per Role

```
Full credential chain:
  L1: [identity, pan_last_four, scheme, email, cnf.jwk]
  L2: [checkout_mandate, payment_mandate, prompt_summary]
  L3a: [payment details]
  L3b: [checkout details]

                    ┌──────────────────────┐
                    │   What Merchant Sees  │
                    │                       │
                    │   L1: pan_last_four,  │
                    │       scheme, cnf     │
                    │   L2: checkout only   │
                    │   L3b: checkout_jwt,  │
                    │        items          │
                    │                       │
                    │   HIDDEN: payment     │
                    │   amount, payee,      │
                    │   card details        │
                    └──────────────────────┘

                    ┌──────────────────────┐
                    │   What Network Sees   │
                    │                       │
                    │   L1: pan_last_four,  │
                    │       scheme, cnf     │
                    │   L2: payment only    │
                    │   L3a: amount,        │
                    │        currency,      │
                    │        payee,         │
                    │        transaction_id │
                    │                       │
                    │   HIDDEN: checkout    │
                    │   items, product      │
                    │   details, merchant   │
                    │   specifics           │
                    └──────────────────────┘

                    ┌──────────────────────┐
                    │  Dispute Resolution   │
                    │  (sees everything)    │
                    │                       │
                    │  Both mandates        │
                    │  Full chain           │
                    │  checkout_hash ==     │
                    │  transaction_id ✓     │
                    └──────────────────────┘
```

---

## Summary

Verifiable Intent provides **cryptographic agent authorization** for commerce:

1. **Layered SD-JWT delegation chain** — Credential Provider → User → Agent, each layer binding the next through key confirmation claims (ES256 signatures, `cnf.jwk`)
2. **Two execution modes** — Immediate (user confirms exact values, 2 layers) and Autonomous (user sets constraints, agent acts within them, 3 layers)
3. **8 constraint types** — amount range, allowed merchants, allowed payees, line items, budget, recurrence, agent recurrence, payment reference
4. **Selective disclosure** — merchant sees checkout details only, network sees payment details only, dispute resolution sees everything
5. **Checkout-payment integrity** — `checkout_hash` == `transaction_id` binding ensures payment corresponds to specific checkout across split disclosures
6. **Payment network native** — designed to integrate with existing card network infrastructure (Mastercard reference profile), not a separate on-chain system
7. **Reference implementation** — Python SDK with issuance, verification, constraint checking, and selective disclosure (`pip install verifiable-intent`)

The core innovation: **extending payment authentication from "is this the cardholder?" to "is this agent authorized by the cardholder, within what scope, and can each party verify independently while seeing only their relevant claims?"** VI doesn't replace payment infrastructure — it adds an agent authorization layer on top of it.
