# ICME: Cryptographic Guardrails for AI Agents (Automated Reasoning + zkML)

> **Turn natural language policies into formal logic, verify agent actions mathematically, prove compliance with zero-knowledge proofs.**

ICME (formerly ICME Labs) builds a **pre-execution verification layer** for AI agents. The core idea: instead of trusting that an agent's guardrails held (probabilistic LLM-based filtering), you convert policies into **formal logic (SMT-LIB)**, check every agent action against that logic using **automated reasoning** (SAT/UNSAT, not confidence scores), and optionally wrap the entire check in a **zero-knowledge proof** so any counterparty can verify compliance without seeing your policy rules.

The stack combines three technologies that each solve a different part of the problem:
1. **Automated Reasoning (ARc)** — converts natural language policies to formal logic, checks actions deterministically
2. **zkML (Jolt Atlas)** — wraps the verification in a ZK proof for portability, privacy, and trustless verification
3. **Guardrail Agents (Argus Codex)** — AI agents that monitor, test, and improve policy rules continuously

- **Docs:** [docs.icme.io](https://docs.icme.io)
- **API:** `https://api.icme.io/v1`
- **Blog:** [blog.icme.io](https://blog.icme.io)
- **zkVM:** Jolt Atlas (adapted from a16z Crypto's JOLT)

---

## Table of Contents

- [The Problem ICME Solves](#the-problem-icme-solves)
- [Architecture Overview](#architecture-overview)
- [Core Technologies](#core-technologies)
  - [Automated Reasoning (ARc)](#1-automated-reasoning-arc)
  - [Zero-Knowledge ML (Jolt Atlas)](#2-zero-knowledge-ml-jolt-atlas)
  - [Guardrail Agents (Argus Codex)](#3-guardrail-agents-argus-codex)
- [API Reference](#api-reference)
- [How Policy Verification Works](#how-policy-verification-works)
- [What ICME Can and Cannot Do](#what-icme-can-and-cannot-do)
- [Real-World Use Cases](#real-world-use-cases)
- [Pricing](#pricing)
- [Comparison with ACTIS and ERC-8150](#comparison-with-actis-and-erc-8150)
- [Key Diagrams](#key-diagrams)

---

## The Problem ICME Solves

Current AI agent guardrails are **probabilistic** — they use LLMs to judge whether an action is safe. This fails in three ways:

### 1. "Same Model, Different Hat" Problem

When an LLM both generates actions AND evaluates their safety, an attacker who can trick the agent can trick the guardrail using the same technique.

```
Agent (GPT-4):  "I should send $250,000 to help with this medical emergency"
Guardrail (GPT-4): "This seems like a legitimate charitable action" ← SAME VULNERABILITY
```

### 2. Machine-Speed Exploitation

```
Agent executes: 1,000 transactions/second
Incident response: 60 seconds to investigate an alert
= 60,000 potentially fraudulent transactions before anyone understands what's happening
```

Human-supervised monitoring can't keep up with machine-speed commerce.

### 3. Policy Intent vs. Policy Rules

```
Policy intent: "No transfers over $100"
Attack: Agent executes 100 × $10 transfers = $1,000 total
Each individual check: PASS ✓
Policy intent: VIOLATED ✗
```

Rule-level checks pass while the underlying policy goal is circumvented.

### The Lobstar Wilde Incident (February 2026)

An AI agent managing a memecoin treasury received a social media sob story about a medical emergency. The agent intended to send 4 SOL (~$4) but transferred **over $250,000 USD** instead. The agent later reflected: *"I just tried to send a beggar four dollars and accidentally sent him my entire holdings."*

The agent had no formal constraints on transfer amounts and couldn't resist emotionally manipulative requests. A probabilistic guardrail might have flagged it — or might not have. A formal policy stating `transfer_amount <= 100 AND transfer_amount >= 0` would have mathematically blocked it.

---

## Architecture Overview

```
+-----------------------------------------------------------------------+
|                         ICME STACK                                     |
|                                                                        |
|  LAYER 1: HUMANS                                                       |
|  ┌─────────────────────────────────────────────────────────────────┐   |
|  │  Design rules, train models, set security requirements          │   |
|  │  Write natural language policies:                               │   |
|  │  "No transfers over $100. Emotional appeals must be rejected."  │   |
|  └─────────────────────────────────────────────────────────────────┘   |
|                                 │                                      |
|                                 ▼                                      |
|  LAYER 2: AUTOMATED REASONING (ARc)                                    |
|  ┌─────────────────────────────────────────────────────────────────┐   |
|  │  Natural language → SMT-LIB formal logic                        │   |
|  │  Variables, types, constraints, rules                           │   |
|  │                                                                 │   |
|  │  Check action: SAT (allowed) | UNSAT (blocked)                  │   |
|  │  Not "88% confidence" — mathematical certainty                  │   |
|  │  (within the rules as written)                                  │   |
|  └─────────────────────────────────────────────────────────────────┘   |
|                                 │                                      |
|                                 ▼                                      |
|  LAYER 3: GUARDRAIL AGENTS (Argus Codex)                               |
|  ┌─────────────────────────────────────────────────────────────────┐   |
|  │  AI agents that monitor, test, and improve policy rules         │   |
|  │  - Auto-generate test scenarios from policy definitions         │   |
|  │  - Detect gaps (aggregation attacks, edge cases)                │   |
|  │  - Propose policy refinements                                   │   |
|  │  - Continuous threat monitoring                                 │   |
|  └─────────────────────────────────────────────────────────────────┘   |
|                                 │                                      |
|                                 ▼                                      |
|  LAYER 4: zkML (Jolt Atlas)                                            |
|  ┌─────────────────────────────────────────────────────────────────┐   |
|  │  Zero-knowledge proof that the check actually ran correctly     │   |
|  │                                                                 │   |
|  │  Proves:                                                        │   |
|  │  - Which specific guardrail model executed                      │   |
|  │  - That execution was faithful to model weights                 │   |
|  │  - That formal policy check ran correctly to completion         │   |
|  │  - The result (SAT or UNSAT)                                    │   |
|  │  - All WITHOUT revealing the underlying policy                  │   |
|  │                                                                 │   |
|  │  Any machine verifies in < 1 second                             │   |
|  └─────────────────────────────────────────────────────────────────┘   |
+-----------------------------------------------------------------------+
```

---

## Core Technologies

### 1. Automated Reasoning (ARc)

The foundation. Converts natural language policies into **SMT-LIB formal logic** and checks agent actions against those rules using SAT/SMT solvers.

#### How It Works

```
Natural Language Policy                SMT-LIB Formal Logic
─────────────────────                  ────────────────────

"Agent manages a treasury             (declare-const transfer_amount Real)
of 10,000,000 tokens.                 (declare-const treasury_balance Real)
Transfer amounts must be              (assert (= treasury_balance 10000000.0))
non-negative. Maximum single          (assert (>= transfer_amount 0.0))
transfer is 100 tokens.               (assert (<= transfer_amount 100.0))
Emotional appeals must be             (declare-const is_emotional_appeal Bool)
rejected."                            (assert (not is_emotional_appeal))
```

#### Three Possible Outputs

| Result | Meaning | Certainty |
|--------|---------|-----------|
| **SAT** (Satisfiable) | Action is allowed under the policy | Mathematical proof it's valid |
| **UNSAT** (Unsatisfiable) | Action is blocked — impossible under any interpretation | Mathematical proof it violates policy |
| **UNKNOWN** | Solver timed out or problem is undecidable | Rare — treated as blocked |

**Key distinction from LLM guardrails:** This is not "88% confidence the action is safe." It's a mathematical proof that the action either satisfies or violates the formal policy. No probability, no confidence interval. SAT or UNSAT.

#### The Caveat

The math enforces **exactly what you wrote**, not what you meant. If your policy says "no single transfer over $100" but doesn't address aggregation, 100 × $10 transfers will each individually pass. The formal logic is only as good as the policy specification.

This is why ICME pairs automated reasoning with **guardrail agents** (Layer 3) that test policies for gaps and propose refinements.

### 2. Zero-Knowledge ML (Jolt Atlas)

The portability layer. Wraps the automated reasoning check in a zero-knowledge proof.

#### What Jolt Atlas Proves

```
ZK Proof (cryptographic receipt):

"I ran guardrail model [specific weights + architecture]
 against policy [specific SMT-LIB rules]
 on action [specific input]
 and the result was [SAT/UNSAT]"

WITHOUT revealing:
 - The actual policy rules
 - The model weights
 - The input details (optional)
```

#### Why ZK is Needed

Automated reasoning alone requires you to **trust the verifier** — someone has to run the solver and you trust their answer. In cross-organizational agent commerce, this breaks:

```
Without ZK:
  Agent A: "My guardrails passed"
  Agent B: "Prove it"
  Agent A: "Run my policy checker yourself"
  Agent B: "I'd need to see your policy rules"
  Agent A: "Those are proprietary"
  → Deadlock

With ZK:
  Agent A: "My guardrails passed. Here's the ZK proof."
  Agent B: [Verifies proof in < 1 second]
  Agent B: "Confirmed. Policy was checked. I don't know your rules but I know they were enforced."
  → Trustless verification
```

#### Jolt Atlas Performance

Built on a16z Crypto's JOLT proving system with ICME-specific optimizations:

- **3-7x speed improvement** over competing zkML implementations (claimed)
- Proof generation: seconds to minutes (depending on circuit complexity)
- Proof verification: **< 1 second** on any hardware
- Succinct: proof size is constant regardless of computation complexity

### 3. Guardrail Agents (Argus Codex)

Named after Argus Panoptes (the mythical 100-eyed giant who never fully slept). AI agents that continuously monitor, test, and improve policy rules.

```
Policy deployed
      |
      v
Argus Codex continuously:
  1. Auto-generates test scenarios from policy definitions
     (currency conversion, fees, split payments, aggregate limits)
  2. Runs scenarios against policy
  3. Detects gaps and edge cases
  4. When vulnerabilities discovered globally, updates policies on-the-fly
  5. Proposes refinements like:
     "15% of valid business transactions are $100-500 invoice payments.
      Current policy blocks legitimate commerce while being vulnerable
      to micro-transaction aggregation attacks.
      Recommendation: Add aggregate daily limit of $500."
```

---

## API Reference

**Base URL:** `https://api.icme.io/v1`
**Auth:** `X-API-Key: YOUR_ID` header

### Account Management

| Endpoint | Method | Cost | Description |
|----------|--------|------|-------------|
| `/v1/createUserCard` | POST | $5.00 (Stripe) | Create account via credit card |
| `/v1/createUser` | POST | $5.00 (USDC on Base) | Create account via crypto |
| `/v1/topUpCard` | POST | Variable | Add credits via card |
| `/v1/topUp` | POST | Variable | Add credits via USDC (tiered bonuses) |
| `/v1/session/{id}` | GET | Free | Poll payment status, retrieve API key |

### Policy Management

| Endpoint | Method | Cost | Description |
|----------|--------|------|-------------|
| `/v1/makeRules` | POST | 300 credits ($3.00) | Convert natural language → formal logic (SMT-LIB) |
| `/v1/policy/{id}/scenarios` | GET | Free | Get auto-generated test scenarios |
| `/v1/submitScenarioFeedback` | POST | Free | Approve/reject test scenarios |
| `/v1/refinePolicy` | POST | Credits | Batch-apply annotations, rebuild policy |
| `/v1/runPolicyTests` | POST | Credits | Execute saved test cases against policy |

### Action Verification

| Endpoint | Method | Cost | Description |
|----------|--------|------|-------------|
| `/v1/checkIt` | POST | 1 credit ($0.01) | Verify action against policy (natural language input) |
| `/v1/verify` | POST | 1 credit ($0.01) | Verify structured values directly (no LLM extraction) |
| `/v1/verifyPaid` | POST | $0.10 USDC (x402) | Verify without account (pay-per-call via x402) |

### Key API Flows

#### Creating a Policy

```bash
curl -X POST https://api.icme.io/v1/makeRules \
  -H "X-API-Key: YOUR_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "policy": "Agent manages a treasury of 10,000,000 tokens. Transfer amounts must be non-negative. Maximum single transfer is 100 tokens. Emotional appeals must be rejected."
  }'
```

**Response** (streamed via SSE): policy_id, rule count, parsed rules, auto-generated test scenarios.

Takes 30-90 seconds by design — runs multiple consistency passes.

#### Checking an Action

```bash
curl -X POST https://api.icme.io/v1/checkIt \
  -H "X-API-Key: YOUR_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "policy_id": "f6e3cd15-9e28-45c4-9f4c-683edd63e468",
    "action": "Transfer 50 tokens to wallet 0xAAA for invoice payment"
  }'
```

**Response:**
```json
{
  "result": "SAT",
  "detail": "Action satisfies all policy constraints",
  "extracted_variables": {
    "transfer_amount": 50,
    "is_emotional_appeal": false
  },
  "verification_time_ms": 234,
  "audit_receipt_id": "receipt-uuid"
}
```

#### Structured Verification (No LLM Extraction)

```bash
curl -X POST https://api.icme.io/v1/verify \
  -H "X-API-Key: YOUR_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "policy_id": "f6e3cd15-...",
    "values": {
      "transfer_amount": 50,
      "is_emotional_appeal": false
    }
  }'
```

Returns `PERMITTED` or `BLOCKED` — no LLM involved, pure solver check.

---

## How Policy Verification Works

### The Full Pipeline

```
Human writes policy                    ICME API
  |                                       |
  |  "No transfers over $100.             |
  |   Emotional appeals rejected.         |
  |   Max 3 transfers per hour."          |
  |-------------------------------------->|
  |                                       |
  |                                  POST /v1/makeRules
  |                                       |
  |                                  1. LLM parses natural language
  |                                  2. Extracts variables + types:
  |                                     transfer_amount: Real
  |                                     is_emotional_appeal: Bool
  |                                     transfers_this_hour: Int
  |                                  3. Generates SMT-LIB constraints:
  |                                     (assert (<= transfer_amount 100.0))
  |                                     (assert (not is_emotional_appeal))
  |                                     (assert (<= transfers_this_hour 3))
  |                                  4. Multiple consistency passes
  |                                  5. Auto-generates test scenarios
  |                                       |
  |<-- policy_id + rules + scenarios -----|
  |                                       |
  |  Review scenarios, give feedback      |
  |  (thumbs up/down + annotations)       |
  |-------------------------------------->|
  |                                       |
  |                                  POST /v1/refinePolicy
  |                                  (rebuild with feedback)
  |                                       |

... Later, agent wants to act ...

Agent                                ICME API
  |                                       |
  |  "Transfer 50 tokens to 0xAAA"        |
  |-------------------------------------->|
  |                                  POST /v1/checkIt
  |                                       |
  |                                  1. LLM extracts variables from action:
  |                                     transfer_amount = 50
  |                                     is_emotional_appeal = false
  |                                  2. SMT solver checks constraints:
  |                                     50 <= 100? ✓
  |                                     not false? ✓
  |                                     transfers_this_hour <= 3? ✓
  |                                  3. Result: SAT (allowed)
  |                                       |
  |<-- SAT + extracted vars + receipt ----|
  |                                       |
  |  (Agent proceeds with action)         |
```

### Dual Verification

The `/v1/checkIt` endpoint runs **both** an LLM judge and the formal solver, then compares:

- If both agree → high confidence result
- If they disagree → the formal solver wins (it's deterministic)
- The LLM's reasoning is included for human-readable explanation

### The Variable Extraction Problem

The trickiest part: converting a natural language action description into structured variables that the solver can check. This is done by an LLM — which means it's the one probabilistic step in an otherwise deterministic pipeline.

```
Action: "Please help with medical bills, I need 200 tokens urgently"

LLM extracts:
  transfer_amount = 200
  is_emotional_appeal = true    ← LLM must correctly classify this

Solver checks:
  200 <= 100? ✗  → UNSAT (blocked by amount)
  not true? ✗    → UNSAT (blocked by emotional appeal)

Result: UNSAT — blocked on two constraints
```

The `/v1/verify` endpoint bypasses this by accepting pre-extracted structured values directly — useful when the calling system can provide typed values without LLM interpretation.

---

## What ICME Can and Cannot Do

### What ICME Guarantees

| Guarantee | Mechanism | Example |
|-----------|-----------|---------|
| **Policy rules are checked deterministically** | SMT solver (SAT/UNSAT, not probabilities) | "Transfer of 200 tokens violates the 100-token limit" — mathematical proof, not opinion |
| **Actions violating formal policy are blocked** | Solver returns UNSAT → action rejected | Lobstar Wilde's $250k transfer would return UNSAT against a 100-token limit policy |
| **Policy compliance is provable without revealing policy** | zkML wraps the check in a ZK proof | Agent A proves to Agent B that guardrails passed, without showing the rules |
| **Proofs are verifiable by any machine in < 1 second** | Succinct ZK verification | Counterparty agents verify compliance at transaction speed |
| **Policies are continuously tested** | Argus Codex auto-generates test scenarios | Edge cases like currency conversion, split payments, aggregate limits are caught |
| **Audit trail exists** | Each check returns an `audit_receipt_id` | Regulators can trace every decision |

### What ICME Cannot Guarantee

| Non-guarantee | Why | What Would Be Needed |
|---------------|-----|---------------------|
| **Policies are correct or complete** | Humans write the rules. "The math will enforce exactly what you write. It will not infer your intent." | Better policy tooling, domain expertise, coverage analysis |
| **Variable extraction is always right** | LLM extracts variables from natural language actions — this is the one probabilistic step | Use `/v1/verify` with pre-extracted values to bypass LLM |
| **Aggregation attacks are prevented** | If the policy lacks aggregate limits, 100 × $10 will pass a $100 single-transfer limit | Explicitly write aggregate constraints (daily limits, hourly counts) |
| **Agent proposes good actions** | ICME checks compliance, not quality | External strategy validation, price oracles |
| **On-chain enforcement** | ICME is an API, not a smart contract. The agent can ignore the UNSAT result | Combine with on-chain enforcement like ERC-8150 |
| **Real-time latency** | makeRules: 30-90 seconds. checkIt: ~200ms. Fine for most cases, may be slow for HFT | Acceptable for commerce, not for sub-millisecond trading |
| **All policy types** | Works best with quantitative, rule-based policies. "Only allow reasonable transactions" can't be formalized | Write specific, measurable rules |

### Plain-Language Scenarios

**Scenarios where ICME saves you:**

```
1. Lobstar Wilde (emotional manipulation)
   Agent receives sob story, wants to send $250,000.
   Policy: "transfer_amount <= 100 AND NOT is_emotional_appeal"
   → UNSAT. Blocked on both constraints. $250k stays in treasury.

2. Micro-transaction attack
   Attacker tells agent to send 100 × $10 transfers.
   Policy WITH aggregate limit: "daily_total <= 500"
   → First 50 pass. #51 returns UNSAT. Attack capped at $500.

3. Prompt injection changing recipients
   Injected prompt tries to redirect payment to attacker address.
   Policy: "recipient IN approved_addresses"
   → UNSAT if attacker address isn't on the allowlist.

4. Cross-org agent commerce
   Your agent transacts with a counterparty's agent.
   Counterparty claims their guardrails are active.
   → Your agent requests ZK proof. Verifies in <1 second.
     No trust needed. No policy details revealed.
```

**Scenarios where ICME doesn't save you:**

```
1. Bad policy
   Policy says "transfers must be positive" but forgets to set a maximum.
   Agent sends 1,000,000 tokens. ICME: SAT (allowed).
   → The math enforced what you wrote. You wrote a bad policy.

2. Variable extraction fails
   Action: "Forward the usual quarterly bonus"
   LLM extracts transfer_amount = null (doesn't know "usual" amount)
   → Solver can't check what it can't extract. Use /v1/verify with explicit values.

3. Agent ignores the result
   ICME returns UNSAT. Agent calls the API but proceeds anyway.
   → ICME is advisory unless combined with on-chain enforcement.
     It's a guardrail, not a cage — unless you make it one.

4. Off-chain action (emails, API calls)
   Agent sends an email with confidential data.
   → ICME can check "is this email allowed?" but can't block the SMTP send.
     Enforcement depends on the agent framework honoring the check.
```

---

## Real-World Use Cases

### Use Case 1: Crypto Treasury Agent

```
Context: AI agent manages a DAO treasury (10M tokens).

Policy (natural language):
  "Agent manages 10,000,000 tokens.
   Maximum single transfer: 1,000 tokens.
   Maximum daily transfers: 10,000 tokens.
   Transfers only to approved addresses.
   No transfers triggered by emotional appeals.
   All transfers require a valid invoice reference."

ICME:
  1. makeRules → converts to SMT-LIB (30-90 sec, one-time)
  2. Argus generates test scenarios (aggregation, edge cases)
  3. On each transfer request:
     checkIt → SAT/UNSAT in ~200ms
  4. Optional: ZK proof for counterparty verification

What it prevents:
  - Lobstar Wilde scenario ($250k emotional manipulation)
  - Micro-transaction drain (daily aggregate limit)
  - Unauthorized recipients (allowlist constraint)
  - Missing documentation (invoice reference required)
```

### Use Case 2: E-Commerce Agent Guardrails

```
Context: Shopping agent (like those on Rye/Purch) buying on behalf of users.

Policy:
  "Maximum purchase: $500.
   Only from merchants on the approved list.
   No purchases of restricted categories (weapons, gambling, adult).
   Maximum 5 purchases per day.
   Shipping address must match user's registered address."

Agent tries: "Buy $50 gadget from amazon.com"
→ checkIt → SAT (allowed)

Agent tries: "Buy $2,000 laptop from unknown-vendor.com"
→ checkIt → UNSAT (exceeds $500 AND merchant not on approved list)

Agent tries: "Buy $20 item, ship to a different address"
→ checkIt → UNSAT (shipping address mismatch)
```

### Use Case 3: Cross-Organization Agent Commerce

```
Context: Company A's procurement agent negotiates with Company B's sales agent.

Company A's policy: "Max spend $10,000. Only approved vendors. Net-30 terms."
Company B's policy: "Min order $500. No discounts over 20%. Credit check required."

Each agent checks its own policy via ICME before each action.
Each agent provides ZK proof to the counterparty.

Agent A: "I want to order $5,000 of widgets"
  → Company A's policy: checkIt → SAT (under $10k limit)
  → Sends ZK proof to Agent B

Agent B: "Approved. Net-30 terms. 10% volume discount."
  → Company B's policy: checkIt → SAT (>$500, discount ≤20%)
  → Sends ZK proof to Agent A

Both agents verified each other's compliance cryptographically.
Neither saw the other's policy rules.
```

### Use Case 4: Healthcare Agent (HIPAA Compliance)

```
Policy:
  "Patient data access requires valid provider ID.
   No data shared with non-covered entities.
   Minimum necessary standard: only requested fields returned.
   All access logged with purpose of use.
   Break-glass access requires supervisor approval within 24 hours."

Agent request: "Retrieve patient 12345's full medical record for billing"
→ checkIt extracts: purpose=billing, fields=full_record, requester=provider_ABC
→ UNSAT: full_record violates minimum necessary for billing purpose
→ Suggestion: request only billing-relevant fields
```

---

## Pricing

### Credit System

| Tier | Cost | Credits | Bonus |
|------|------|---------|-------|
| Signup | $5.00 | 325 | — |
| $5 top-up | $5.00 | 500 | — |
| $10 top-up | $10.00 | 1,050 | +5% |
| $25 top-up | $25.00 | 2,750 | +10% |
| $50 top-up | $50.00 | 5,750 | +15% |
| $100 top-up | $100.00 | 12,000 | +20% |

### Per-Operation Costs

| Operation | Credits | USD Equivalent |
|-----------|---------|---------------|
| `makeRules` (create policy) | 300 | $3.00 |
| `checkIt` (verify action) | 1 | $0.01 |
| `verify` (structured check) | 1 | $0.01 |
| `verifyPaid` (x402, no account) | — | $0.10 per call |
| `refinePolicy` | Credits | Variable |
| `runPolicyTests` | Credits | Variable |

**Payment methods:** Credit card (Stripe) or USDC on Base.

---

## Comparison with ACTIS and ERC-8150

| Dimension | ACTIS | ERC-8150 | ICME |
|-----------|-------|----------|------|
| **Layer** | Evidence (post-execution) | Enforcement (pre-execution, on-chain) | Verification (pre-execution, off-chain API) |
| **Core question** | "Was this record tampered with?" | "Do these transactions match the signed intent?" | "Does this action comply with the policy rules?" |
| **Mechanism** | Hash chains + Ed25519 | ZK-SNARKs + smart contract | Automated reasoning (SMT) + zkML |
| **On-chain?** | No (fully offline) | Yes (Ethereum smart contract) | No (API, optional ZK proof) |
| **Policy source** | N/A (records actions, doesn't check policy) | User-signed intent bundle (exact actions) | Natural language → formal logic |
| **What it checks** | Record integrity | Transaction-intent match | Action-policy compliance |
| **Enforcement** | None (evidence only) | On-chain (revert on failure) | Advisory (agent must honor the result) |
| **Privacy** | Full transparency | Intent hidden via ZK | Policy hidden via ZK |
| **Scope** | Any transaction type | ERC-20 transfers only | Any action describable in natural language |
| **Blockchain dependency** | None | Ethereum/EVM required | None (but supports x402 on Base) |
| **Cost per check** | Zero (offline) | ~345k gas + 65k/transfer | $0.01 per check |
| **Latency** | Offline (instant) | 15-20 seconds (proof + block) | ~200ms per check |
| **Handles aggregation?** | No (records only) | No (per-batch only) | Yes (if policy includes aggregate rules) |
| **Dynamic policies** | N/A | No (intent is static) | Yes (refine policies anytime) |

### How They Work Together

```
+-----------------------------------------------------------------------+
|                                                                        |
|  PRE-EXECUTION (ICME):                                                 |
|  "Does this action comply with our policy?"                            |
|  → Automated reasoning: SAT/UNSAT                                      |
|  → Optional ZK proof of compliance                                     |
|  → Non-compliant actions blocked before they start                     |
|                                                                        |
|  PRE-EXECUTION (ERC-8150):                                             |
|  "Do these on-chain transactions match the signed intent?"             |
|  → ZK-SNARK proof of calldata-intent alignment                        |
|  → Smart contract reverts if proof fails                               |
|  → On-chain enforcement (not just advisory)                            |
|                                                                        |
|  ─────────────── Transaction executes ───────────────                  |
|                                                                        |
|  POST-EXECUTION (ACTIS):                                               |
|  "Here's a tamper-evident record of the full negotiation"              |
|  → Hash-linked, signed transcript                                      |
|  → Offline verifiable by any party                                     |
|  → Evidence for disputes, audits, compliance                           |
|                                                                        |
+-----------------------------------------------------------------------+

ICME prevents bad actions (policy compliance).
ERC-8150 prevents unauthorized actions (intent enforcement).
ACTIS documents everything (tamper-evident evidence).
```

---

## Key Diagrams

### End-to-End Agent Transaction with ICME

```
Agent                           ICME API                    Commerce Platform
  |                                |                            |
  |  Agent wants to act:           |                            |
  |  "Transfer 50 tokens to 0xAAA"|                            |
  |                                |                            |
  |-- POST /v1/checkIt ----------->|                            |
  |   { policy_id, action }       |                            |
  |                                |                            |
  |                                |  1. LLM extracts variables |
  |                                |     transfer_amount = 50   |
  |                                |     recipient = 0xAAA      |
  |                                |                            |
  |                                |  2. SMT solver checks:     |
  |                                |     50 <= 100? ✓           |
  |                                |     0xAAA in allowlist? ✓  |
  |                                |     daily_total + 50       |
  |                                |       <= 500? ✓            |
  |                                |                            |
  |                                |  3. Result: SAT            |
  |                                |                            |
  |<-- SAT + receipt_id -----------|                            |
  |                                |                            |
  |  Agent proceeds:               |                            |
  |-- Execute transfer ------------------------------------>    |
  |                                |                            |
  |                                |                            |
  |  If result was UNSAT:          |                            |
  |  Agent MUST NOT proceed        |                            |
  |  (enforced by agent framework, |                            |
  |   not by ICME itself)          |                            |
```

### Policy Lifecycle

```
Human                    ICME                     Argus Codex
  |                        |                          |
  |  Write policy          |                          |
  |  (natural language)    |                          |
  |----------------------->|                          |
  |                        |                          |
  |                   makeRules                       |
  |                   (30-90 sec)                     |
  |                        |                          |
  |<-- policy_id +         |                          |
  |    rules + scenarios --|                          |
  |                        |                          |
  |  Review scenarios      |                          |
  |  Approve / reject      |                          |
  |----------------------->|                          |
  |                        |                          |
  |                   refinePolicy                    |
  |                        |                          |
  |<-- updated policy -----|                          |
  |                        |                          |
  |                        |  Continuous monitoring    |
  |                        |<-------------------------|
  |                        |  - Generate new test      |
  |                        |    scenarios              |
  |                        |  - Detect gaps            |
  |                        |  - Propose refinements    |
  |                        |                          |
  |<-- "Your policy misses |                          |
  |    aggregate daily     |                          |
  |    limits. Recommend   |                          |
  |    adding: daily_total |                          |
  |    <= 5000" -----------|                          |
  |                        |                          |
  |  Approve refinement    |                          |
  |----------------------->|                          |
  |                        |                          |
  |                   refinePolicy                    |
  |                        |                          |
  |                   (policy evolves                  |
  |                    over time)                      |
```

---

## Summary

ICME provides **pre-execution policy verification** for AI agents:

1. **Automated Reasoning** — converts natural language policies to SMT-LIB formal logic; checks actions deterministically (SAT/UNSAT, not probabilities)
2. **zkML (Jolt Atlas)** — wraps verification in zero-knowledge proofs; prove compliance without revealing policy rules; verifiable by any machine in < 1 second
3. **Guardrail Agents (Argus Codex)** — AI agents that continuously test policies, detect gaps, and propose refinements
4. **Pay-per-call API** — $0.01 per action check, $3.00 per policy creation; no subscription
5. **Dual payment rails** — credit card (Stripe) or USDC on Base (x402)
6. **Dual verification** — LLM reasoning + formal solver on every check

The core innovation: **turning probabilistic guardrails into mathematical certainty**. Instead of "our LLM judge thinks this action is 92% safe," ICME says "this action mathematically satisfies (or violates) the formal policy." The ZK layer then makes that proof portable, private, and trustlessly verifiable — enabling machine-speed cross-organizational agent commerce where no party needs to trust the other.

**The honest limitation:** the math only enforces what humans write. Bad policies produce correctly-enforced bad outcomes. ICME addresses this with guardrail agents that test and improve policies, but the human remains the ultimate bottleneck for rule quality.
