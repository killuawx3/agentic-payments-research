# ACTIS: Autonomous Coordination & Transaction Integrity Standard

> **A deterministic, vendor-neutral standard for producing and verifying cryptographically intact evidence of autonomous agent transactions.**

ACTIS defines a minimal format for recording what AI agents did during a transaction — procurement, negotiation, settlement — in a way that is **tamper-evident**, **offline-verifiable**, and **deterministic**. It uses hash-linked, Ed25519-signed JSON records packaged into portable ZIP bundles that any party can verify independently without trusting the producing system, querying online services, or depending on any blockchain.

ACTIS v1.0 (March 2026) ships with three independent conformant implementations (Rust, TypeScript, Python) derived from spec alone with no shared code, and an 11-vector conformance corpus.

- **Spec:** [github.com/actis-spec/actis](https://github.com/actis-spec/actis)
- **Whitepaper:** [actis.world/whitepaper](https://actis.world/whitepaper)
- **License:** Apache 2.0, patent non-assert for all implementers

---

## Table of Contents

- [Why ACTIS Exists](#why-actis-exists)
- [What ACTIS Is (and Isn't)](#what-actis-is-and-isnt)
- [Architecture Overview](#architecture-overview)
- [Core Concepts](#core-concepts)
  - [Transcripts](#1-transcripts)
  - [Rounds](#2-rounds)
  - [Bundles](#3-bundles)
  - [Verification Reports](#4-verification-reports)
- [The Hash Chain](#the-hash-chain)
- [The Signature Model](#the-signature-model)
- [The Verification Model](#the-verification-model)
- [Security Model](#security-model)
- [Conformance Corpus](#conformance-corpus)
- [Where ACTIS Fits in the Agentic Commerce Stack](#where-actis-fits-in-the-agentic-commerce-stack)
- [Key Diagrams](#key-diagrams)

---

## Why ACTIS Exists

When AI agents autonomously execute transactions on behalf of humans, an evidence gap emerges. If a dispute arises — "my agent paid too much", "the counterparty changed terms", "the settlement was wrong" — how do you prove what actually happened?

### The Evidence Problem

```
Traditional Commerce:                  Agentic Commerce:

Human signs contract     ──→           Agent negotiates at machine speed
Human reviews wire       ──→           Agent executes settlement autonomously
Human confirms receipt   ──→           Agent acts across trust boundaries
                                       No human in the loop
Paper trail exists       ──→           What's the evidence?
```

### Why Existing Approaches Fail

| Approach | Problem |
|----------|---------|
| **Application logs** | Mutable by any party with write access. The operator controls the record — conflict of interest in disputes |
| **API receipts** | Unilateral attestation. Proves a request was received, not that processing was correct |
| **Audit databases** | Append-only by convention, not by cryptographic construction. Tamper-evidence requires careful implementation |
| **Blockchain** | Solves immutability but introduces inappropriate dependencies — specific infrastructure, token economics, consensus overhead, vendor lock-in. Overkill for bilateral evidence |

### What's Needed

A practical evidence standard requires four properties:

1. **Tamper-evidence** — modification of any record part must be detectable by any verifier
2. **Offline verifiability** — verification requires no trust in online services
3. **Determinism** — independent verifiers given the same evidence must produce identical verdicts
4. **Vendor neutrality** — implementable by any party without proprietary dependencies

ACTIS provides exactly these four properties and nothing more.

---

## What ACTIS Is (and Isn't)

### ACTIS defines:

- **Transcript format** — hash-linked, signed JSON record of transaction sessions
- **Bundle format** — portable ZIP evidence container with integrity verification
- **Verification model** — deterministic, offline checks with canonical output reports
- **Conformance corpus** — 11 test vectors any implementation must pass

### ACTIS explicitly does NOT define:

- Negotiation protocols
- Settlement logic
- Identity systems (key-to-person binding)
- Reputation scoring
- Adjudication or dispute resolution
- Content semantics (what messages mean)
- Timestamps bound to external time sources

ACTIS is **only the evidence layer**. It proves the record is intact, not that the transaction was correct.

### Cryptographic Stack

ACTIS uses deliberately simple, widely available cryptography. No ZK proofs, no blockchain, no trusted setup.

| Primitive | Standard | Purpose |
|-----------|----------|---------|
| **SHA-256** | NIST FIPS 180-4 | Hash chain linking, file checksums, final hash seal |
| **Ed25519** | RFC 8032 | Per-round digital signatures (participant authorization) |
| **RFC 8785 (JCS)** | JSON Canonicalization Scheme | Deterministic serialization so hashes are reproducible across implementations |
| **Base58** | Bitcoin-style encoding | Public keys and signature encoding |

That's the entire crypto stack. Every primitive is available in every language's standard library. No specialized circuits, no prover infrastructure, no trusted setup ceremonies.

**Why not ZK proofs?** ZK solves a different problem — proving you know something *without revealing it*. ACTIS has the opposite goal: **full transparency**. The whole point is that anyone can read every round, verify every hash, check every signature. There's nothing to hide.

### What ACTIS Can Verify

| Check | What It Proves | Example |
|-------|---------------|---------|
| **Hash chain intact** | No rounds were added, removed, reordered, or modified after recording | "The transcript shows round 3 was a BID of $49.99 — and no one changed that number after the fact" |
| **Signatures valid** | The stated party (public key) authorized each round | "The agent identified by key `7nYBb...` signed the ACCEPT round" |
| **Bundle integrity** | The ZIP file hasn't been corrupted or selectively edited | "The transcript file inside the bundle matches its recorded checksum" |
| **Final hash seal** | The entire transcript hasn't been modified since it was sealed | "This is the complete, unaltered record" |
| **Evidence completeness** | All referenced evidence artifacts are present in the bundle | "No one stripped attachments from the bundle after the fact" |

### What ACTIS Cannot Verify

| Non-check | Why Not | What Would Be Needed |
|-----------|---------|---------------------|
| **Was the agent's decision correct?** | ACTIS records WHAT happened, not WHETHER it was right | Application-layer business logic, auditor judgment |
| **Did the agent get a good price?** | ACTIS doesn't know market prices | External price oracle or benchmark |
| **Is the signer who they claim to be?** | ACTIS verifies key → signature, not key → real-world identity | PKI, DID, or identity provider |
| **Did the counterparty actually deliver?** | ACTIS records the negotiation, not the fulfillment | Order tracking, delivery confirmation system |
| **Is this the ONLY transcript?** | A bad actor could produce two valid transcripts for the same deal (equivocation attack) | Transparency log, bilateral counter-signing |
| **Was anything omitted BEFORE recording?** | The producer controls what rounds go into the transcript | Higher-level protocol commitments |
| **Are the timestamps real?** | Timestamps come from the producer, not an external clock | Trusted timestamping service (RFC 3161) |
| **Is the content truthful?** | ACTIS verifies `message_hash` exists, not that the message is true | Out of scope — application-layer concern |

### The One-Sentence Summary

ACTIS answers one question: **"Given this record of what an agent did, can we prove the record hasn't been tampered with?"** Everything else — identity, correctness, reputation, adjudication — is explicitly out of scope.

---

## Real-World Use Cases

### Use Case 1: Agent Overpaid — Dispute Resolution

```
Your agent bought office supplies via Rye/Purch/Sponge.
The invoice says $500. You think it should have been $300.

Without ACTIS:
  - You check the platform's logs (but they control those logs)
  - You check your agent's logs (but those are just your side)
  - Nobody has a mutually trusted record
  - It's your word vs theirs

With ACTIS:
  - Both sides have a signed transcript bundle
  - Round 1 (ASK):        Seller offered at $500
  - Round 2 (COUNTER):    Your agent countered at $250
  - Round 3 (COUNTER):    Seller countered at $300
  - Round 4 (ACCEPT):     Your agent accepted $300
  - Round 5 (SETTLEMENT): Payment executed for $500  ← MISMATCH

  - Any third party can verify the bundle offline
  - The hash chain proves round 4 accepted $300
  - The signature proves YOUR agent signed that acceptance
  - Now you have cryptographic evidence the settlement
    didn't match the agreed terms
```

### Use Case 2: Regulatory Compliance (EU AI Act)

```
EU AI Act requires "traceability" for high-risk AI systems.
A regulator asks: "Show me what your agent did."

Without ACTIS:
  - Export application logs (mutable, not standardized)
  - Auditor has to trust your infrastructure
  - No way to prove logs weren't modified after the fact

With ACTIS:
  - Hand over the bundle.zip
  - Regulator runs their own verifier (Rust, TypeScript, or Python)
  - Gets ACTIS_COMPATIBLE → record is structurally intact
  - Can read every round of the transaction in plain JSON
  - No trust in your infrastructure needed
  - Standardized format — same verifier works for any ACTIS producer
```

### Use Case 3: Multi-Agent Negotiation

```
Agent A (buyer) and Agent B (seller) negotiate a deal.
Later, Agent B claims different terms were agreed.

Without ACTIS:
  - Each agent has its own logs
  - Logs disagree
  - No resolution possible

With ACTIS:
  - Both agents sign each round with their Ed25519 keys
  - Agent A signed ACCEPT on specific terms
  - Agent B signed the ASK with those same terms
  - The hash chain links them in order
  - A third party can verify both signatures offline
  - The signed record IS the agreement — cryptographic proof
    of who said what and when
```

### Use Case 4: Audit Trail for Autonomous Procurement

```
A company deploys agents to autonomously procure cloud
infrastructure, negotiate SaaS contracts, or purchase supplies.

CFO asks: "What did our agents commit us to this quarter?"

Without ACTIS:
  - Scattered logs across multiple platforms
  - Each platform has its own format
  - No guarantee logs are complete or unmodified

With ACTIS:
  - Each procurement generates a sealed bundle
  - Bundles are standardized (same format, same verifier)
  - Auditor batch-verifies all bundles
  - Can reconstruct the full decision trail:
    what was offered, what was countered, what was accepted
  - Signatures prove which agent authorized each commitment
```

### When ACTIS Is Useful vs Unnecessary

**Useful when:**
- Agents transact across **trust boundaries** (different owners, different platforms)
- **Disputes are possible** and need resolution with evidence
- **Regulators** require standardized audit trails (EU AI Act, NIST AI RMF)
- You need a **format multiple parties can verify independently**
- Transactions are **high-value enough** to warrant formal evidence

**Unnecessary when:**
- All agents are under your control (just use your own logs)
- Transactions are low-value and disputes don't matter
- You already have on-chain settlement that serves as the record (e.g., everything on a blockchain already)
- There's no counterparty (single-agent actions with no dispute risk)
- You need real-time enforcement, not after-the-fact evidence

---

## Architecture Overview

```
+------------------------------------------------------------------+
|                    PRODUCER LAYER                                  |
|                    (out of ACTIS scope)                            |
|                                                                    |
|  +------------------+  +------------------+  +------------------+  |
|  | Agent Runtime    |  | Negotiation      |  | Settlement       |  |
|  | (Claude, GPT,    |  | Logic            |  | Logic            |  |
|  |  custom agent)   |  |                  |  |                  |  |
|  +--------+---------+  +--------+---------+  +--------+---------+  |
|           |                     |                     |            |
|           +---------------------+---------------------+            |
|                                 |                                  |
|                      Produces ACTIS transcript                     |
|                      (hash-linked, signed rounds)                  |
+----------------------------------+---------------------------------+
                                   |
                                   v
+------------------------------------------------------------------+
|                    EVIDENCE ARTIFACTS                              |
|                    (ACTIS scope)                                   |
|                                                                    |
|  +---------------------------+  +------------------------------+   |
|  | transcript.json           |  | bundle.zip                   |   |
|  |                           |  |                              |   |
|  | transcript_version        |  | ├── manifest.json            |   |
|  | transcript_id             |  | ├── input/transcript.json    |   |
|  | intent_id                 |  | └── checksums.sha256         |   |
|  | rounds: [                 |  |                              |   |
|  |   { round_hash,           |  | Self-contained, portable,    |   |
|  |     previous_round_hash,  |  | offline-verifiable           |   |
|  |     signature,            |  |                              |   |
|  |     message_hash, ... }   |  |                              |   |
|  | ]                         |  |                              |   |
|  | final_hash                |  |                              |   |
|  +---------------------------+  +------------------------------+   |
+----------------------------------+---------------------------------+
                                   |
                                   v
+------------------------------------------------------------------+
|                    VERIFIER LAYER                                  |
|                    (ACTIS scope)                                   |
|                                                                    |
|  +---------------------------+                                     |
|  | ACTIS Verifier            |  Offline, deterministic             |
|  | (Rust / TS / Python)      |  No network, no trusted service     |
|  |                           |                                     |
|  | Checks:                   |                                     |
|  | 1. Schema valid?          |  → schema_ok                       |
|  | 2. Hash chain intact?     |  → hash_chain_ok                   |
|  | 3. Signatures valid?      |  → signatures_ok                   |
|  | 4. Checksums match?       |  → checksums_ok                    |
|  | 5. Final hash replays?    |  → replay_ok                       |
|  |                           |                                     |
|  | Output:                   |                                     |
|  | ACTIS_COMPATIBLE          |  (all checks pass)                 |
|  | ACTIS_PARTIAL             |  (structure ok, sigs invalid)      |
|  | ACTIS_NONCOMPLIANT        |  (structural integrity broken)     |
|  +---------------------------+                                     |
+------------------------------------------------------------------+
```

---

## Core Concepts

### 1. Transcripts

A transcript is the primary evidence artifact — a hash-linked, signed JSON record of one transaction session.

```json
{
  "transcript_version": "actis-transcript/1.0",
  "transcript_id": "transcript-<sha256hex>",
  "intent_id": "purchase-order-2026-001",
  "intent_type": "procurement",
  "created_at_ms": 1710800000000,
  "policy_hash": "<sha256hex>",
  "strategy_hash": "<sha256hex>",
  "identity_snapshot_hash": "<sha256hex>",
  "rounds": [ ... ],
  "final_hash": "<sha256hex>"
}
```

**Key fields:**

| Field | Purpose |
|-------|---------|
| `transcript_version` | Must be `actis-transcript/1.0` |
| `transcript_id` | Unique identifier for this transcript |
| `intent_id` | What transaction this records (bound into genesis hash) |
| `created_at_ms` | Creation timestamp in milliseconds (bound into genesis hash) |
| `policy_hash` | Hash of the policy governing this transaction |
| `strategy_hash` | Hash of the agent's strategy configuration |
| `identity_snapshot_hash` | Hash of participant identity state at creation |
| `rounds` | Array of hash-linked, signed round objects |
| `final_hash` | Commitment over the entire transcript (tamper seal) |

### 2. Rounds

Each round represents one discrete step in the transaction. Rounds are hash-linked — each round references the previous round's hash, forming a chain.

```json
{
  "round_number": 2,
  "round_type": "BID",
  "message_hash": "<sha256hex>",
  "envelope_hash": "<sha256hex>",
  "signature": {
    "signer_public_key_b58": "<base58-ed25519-pubkey>",
    "signature_b58": "<base58-ed25519-signature>",
    "scheme": "ed25519"
  },
  "timestamp_ms": 1710800002000,
  "previous_round_hash": "<sha256hex of round 1>",
  "round_hash": "<sha256hex>",
  "agent_id": "buyer",
  "content_summary": {}
}
```

**Round types:**

| Type | Meaning |
|------|---------|
| `INTENT` | Transaction initiation (always round 0) |
| `ASK` | Seller's offer/request |
| `BID` | Buyer's offer/counter |
| `COUNTER` | Counter-proposal from either party |
| `ACCEPT` | Agreement to terms |
| `REJECT` | Rejection of terms |
| `ABORT` | Transaction cancellation |

### 3. Bundles

A bundle is a ZIP archive — the portable unit of evidence exchange. Self-contained, transmittable, storable, and verifiable independently.

```
bundle.zip
├── manifest.json              ← Standard ID + version + core file list
├── input/
│   └── transcript.json        ← The ACTIS transcript
└── checksums.sha256           ← SHA-256 checksums of all core files
```

**manifest.json:**
```json
{
  "standard": { "name": "ACTIS", "version": "1.0" },
  "core_files": [
    "checksums.sha256",
    "manifest.json",
    "input/transcript.json"
  ]
}
```

**Bundle security rules:**
- Path traversal (`..` segments) → rejected
- Duplicate ZIP entries → rejected
- Missing manifest → `ACTIS_NONCOMPLIANT`
- Unreadable/malformed ZIP → `ACTIS_NONCOMPLIANT`
- Extra files beyond core set → ignored (allows supplementary evidence)

### 4. Verification Reports

The canonical output of verification — a JSON object with deterministic fields.

```json
{
  "actis_version": "1.0",
  "actis_status": "ACTIS_COMPATIBLE",
  "schema_ok": true,
  "hash_chain_ok": true,
  "signatures_ok": true,
  "checksums_ok": true,
  "replay_ok": true,
  "warnings": [],
  "errors": []
}
```

**Status values:**

| Status | Meaning | When |
|--------|---------|------|
| `ACTIS_COMPATIBLE` | Fully verified | All 4 checks pass |
| `ACTIS_PARTIAL` | Structure intact, signatures invalid | Schema + hash chain + checksums pass; signatures fail |
| `ACTIS_NONCOMPLIANT` | Structural integrity broken | Any of schema, hash chain, or checksums fail |

---

## The Hash Chain

The hash chain is the core integrity mechanism. It creates an ordered, tamper-evident sequence of rounds that cannot be reordered, inserted into, or removed from without detection.

### How It Works

```
Transaction metadata
  intent_id = "purchase-order-2026-001"
  created_at_ms = 1710800000000
        |
        v
  Genesis Hash = SHA-256("purchase-order-2026-001:1710800000000")
        |
        v
+-------+------------------------------------------------------------------+
| Round 0 (INTENT)                                                          |
|                                                                           |
|   previous_round_hash = Genesis Hash                                      |
|   round_type = "INTENT"                                                   |
|   message_hash = SHA-256(message content)                                 |
|   envelope_hash = SHA-256(envelope)                                       |
|   timestamp_ms = 1710800000000                                            |
|                                                                           |
|   round_hash = SHA-256(JCS(round \ {round_hash, signature}))             |
|   ──────────────────────────────────────────────────────────────────────  |
|        |                                                                  |
+--------+------------------------------------------------------------------+
         |
         v  (round_hash of round 0 becomes previous_round_hash of round 1)
+--------+------------------------------------------------------------------+
| Round 1 (ASK)                                                             |
|                                                                           |
|   previous_round_hash = round_hash of Round 0                             |
|   round_type = "ASK"                                                      |
|   ...                                                                     |
|   round_hash = SHA-256(JCS(round \ {round_hash, signature}))             |
|   ──────────────────────────────────────────────────────────────────────  |
|        |                                                                  |
+--------+------------------------------------------------------------------+
         |
         v
+--------+------------------------------------------------------------------+
| Round 2 (BID)                                                             |
|                                                                           |
|   previous_round_hash = round_hash of Round 1                             |
|   ...                                                                     |
|   round_hash = SHA-256(JCS(round \ {round_hash, signature}))             |
+--------+------------------------------------------------------------------+
         |
         v
+--------+------------------------------------------------------------------+
| Round 3 (ACCEPT)                                                          |
|                                                                           |
|   previous_round_hash = round_hash of Round 2                             |
|   ...                                                                     |
|   round_hash = SHA-256(JCS(round \ {round_hash, signature}))             |
+--------+------------------------------------------------------------------+
         |
         v
  final_hash = SHA-256(JCS(transcript \ {final_hash, model_context}))
         |
         v
  Tamper-evident seal over the entire transcript
```

### Hash Formulas

| Hash | Formula |
|------|---------|
| **Genesis** `previous_round_hash` | `SHA-256(UTF-8(intent_id + ":" + decimal(created_at_ms)))` |
| **Round** `round_hash` | `SHA-256(JCS(round_object \ {round_hash, signature}))` |
| **Transcript** `final_hash` | `SHA-256(JCS(transcript \ {final_hash, model_context}))` |

**All hashes:** 64 lowercase hex characters. **JCS:** RFC 8785 JSON Canonicalization Scheme (deterministic key ordering, number serialization, no whitespace ambiguity).

### Why Signatures Are Excluded from round_hash

This is a critical design decision. The `signature` field is explicitly excluded from the `round_hash` input. This means:

- `hash_chain_ok` is **independent** of `signatures_ok`
- A transcript can have valid hash chains but invalid signatures → `ACTIS_PARTIAL`
- A transcript can have valid signatures but broken hash chains → `ACTIS_NONCOMPLIANT`

This separation preserves **two independent failure dimensions**, enabling diagnostically useful partial results. If signatures were included in `round_hash`, a single bad signature would break the entire hash chain, collapsing structural integrity and authentication into one check.

### What the Genesis Hash Prevents

The genesis hash binds the chain to the specific `intent_id` and `created_at_ms`. This prevents **chain transplantation attacks** — taking valid rounds from one transcript and inserting them into another. The genesis hash would mismatch, breaking the chain at round 0.

---

## The Signature Model

Each round is individually signed using Ed25519. Signatures prove that a specific participant (identified by public key) authorized that specific round.

### Signing Process

```
                 Domain string          Envelope hash (from round)
                 "ACTIS/v1"             (32 bytes, hex-decoded)
                 (8 bytes)
                     |                       |
                     v                       v
              +------+------+  +-------------+-------------+
              | A C T I S / |  | raw 32-byte envelope_hash |
              | v 1         |  | (hex-decoded, not hex str)|
              +------+------+  +-------------+-------------+
                     |                       |
                     +----------+------------+
                                |
                                v
                     40-byte signing message
                                |
                     Ed25519.sign(signing_message, private_key)
                                |
                                v
                     signature_b58 (Base58-encoded)
```

**Signing message:** `utf8("ACTIS/v1") || hex_decode(envelope_hash)` = 40 bytes

The `"ACTIS/v1"` domain prefix prevents **cross-protocol signature reuse** — a signature valid in ACTIS cannot be replayed in another protocol, and vice versa.

### Verification

```
For each round:
  1. Decode signer_public_key_b58 (Base58 → Ed25519 public key)
  2. Decode signature_b58 (Base58 → Ed25519 signature)
  3. Construct signing_message = utf8("ACTIS/v1") || hex_decode(envelope_hash)
  4. Ed25519.verify(signing_message, signature, public_key)
  5. If ANY round fails → signatures_ok = false → ACTIS_PARTIAL
```

### What Signatures Prove (and Don't)

| Proves | Does NOT Prove |
|--------|---------------|
| Signature was produced by the stated public key | That the key belongs to any real-world identity |
| Participant authorized this specific round | That the participant is who they claim to be |
| Round content hasn't changed since signing | That the content is correct or truthful |

Key-to-identity binding requires a **separate identity layer** outside ACTIS scope.

---

## The Verification Model

Verification is four independent checks that map to boolean fields in the report.

```
                         ACTIS Bundle (.zip)
                                |
                                v
                    +-----------+-----------+
                    |                       |
                    v                       v
            +-------+-------+      +-------+-------+
            | Check 1:      |      | Check 4:      |
            | Schema &      |      | Checksums     |
            | Layout        |      |               |
            | (transcript   |      | (SHA-256 of   |
            |  fields, ZIP  |      |  core files   |
            |  structure)   |      |  vs stored)   |
            +-------+-------+      +-------+-------+
                    |                       |
                    v                       v
              schema_ok               checksums_ok
                    |
                    v
            +-------+-------+
            | Check 2:      |
            | Hash Chain    |
            |               |
            | (genesis →    |
            |  linkage →    |
            |  round_hash → |
            |  final_hash)  |
            +-------+-------+
                    |
                    v
            hash_chain_ok
            replay_ok
                    |
                    v
            +-------+-------+
            | Check 3:      |
            | Signatures    |
            |               |
            | (Ed25519      |
            |  per round)   |
            +-------+-------+
                    |
                    v
            signatures_ok
                    |
                    v
            +-------+-------+
            | Status        |
            | Derivation    |
            +-------+-------+
                    |
         +----------+----------+
         |          |          |
         v          v          v
  ACTIS_COMPATIBLE  ACTIS_PARTIAL  ACTIS_NONCOMPLIANT
  (all pass)        (struct ok,    (struct broken)
                     sigs fail)
```

### Status Derivation Rules

| schema_ok | hash_chain_ok | checksums_ok | signatures_ok | → actis_status |
|-----------|--------------|-------------|--------------|----------------|
| true | true | true | true | `ACTIS_COMPATIBLE` |
| true | true | true | **false** | `ACTIS_PARTIAL` |
| **false** | any | any | any | `ACTIS_NONCOMPLIANT` |
| any | **false** | any | any | `ACTIS_NONCOMPLIANT` |
| any | any | **false** | any | `ACTIS_NONCOMPLIANT` |

**Key invariant:** `ACTIS_PARTIAL` is only possible when structural integrity is fully intact but signatures fail. This preserves evidentiary value — the record is structurally sound but not fully authenticated.

---

## Security Model

### What ACTIS Proves

| Property | Guarantee | Mechanism |
|----------|-----------|-----------|
| **Structural integrity** | Transcript not modified since `final_hash` was computed | Hash chain + final hash |
| **Round linkage** | Rounds in correct order, none inserted/removed | `previous_round_hash` chain |
| **Participant authorization** | Each round signed by stated public key | Ed25519 signatures |
| **Bundle integrity** | Files match recorded checksums | `checksums.sha256` |

### What ACTIS Does NOT Prove

| Non-guarantee | Why | What would be needed |
|---------------|-----|---------------------|
| **Content correctness** | Verifies `message_hash` exists, not message truth | Application-layer validation |
| **Key authenticity** | Verifies signature matches key, not key identity | External identity layer (PKI, DID, etc.) |
| **Liveness** | Timestamps present but not bound to external time | Trusted timestamping service |
| **Completeness** | Producer could omit rounds before bundling | Higher-level protocol commitments |

### Threat Model

```
Attack                         Defense                          Detection
──────                         ───────                          ─────────

Modify transcript field    →   round_hash changes           →   hash_chain_ok = false
                                                                 → ACTIS_NONCOMPLIANT

Reorder rounds             →   previous_round_hash breaks   →   hash_chain_ok = false
                                                                 → ACTIS_NONCOMPLIANT

Insert/remove round        →   Chain linkage breaks         →   hash_chain_ok = false
                                                                 → ACTIS_NONCOMPLIANT

Strip signatures           →   Signature verification fails →   signatures_ok = false
                                                                 → ACTIS_PARTIAL

Transplant rounds from     →   Genesis hash mismatch        →   hash_chain_ok = false
another transcript              (bound to intent_id)             → ACTIS_NONCOMPLIANT

Modify bundle files        →   Checksum mismatch            →   checksums_ok = false
                                                                 → ACTIS_NONCOMPLIANT

ZIP path traversal         →   Rejected before processing   →   ACTIS_NONCOMPLIANT

ZIP duplicate entries      →   Rejected before processing   →   ACTIS_NONCOMPLIANT

Cross-protocol sig reuse   →   Domain prefix "ACTIS/v1"     →   Signature invalid
                                prevents reuse                   → ACTIS_PARTIAL
```

### Known Limitation: Equivocation

ACTIS verifies that a **presented** bundle is internally intact. It does **not** prove that no alternate valid transcript exists for the same transaction. An actor with a valid signing key could produce multiple transcripts that each verify independently.

Preventing equivocation (split-view attacks) requires external mechanisms:
- Bilateral counter-signing
- Append-only transparency log anchoring
- External transaction registry binding

This is explicitly out of ACTIS v1.0 scope.

---

## Conformance Corpus

ACTIS ships with 11 test vectors. An implementation is conformant if and only if it passes all 11.

### Test Vectors

| Vector | Description | Expected Status | Tests |
|--------|-------------|-----------------|-------|
| **tv-001** | Compatible minimal transcript | `ACTIS_COMPATIBLE` | Happy path |
| **tv-002** | Invalid signature, intact chain | `ACTIS_PARTIAL` | Signature ≠ hash chain independence |
| **tv-003** | Wrong `transcript_version` | `ACTIS_NONCOMPLIANT` | Schema validation |
| **tv-004** | Hash chain break | `ACTIS_NONCOMPLIANT` | Chain linkage |
| **tv-005** | Checksum tamper | `ACTIS_NONCOMPLIANT` | Bundle integrity |
| **tv-006** | Missing manifest | `ACTIS_NONCOMPLIANT` | Bundle layout |
| **tv-007** | Compatible with failure event | `ACTIS_COMPATIBLE` | REJECT/ABORT rounds are valid |
| **tv-008** | All signatures cryptographically invalid | `ACTIS_PARTIAL` | Zero valid sigs ≠ noncompliant |
| **tv-009** | Incorrect `final_hash` | `ACTIS_NONCOMPLIANT` | Final hash verification |
| **tv-010** | Duplicate core file in ZIP | `ACTIS_NONCOMPLIANT` | ZIP security |
| **tv-011** | Path traversal in ZIP entry | `ACTIS_NONCOMPLIANT` | ZIP security |

### Critical Invariants Encoded in Corpus

- **tv-002, tv-008:** `hash_chain_ok` MUST be `true` even when signatures are invalid — checks are independent
- **tv-002, tv-008:** Status MUST be `ACTIS_PARTIAL` (not `ACTIS_NONCOMPLIANT`) — bad sigs don't break structural integrity
- **tv-006:** Missing manifest → `schema_ok = false` — layout is part of schema check

### Running Conformance

```bash
ACTIS_VERIFIER_CMD="your-verifier <zip-path>" \
  bash actis/test-vectors/run_conformance.sh
```

### Implementations

Three independent conformant implementations exist (no shared code):

| Language | Purpose |
|----------|---------|
| **Rust** | Performance-critical / production |
| **TypeScript** | Node.js / web integration |
| **Python** | Data science / scripting |

---

## Where ACTIS Fits in the Agentic Commerce Stack

ACTIS is not a payment system, a commerce platform, or a settlement protocol. It's an **evidence layer** that sits alongside or on top of transaction systems.

```
+-----------------------------------------------------------------------+
|                        AGENTIC COMMERCE STACK                          |
|                                                                        |
|  LAYER 4: EVIDENCE & ACCOUNTABILITY                                    |
|  ┌─────────────────────────────────────────────────────────────────┐   |
|  │  ACTIS                                                          │   |
|  │  Tamper-evident transaction transcripts                         │   |
|  │  Hash-linked, signed, offline-verifiable                        │   |
|  │                                                                 │   |
|  │  Consumers: auditors, regulators, dispute resolution,           │   |
|  │  compliance teams, counterparties                               │   |
|  └─────────────────────────────────────────────────────────────────┘   |
|                                 ▲                                      |
|                                 │ Records what happened                |
|                                 │                                      |
|  LAYER 3: SHOPPING & CHECKOUT                                          |
|  ┌─────────────────────────────────────────────────────────────────┐   |
|  │  Rye          │  Purch         │  Sponge (Amazon)               │   |
|  │  Universal    │  x402 + AI     │  Amazon checkout               │   |
|  │  checkout via │  discovery +   │  via stored                    │   |
|  │  RyeBot       │  Crossmint     │  credit card                   │   |
|  └─────────────────────────────────────────────────────────────────┘   |
|                                 ▲                                      |
|                                 │ Executes purchases                   |
|                                 │                                      |
|  LAYER 2: WALLET & PAYMENT INFRASTRUCTURE                              |
|  ┌─────────────────────────────────────────────────────────────────┐   |
|  │  Locus         │  Sponge        │  Crossmint                    │   |
|  │  ERC-4337      │  Managed       │  Smart wallets                │   |
|  │  smart wallets │  EVM + Solana  │  50+ chains                   │   |
|  │  on Base       │  wallets       │  + fiat                       │   |
|  │  (USDC)        │  (multi-token) │                               │   |
|  └─────────────────────────────────────────────────────────────────┘   |
|                                 ▲                                      |
|                                 │ Holds and moves money                |
|                                 │                                      |
|  LAYER 1: AGENT RUNTIME                                                |
|  ┌─────────────────────────────────────────────────────────────────┐   |
|  │  Claude  │  GPT  │  Custom agents  │  Multi-agent systems       │   |
|  │  MCP     │  Tool calling  │  Autonomous decision-making         │   |
|  └─────────────────────────────────────────────────────────────────┘   |
+-----------------------------------------------------------------------+
```

### Example: ACTIS + Agentic Purchase

```
Agent (Claude)                   Rye/Purch/Sponge          ACTIS Producer
     |                                |                         |
     |  1. "Buy wireless earbuds"     |                         |
     |------------------------------->|                         |
     |                                |                         |
     |                                |  2. Record INTENT round |
     |                                |------------------------>|
     |                                |                         |
     |  3. Offer: $49.99 + tax        |                         |
     |<-------------------------------|                         |
     |                                |  4. Record ASK round    |
     |                                |------------------------>|
     |                                |                         |
     |  5. Agent approves             |                         |
     |------------------------------->|                         |
     |                                |  6. Record ACCEPT round |
     |                                |------------------------>|
     |                                |                         |
     |  7. Payment + order placed     |                         |
     |<-------------------------------|                         |
     |                                |  8. Seal transcript     |
     |                                |  (compute final_hash,   |
     |                                |   create bundle.zip)    |
     |                                |------------------------>|
     |                                |                         |
     |                                |                    bundle.zip
     |                                |                    (evidence)
     |                                |                         |
     |                                |                         v
     |                                |              Stored for audit,
     |                                |              dispute resolution,
     |                                |              regulatory review
```

### Regulatory Alignment

The ACTIS repo includes mappings to:
- **EU AI Act** (`ACTIS_EU_AI_ACT_ALIGNMENT.md`) — how ACTIS evidence satisfies AI system traceability requirements
- **NIST AI RMF** (`ACTIS_NIST_AI_RMF_MAPPING.md`) — mapping to NIST AI Risk Management Framework

---

## Key Diagrams

### Complete Verification Pipeline

```
                    bundle.zip
                        |
                        v
              +---------+---------+
              | Unzip + Security  |
              | (path traversal,  |
              |  duplicates,      |
              |  missing manifest)|
              +---------+---------+
                        |
              +---------+---------+
              |                   |
              v                   v
     manifest.json        input/transcript.json        checksums.sha256
              |                   |                          |
              v                   v                          v
     +--------+------+   +-------+--------+         +-------+--------+
     | Validate core |   | Check 1:       |         | Check 4:       |
     | file list     |   | Schema         |         | SHA-256 of     |
     +--------+------+   | validation     |         | each core file |
              |           +-------+--------+         | vs stored hash |
              |                   |                  +-------+--------+
              |                   v                          |
              |           +-------+--------+                 v
              |           | Check 2:       |           checksums_ok
              |           | Hash Chain     |
              |           | (genesis →     |
              |           |  linkage →     |
              |           |  round_hash →  |
              |           |  final_hash)   |
              |           +-------+--------+
              |                   |
              |                   v
              |           hash_chain_ok
              |           replay_ok
              |                   |
              |                   v
              |           +-------+--------+
              |           | Check 3:       |
              |           | Signatures     |
              |           | (Ed25519 per   |
              |           |  round)        |
              |           +-------+--------+
              |                   |
              |                   v
              |           signatures_ok
              |                   |
              +-------------------+
                        |
                        v
              +---------+---------+
              | Status Derivation |
              |                   |
              | All pass →        |
              |   ACTIS_COMPATIBLE|
              |                   |
              | Struct ok, sigs   |
              | fail →            |
              |   ACTIS_PARTIAL   |
              |                   |
              | Struct broken →   |
              |   ACTIS_          |
              |   NONCOMPLIANT    |
              +---------+---------+
                        |
                        v
              Verification Report (JSON)
```

### Hash Chain Integrity

```
intent_id + ":" + created_at_ms
              |
              v
        SHA-256 (genesis)
              |
              v
+─────────────+──────────────+
| Round 0     |              |
| INTENT      | previous =   |
|             | genesis_hash |
|             |              |
| round_hash =|              |
| SHA-256(JCS)|              |
+─────────────+──────────────+
              |
              v  round_hash flows down
+─────────────+──────────────+
| Round 1     |              |
| ASK         | previous =   |
|             | round_0_hash |
|             |              |
| round_hash =|              |
| SHA-256(JCS)|              |
+─────────────+──────────────+
              |
              v
+─────────────+──────────────+
| Round 2     |              |
| BID         | previous =   |
|             | round_1_hash |
+─────────────+──────────────+
              |
              v
+─────────────+──────────────+
| Round 3     |              |
| ACCEPT      | previous =   |
|             | round_2_hash |
+─────────────+──────────────+
              |
              v
        final_hash = SHA-256(JCS(entire transcript))
              |
              v
        TAMPER-EVIDENT SEAL

  Modify ANY field in ANY round
        → round_hash changes
        → next round's previous_round_hash mismatches
        → hash_chain_ok = false
        → ACTIS_NONCOMPLIANT
```

---

## Summary

ACTIS provides the **evidence layer** for agentic commerce:

1. **Transcript format** — hash-linked, Ed25519-signed JSON records of agent transaction sessions with 7 round types (INTENT, ASK, BID, COUNTER, ACCEPT, REJECT, ABORT)
2. **Hash chain integrity** — SHA-256 linked rounds with genesis binding (prevents transplantation), final hash seal (commits entire transcript), and JCS canonicalization (deterministic across all implementations)
3. **Signature model** — domain-prefixed Ed25519 signatures (`"ACTIS/v1"`) over envelope hashes, independent from hash chain verification (two separate failure dimensions)
4. **Bundle format** — portable ZIP archives (manifest + transcript + checksums) with security rules against path traversal and duplicate entries
5. **Verification model** — four independent checks (schema, hash chain, signatures, checksums) producing deterministic three-tier status (COMPATIBLE / PARTIAL / NONCOMPLIANT)
6. **Conformance corpus** — 11 test vectors encoding critical invariants; binary pass/fail; three independent implementations (Rust, TypeScript, Python)
7. **Offline verifiable** — no network, no trusted service, no blockchain required
8. **Vendor neutral** — Apache 2.0 licensed, patent non-assert, implementable from spec alone

ACTIS answers one question: **given this record of what an agent did, can we prove the record hasn't been tampered with?** Everything else — identity, correctness, reputation, adjudication — is explicitly out of scope.
