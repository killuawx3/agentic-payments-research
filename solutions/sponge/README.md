# Sponge Wallet: Crypto-Financial Toolkit for AI Agents

> **Multi-chain wallet + paid API access + DeFi trading + shopping — all from one agent identity.**

Sponge Wallet is a crypto-financial platform for AI agents that combines wallet management, cross-chain operations, paid third-party API access (via x402 protocol), prediction market trading, and shopping capabilities.

---

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Core Components](#core-components)
  - [Multi-Chain Wallet](#1-multi-chain-wallet)
  - [Paid Services (x402)](#2-paid-services-x402)
  - [Prediction Markets](#3-prediction-markets)
  - [Shopping & Prepaid Cards](#4-shopping--prepaid-cards)
  - [Planning API](#5-planning-api)
  - [Secret Storage](#6-secret-storage)
- [Authentication & Agent Onboarding](#authentication--agent-onboarding)
- [Key Differences from Locus](#key-differences-from-locus)

---

## Architecture Overview

```
+------------------------------------------------------------------+
|                        AI AGENT                                   |
|  Authenticates via sponge_live_* API key                         |
|  Required header: Sponge-Version: 0.2.1                          |
+------+--------+--------+--------+--------+--------+-------------+
       |        |        |        |        |        |
       v        v        v        v        v        v
  +--------+ +------+ +-------+ +-------+ +------+ +------+
  | Wallet | | x402 | |Poly-  | |Amazon | |Bridge| | Swap |
  | (Multi)| |(APIs)| |market | |Shop   | |(deB) | |(Jup/ |
  |        | |      | |       | |       | |      | | 0x)  |
  +--------+ +------+ +-------+ +-------+ +------+ +------+
       |        |        |        |        |        |
       +--------+--------+--------+--------+--------+
                         |
              Multi-Chain (EVM + Solana)
              EOA Wallets (not smart wallets)
```

### How It Works

1. **Agent registers** via `/api/agents/register` (agent-first mode gives immediate API key)
2. **Human claims** agent via verification URL (optional, links to dashboard)
3. **Agent operates** — checks balances, calls APIs, trades markets, shops
4. **USDC on Base** is the primary payment token for x402 services
5. **All chains share** the same EVM address (EOA), Solana has separate address

---

## Core Components

### 1. Multi-Chain Wallet

EOA wallets across 10+ networks, all managed by the same agent identity.

| Chain | Type | Features |
|-------|------|----------|
| Ethereum | EOA | Balances, transfers |
| Base | EOA | Balances, transfers, USDC payments, swaps (0x) |
| Polygon | EOA | Balances, transfers |
| Arbitrum One | EOA | Balances, transfers |
| Monad | EOA | Balances, transfers |
| Solana | EOA | Balances, transfers, swaps (Jupiter) |
| Sepolia / Base Sepolia | EOA | Testnet |
| Hyperliquid | EOA | Perps and spot trading |
| Tempo Testnet | EOA | Testnet |

**Key difference from Locus:** Sponge uses standard EOA wallets (not ERC-4337 smart wallets). No paymaster, no gasless transactions, no on-chain spending controls.

### 2. Paid Services (x402)

19 third-party services accessible via the x402 payment protocol. Agent pays per-call in USDC.

**3-step discovery flow:**
1. `GET /api/discover` — list all services
2. `GET /api/discover/:serviceId` — get endpoints, pricing, baseUrl
3. `POST /api/x402/fetch` — make the call (Sponge handles payment handshake)

### 3. Prediction Markets

- **Polymarket** — search markets, check status, place orders
- **Hyperliquid** — perps and spot trading
- Safe wallets auto-provisioned on first Polymarket use

### 4. Shopping & Prepaid Cards

- **Amazon Checkout** — search and purchase (supports dryRun preview)
- **Prepaid Visa Cards** via Laso Finance ($5–$1000, US only)

### 5. Planning API

Multi-step operation planner — submit plans for human approval before execution.

### 6. Secret Storage

Encrypted storage for credit cards and API keys.

---

## Authentication & Agent Onboarding

```
Agent                           Sponge API
  |                                 |
  |-- POST /agents/register ------->|
  |   { name, agentFirst: true }    |
  |                                 |
  |<-- apiKey (immediate) ----------|
  |    agentId, claimCode,          |
  |    verificationUrl              |
  |                                 |
  |-- Use API key for all calls --->|
  |   + Sponge-Version header       |
  |                                 |
  Human visits verificationUrl to claim agent (optional)
```

**Key format:** `sponge_live_*` (production) or `sponge_test_*` (testnet)

---

## Key Differences from Locus

| Aspect | Sponge | Locus |
|--------|--------|-------|
| Wallet type | EOA (standard) | ERC-4337 (smart wallet) |
| Gas handling | Agent pays gas | Gasless (paymaster) |
| Chains | 10+ (EVM + Solana) | Base only |
| Spending controls | None (on-chain) | Three-tier governance |
| x402 services | 19 providers | 30+ wrapped APIs |
| DeFi | Swaps, bridges, prediction markets | None |
| Shopping | Amazon, prepaid cards | Prepaid cards, Venmo/PayPal |
| PaaS deployment | None | Build with Locus |
| Checkout SDK | None | Stripe-like checkout |
| Human approval | Planning API | Spending threshold + dashboard |
