# Accountability Frameworks Comparison: ACTIS vs ERC-8150 vs ICME vs Verifiable Intent

> Four approaches to the same meta-problem: **how do you trust AI agents that spend money on your behalf?**

Each framework attacks a different facet of this problem — evidence, enforcement, verification, or authorization. None is a substitute for the others. This document compares them across architecture, mechanism, scope, trust model, and practical tradeoffs.

---

## At a Glance

| | **ACTIS** | **ERC-8150** | **ICME** | **Verifiable Intent** |
|---|---|---|---|---|
| **One-liner** | Tamper-evident transaction records | ZK-proof that agent txs match signed intent | Policy → formal logic → SAT/UNSAT + ZK proof | SD-JWT delegation chain binding agent to user scope |
| **Layer** | Evidence (post-execution) | Enforcement (pre-execution, on-chain) | Verification (pre-execution, off-chain API) | Authorization (pre-execution, credential chain) |
| **Core question** | "Was this record tampered with?" | "Do these txs match what the user signed?" | "Does this action comply with formal policy?" | "Is this agent authorized by the user within this scope?" |
| **When it acts** | After transaction | Before transaction | Before transaction | Before transaction |
| **Prevents bad actions?** | No (evidence only) | Yes (smart contract reverts) | Advisory (agent should honor result) | Yes (network/merchant rejects unauthorized) |
| **Blockchain required?** | No | Yes (Ethereum/EVM) | No | No |
| **Industry backing** | Open source (independent) | ERC draft (Ethereum community) | Startup (ICME Labs) | Mastercard |
| **Status** | v1.0 (March 2026) | Draft ERC | Live API | Draft v0.1 (Feb 2026) |

---

## 1. What Problem Does Each Solve?

### ACTIS: "We can't prove what happened"

```
Problem: Agent executed a transaction. A dispute arises.
         Both sides have their own logs. Logs disagree.
         Nobody has a mutually trusted record.

ACTIS:   Hash-linked, signed transcript of the negotiation.
         Any party can verify it hasn't been tampered with.
         Offline. No trust in the producing system.
```

ACTIS doesn't prevent anything. It **documents** what happened in a way that's cryptographically verifiable after the fact.

### ERC-8150: "The agent might deviate from what I approved"

```
Problem: User approves a batch of transfers.
         Agent submits the transactions to the blockchain.
         How do we know the agent didn't change amounts or recipients?

ERC-8150: Agent produces a ZK-SNARK proving its submitted calldata
          derives from the user's signed intent bundle.
          Smart contract verifies proof on-chain.
          Mismatch → entire transaction reverts. Zero funds moved.
```

ERC-8150 **enforces** exact match between signed intent and executed transactions, on-chain.

### ICME: "Our guardrails are probabilistic — an LLM judging an LLM"

```
Problem: Agent's guardrails are another LLM.
         Attacker tricks the agent → probably tricks the guardrail too.
         "88% confidence it's safe" isn't good enough for $250,000.

ICME:    Convert natural language policy to formal logic (SMT-LIB).
         Check each action: SAT (mathematically allowed) or UNSAT (blocked).
         Optionally wrap in ZK proof for trustless cross-org verification.
```

ICME replaces **probabilistic** guardrails with **deterministic** formal verification. Advisory — the agent must choose to honor the result.

### Verifiable Intent: "Nobody can prove the agent was authorized"

```
Problem: Agent buys something. Merchant asks: who authorized this?
         User claims they didn't. Agent platform says user did.
         Payment network doesn't know who to believe.
         Chargebacks, disputes, liability ambiguity.

VI:      Cryptographic delegation chain:
         Credential Provider → User → Agent
         Each layer signed, each layer constrained.
         Merchant verifies checkout authorization.
         Network verifies payment authorization.
         Selective disclosure — each party sees only their relevant claims.
```

VI creates a **provable authorization chain** for payment networks — extending "is this the cardholder?" to "is this agent authorized by the cardholder, within what scope?"

---

## 2. Architecture Comparison

### ACTIS Architecture

