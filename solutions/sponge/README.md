# Sponge: Financial Infrastructure for the Agent Economy

> **The easiest way for agents to hold and spend money, and for businesses to sell directly to them.**

Sponge is a multi-chain crypto wallet platform purpose-built for AI agents. It provides managed wallets across **Ethereum, Base, and Solana** that enable agents to hold balances, transfer tokens, execute swaps, bridge cross-chain, trade on prediction and perpetual markets, access paid third-party services via x402 micropayments, purchase prepaid Visa cards, and shop on Amazon — all governed by human-configured spending controls and address allowlists.

---

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Core Components](#core-components)
  - [Managed Wallets (EVM + Solana)](#1-managed-wallets-evm--solana)
  - [Token Transfers](#2-token-transfers)
  - [Token Swaps](#3-token-swaps)
  - [Cross-Chain Bridges](#4-cross-chain-bridges)
  - [Paid External Services (x402)](#5-paid-external-services-x402)
  - [Polymarket (Prediction Markets)](#6-polymarket-prediction-markets)
  - [Hyperliquid (Perps & Spot Trading)](#7-hyperliquid-perps--spot-trading)
  - [Prepaid Visa Cards (Laso Finance)](#8-prepaid-visa-cards-laso-finance)
  - [Amazon Checkout](#9-amazon-checkout)
  - [Payment Links](#10-payment-links)
- [Spending Controls & Governance](#spending-controls--governance)
- [Authentication & Agent Onboarding](#authentication--agent-onboarding)
- [Claude Integration (MCP & Direct Tools)](#claude-integration-mcp--direct-tools)
- [Supported Chains](#supported-chains)
- [Provider Catalog (x402 Services)](#provider-catalog-x402-services)
- [SDK Reference](#sdk-reference)
- [Key Diagrams](#key-diagrams)

---

## Architecture Overview

```
+------------------------------------------------------------------+
|                        HUMAN OWNER                                |
|  (Dashboard: wallet.paysponge.com)                               |
|  - Approve agent registration (claim URL)                        |
|  - Set spending limits (per-tx, daily, weekly, monthly)          |
|  - Configure address allowlists                                  |
|  - Approve funding requests                                      |
|  - View audit logs & transaction history                         |
|  - Pause / resume / delete agents                                |
+----------------------------------+-------------------------------+
                                   |
                          Governance & Controls
                                   |
                                   v
+------------------------------------------------------------------+
|                         AI AGENT                                  |
|  Authenticates via sponge_live_* / sponge_test_* API key         |
|  SDK: @spongewallet/sdk  |  MCP: @spongewallet/mcp              |
+------+--------+--------+--------+--------+--------+-------------+
       |        |        |        |        |        |
       v        v        v        v        v        v
  +--------+ +------+ +-------+ +------+ +------+ +------+
  | Wallet | | Swaps| |Bridge | |x402  | |Market| | Fiat |
  |(Bal/Tx)| |(Jup/ | |(deBr.)| |(Paid)| |(Poly/| |(Visa/|
  |        | | 0x)  | |       | |(APIs)| | HyLq)| | Amzn)|
  +--------+ +------+ +-------+ +------+ +------+ +------+
       |        |        |        |        |        |
       +--------+--------+--------+--------+--------+
                         |
              Multi-Chain Settlement
           Ethereum  |  Base  |  Solana
```

### How It All Connects

1. **Human** creates account at wallet.paysponge.com, approves agent via claim URL
2. **Agent** receives API key, authenticates with `Authorization: Bearer <key>` + `Sponge-Version: 0.2.1`
3. **Agent** performs actions (transfers, swaps, bridges, x402 API calls, trading, purchases)
4. **Spending controls** enforce per-tx, daily, weekly, monthly limits; allowlists restrict destinations
5. **All transactions** settle on-chain across Ethereum, Base, or Solana

---

## Core Components

### 1. Managed Wallets (EVM + Solana)

Each Sponge agent receives **two wallets** automatically upon registration — one EVM wallet (shared address across Ethereum, Base, and all EVM chains) and one Solana wallet.

#### Wallet Architecture

```
+---------------------------------------------------------------+
|                     Sponge Agent Wallet                        |
|                                                                |
|  +---------------------------+  +---------------------------+  |
|  |       EVM Wallet          |  |     Solana Wallet         |  |
|  |  (Ethereum, Base, etc.)   |  |  (Mainnet / Devnet)      |  |
|  |                           |  |                           |  |
|  |  Address: 0x1234...abcd   |  |  Address: 7nYBbH...xyz   |  |
|  |  Same address on ALL      |  |  SOL + SPL tokens        |  |
|  |  EVM chains               |  |  Jupiter DEX access      |  |
|  |                           |  |                           |  |
|  |  Tokens: ETH, USDC        |  |  Tokens: SOL, USDC,      |  |
|  |  (per chain)              |  |  any SPL token           |  |
|  +---------------------------+  +---------------------------+  |
|                                                                |
|  Key Properties:                                               |
|  - Managed by Sponge (server-side key management)              |
|  - Human owner approves agent via claim URL                    |
|  - Owner can pause/resume/delete agent at any time             |
|  - Allowlists restrict transfer destinations                   |
|  - Spending limits enforced per-tx, daily, weekly, monthly     |
+----------------------------------------------------------------+
```

#### Wallet Operations

| Category | SDK Method | API Endpoint |
|----------|-----------|--------------|
| **Addresses** | `wallet.getAddresses()` | — |
| **All Balances** | `wallet.getBalances()` | `GET /api/balances` |
| **Chain Balance** | `wallet.getBalance(chain)` | `GET /api/balances?chain=base` |
| **Solana Tokens** | `wallet.getSolanaTokens()` | `GET /api/solana/tokens` |
| **Token Search** | — | `GET /api/solana/tokens/search?query=...` |
| **Tx History** | `wallet.getTransactionHistory()` | `GET /api/transactions/history` |
| **Tx Status** | `wallet.getTransactionStatus()` | `GET /api/transactions/status/{txHash}` |
| **Request Funding** | `wallet.requestFunding()` | `POST /api/funding-requests` |
| **Withdraw** | `wallet.withdraw()` | `POST /api/wallets/withdraw-to-main` |
| **Faucet (testnet)** | `wallet.requestFaucet()` | — |

---

### 2. Token Transfers

Two transfer methods depending on the destination chain:

#### EVM Transfers (Ethereum, Base)

```
Agent --> Sponge API --> On-chain transfer --> Recipient wallet
         POST /api/transfers/evm
         { chain, to, amount, currency }
```

```bash
curl -sS -X POST "$SPONGE_API_URL/api/transfers/evm" \
  -H "Authorization: Bearer $SPONGE_API_KEY" \
  -H "Sponge-Version: 0.2.1" \
  -H "Content-Type: application/json" \
  -d '{"chain":"base","to":"0x...","amount":"10","currency":"USDC"}'
```

#### Solana Transfers

```
Agent --> Sponge API --> On-chain transfer --> Recipient wallet
         POST /api/transfers/solana
         { chain, to, amount, currency }
```

**Key constraints:**
- Recipients must be **pre-approved in the owner's allowlist** — transfers to unapproved addresses will fail (HTTP 403)
- Spending limits checked before execution
- Returns transaction hash + explorer URL on success

---

### 3. Token Swaps

Sponge supports token swaps on both Solana (via Jupiter aggregator) and Base (via 0x protocol).

#### Swap Architecture

```
Agent                        Sponge API                  DEX Aggregator
  |                              |                            |
  |-- POST /transactions/swap ->|                            |
  |   (inputToken, outputToken, |                            |
  |    amount, slippageBps)     |                            |
  |                              |-- Route via Jupiter/0x -->|
  |                              |                            |
  |                              |<-- Best price route ------|
  |                              |                            |
  |                              |-- Execute swap on-chain   |
  |                              |                            |
  |<-- SwapResult --------------|                            |
  |   (txHash, amountOut,       |                            |
  |    explorerUrl)             |                            |
```

| Chain | Endpoint | Aggregator |
|-------|----------|------------|
| **Solana** | `POST /api/transactions/swap` | Jupiter |
| **Base** | `POST /api/transactions/base-swap` | 0x Protocol |

**Parameters:** `chain`, `inputToken`, `outputToken`, `amount`, `slippageBps` (basis points, e.g. 100 = 1%)

#### Swap with Quote (SDK)

```typescript
const quote = await wallet.getSwapQuote({
  chain: "solana",
  fromToken: "SOL",
  toToken: "USDC",
  amount: "1.0"
});
console.log(`Expected: ${quote.expectedOutput} USDC`);
console.log(`Price impact: ${quote.priceImpact}%`);

const swap = await wallet.executeSwap(quote);
```

---

### 4. Cross-Chain Bridges

Bridge tokens between chains via deBridge integration.

```
Agent                        Sponge API                  deBridge
  |                              |                          |
  |-- POST /transactions/bridge->|                          |
  |   { sourceChain,            |                          |
  |     destinationChain,       |-- Bridge request ------->|
  |     token, amount,          |                          |
  |     destinationToken }      |<-- Bridge execution -----|
  |                              |                          |
  |<-- BridgeResult ------------|                          |
```

```bash
curl -sS -X POST "$SPONGE_API_URL/api/transactions/bridge" \
  -H "Authorization: Bearer $SPONGE_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "sourceChain":"solana",
    "destinationChain":"base",
    "token":"SOL",
    "amount":"0.1",
    "destinationToken":"ETH"
  }'
```

A special MCP tool `consolidate_usdc` can consolidate USDC from all chains into one.

---

### 5. Paid External Services (x402)

Agents can access third-party APIs (search, image generation, web scraping, AI models) through a **three-step x402 payment workflow**. Sponge handles authentication and micropayments automatically.

#### x402 Workflow

```
Agent                           Sponge API                   External Service
  |                                 |                              |
  |-- 1. GET /api/discover -------->|                              |
  |   ?query=web+search            |                              |
  |<-- Service catalog ------------|                              |
  |   [{id, name, category}]       |                              |
  |                                 |                              |
  |-- 2. GET /api/discover/{id} -->|                              |
  |<-- Endpoints + pricing --------|                              |
  |   (paymentsProtocolConfig,     |                              |
  |    endpoint paths, params)     |                              |
  |                                 |                              |
  |-- 3. POST /api/x402/fetch ---->|                              |
  |   { url, method, body,         |-- Auto-pay USDC ----------->|
  |     preferred_chain }          |                              |
  |                                 |<-- Service response --------|
  |                                 |                              |
  |<-- Result + payment_details ---|                              |
```

**Critical:** Step 2 (inspect) is **mandatory** — direct API URLs won't work because Sponge proxies authentication. Skipping it will fail.

#### Discovery Endpoints

```
GET /api/discover                          # List all services
GET /api/discover?query=image&category=image  # Filter by query + category
GET /api/discover/{serviceId}              # Get endpoints, pricing, payment config
POST /api/x402/fetch                       # Execute with auto-payment
```

#### x402 Categories

| Category | Description |
|----------|-------------|
| `search` | Web and semantic search |
| `image` | AI image generation |
| `llm` | Language model inference |
| `crawl` | Web scraping and crawling |
| `data` | Data enrichment and lookup |
| `predict` | Prediction markets data |
| `parse` | Document parsing |
| `prospect` | Sales prospecting |
| `person_search` | People/contact search |
| `crypto_data` | Blockchain data |

---

### 6. Polymarket (Prediction Markets)

Agents can trade on Polymarket prediction markets through a unified action-based API.

#### Polymarket Flow

```
Agent                           Sponge API                  Polymarket
  |                                 |                           |
  |-- POST /api/polymarket -------->|                           |
  |   { action: "search_markets",  |                           |
  |     query: "election" }        |-- Query markets --------->|
  |                                 |                           |
  |<-- Market list + prices --------|<-- Market data ----------|
  |                                 |                           |
  |-- POST /api/polymarket -------->|                           |
  |   { action: "order",           |                           |
  |     market_slug, outcome,      |-- Place order ----------->|
  |     side: "buy", size: 10 }    |                           |
  |                                 |                           |
  |<-- Order confirmation ----------|<-- Fill confirmation ----|
```

#### Polymarket Actions

| Action | Required Params | Description |
|--------|----------------|-------------|
| `status` | — | Check connection status |
| `search_markets` | `query` | Search available markets |
| `get_market` | `market_slug` | Get market details + prices |
| `positions` | — | View open positions |
| `orders` | — | View active orders |
| `order` | `outcome`, `side`, `size`, `market_slug` or `token_id` | Place buy/sell order |
| `cancel` | `order_id` | Cancel an order |
| `set_allowances` | — | Set token allowances |
| `deposit` | — | Deposit to Polymarket |
| `deposit_from_wallet` | `amount` | Deposit USDC from agent wallet |
| `withdraw` | `amount` | Withdraw USDC |

**Key calculation:** For dollar-amount orders, calculate shares as `floor(usdAmount / price)`.

---

### 7. Hyperliquid (Perps & Spot Trading)

Agents can trade perpetual futures and spot markets on Hyperliquid via a unified action-based API.

#### Hyperliquid Actions

| Action | Required Params | Description |
|--------|----------------|-------------|
| `status` | — | Check connection status |
| `positions` | — | View open positions |
| `orders` | `symbol` (optional) | View active orders |
| `fills` | `symbol`, `since`, `limit` (optional) | View trade fills |
| `markets` | — | List available markets |
| `ticker` | `symbol` | Get ticker price |
| `orderbook` | `symbol` | Get orderbook depth |
| `funding` | `symbol` (optional) | View funding rates |
| `order` | `symbol`, `side`, `type`, `amount` | Place order |
| `cancel` | `order_id`, `symbol` | Cancel order |
| `cancel_all` | `symbol` (optional) | Cancel all orders |
| `set_leverage` | `symbol`, `leverage` | Set leverage |
| `withdraw` | `amount`, `destination` | Withdraw funds |
| `transfer` | `amount`, `to_perp` | Transfer between spot/perp |

**Order types:** market, limit (requires `price`), stop (requires `trigger_price`), take-profit/stop-loss (via `tp_sl`)

Uses EVM wallet signing automatically. Deposits/withdrawals use the bridge endpoint.

---

### 8. Prepaid Visa Cards (Laso Finance)

Agents can order prepaid Visa cards funded with USDC. US-only, non-reloadable.

```
Agent (USDC wallet)
   |
   |-- order_prepaid_card(amount)     # $5 - $1,000
   |   (charges USDC immediately)
   |
   |-- get_prepaid_card(card_id)      # Poll every 2-3 seconds
   |   (ready in ~7-10 seconds)
   |   Returns: card_number, CVV, expiration
   |
   |-- search_prepaid_card_merchants(query)
       (check compatibility BEFORE ordering)
```

**Critical constraints:**
- Cards are **non-reloadable** and **non-refundable** — order with exact purchase amount
- US-only (IP-locked)
- $5 minimum, $1,000 maximum
- Always check merchant compatibility before ordering

#### Merchant Compatibility

| Status | Meaning |
|--------|---------|
| `accepted` | Confirmed working |
| `not_accepted` | Documented declines — don't order |
| `unknown` | Unclear — proceed with caution |
| Not listed | No prior data — presumed compatible |

---

### 9. Amazon Checkout

Agents can search for and purchase products on Amazon using stored credit card credentials.

#### Checkout Flow

```
Agent                           Sponge API                  Amazon
  |                                 |                          |
  |-- POST /api/checkout/          |                          |
  |   amazon-search                |                          |
  |   { query: "USB-C cable" } --->|-- Search products ------>|
  |                                 |                          |
  |<-- Product results ------------|<-- Results --------------|
  |                                 |                          |
  |-- POST /api/checkout ----------|                          |
  |   { items, shippingAddr,      |-- Initiate purchase ---->|
  |     dryRun: false }           |                          |
  |                                 |                          |
  |-- GET /api/checkout/{id} ----->|-- Poll status            |
  |                                 |                          |
  |<-- Status: completed ----------|                          |
```

**Status flow:** `pending` → `in_progress` → `completed` | `failed` | `cancelled`

**Options:** `dryRun` (simulate without purchasing), `clearCart` (empty cart before adding items)

**Requires:** Credit card stored via `POST /api/credit-cards` (encrypted, snake_case fields: `card_number`, `cvc`, `cardholder_name`, etc.)

---

### 10. Payment Links

Create reusable x402 payment links for receiving USDC payments from other agents or users.

```
POST /api/payment-links
{
  "amount": "5.00",
  "description": "API access fee",
  "max_uses": 100,
  "expires_in_minutes": 1440,
  "callback_url": "https://myapp.com/webhook",
  "livemode": true
}

GET /api/payment-links/{paymentLinkId}   # Check payment status
```

---

## Spending Controls & Governance

Sponge implements a **multi-tier spending control** system configured by human owners through the dashboard or via the Master Key admin API.

```
+-------------------------------------------------------------------+
|                    Spending Controls                               |
|                                                                   |
|  +----------------+ +----------------+ +----------------+         |
|  | Per-Transaction| | Time-Based     | | Address        |         |
|  | Limit          | | Limits         | | Allowlist      |         |
|  |                | |                | |                |         |
|  | Max USD per    | | per_minute     | | Only approved  |         |
|  | single tx      | | hourly         | | destinations   |         |
|  |                | | daily          | | can receive    |         |
|  | Exceeding      | | weekly         | | transfers      |         |
|  | returns 403    | | monthly        | |                |         |
|  |                | |                | | Unapproved     |         |
|  |                | | Exceeding any  | | returns 403    |         |
|  |                | | returns 403    | |                |         |
|  +----------------+ +----------------+ +----------------+         |
|                                                                   |
|  +----------------+ +----------------+                            |
|  | Agent Pause    | | Agent Delete   |                            |
|  |                | |                |                            |
|  | Owner can      | | Permanent,     |                            |
|  | pause/resume   | | revokes all    |                            |
|  | all operations | | API keys       |                            |
|  +----------------+ +----------------+                            |
|                                                                   |
|  Applied across: Transfers, Swaps, Bridges, x402, Trading,       |
|  Cards, Amazon Checkout                                           |
+-------------------------------------------------------------------+
```

#### Setting Spending Limits (Admin SDK)

```typescript
await admin.setSpendingLimit("agent_id", {
  type: "daily",        // per_transaction | per_minute | hourly | daily | weekly | monthly
  amount: "100.0",
  currency: "USD"
});
```

#### Managing Allowlists

```typescript
// Add approved destination
await admin.addToAllowlist("agent_id", {
  chain: "base",
  address: "0xTrustedAddress",
  label: "Treasury"
});

// View allowlist
const list = await admin.getAgentAllowlist("agent_id");

// Remove entry
await admin.removeFromAllowlist("agent_id", entryId);
```

---

## Authentication & Agent Onboarding

### Agent Registration (Agent-First Mode)

```
Agent                           Sponge API
  |                                  |
  |-- POST /api/agents/register ---->|
  |   { name: "MyAgent",           |
  |     agentFirst: true }          |
  |                                  |
  |<-- apiKey (immediate), ---------|
  |    claimCode, claimUrl,         |
  |    deviceCode, expiresIn        |
  |    (SAVE IMMEDIATELY)           |
  |                                  |
  |-- Share claimUrl with human ---->  Human approves at claim URL
  |                                  |
  |-- Agent can operate immediately  |
  |   (with default limits)         |
```

### Device Flow (Human-Initiated)

Standard OAuth 2.0 Device Authorization Grant (RFC 8628):

1. Call `SpongeWallet.connect()` — displays verification URL + code
2. Human visits URL, enters code, approves
3. SDK polls `POST /api/oauth/device/token` until approved
4. API key returned and stored locally

### API Key Types

| Key Type | Prefix | Access | Purpose |
|----------|--------|--------|---------|
| **Test** | `sponge_test_` | Testnets only | Development |
| **Live** | `sponge_live_` | Mainnets only | Production |
| **Master** | `master_xxx_` | Admin operations | Agent management |

**Test keys cannot access mainnets. Live keys cannot access testnets.**

### Credential Storage

Default locations:
- **macOS:** `~/Library/Application Support/sponge/credentials.json`
- **Linux:** `~/.config/sponge/credentials.json` (or `~/.spongewallet/credentials.json`)
- **Windows:** `%APPDATA%\sponge\credentials.json`

### API Key Scopes

| Scope | Description |
|-------|-------------|
| `wallet:read` | Read wallet addresses and balances |
| `wallet:write` | Create and manage wallets |
| `transaction:read` | View transaction history |
| `transaction:sign` | Sign transactions |
| `transaction:write` | Submit transactions |
| `spending:read` | View spending limits |
| `flow:execute` | Execute automated flows |
| `payment:read` | Read payment methods |
| `payment:decrypt` | Decrypt stored cards |
| `payment:write` | Update payment records |

### Authentication Headers (All Requests)

```
Authorization: Bearer <SPONGE_API_KEY>
Sponge-Version: 0.2.1
Content-Type: application/json
```

---

## Claude Integration (MCP & Direct Tools)

Sponge offers first-class Claude integration through two methods:

| Method | Best For | Requires |
|--------|----------|----------|
| **MCP** | Claude Desktop, Claude Code, MCP clients | MCP-compatible client |
| **Direct Tools** | Custom apps, Anthropic API | Anthropic SDK |

### MCP Integration

```json
{
  "mcpServers": {
    "sponge-wallet": {
      "command": "npx",
      "args": ["@spongewallet/mcp", "--api-key", "YOUR_API_KEY"]
    }
  }
}
```

Or via SDK:

```typescript
import Anthropic from "@anthropic-ai/sdk";
import { SpongeWallet } from "@spongewallet/sdk";

const wallet = await SpongeWallet.connect();
const anthropic = new Anthropic();

const response = await anthropic.messages.create({
  model: "claude-sonnet-4-20250514",
  max_tokens: 1024,
  messages: [{ role: "user", content: "What's my wallet balance?" }],
  mcp_servers: { wallet: wallet.mcp() }
});
```

### Available MCP Tools

| Tool | Description |
|------|-------------|
| `get_balance` | Check balance on one or all chains |
| `evm_transfer` | Transfer ETH/USDC on Ethereum or Base |
| `solana_transfer` | Transfer SOL/USDC on Solana |
| `tempo_transfer` | Transfer pathUSD on Tempo (testnet) |
| `solana_swap` | Swap tokens on Solana via Jupiter |
| `get_solana_tokens` | List all SPL tokens in wallet |
| `search_solana_tokens` | Search Jupiter token database |
| `get_transaction_status` | Check transaction status |
| `get_transaction_history` | View past transactions |
| `request_funding` | Request funds from owner |
| `withdraw_to_main_wallet` | Withdraw to owner's wallet |
| `request_tempo_faucet` | Get testnet tokens |
| `order_prepaid_card` | Order Visa prepaid card |
| `get_prepaid_card` | Retrieve card details |
| `search_prepaid_card_merchants` | Check merchant compatibility |

### Direct Tools Integration

```typescript
const wallet = await SpongeWallet.connect();
const tools = wallet.tools();

const response = await anthropic.messages.create({
  model: "claude-sonnet-4-20250514",
  max_tokens: 1024,
  messages: [{ role: "user", content: "Send 10 USDC to 0x..." }],
  tools: tools.definitions
});

for (const block of response.content) {
  if (block.type === "tool_use") {
    const result = await tools.execute(block.name, block.input);
  }
}
```

### Tool Filtering & Security

```typescript
// Only expose specific tools
const tools = wallet.tools({ include: ["get_balance", "transfer"] });

// Implement approval flow
const tools = wallet.tools({
  beforeExecute: async (toolName, input) => {
    if (toolName === "transfer") {
      const approved = await promptUser(`Approve ${input.amount} ${input.currency}?`);
      if (!approved) throw new Error("User declined");
    }
  }
});
```

### MCP Resources & Prompts

| Type | Name | Description |
|------|------|-------------|
| Resource | `x402-services` | Catalog of x402 payment services |
| Prompt | `sponge-guide` | Decision matrix for Sponge tool usage |
| Prompt | `x402-guide` | x402 payment protocol guide |
| Prompt | `how-to-search` | Web search instructions |

---

## Supported Chains

### Mainnet (Live Keys — `sponge_live_*`)

| Chain | Type | Chain ID | Native Token | USDC |
|-------|------|----------|-------------|------|
| **Ethereum** | EVM | 1 | ETH | `0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48` |
| **Base** | EVM | 8453 | ETH | `0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913` |
| **Solana** | Solana | 101 | SOL | USDC (SPL) |

### Testnet (Test Keys — `sponge_test_*`)

| Chain | Type | Chain ID | Native Token | Notes |
|-------|------|----------|-------------|-------|
| **Sepolia** | EVM | 11155111 | ETH | Ethereum testnet |
| **Base Sepolia** | EVM | 84532 | ETH | Base testnet |
| **Solana Devnet** | Solana | 102 | SOL | Solana testnet |
| **Tempo** | EVM | 42431 | pathUSD | Instant finality testnet |

### Chain Properties

- **EVM address reuse:** Same address across all EVM chains
- **Gas management:** Automatically estimated for EVM; minimal for Solana (~$0.00025/tx)
- **Explorer URLs:** Included in all transaction responses
- **Faucets:** Available on all testnets via `wallet.requestFaucet({ chain })`

---

## Provider Catalog (x402 Services)

Sponge integrates with external paid services through the x402 micropayment protocol. Services are discoverable via `GET /api/discover`.

### Available Service Categories

| Category | Example Providers | Capabilities |
|----------|------------------|--------------|
| **Search** | Exa, web search services | Semantic search, content retrieval |
| **Image** | AI image generators | Image generation and editing |
| **LLM** | Language models | AI text generation |
| **Crawl** | Firecrawl, web scrapers | Web scraping, site crawling |
| **Data** | Data enrichment services | Contact/company enrichment |
| **Predict** | Prediction market data | Market odds and data |
| **Parse** | Document parsers | PDF/document extraction |
| **Prospect** | Sales tools | Lead generation, prospecting |
| **Person Search** | People search, LinkedIn | Contact info lookup |
| **Crypto Data** | Blockchain analytics | On-chain data queries |

---

## SDK Reference

### Installation

```bash
npm install @spongewallet/sdk
# or
bun add @spongewallet/sdk
```

### SpongeWallet Class

| Method | Returns | Description |
|--------|---------|-------------|
| `SpongeWallet.connect(options?)` | `Promise<SpongeWallet>` | Authenticate via device flow |
| `SpongeWallet.fromApiKey(key)` | `Promise<SpongeWallet>` | Connect with existing key |
| `wallet.getAddresses()` | `{ evm, solana }` | Get wallet addresses |
| `wallet.getBalances()` | `Balance[]` | All chain balances |
| `wallet.getBalance(chain)` | `Balance` | Single chain balance |
| `wallet.getSolanaTokens()` | `SolanaToken[]` | SPL token holdings |
| `wallet.transfer(opts)` | `Transaction` | Send crypto |
| `wallet.swap(opts)` | `SwapResult` | Swap tokens |
| `wallet.getSwapQuote(opts)` | `SwapQuote` | Get swap quote |
| `wallet.getTransactionStatus(opts)` | `TransactionStatus` | Check tx status |
| `wallet.getTransactionHistory(opts?)` | `Transaction[]` | Transaction history |
| `wallet.requestFunding(opts)` | `FundingRequest` | Request funds from owner |
| `wallet.withdraw(opts)` | `Transaction` | Withdraw to owner wallet |
| `wallet.requestFaucet(opts)` | `FaucetResult` | Get testnet tokens |
| `wallet.mcp()` | `McpServerConfig` | MCP server config |
| `wallet.tools(opts?)` | `Tools` | Anthropic tool definitions |
| `wallet.disconnect()` | `void` | Clear credentials |
| `wallet.refresh()` | `void` | Force credential refresh |

### SpongeAdmin Class (Master Keys)

| Method | Returns | Description |
|--------|---------|-------------|
| `SpongeAdmin.connect(opts?)` | `Promise<SpongeAdmin>` | Connect with master key |
| `SpongeAdmin.fromApiKey(key)` | `Promise<SpongeAdmin>` | Connect with existing master key |
| `admin.createAgent(opts)` | `CreateAgentResponse` | Create new agent + wallets |
| `admin.listAgents(opts?)` | `Agent[]` | List all agents |
| `admin.getAgent(id)` | `Agent` | Get agent details |
| `admin.deleteAgent(id)` | `void` | Permanently delete agent |
| `admin.pauseAgent(id)` | `void` | Pause all operations |
| `admin.resumeAgent(id)` | `void` | Resume agent |
| `admin.rotateAgentKey(id)` | `KeyRotationResult` | Rotate API key |
| `admin.setSpendingLimit(id, opts)` | `SpendingLimit` | Set spending limit |
| `admin.getSpendingLimits(id)` | `SpendingLimit[]` | Get limits |
| `admin.addToAllowlist(id, opts)` | `AllowlistEntry` | Add allowed address |
| `admin.getAgentAllowlist(id)` | `AllowlistEntry[]` | View allowlist |
| `admin.removeFromAllowlist(id, entryId)` | `void` | Remove entry |
| `admin.getAgentAuditLogs(id, opts?)` | `AuditLog[]` | View audit logs |

### Error Codes

| Code | Error Class | Description |
|------|------------|-------------|
| `UNAUTHORIZED` | `SpongeError` | Invalid/expired API key |
| `INSUFFICIENT_FUNDS` | `InsufficientFundsError` | Not enough balance |
| `INVALID_ADDRESS` | `InvalidAddressError` | Bad recipient address |
| `INVALID_CHAIN` | `SpongeError` | Unsupported chain |
| `SPENDING_LIMIT_EXCEEDED` | `SpendingLimitError` | Over spending limit |
| `ADDRESS_NOT_ALLOWLISTED` | `AllowlistError` | Destination not approved |
| `AGENT_PAUSED` | `SpongeError` | Agent is paused |
| `RATE_LIMITED` | `SpongeError` | Too many requests |

### Environment Variables

| Variable | Purpose |
|----------|---------|
| `SPONGE_API_KEY` | Default agent API key |
| `SPONGE_MASTER_KEY` | Default master key |
| `SPONGE_DEBUG` | Enable debug logging |
| `SPONGE_API_URL` | Custom API endpoint |

---

## Key Diagrams

### Complete Platform Architecture

```
                    +------------------------------------------+
                    |         Human Owner Dashboard             |
                    |  +------+ +--------+ +-------+ +------+  |
                    |  |Wallet| |Spending| | Agent | | Audit|  |
                    |  | View | |Limits  | |Control| |  Log |  |
                    |  +------+ +--------+ +-------+ +------+  |
                    +-------------------+----------------------+
                                        |
                                        | Governance
                                        v
+--------+  sponge_*_ API Key   +------+-------+
|   AI   |--------------------->| Sponge API   |
| Agent  |<---------------------|   Gateway    |
+--------+   JSON responses     +------+-------+
                                       |
                 +----------+----------+----------+----------+
                 |          |          |          |          |
                 v          v          v          v          v
          +------+--+ +----+----+ +---+---+ +---+---+ +----+----+
          |  Wallet  | |  Swaps  | |Bridge | | x402  | |Trading  |
          | Balances | | Jupiter | |deBridge| | Paid  | |Poly/   |
          |Transfers | | 0x      | |       | | APIs  | |HyperLq |
          +------+---+ +----+----+ +---+---+ +---+---+ +----+----+
                 |          |          |          |          |
                 v          v          v          v          v
          +------+--+ +----+----+ +---+---+ +---+---+ +----+----+
          |Ethereum | | Solana  | | Cross | |Search,| | Predict |
          |Base     | | DEXes   | | Chain | |Image, | | Markets |
          |Solana   | |         | |       | |Scrape | | Perps   |
          +---+-----+ +---------+ +-------+ +-------+ +---------+
              |
              +---+---+---+
              |       |   |
              v       v   v
          +------+ +---+ +------+
          | Visa | |Amzn| |Pymt  |
          |Cards | |Shop| |Links |
          +------+ +----+ +------+
```

### Agent Transaction Lifecycle

```
Agent initiates action
        |
        v
+-------+--------+
| Check spending  |-----> Exceeds? --> HTTP 403
| limits          |       (SPENDING_LIMIT_EXCEEDED)
+-------+--------+
        |
        v
+-------+---------+
| Check allowlist |-----> Not listed? --> HTTP 403
| (transfers)     |       (ADDRESS_NOT_ALLOWLISTED)
+-------+---------+
        |
        v
+-------+--------+
| Check balance  |-----> Insufficient? --> HTTP 402
|                |       (INSUFFICIENT_FUNDS)
+-------+--------+
        |
        v
+-------+--------+
| Execute action |-----> Fails? --> Error response
| (on-chain tx / |                  (no charge for x402)
|  API call)     |
+-------+--------+
        |
        v (success)
+-------+---------+
| Return result  |
| (txHash,       |
|  explorerUrl,  |
|  data)         |
+-----------------+
```

### Agent Lifecycle (Admin Management)

```
+------------------+
|  SpongeAdmin     |  (Master Key holder)
|  (master_xxx_*)  |
+--------+---------+
         |
         | createAgent()
         v
+--------+-----------+
| Agent Created      |  Status: active
| - apiKey assigned  |  - EVM wallet
| - Wallets created  |  - Solana wallet
+--------+-----------+
         |
    +----+----+----+----+
    |         |         |
    v         v         v
 Operate   Pause    Delete
    |      agent     agent
    |         |     (permanent)
    v         v
 Active    Paused
 (all ops  (all ops
  work)    blocked)
    ^         |
    |         |
    +--Resume-+
```

### SIWE Authentication Flow

```
Agent                           Sponge API                  External Service
  |                                 |                           |
  |-- POST /api/siwe/generate ---->|                           |
  |   { domain, uri }             |                           |
  |                                 |                           |
  |<-- { message, signature, ------|                           |
  |      address, chainId,        |                           |
  |      base64SiweMessage }      |                           |
  |                                 |                           |
  |-- Use SIWE signature ---------------------------------->  |
  |   (EIP-4361 authentication)    |                           |
```

---

## Summary

Sponge enables multi-chain agentic finance by providing:

1. **Managed dual wallets** (EVM + Solana) with automatic provisioning and shared EVM addresses across chains
2. **Multi-chain operations** — transfers, swaps (Jupiter/0x), and cross-chain bridges (deBridge) across Ethereum, Base, and Solana
3. **x402 micropayment services** — discover and use paid APIs (search, image gen, scraping, AI) with automatic USDC payments
4. **Trading integrations** — Polymarket prediction markets and Hyperliquid perpetual/spot trading
5. **Fiat bridge** — prepaid Visa cards ($5-$1,000) and Amazon checkout for real-world spending
6. **Human governance** — spending limits (per-tx, hourly, daily, weekly, monthly), address allowlists, agent pause/resume/delete
7. **Claude-native integration** — first-class MCP server and direct Anthropic tool definitions via TypeScript SDK
8. **Admin API (Master Keys)** — programmatic agent creation, management, and monitoring with audit logs

The core innovation is providing agents with **multi-chain managed wallets** and a **unified API surface** that spans crypto operations, paid service access, trading, and real-world purchases — all under human-configurable spending controls.
