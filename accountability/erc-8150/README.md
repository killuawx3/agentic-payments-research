# ERC-8150: Zero-Knowledge Agent Payment Verification

> **Pre-execution verification of agent-mediated payments through zero-knowledge proofs — agents can only spend what users cryptographically pre-approve.**

ERC-8150 is an Ethereum standard (draft) that uses **ZK-SNARKs** to solve the agent trust dilemma: users either trust agents with their funds (risky) or approve every transaction manually (defeats the purpose). ERC-8150 introduces a third option — users sign an **intent bundle** describing exactly what the agent may do, and the agent must produce a **zero-knowledge proof** that its transactions match the signed intent before any funds move.

- **EIP:** [ERC-8150](https://github.com/jqhc/ERCs/blob/master/ERCS/erc-8150.md)
- **Status:** Draft
- **Authors:** Justin Cheng (@jqhc), Nathan Liang (@vincentcaptain)
- **Requires:** EIP-712
- **Discussion:** [ethereum-magicians.org](https://ethereum-magicians.org/t/erc-8150-zero-knowledge-agent-payment-verification/27665)

---

## Table of Contents

- [The Problem](#the-problem)
- [How ERC-8150 Solves It](#how-erc-8150-solves-it)
- [Architecture Overview](#architecture-overview)
- [Core Components](#core-components)
  - [Intent Bundles](#1-intent-bundles)
  - [ZK Circuit](#2-zk-circuit)
  - [Agent Wallet Contract](#3-agent-wallet-contract)
- [End-to-End Flow](#end-to-end-flow)
- [What It Can and Cannot Do](#what-it-can-and-cannot-do)
- [Security Model](#security-model)
- [Performance](#performance)
- [Comparison with ACTIS](#comparison-with-actis)
- [Comparison with Existing Agent Wallet Approaches](#comparison-with-existing-agent-wallet-approaches)
- [Real-World Use Cases](#real-world-use-cases)

---

## The Problem

When you let an AI agent manage your on-chain finances, you face a fundamental tradeoff:

```
Option A: Trust the agent fully
  → Deposit funds into agent-controlled wallet
  → Agent can do anything — including drain your funds
  → Risk: theft, hallucination, compromise

Option B: Approve everything manually
  → Agent proposes, you sign each transaction
  → You're the bottleneck
  → Defeats the purpose of automation

Option C (ERC-8150): Pre-approve the EXACT actions
  → Sign an intent bundle: "agent may transfer X USDC to Y, Z USDC to W"
  → Agent MUST prove (via ZK) its transactions match your intent
  → Funds only move after proof verification
  → Non-custodial — funds never leave your wallet until execution
```

**The key insight:** instead of trusting the agent OR approving each action, you approve a **batch of specific actions** upfront, and the smart contract **mathematically enforces** that the agent can only execute exactly those actions.

---

## How ERC-8150 Solves It

```
USER                              AGENT                         ON-CHAIN
  |                                  |                              |
  |  1. Agent proposes intent bundle |                              |
  |  "I want to send:               |                              |
  |   - 10 USDC to 0xAAA            |                              |
  |   - 150 USDC to 0xBBB           |                              |
  |   - 35 USDC to 0xCCC"           |                              |
  |<---------------------------------|                              |
  |                                  |                              |
  |  2. User reviews exact amounts,  |                              |
  |     recipients, tokens           |                              |
  |                                  |                              |
  |  3. User signs commitment hash   |                              |
  |     (EIP-712 typed data)         |                              |
  |--------------------------------->|                              |
  |                                  |                              |
  |  4. User approves Agent Wallet   |                              |
  |     for token spending           |                              |
  |     (standard ERC-20 approve)    |                              |
  |----------------------------------------------------->          |
  |                                  |                              |
  |                                  |  5. Agent generates          |
  |                                  |     ZK-SNARK proof           |
  |                                  |     (off-chain, 2-5 sec)     |
  |                                  |                              |
  |                                  |  6. Agent submits to chain:  |
  |                                  |     proof + signature +      |
  |                                  |     public inputs + calls    |
  |                                  |----------------------------->|
  |                                  |                              |
  |                                  |           7. Contract verifies:
  |                                  |              - ZK proof valid?
  |                                  |              - Signature valid?
  |                                  |              - Nonce fresh?
  |                                  |              - Not expired?
  |                                  |              - Calldata hash matches?
  |                                  |                              |
  |                                  |           8. ALL checks pass →
  |                                  |              Execute transfers
  |                                  |              atomically via
  |                                  |              multicall
  |                                  |                              |
  |                                  |           ANY check fails →
  |                                  |              Revert entirely
  |                                  |              (no funds moved)
```

**The ZK proof guarantees:** the transactions the agent submits are *mathematically identical* to what the user signed. Not "similar", not "within bounds" — identical. If the agent tries to change even one recipient address or amount, the proof fails and the contract reverts.

---

## Architecture Overview

```
+------------------------------------------------------------------+
|                    USER (EOA or Smart Wallet)                      |
|                                                                    |
|  1. Reviews proposed intent bundle                                 |
|  2. Signs commitment hash (EIP-712)                                |
|  3. Approves Agent Wallet for ERC-20 spending                      |
|  4. Can cancel intents or invalidate nonce ranges                  |
|  5. Can revoke approval at any time                                |
+----------------------------------+-------------------------------+
                                   |
                          Signed commitment +
                          ERC-20 approval
                                   |
                                   v
+------------------------------------------------------------------+
|                    AI AGENT (off-chain)                            |
|                                                                    |
|  1. Proposes intent bundle to user                                 |
|  2. Receives signed commitment                                     |
|  3. Generates ZK-SNARK proof (off-chain, 2-5 sec)                 |
|     - Proves: calldata matches signed intent                       |
|     - Private witness: full intent bundle (hidden)                 |
|     - Public inputs: commitment, chainId, signer, multicallHash    |
|  4. Submits proof + signature + calls to Agent Wallet              |
+----------------------------------+-------------------------------+
                                   |
                         proof + signature +
                         publicInputs + calls[]
                                   |
                                   v
+------------------------------------------------------------------+
|                    AGENT WALLET CONTRACT (on-chain)                |
|                                                                    |
|  +------------------+  +------------------+  +------------------+  |
|  | ZK Verifier      |  | Signature Check  |  | Nonce/Expiry     |  |
|  | (SNARK verify)   |  | (ECDSA recover)  |  | Management       |  |
|  +------------------+  +------------------+  +------------------+  |
|                                                                    |
|  Verification passes → Atomic multicall execution                  |
|  Any check fails    → Full revert (no funds moved)                 |
|                                                                    |
|  Key property: NON-CUSTODIAL                                       |
|  - Never holds user funds                                          |
|  - Uses transferFrom (ERC-20 approval pattern)                     |
|  - User can revoke approval instantly                              |
+------------------------------------------------------------------+
                                   |
                         transferFrom calls
                                   |
                                   v
+------------------------------------------------------------------+
|                    ERC-20 TOKEN CONTRACTS                          |
|                                                                    |
|  USDC.transferFrom(user, recipientA, 10_000_000)                   |
|  USDC.transferFrom(user, recipientB, 150_000_000)                  |
|  USDC.transferFrom(user, recipientC, 35_000_000)                   |
|                                                                    |
|  Funds move directly from user → recipients                        |
|  (never through Agent Wallet)                                      |
+------------------------------------------------------------------+
```

---

## Core Components

### 1. Intent Bundles

The intent bundle is a structured JSON/Solidity object that describes exactly what the agent is allowed to do.

```solidity
struct IntentBundle {
    string version;        // Protocol version ("1.0")
    uint256 chainId;       // Target blockchain (1 = Ethereum mainnet)
    bytes32 nonce;         // Unique nonce (replay protection)
    uint256 expiry;        // Unix timestamp — intent expires after this
    address payer;         // User's wallet address
    Action[] actions;      // Array of permitted actions
}

struct Action {
    string actionType;     // Currently only "ERC20_TRANSFER"
    address token;         // Token contract address
    address to;            // Recipient address
    uint256 amount;        // Amount in smallest unit (e.g., 6 decimals for USDC)
}
```

**Example — agent rebalancing a DeFi portfolio:**

```json
{
  "version": "1.0",
  "chainId": 1,
  "nonce": "0x42",
  "expiry": 1736700000,
  "payer": "0xAlice...",
  "actions": [
    { "actionType": "ERC20_TRANSFER", "token": "0xUSDC", "to": "0xAAA", "amount": "10000000" },
    { "actionType": "ERC20_TRANSFER", "token": "0xUSDC", "to": "0xBBB", "amount": "150000000" },
    { "actionType": "ERC20_TRANSFER", "token": "0xUSDC", "to": "0xCCC", "amount": "35000000" }
  ]
}
```

The user reviews this, signs it (EIP-712 typed data for wallet readability), and the commitment hash becomes the public input to the ZK circuit.

### 2. ZK Circuit

The ZK-SNARK circuit proves that the agent's submitted transactions are derived correctly from the user's signed intent — without revealing the full intent bundle on-chain.

#### What the Proof Proves (7 Constraints)

```
+-----------------------------------------------------------------------+
|                    ZK CIRCUIT CONSTRAINTS                               |
|                                                                        |
|  PUBLIC INPUTS (visible on-chain):                                     |
|  - commitment (keccak256 of intent bundle)                             |
|  - chainId                                                             |
|  - signerAddress (user)                                                |
|  - multicallDataHash (hash of all calldata)                            |
|  - nonce                                                               |
|  - expiry                                                              |
|                                                                        |
|  PRIVATE WITNESS (hidden, only prover knows):                          |
|  - Full intent bundle (actions, amounts, recipients)                   |
|                                                                        |
|  CONSTRAINTS (all must hold for proof to be valid):                    |
|                                                                        |
|  1. keccak256(intentBundle) == commitment                              |
|     "The hidden intent bundle hashes to the public commitment"         |
|                                                                        |
|  2. intentBundle.chainId == chainId                                    |
|     "Intent targets the correct chain"                                 |
|                                                                        |
|  3. intentBundle.payer == signerAddress                                 |
|     "Intent was created for this specific user"                        |
|                                                                        |
|  4. For each action[i]:                                                |
|     deriveCalldata(action[i], payer) == calls[i]                       |
|     "Each submitted transaction matches the intent exactly"            |
|                                                                        |
|  5. keccak256(concat(allCalldata)) == multicallDataHash                |
|     "The complete calldata set matches the public hash"                |
|                                                                        |
|  6. intentBundle.nonce == nonce                                        |
|     "Nonce matches (replay protection)"                                |
|                                                                        |
|  7. block.timestamp < intentBundle.expiry                              |
|     "Intent hasn't expired"                                            |
+-----------------------------------------------------------------------+
```

#### Why ZK and Not Just a Signature?

A signature alone proves the user approved *something*. The ZK proof additionally proves the agent's *submitted calldata* matches that approval — without revealing the full intent on-chain.

```
Signature only:
  User signed commitment X.
  Agent submits transactions Y.
  How do we know Y derives from X?
  → We'd have to put the full intent on-chain to check (expensive, no privacy)

ZK proof:
  User signed commitment X.
  Agent submits transactions Y + ZK proof.
  Proof mathematically guarantees: Y derives from X.
  → Full intent stays off-chain (cheap, private)
```

### 3. Agent Wallet Contract

The on-chain smart contract that verifies proofs and executes transactions.

```solidity
interface IAgentWallet {
    function executeWithProof(
        bytes calldata proof,              // ZK-SNARK proof
        bytes calldata signature,          // User's ECDSA signature on commitment
        PublicInputs calldata publicInputs, // Public circuit inputs
        Call[] calldata calls              // Actual transactions to execute
    ) external returns (bool success);
}

struct PublicInputs {
    bytes32 commitment;         // Hash of intent bundle
    uint256 chainId;            // Target blockchain
    address signerAddress;      // User's address
    bytes32 multicallDataHash;  // Hash of calldata
    bytes32 nonce;              // Unique nonce
    uint256 expiry;             // Expiry timestamp
}

struct Call {
    address target;  // Token contract address
    bytes data;      // transferFrom calldata
}
```

**Contract verification sequence:**

```
executeWithProof() called
        |
        v
1. Verify ZK proof against public inputs
   (Is the proof mathematically valid?)
        |
        v
2. Recover signer from ECDSA signature + commitment
   (Did this user actually sign this commitment?)
        |
        v
3. Check nonce: not used, not below minimum
   (Replay protection)
        |
        v
4. Check expiry: block.timestamp < expiry
   (Intent not expired)
        |
        v
5. Check multicallDataHash == keccak256(calls)
   (Submitted calls match the proven hash)
        |
        v
ALL PASS → Execute all calls atomically
           (transferFrom for each action)
           Mark nonce as used
           Emit IntentExecuted event

ANY FAIL → Revert entire transaction
           No funds moved
           Emit IntentFailed event
```

**Key property:** The Agent Wallet is **non-custodial**. It never holds user funds. It uses `transferFrom` to move tokens directly from the user's wallet to recipients. The user can revoke the ERC-20 approval at any time with `approve(agentWallet, 0)`.

---

## End-to-End Flow

```
Alice (user)              Agent (AI)               Agent Wallet (chain)     USDC Contract
  |                          |                          |                       |
  |                          |                          |                       |
  |  "Rebalance my portfolio"|                          |                       |
  |------------------------->|                          |                       |
  |                          |                          |                       |
  |  Intent bundle:          |                          |                       |
  |  - 10 USDC → 0xAAA      |                          |                       |
  |  - 150 USDC → 0xBBB     |                          |                       |
  |  - 35 USDC → 0xCCC      |                          |                       |
  |  nonce: 0x42             |                          |                       |
  |  expiry: +1 hour         |                          |                       |
  |<-------------------------|                          |                       |
  |                          |                          |                       |
  |  Reviews, signs (EIP-712)|                          |                       |
  |  commitment = keccak256( |                          |                       |
  |    encode(intentBundle)) |                          |                       |
  |------------------------->|                          |                       |
  |                          |                          |                       |
  |  approve(agentWallet,    |                          |                       |
  |    195_000_000)          |  (exact total amount)    |                       |
  |--------------------------------------------------------------------->      |
  |                          |                          |                       |
  |                          |  Generate ZK proof       |                       |
  |                          |  (off-chain, 2-5 sec)    |                       |
  |                          |                          |                       |
  |                          |  Submit: proof +         |                       |
  |                          |  signature + inputs +    |                       |
  |                          |  calls[]                 |                       |
  |                          |------------------------->|                       |
  |                          |                          |                       |
  |                          |                          |  Verify ZK proof      |
  |                          |                          |  Verify signature     |
  |                          |                          |  Check nonce/expiry   |
  |                          |                          |  Check calldata hash  |
  |                          |                          |                       |
  |                          |                          |  ALL PASS:            |
  |                          |                          |                       |
  |                          |                          |  transferFrom(Alice,  |
  |                          |                          |    0xAAA, 10M) ------>|
  |                          |                          |  transferFrom(Alice,  |
  |                          |                          |    0xBBB, 150M) ----->|
  |                          |                          |  transferFrom(Alice,  |
  |                          |                          |    0xCCC, 35M) ------>|
  |                          |                          |                       |
  |                          |  IntentExecuted event    |                       |
  |                          |<-------------------------|                       |
  |                          |                          |                       |
  |  "Done! 3 transfers      |                          |                       |
  |   executed atomically"   |                          |                       |
  |<-------------------------|                          |                       |
```

---

## What It Can and Cannot Do

### What ERC-8150 Guarantees

| Guarantee | How | Mechanism |
|-----------|-----|-----------|
| **Agent can ONLY execute pre-approved actions** | ZK proof must match signed commitment | Circuit constraint: `keccak256(intent) == commitment` |
| **No amount tampering** | Amounts are part of the committed intent | Circuit constraint: calldata derivation check |
| **No recipient tampering** | Recipients are part of the committed intent | Circuit constraint: calldata derivation check |
| **No replay** | Nonce consumed on execution | On-chain nonce tracking + `minValidNonce` |
| **Time-bounded** | Intent has expiry timestamp | Contract checks `block.timestamp < expiry` |
| **Atomic execution** | All transfers succeed or all revert | Multicall with revert-on-failure |
| **Non-custodial** | Funds stay in user's wallet until execution | `transferFrom` pattern, not deposits |
| **User can cancel anytime** | Revoke approval or cancel specific intents | `approve(0)` or `cancelIntent()` |
| **Privacy of intent details** | Full intent bundle stays off-chain | ZK proof hides private witness |

### What ERC-8150 Does NOT Guarantee

| Non-guarantee | Why | What Would Be Needed |
|---------------|-----|---------------------|
| **Agent proposes good transactions** | ERC-8150 enforces what the user approved, not whether the agent's proposal was wise | Agent reputation system, price oracles, strategy validation |
| **User understands what they signed** | User could sign a malicious bundle without reading it | Better UX, simulation tools, human-readable intent display |
| **Agent isn't compromised** | A compromised agent proposes bad intents — but user still must sign | Agent security auditing, sandboxing |
| **Cross-chain safety** | Only covers one chain per intent (chainId bound) | Multi-chain intent protocols |
| **Non-ERC20 actions** | Currently only supports `ERC20_TRANSFER` action type | Future extension to swaps, approvals, arbitrary calls |
| **MEV protection** | Transactions are submitted normally (no private mempool) | Flashbots, MEV-resistant submission |
| **Identity of counterparties** | Proves user signed, not who the recipients are | External identity/KYC layer |
| **Post-execution correctness** | Proves execution matched intent, not that the outcome was good | Outcome validation, price impact checks |

### Plain-Language Scenarios

**Scenarios where ERC-8150 saves you:**

```
1. Agent gone rogue
   You told the agent to pay 3 vendors. The agent tries to also
   send 10,000 USDC to its operator's wallet.
   → ZK proof fails (extra transfer not in intent) → REVERT. Zero funds lost.

2. Agent hallucinated an amount
   Agent meant to send $50 but the LLM outputted $5,000 in the calldata.
   → ZK proof fails (amount doesn't match signed intent) → REVERT.

3. Replay attack
   Someone captures the agent's previous valid proof+tx and resubmits it.
   → Nonce already consumed → REVERT.

4. Agent compromised after signing
   Attacker gets access to the agent after user signed, tries to
   change recipients.
   → ZK proof fails (recipients don't match commitment) → REVERT.
```

**Scenarios where ERC-8150 does NOT save you:**

```
1. Agent proposes a bad deal
   Agent suggests buying a token at 50% above market price.
   User signs without checking. ERC-8150 faithfully executes the bad trade.
   → ERC-8150 enforces what you approved, not whether it was smart.

2. Social engineering
   Malicious agent shows "Send 100 USDC to Vendor" but the actual
   intent bundle says "Send 100 USDC to Attacker."
   User signs without reading the EIP-712 details.
   → ERC-8150 can't protect users from signing things they didn't read.

3. Front-running / MEV
   Agent submits the proven transaction. A MEV bot sees it in the
   mempool and front-runs.
   → ERC-8150 doesn't touch mempool privacy. Need Flashbots etc.

4. You need to do something other than ERC-20 transfers
   Agent needs to call Uniswap swap(), stake on Aave, mint an NFT.
   → ERC-8150 v1 only supports ERC20_TRANSFER. Can't help.

5. Cross-chain agent
   Agent operates on Ethereum AND Arbitrum AND Base.
   → Each intent is single-chain. No atomic cross-chain support.
```

### The Honest Assessment

ERC-8150 is strong at **constraining what an agent CAN do** (enforcement). It's not designed for proving what an agent DID do after the fact (evidence/audit). It prevents bad transactions before they happen, rather than documenting them after.

**The biggest current limitation:** only `ERC20_TRANSFER` is supported. No swaps, no staking, no approvals, no arbitrary contract calls. This means the DeFi portfolio rebalancing use case — the motivating example in the spec — actually can't be fully implemented yet. The standard would need extension to action types like `ERC20_APPROVE`, `UNISWAP_SWAP`, or a generic `ARBITRARY_CALL` (which introduces much harder verification problems).

**The gas overhead reality:** 345,000 gas for ZK verification is roughly $1-3 on Ethereum mainnet at current prices. This makes single transfers uneconomical (a normal transfer is ~65k gas). The break-even is around 3-5 transfers per batch where the per-transfer cost dilutes the fixed overhead enough to be worth the security guarantee.

---

## Security Model

### Threat Model

```
Attack                              Defense                         Result
──────                              ───────                         ──────

Agent changes amounts               ZK proof fails                  Revert (no funds moved)
after user signs                     (calldata ≠ commitment)

Agent changes recipients             ZK proof fails                  Revert
after user signs                     (calldata ≠ commitment)

Agent replays old intent             Nonce already consumed          Revert

Agent uses expired intent            block.timestamp > expiry        Revert

Agent submits to wrong chain         chainId mismatch                Revert

Agent submits extra transfers        multicallDataHash mismatch      Revert
not in intent

Compromised agent proposes           User must still sign            User is the last
bad intent                           (human review step)             line of defense

User wants to cancel                 cancelIntent() or               Intent becomes
after signing                        invalidateNonceRange()          unexecutable
                                     or revoke ERC-20 approval

Proof malleability                   Use non-malleable proof         PLONK recommended
                                     systems
```

### Non-Custodial Design

This is a critical property. The Agent Wallet contract **never holds user funds**:

```
Traditional agent wallet:          ERC-8150 Agent Wallet:

User → deposits funds → Agent      User → approves spending → Agent Wallet
       Wallet holds $$$                    User holds $$$

Agent controls funds                Agent Wallet uses transferFrom
(custodial risk)                    (non-custodial, revocable)

Agent can drain wallet              Agent can only execute
                                    proven-matching transactions

Recovery: hope agent is honest      Recovery: revoke approval instantly
                                    with approve(agentWallet, 0)
```

### Nonce Management

```
Per-user nonce tracking:

usedNonces[Alice][0x42] = true     ← nonce consumed on execution
minValidNonce[Alice] = 0x100       ← batch invalidate all nonces < 0x100

cancelledIntents[Alice][commitment] = true  ← cancel specific intent
```

Three cancellation mechanisms:
1. **Cancel specific intent:** `cancelIntent(nonce, commitment)` — marks nonce as used
2. **Batch invalidate:** `invalidateNonceRange(newMinNonce)` — invalidates all nonces below threshold
3. **Revoke approval:** `approve(agentWallet, 0)` — nuclear option, blocks all transfers

---

## Performance

### Gas Costs

| Operation | Gas | Notes |
|-----------|-----|-------|
| **Fixed overhead** | ~345,000 | ZK proof verification + state checks |
| **Per transfer** | ~65,000 | ERC-20 `transferFrom` |
| **1 transfer** | ~410,000 | Fixed + 1 × 65k |
| **3 transfers** | ~540,000 | Fixed + 3 × 65k |
| **10 transfers** | ~995,000 | Fixed + 10 × 65k |

**Break-even:** The 345k fixed overhead means this is most efficient for **batch payments** (3+ transfers). For single transfers, a direct `transfer` call (21k-65k gas) is much cheaper.

### Latency

| Step | Time |
|------|------|
| Proof generation (off-chain) | 2-5 seconds |
| Block confirmation | ~12 seconds (Ethereum) |
| **End-to-end** | ~15-20 seconds |

---

## Comparison with ACTIS

ACTIS and ERC-8150 operate at different layers and solve different problems:

| Dimension | ACTIS | ERC-8150 |
|-----------|-------|----------|
| **Layer** | Evidence (post-transaction) | Enforcement (pre-transaction) |
| **Question answered** | "Can we prove this record hasn't been tampered with?" | "Can we prevent the agent from deviating from approved actions?" |
| **When it acts** | After the transaction — records what happened | Before the transaction — blocks unauthorized actions |
| **Mechanism** | Hash chains + Ed25519 signatures | ZK-SNARKs + on-chain verification |
| **On-chain?** | No (fully offline) | Yes (smart contract on Ethereum) |
| **Blockchain dependency** | None | Requires Ethereum (or EVM-compatible) |
| **Cryptography** | SHA-256, Ed25519 (simple) | ZK-SNARKs, keccak256, ECDSA (complex) |
| **What it prevents** | Nothing (evidence only, not enforcement) | Unauthorized transactions (enforcement) |
| **What it proves** | Record integrity (tamper-evidence) | Transaction-intent alignment (mathematical proof) |
| **Privacy** | Full transparency (record is readable) | Intent details hidden (ZK) |
| **Vendor neutral** | Yes (any language, offline) | Ethereum-specific (EVM) |
| **Gas cost** | Zero (off-chain) | 345k + 65k per transfer |

### How They Complement Each Other

```
+---------------------------------------------------------------+
|                                                                |
|  PRE-EXECUTION (ERC-8150):                                     |
|  "Agent, prove your transactions match my signed intent"       |
|  → ZK proof verification on-chain                              |
|  → Unauthorized transactions blocked                           |
|                                                                |
|  ─────────────── Transaction executes ───────────────          |
|                                                                |
|  POST-EXECUTION (ACTIS):                                       |
|  "Here's a tamper-evident record of the full negotiation"      |
|  → Hash-linked transcript                                      |
|  → Offline verifiable by any party                              |
|  → Evidence for disputes, audits, compliance                   |
|                                                                |
+---------------------------------------------------------------+
```

ERC-8150 **prevents** bad transactions. ACTIS **documents** what happened. Together they provide both enforcement and evidence.

---

## Comparison with Existing Agent Wallet Approaches

| Approach | Trust Model | Custody | Risk | When Bad Things Are Caught |
|----------|------------|---------|------|---------------------------|
| **Direct deposit to agent wallet** | Full trust in agent | Custodial | Agent can drain funds | After funds are gone |
| **Per-transaction approval** | Zero trust, full control | Non-custodial | None (but kills automation) | Before each action |
| **Spending limits (Sponge/Locus)** | Bounded trust | Custodial (Sponge) / Non-custodial (Locus) | Agent can spend up to limit | At the limit boundary |
| **ERC-8004 (reputation)** | Trust based on track record | Varies | Reputable agents can still be compromised | After bad execution |
| **ERC-8150 (ZK verification)** | Zero trust, batch approval | Non-custodial | None within approved intent | Before execution (blocked) |

---

## Real-World Use Cases

### Use Case 1: AI Portfolio Rebalancing

```
User: "Rebalance my portfolio to 40% ETH, 30% USDC, 30% DAI"

Agent proposes intent bundle:
  - Sell 500 USDC for ETH on Uniswap  [future extension]
  - Transfer 300 USDC to DAI pool     [future extension]
  - Transfer 200 USDC to yield vault

User reviews exact amounts and recipients.
Signs commitment.

Agent generates ZK proof, submits.
Contract verifies and executes atomically.

If agent tried to slip in an extra transfer to its own address
→ ZK proof would fail → entire transaction reverts.
```

### Use Case 2: Batch Payroll by AI Agent

```
Company agent processes monthly payroll:

Intent bundle:
  - 5,000 USDC → Employee A
  - 7,500 USDC → Employee B
  - 3,200 USDC → Employee C
  - 12,000 USDC → Employee D

CFO signs the bundle.
Agent submits with ZK proof.
All 4 transfers execute atomically.

Cost: ~605,000 gas (345k + 4 × 65k)
vs 4 separate transfers: ~260,000 gas (4 × 65k)

Premium: ~345k gas for ZK verification overhead.
Worth it when the CFO can't trust the agent to not
change amounts or add extra recipients.
```

### Use Case 3: Agent-Mediated E-Commerce Settlement

```
Shopping agent (on Sponge/Locus) negotiates deals with 3 vendors.
Total settlement: 195 USDC across 3 recipients.

Without ERC-8150:
  Option A: Give the agent your wallet key → custodial risk
  Option B: Agent proposes, you sign 3 separate transactions → defeats automation
  Option C: Use spending limits → agent could send to wrong recipients within limit

With ERC-8150:
  Agent proposes intent bundle with exact recipients + amounts.
  You review once, sign once.
  Agent generates ZK proof, submits.
  All 3 transfers execute atomically.
  Agent can't change a single recipient or amount.
```

### Use Case 4: DAO Treasury Agent

```
A DAO votes to allocate funds. An AI agent executes the allocation.

Intent bundle:
  - 50,000 USDC → Development team multisig
  - 20,000 USDC → Marketing budget wallet
  - 10,000 USDC → Auditor payment address
  - 5,000 USDC → Community grants pool

DAO multisig signs the bundle.
Agent executes with ZK proof.

Why this matters:
  - DAO members can verify the EXACT allocation was executed
  - Agent can't skim, reroute, or add extra recipients
  - Atomic: if one transfer fails (e.g., contract paused), ALL revert
  - No trust in the agent operator required
```

### Use Case 5: Subscription/Recurring Payments via Agent

```
User authorizes agent to make monthly SaaS payments:

Month 1 intent bundle:
  - 99 USDC → Notion (0xNotion)
  - 20 USDC → GitHub Copilot (0xGitHub)
  - 49 USDC → Vercel (0xVercel)

User signs. Agent executes.

Next month, agent proposes a new bundle (amounts may change).
User reviews and signs again.

Key: user re-approves each month's exact amounts.
Agent can't auto-escalate prices or add new services
without a new signature.
```

### When ERC-8150 Is Useful vs Unnecessary

**Useful when:**
- Agent handles **on-chain ERC-20 transfers** on your behalf
- You want to **batch-approve** multiple transactions with one signature
- **Non-custodial** control matters (funds stay in your wallet)
- You need **mathematical guarantees** the agent can't deviate from approved actions
- **Atomic execution** matters (all-or-nothing for a batch)
- Intent **privacy** matters (details stay off-chain via ZK)
- **Multiple recipients** in one batch (amortizes the 345k gas overhead)
- Trust boundary exists between you and the agent operator

**Unnecessary when:**
- **Single transactions** (345k gas overhead makes it 5x more expensive than a direct transfer)
- You **already trust the agent fully** (e.g., your own infrastructure, same org)
- **Off-chain transactions** (ERC-8150 is on-chain Ethereum only)
- **Non-ERC20 actions** needed (swaps, staking, NFT mints — only `ERC20_TRANSFER` supported today)
- Agent needs to make **dynamic decisions** (ERC-8150 requires pre-approval of exact amounts — can't handle "buy ETH at best available price")
- **Speed matters** — 15-20 seconds end-to-end (proof generation + block confirmation) may be too slow for time-sensitive trades
- **L2 with cheap gas** — the security guarantee still holds but the gas overhead matters less, and the ZK circuit complexity remains

---

## Summary

ERC-8150 provides **pre-execution enforcement** for agent-mediated payments:

1. **Intent bundles** — structured description of exactly what the agent may do (tokens, amounts, recipients, chain, expiry)
2. **ZK-SNARK proofs** — agent proves its submitted transactions derive from the user's signed intent, without revealing full intent on-chain
3. **Agent Wallet contract** — on-chain verifier that checks proof + signature + nonce + expiry, then executes atomically via `transferFrom`
4. **Non-custodial** — funds never leave the user's wallet until verified execution; user can revoke at any time
5. **Replay protection** — per-user nonce tracking with batch invalidation and individual intent cancellation
6. **Atomic execution** — all transfers succeed or all revert; no partial execution

The core innovation: **zero-trust delegation**. Users delegate execution to agents without trusting them — the smart contract mathematically enforces that the agent can only do what was pre-approved. This sits at the enforcement layer of the agentic commerce stack, complementing evidence layers like ACTIS.