```
Agent Runtime (out of scope)
        |
        | Produces rounds: INTENT → ASK → BID → ACCEPT
        v
+------------------+
| ACTIS Transcript |  Hash-linked, Ed25519-signed JSON
| (evidence)       |  Each round references previous round's hash
+------------------+  Genesis hash binds to intent_id
        |
        v
+------------------+
| ACTIS Bundle     |  ZIP: manifest.json + transcript.json + checksums.sha256
| (portable)       |  Self-contained, offline-verifiable
+------------------+
        |
        v
+------------------+
| ACTIS Verifier   |  4 checks: schema, hash chain, signatures, checksums
| (offline)        |  → COMPATIBLE | PARTIAL | NONCOMPLIANT
+------------------+
```

**Stack:** SHA-256 + Ed25519 + RFC 8785 (JCS). No blockchain, no network, no ZK.

### ERC-8150 Architecture

```
User                          Agent                    Ethereum
  |                              |                        |
  | Sign intent bundle           |                        |
  | (EIP-712 typed data)         |                        |
  |----------------------------->|                        |
  |                              |                        |
  | Approve Agent Wallet         |                        |
  | (ERC-20 approve)             |                        |
  |-------------------------------------------------->    |
  |                              |                        |
  |                              | Generate ZK-SNARK     |
  |                              | (off-chain, 2-5 sec)  |
  |                              |                        |
  |                              | Submit proof + calls   |
  |                              |----------------------->|
  |                              |                        |
  |                              |    Verify ZK proof     |
  |                              |    Verify signature    |
  |                              |    Check nonce/expiry  |
  |                              |    Execute atomically  |
  |                              |    (or revert ALL)     |
```

**Stack:** ZK-SNARKs (Circom) + keccak256 + ECDSA + EIP-712 + Solidity smart contract. Ethereum-only.

### ICME Architecture

```
Human writes policy               ICME API
  |                                   |
  | "No transfers over $100"          |
  |---------------------------------->|
  |                              makeRules (30-90 sec)
  |                              NL → SMT-LIB formal logic
  |<-- policy_id + rules ------------|
  |                                   |
  |                                   |
Agent wants to act                    |
  |                                   |
  | "Transfer 50 tokens to 0xAAA"     |
  |---------------------------------->|
  |                              checkIt (~200ms)
  |                              LLM extracts variables
  |                              SMT solver checks constraints
  |<-- SAT (allowed) ----------------|
  |                                   |
  | Agent proceeds                    |
  |                                   |
  |                              Optional: ZK proof
  |                              (Jolt Atlas zkVM)
  |                              Prove check ran correctly
  |                              without revealing policy
```

**Stack:** SMT-LIB solvers + LLM extraction + Jolt Atlas zkVM. Off-chain API. Optional ZK.

### Verifiable Intent Architecture

```
Credential Provider              User                   Agent
(Mastercard)                       |                       |
  |                                |                       |
  | Issue L1 (SD-JWT)              |                       |
  | cnf.jwk = user key             |                       |
  |------------------------------->|                       |
  |                                |                       |
  |                                | Create L2 (KB-SD-JWT) |
  |                                | constraints +         |
  |                                | agent cnf.jwk         |
  |                                |---------------------->|
  |                                |                       |
  |                                |                       | Shop within
  |                                |                       | constraints
  |                                |                       |
  |                                |                       | Create L3a → Network
  |                                |                       | Create L3b → Merchant
  |                                |                       |
  |                                |        Merchant verifies L3b (checkout)
  |                                |        Network verifies L3a (payment)
  |                                |        Each sees only relevant claims
```

**Stack:** SD-JWT (RFC 9901) + ES256 + JWS + JWT + RFC 7800 (`cnf`). No blockchain, no ZK, no SMT solvers.

---

## 3. Cryptographic Primitives

| Primitive | ACTIS | ERC-8150 | ICME | VI |
|-----------|-------|----------|------|-----|
| **Hash function** | SHA-256 | keccak256 | SHA-256 (in zkVM) | SHA-256 |
| **Signatures** | Ed25519 | ECDSA (secp256k1) | N/A (API-based) | ES256 (ECDSA P-256) |
| **ZK proofs** | No | ZK-SNARKs (Circom/PLONK) | Jolt Atlas zkVM (optional) | No |
| **Canonicalization** | RFC 8785 (JCS) | EIP-712 typed data | N/A | SD-JWT (RFC 9901) |
| **Formal logic** | No | No | SMT-LIB (Z3/CVC5 class) | No |
| **Key binding** | Per-round signer key | ECDSA recovery from signature | N/A | `cnf.jwk` delegation chain |
| **Selective disclosure** | No (full transparency) | Intent hidden (ZK) | Policy hidden (ZK) | Role-based (SD-JWT) |
| **Replay protection** | N/A (evidence only) | Nonce + chainId | N/A | `nonce` + `aud` + expiry |

**Complexity gradient:** ACTIS (simplest) → VI (moderate) → ICME (moderate + optional ZK) → ERC-8150 (most complex, ZK circuits + on-chain)

---

## 4. Trust Model

### Who Trusts Whom?

| | ACTIS | ERC-8150 | ICME | VI |
|---|---|---|---|---|
| **Trust in agent** | N/A (records, doesn't constrain) | Zero trust (math enforces) | Advisory (agent should honor UNSAT) | Constrained trust (key-bound, scope-limited) |
| **Trust in platform** | None (offline verifiable) | Trust Ethereum consensus | Trust ICME API (or verify ZK proof) | Trust Credential Provider (JWKS) |
| **Trust in counterparty** | None (each verifies independently) | Smart contract is the trust | ZK proof eliminates trust | Each party verifies independently |
| **User must...** | N/A | Sign exact intent bundle | Write good policy rules | Set good constraints |
| **Agent can...** | Do anything (ACTIS just records) | Only execute proven-matching txs | Do anything if it ignores UNSAT | Only act within L2 constraint scope |

### Enforcement Strength

```
Strongest ──────────────────────────────────────────── Weakest

ERC-8150          VI                ICME              ACTIS
(on-chain         (network/merchant (advisory API —    (no enforcement —
 revert —         reject —          agent must honor   evidence only)
 impossible to    verifier-enforced) the result)
 bypass)
```

**ERC-8150:** Agent literally cannot execute unauthorized transactions — the smart contract reverts. Hardest enforcement.

**VI:** Merchant and payment network independently reject L3 credentials that fail verification. Agent can't bypass the network.

**ICME:** Returns SAT/UNSAT, but the agent's runtime must choose to honor it. A malicious or buggy agent could ignore the result.

**ACTIS:** Zero enforcement. Records what happened for later review.

---

## 5. Scope & Flexibility

### What Actions Can Each Framework Handle?

| Action Type | ACTIS | ERC-8150 | ICME | VI |
|-------------|-------|----------|------|-----|
| **ERC-20 transfers** | Record ✓ | Enforce ✓ | Check ✓ | Authorize ✓ |
| **Token swaps (DeFi)** | Record ✓ | Not yet (v1 = transfers only) | Check ✓ | Not designed for DeFi |
| **Physical product purchase** | Record ✓ | Not applicable (on-chain only) | Check ✓ | Core use case ✓ |
| **Multi-merchant shopping** | Record ✓ | No (single-chain atomic) | Check ✓ | Core use case ✓ |
| **Subscription/recurring** | Record ✓ | No (per-batch only) | Check ✓ (with aggregate rules) | `payment.recurrence` constraint ✓ |
| **Email/API actions** | Record ✓ | No (on-chain only) | Check ✓ | No (payment-specific) |
| **Cross-chain** | Record ✓ | No (single chainId) | Check ✓ | N/A (not blockchain-based) |
| **Negotiation** | Core use case ✓ | No | No | No |
| **Arbitrary policy rules** | No | No | Core use case ✓ | Structured constraints only |

### Constraint Expressiveness

| | ACTIS | ERC-8150 | ICME | VI |
|---|---|---|---|---|
| **Amount limits** | N/A | `maxTotalPrice` (cents) | Any arithmetic constraint | `payment.amount` (min/max) |
| **Recipient/merchant restrictions** | N/A | Exact addresses in intent | Any allowlist via formal logic | `allowed_merchant`, `allowed_payee` |
| **Aggregate/budget limits** | N/A | No (per-batch only) | Yes (if policy includes it) | `payment.budget` (cumulative) |
| **Time-based** | N/A | `expiry` timestamp | No built-in (could be in policy) | `payment.recurrence` (frequency, date range) |
| **Product-level** | N/A | No | Possible via policy rules | `mandate.checkout.line_items` (SKUs, quantity, brand) |
| **Custom rules** | N/A | No | Any natural language → formal logic | URI-namespaced extensions |

**ICME is the most flexible** — any rule expressible in natural language that can be converted to formal logic. **VI has the most structured constraints** — 8 registered types designed specifically for commerce. **ERC-8150 has the least** — exact amounts/recipients only, no aggregation.

---

## 6. Privacy Model

| | ACTIS | ERC-8150 | ICME | VI |
|---|---|---|---|---|
| **Transaction details** | Fully visible in transcript | Hidden (ZK proof) | Hidden (ZK proof, optional) | Role-based selective disclosure |
| **Policy/rules** | N/A | Intent bundle hidden (ZK) | Policy hidden (ZK) | Constraints selectively disclosed per role |
| **User identity** | Public key only (no identity binding) | Ethereum address | N/A | SD-JWT selective disclosure (email, card) |
| **What merchant sees** | Everything | N/A (on-chain) | N/A (API) | Checkout only (not payment) |
| **What network sees** | Everything | N/A (on-chain) | N/A (API) | Payment only (not checkout) |
| **Dispute resolution** | Full record | On-chain events | Audit receipts | Full disclosure of both mandates |

**VI has the most sophisticated privacy model** — built on SD-JWT's selective disclosure, each party sees exactly what they need and nothing more. **ERC-8150 and ICME use ZK** for binary hide/reveal. **ACTIS is fully transparent** by design.

---

## 7. Performance & Cost

| | ACTIS | ERC-8150 | ICME | VI |
|---|---|---|---|---|
| **Cost per check** | Free (offline) | ~345k gas + 65k/transfer (~$1-3 on mainnet) | $0.01 per action check | Free (verification is math, no fees) |
| **Proof generation** | N/A | 2-5 seconds (ZK circuit) | 30-90 sec (makeRules); ~200ms (checkIt) | N/A (no proofs) |
| **Verification latency** | Instant (offline) | ~12 sec (block confirmation) | ~200ms (API call) | Milliseconds (signature verification) |
| **End-to-end** | Offline | 15-20 seconds | ~200ms per check | Milliseconds |
| **Scales with** | Record size | Number of transfers | Number of policy rules | Number of constraints |
| **Policy creation cost** | N/A | N/A | $3.00 per policy | Free (user creates L2) |

**VI and ACTIS are cheapest** (no gas, no API fees — just cryptographic math). **ERC-8150 is most expensive** (on-chain gas). **ICME is pay-per-call** ($0.01/check).

---

## 8. Integration Model

### How Does Each Fit Into Existing Systems?

| | ACTIS | ERC-8150 | ICME | VI |
|---|---|---|---|---|
| **Integration point** | Post-transaction logging | Smart contract wallet | Pre-action API call | Payment authorization flow |
| **Requires new infra** | Bundle storage | Ethereum deployment + ZK circuit | API integration | SD-JWT support in wallets + networks |
| **Works with existing payments** | Yes (records any system) | No (on-chain ERC-20 only) | Yes (advisory for any system) | Designed for card networks |
| **Agent framework changes** | Producer SDK needed | Proof generation + submission | API call before each action | L3 creation + presentation |
| **Merchant changes** | None | None | None | Verify L3b credentials |
| **Network changes** | None | None | None | Verify L3a + constraint checking |

**ACTIS and ICME** are the easiest to adopt (no changes to merchants/networks). **VI requires the deepest integration** (merchants and networks must verify credentials). **ERC-8150 is isolated** to the Ethereum ecosystem.

---

## 9. What Each Can and Cannot Do

### Capability Matrix

| Capability | ACTIS | ERC-8150 | ICME | VI |
|------------|-------|----------|------|-----|
| Prove record wasn't tampered | **Yes** | No | No | No |
| Prevent unauthorized execution | No | **Yes** (on-chain revert) | Advisory | **Yes** (network rejects) |
| Verify policy compliance | No | No | **Yes** (SAT/UNSAT) | Structured constraints only |
| Prove agent was authorized | No | No | No | **Yes** (delegation chain) |
| Handle aggregation attacks | No | No | **Yes** (if policy includes it) | **Yes** (`payment.budget`) |
| Privacy per party | No | ZK (binary) | ZK (binary) | **Yes** (role-based SD-JWT) |
| Work offline | **Yes** | No (needs Ethereum) | No (needs API) | **Yes** (after credential issuance) |
| Dynamic policy updates | N/A | No (intent is static) | **Yes** (refinePolicy) | New L2 per delegation |
| Multi-transaction scope | N/A | No (per-batch) | **Yes** (aggregate rules) | **Yes** (budget, recurrence) |
| Cross-org verification | **Yes** (anyone verifies) | Smart contract only | **Yes** (ZK proof) | **Yes** (each party verifies) |
| Regulatory audit trail | **Yes** (core purpose) | On-chain events | Audit receipts | Full chain for disputes |

### Limitation Matrix

| Limitation | ACTIS | ERC-8150 | ICME | VI |
|------------|-------|----------|------|-----|
| No enforcement | **Yes** — evidence only | No | Partially — advisory | No |
| Blockchain required | No | **Yes** — Ethereum | No | No |
| Limited action types | No | **Yes** — ERC-20 only | No | **Yes** — payments only |
| Gas costs | No | **Yes** — 345k+ gas | No | No |
| API dependency | No | No | **Yes** — ICME API | No (after issuance) |
| LLM in critical path | No | No | **Yes** — variable extraction | No |
| Complex cryptography | No | **Yes** — ZK circuits | Moderate | No |
| Requires industry adoption | No | Ethereum ecosystem | No | **Yes** — card networks |

---

## 10. The Equivocation Problem

A key limitation that applies differently to each framework:

**Equivocation:** A bad actor produces two different valid records/credentials for the same transaction.

| | ACTIS | ERC-8150 | ICME | VI |
|---|---|---|---|---|
| **Vulnerable?** | Yes — producer could create two valid transcripts | No — nonce consumed on-chain (single source of truth) | N/A — stateless checks | Partially — network tracks nonce/L3 per mandate pair |
| **Mitigation** | External: transparency logs, bilateral counter-signing | Built-in: blockchain state | N/A | Built-in: network-enforced nonce + per-pair tracking |

ERC-8150 has the strongest anti-equivocation guarantee (blockchain is the single source of truth). VI delegates this to the payment network. ACTIS explicitly acknowledges this as an out-of-scope limitation.

---

## 11. Design Philosophy

| | ACTIS | ERC-8150 | ICME | VI |
|---|---|---|---|---|
| **Philosophy** | "Record everything, let verifiers judge" | "Zero trust — math enforces, not humans" | "Turn probabilistic into deterministic" | "Extend payment auth to agents" |
| **Inspired by** | Audit trails, legal evidence standards | Smart contract security, account abstraction | AWS Automated Reasoning (Cedar, Zelkova) | Card network authorization (EMV, 3DS) |
| **Complexity preference** | Minimal (SHA-256 + Ed25519) | Maximal (ZK-SNARKs + Solidity) | Moderate (SMT solvers + optional ZK) | Moderate (SD-JWT + selective disclosure) |
| **Adoption path** | Any agent framework can add it | Ethereum DeFi agents | API integration, pay-per-call | Card network + wallet + merchant adoption |
| **Maturity** | v1.0 with 3 implementations + conformance corpus | Draft ERC, reference Circom circuit | Live API, early product | Draft v0.1, Mastercard backing |

---

## 12. When to Use Which

### Decision Tree

```
Is this an on-chain (DeFi) agent?
├── Yes → Does it need to batch ERC-20 transfers?
│         ├── Yes → ERC-8150 (ZK-enforced exact-match)
│         └── No → Wait for ERC-8150 extension or use ICME for policy checks
│
└── No (off-chain / traditional commerce)
    │
    ├── Does the agent make purchases via card networks?
    │   ├── Yes → Verifiable Intent (native to payment infrastructure)
    │   └── No → Continue below
    │
    ├── Do you need formal policy verification?
    │   ├── Yes → ICME (natural language → formal logic → SAT/UNSAT)
    │   └── No → Continue below
    │
    ├── Do you need a tamper-evident audit trail?
    │   ├── Yes → ACTIS (hash-linked signed transcripts)
    │   └── No → Maybe you don't need an accountability framework yet
    │
    └── Do you need cross-org trustless verification?
        ├── Yes → ICME (ZK proof of policy compliance)
        │         or VI (selective disclosure credential chain)
        └── No → ACTIS for evidence, ICME for guardrails
```

### Use Case → Framework Mapping

| Use Case | Best Framework | Why |
|----------|---------------|-----|
| **DeFi portfolio rebalancing** | ERC-8150 | On-chain enforcement, atomic batch execution |
| **Agent shopping on Amazon/Shopify** | VI | Payment network authorization, merchant+network verification |
| **Treasury agent with spending rules** | ICME | Formal policy (amount limits, aggregate budgets, allowlists) |
| **Multi-agent negotiation disputes** | ACTIS | Tamper-evident record of who said what |
| **Regulatory audit (EU AI Act)** | ACTIS | Offline-verifiable evidence bundles |
| **Cross-org agent commerce** | VI + ICME | VI for authorization chain, ICME for policy compliance |
| **Subscription renewals** | VI | `payment.recurrence` + `payment.budget` constraints |
| **Batch payroll** | ERC-8150 | Atomic multi-transfer with ZK-verified intent match |
| **Agent guardrails (any domain)** | ICME | Most flexible — any natural language policy |
| **Chargeback dispute defense** | VI | Cryptographic proof of user authorization |

### Combining Frameworks

The four frameworks are complementary, not competing:

```
FULL STACK (maximum accountability):

1. VI authorizes the agent (delegation chain from user)
   → "Is this agent allowed to act on my behalf within these constraints?"

2. ICME verifies policy compliance (formal logic check)
   → "Does this specific action comply with our organizational policy?"

3. ERC-8150 enforces on-chain execution (ZK proof)
   → "Do these blockchain transactions match exactly what was approved?"

4. ACTIS records everything (tamper-evident evidence)
   → "Here's the cryptographic record of the entire transaction lifecycle"
```

In practice, most deployments will use 1-2 of these based on their specific needs, not all four.

---

## Summary Table

| Dimension | ACTIS | ERC-8150 | ICME | VI |
|-----------|-------|----------|------|-----|
| **Type** | Evidence standard | On-chain enforcement | Policy verification API | Authorization credential |
| **When** | Post-execution | Pre-execution | Pre-execution | Pre-execution |
| **Crypto** | SHA-256 + Ed25519 | ZK-SNARKs + ECDSA | SMT solvers + optional zkML | ES256 + SD-JWT |
| **Blockchain** | No | Ethereum required | No | No |
| **Enforcement** | None | Smart contract revert | Advisory | Network/merchant reject |
| **Privacy** | Transparent | ZK (binary) | ZK (optional) | Selective disclosure per role |
| **Scope** | Any transaction type | ERC-20 transfers | Any formalizable policy | Payment commerce |
| **Constraints** | N/A | Exact amounts/recipients | Any NL→formal logic | 8 structured types |
| **Cost** | Free | ~$1-3 gas | $0.01/check | Free |
| **Latency** | Offline | 15-20 sec | ~200ms | Milliseconds |
| **Backing** | Open source | ERC community | Startup | Mastercard |
| **Maturity** | v1.0 + conformance corpus | Draft ERC | Live API | Draft v0.1 |
| **Best for** | Audit trails, disputes | DeFi batch enforcement | Flexible policy guardrails | Card-network agent auth |
