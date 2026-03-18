# Locus vs Sponge: Architectural & Feature Comparison

> Two approaches to the same problem — giving AI agents the ability to hold and spend money autonomously.

---

## At a Glance

| | **Locus** | **Sponge** |
|---|---|---|
| **Tagline** | One USDC balance for wallets, APIs, deployments, checkout, and more | Financial infrastructure for the agent economy |
| **Chain Support** | Base only | Ethereum, Base, Solana |
| **Wallet Model** | Non-custodial ERC-4337 smart wallets (on-chain) | Managed server-side wallets (EVM + Solana) |
| **Currency** | USDC only | ETH, USDC, SOL, any SPL token |
| **Gas Fees** | Fully gasless (paymaster) | Auto-estimated (EVM), minimal (Solana) |
| **External APIs** | Wrapped APIs (static config) | x402 discovery (dynamic catalog) |
| **AI Integration** | SKILL.md reference doc | MCP server + Anthropic tool definitions |
| **SDK** | REST API only | TypeScript SDK (`@spongewallet/sdk`) |
| **Merchant Payments** | Checkout system (React SDK) | Payment links (x402) |
| **Trading** | None | Polymarket, Hyperliquid |
| **Fiat Off-Ramp** | Laso Finance (Visa, Venmo, PayPal) | Laso Finance (Visa), Amazon Checkout |
| **PaaS** | Build (deploy containers to AWS) | None |
| **Human Labor** | Tasks marketplace | None |

---

## 1. Wallet Architecture

This is the most fundamental difference between the two platforms. They take opposite approaches to the custody-vs-convenience tradeoff.

### Locus: Non-Custodial Smart Wallets (ERC-4337)

```
+---------------------------------------------------------------+
|                     Locus Smart Wallet                        |
|                  (ERC-4337 on Base)                            |
|                                                               |
|  +------------------+     +--------------------+              |
|  |    User Key      |     |  Permissioned Key  |              |
|  | (client-side,    |     |  (AWS KMS,         |              |
|  |  never stored    |     |   revocable by     |              |
|  |  on Locus)       |     |   user at any time)|              |
|  +--------+---------+     +---------+----------+              |
|           |                         |                         |
|           |   Both can sign txs     |                         |
|           +------------+------------+                         |
|                        |                                      |
|                        v                                      |
|              Single-Signer Account                            |
|              validates against either key                     |
|                                                               |
|  - Deployed via CREATE2 (deterministic addresses)             |
|  - ERC1967 proxy (Solady ERC4337Factory)                      |
|  - UUPS upgradeability DISABLED (immutable)                   |
|  - Subwallet system for email escrow (up to 100)              |
+---------------------------------------------------------------+
```

**Key properties:**
- Wallet exists as a **smart contract on Base** — verifiable on-chain
- **Dual-key model**: user key (client-side, never touches Locus servers) + permissioned key (AWS KMS, for agent operations)
- User can **revoke agent access at any time** via `revokePermissionedKey()` on-chain
- All transactions go through ERC-4337 EntryPoint as UserOperations
- Fully **gasless** via proprietary paymaster
- Wallet addresses are **predictable before deployment** via `LocusFactory.getAddress()`

**Tradeoff:** More secure and trustless, but locked to a single chain (Base) and a single token (USDC).

### Sponge: Managed Server-Side Wallets

```
+---------------------------------------------------------------+
|                     Sponge Agent Wallet                        |
|                                                                |
|  +---------------------------+  +---------------------------+  |
|  |       EVM Wallet          |  |     Solana Wallet         |  |
|  |  (Ethereum, Base, etc.)   |  |  (Mainnet / Devnet)      |  |
|  |  Same address across ALL  |  |  SOL + any SPL token     |  |
|  |  EVM chains               |  |  Jupiter DEX access      |  |
|  +---------------------------+  +---------------------------+  |
|                                                                |
|  - Keys managed server-side by Sponge                          |
|  - Human owner controls via dashboard/admin API                |
|  - Owner can pause, resume, or delete agent                    |
|  - Allowlists restrict transfer destinations                   |
+----------------------------------------------------------------+
```

**Key properties:**
- Wallets are **managed by Sponge** — private keys live on Sponge's infrastructure
- Each agent gets **two wallets**: one EVM (shared address across all EVM chains) + one Solana
- Multi-chain: Ethereum, Base, Solana (+ testnets including Tempo)
- Multi-token: ETH, USDC, SOL, any SPL token
- No on-chain smart contract wallet — standard EOA-style accounts
- Human controls are **application-level** (pause/resume/delete via API), not on-chain

**Tradeoff:** More flexible (multi-chain, multi-token), but custodial — you're trusting Sponge with private keys.

### Comparison

| Dimension | Locus | Sponge |
|-----------|-------|--------|
| **Custody** | Non-custodial (user holds key) | Custodial (Sponge holds keys) |
| **On-chain presence** | Smart contract wallet (ERC-4337) | Standard accounts (no on-chain contract) |
| **Key revocation** | On-chain (`revokePermissionedKey()`) | Application-level (pause/delete via API) |
| **Chains** | Base only | Ethereum + Base + Solana |
| **Tokens** | USDC only | ETH, USDC, SOL, SPL tokens |
| **Gas** | Gasless (paymaster sponsors all) | Agent/owner covers gas (auto-estimated) |
| **Address predictability** | Yes (`CREATE2` deterministic) | N/A (assigned at registration) |
| **Wallet upgradeability** | Disabled (immutable) | N/A (no contract) |
| **Email transfers** | Yes (subwallet escrow with OTP) | No |

---

## 2. External API Routing: Static vs. Dynamic

This is the difference that the Laso Finance docs directly expose. It stems from fundamentally different architectural choices about how agents discover and call third-party services.

### Locus: Static Endpoint Registry (Wrapped APIs)

Locus uses a **configurable reverse proxy** model. The human operator must manually register every external endpoint in the Locus dashboard before the agent can call it.

```
Human Operator                 Locus Dashboard
     |                              |
     |-- Manually adds endpoint --->|
     |   { url, slug, method,      |
     |     params: [{name,         |
     |       location, required}]  |
     |   }                          |
     |                              v
     |                       Static Route Table
     |                       +----------------------+
     |                       | slug → URL           |
     |                       | slug → HTTP method   |
     |                       | slug → param schema  |
     |                       +----------------------+

Then agent calls:

POST /api/x402/{slug}
     |
     v
Locus looks up slug in route table
     |
     v
Transforms params (body → query if configured)
     |
     v
Forwards to upstream URL with Locus credentials
```

**Why Laso Finance requires manual config on Locus:**

The Laso Finance endpoints are `GET` requests expecting **query parameters** (`?amount=50`). But agents call Locus via `POST` with **JSON body** (`{"amount": 50}`). Locus needs explicit per-parameter configuration to know it must transform `{"amount": 50}` in a POST body into `?amount=50` on a GET request.

Without this config:
1. Locus doesn't know the upstream URL for `laso-get-card`
2. Locus doesn't know the HTTP method (GET vs POST)
3. Locus sends `amount` as JSON body instead of query string
4. Laso Finance rejects with: `{"error": "amount query parameter is required"}`

**Four endpoints that must be manually registered:**

| Slug | URL | Method | Params |
|------|-----|--------|--------|
| `laso-auth` | `https://laso.finance/auth` | GET | (none) |
| `laso-get-card` | `https://laso.finance/get-card` | GET | `amount` (query, required) |
| `laso-push-to-card` | `https://laso.finance/push-to-card` | GET | `amount` (query, required) |
| `laso-send-payment` | `https://laso.finance/send-payment` | GET | `amount`, `platform`, `recipient_id` (all query, required) |

### Sponge: Dynamic Service Registry (x402 Discovery)

Sponge maintains a **server-side service catalog** that agents query at runtime. All routing metadata — URLs, methods, parameters, pricing — is pre-indexed.

```
Agent                         Sponge API                  Service Registry
  |                               |                            |
  |-- GET /api/discover           |                            |
  |   ?query=laso            ---->|---- Query catalog -------->|
  |                               |                            |
  |<-- [{id, name, category,     |<--- Match results ---------|
  |      endpoints}]              |                            |
  |                               |                            |
  |-- GET /api/discover/{id} --->|                            |
  |                               |                            |
  |<-- { endpoints,              |                            |
  |      paymentsProtocolConfig, |  (URL, method, params,    |
  |      pricing }           ----|  pricing all pre-stored)   |
  |                               |                            |
  |-- POST /api/x402/fetch ----->|                            |
  |   { url: baseUrl + path,    |--- Builds correct request  |
  |     method, body }          |    from stored metadata --->|
  |                               |                            |
  |<-- Result + payment_details --|<--- Response --------------|
```

**Why Laso Finance works automatically on Sponge:**

Sponge already knows every Laso Finance endpoint's URL, HTTP method, parameter schema, and payment config. The agent just calls `GET /api/discover?query=laso` and gets back everything needed. Sponge handles the parameter transformation internally — no human configuration step.

### Side-by-Side: Adding a New External Service

```
                    LOCUS                              SPONGE
                    ─────                              ──────

Who adds it?     Human operator via dashboard UI    Sponge team adds to catalog

Steps to add     1. Log into dashboard              Service provider registers
a service        2. Navigate to x402 Endpoints      with Sponge (or Sponge
                 3. Click "+ Add Endpoint"           indexes it). Available
                 4. Enter URL, slug, method         to all agents immediately.
                 5. Add each parameter with
                    name, type, location
                 6. Click "Validate & Add"
                 7. Repeat per endpoint

Agent-side       Agent must know exact slugs        Agent calls discover API,
changes          (e.g. "laso-get-card")              gets slugs dynamically

What breaks      Wrong param location config →       Nothing — metadata is
                 silent failures or upstream         pre-validated server-side
                 errors like "amount query
                 parameter is required"

Flexibility      ANY URL can be added (open)         Only pre-indexed services
                                                     (curated catalog)

Scaling          O(n) manual work per endpoint       O(1) — already in catalog
```

### Why This Difference Exists

**Locus** is architecturally a **configurable reverse proxy with payment middleware**. The mental model: "we'll forward your request to any URL you configure." Generic and flexible — you can point it at any HTTP endpoint — but the cost is manual setup per service, per endpoint, per parameter.

**Sponge** is architecturally an **agent-native service marketplace**. The mental model: "agents discover what's available and we handle the rest." They maintain a curated registry, so agents self-serve. Less flexible (only indexed services), but zero configuration for anything in the catalog.

This is a classic **open-platform vs. curated-marketplace** tradeoff.

---

## 3. Spending Controls & Governance

Both platforms let humans control agent spending, but with different models.

### Locus: Three-Tier Allowance Model

```
+------------------+  +-------------------+  +-----------------+
|   Allowance      |  | Max Transaction   |  | Approval        |
|                  |  | Size              |  | Threshold       |
| Total USDC the   |  | Per-transaction   |  | Amount above    |
| agent can spend  |  | ceiling           |  | which human     |
| (decrements)     |  |                   |  | must approve    |
|                  |  | Exceeds → 403     |  |                 |
| Blank = full     |  |                   |  | Blank = none    |
| balance access   |  |                   |  | $0 = all need   |
+------------------+  +-------------------+  +-----------------+
```

**Unique to Locus:** The **approval threshold** creates an asynchronous human-in-the-loop flow. When a transaction exceeds the threshold:
1. Agent gets HTTP 202 with `approval_url`
2. Human reviews and approves/denies at that URL
3. If approved, action executes automatically

This is a **per-transaction approval** mechanism — the agent pauses and waits for human sign-off.

### Sponge: Multi-Tier Rate Limits + Allowlists

```
+----------------+  +----------------+  +----------------+
| Per-Transaction|  | Time-Based     |  | Address        |
| Limit          |  | Limits         |  | Allowlist      |
|                |  |                |  |                |
| Max USD per    |  | per_minute     |  | Only approved  |
| single tx      |  | hourly         |  | destinations   |
|                |  | daily          |  | can receive    |
| Exceeds → 403  |  | weekly         |  | transfers      |
|                |  | monthly        |  |                |
|                |  | Exceeds → 403  |  | Not listed     |
|                |  |                |  | → 403          |
+----------------+  +----------------+  +----------------+

+ Agent Pause/Resume + Agent Delete (permanent)
```

**Unique to Sponge:**
- **Address allowlists** — transfers can only go to pre-approved addresses. Locus has no equivalent; any Base address can receive.
- **Granular time windows** — per-minute, hourly, daily, weekly, monthly. Locus only has a total allowance (no time windows).
- **Agent lifecycle controls** — pause, resume, delete. Locus has no agent pause mechanism.

### Comparison

| Control | Locus | Sponge |
|---------|-------|--------|
| **Total spending cap** | Allowance (total USDC, decrements) | Daily/weekly/monthly limits |
| **Per-transaction limit** | Max Transaction Size | Per-transaction limit |
| **Human approval flow** | Yes (HTTP 202 + approval URL) | No async approval — hard reject only |
| **Time-based limits** | No | per_minute, hourly, daily, weekly, monthly |
| **Address restrictions** | No (any Base address) | Allowlist required for transfers |
| **Agent pause/resume** | No | Yes (via admin API) |
| **Agent deletion** | No | Yes (permanent, revokes all keys) |
| **Audit logs** | Transaction log in dashboard | Audit logs via admin API |
| **Key rotation** | No (key shown once) | Yes (`admin.rotateAgentKey()`) |

**Key insight:** Locus favors **human-in-the-loop approval** for high-value transactions (async, collaborative). Sponge favors **hard guardrails** that prevent actions entirely (allowlists, rate limits, pause). Different philosophies — "ask permission" vs. "constrain the box."

---

## 4. Authentication & Onboarding

### Locus

```
Production:
  1. Human registers at app.paywithlocus.com
  2. Human creates wallet (deploys to Base ~30s)
  3. Human generates API key (claw_dev_*, shown once)
  4. Human funds wallet with USDC on Base
  5. Human configures spending controls
  6. Human gives agent the API key + SKILL.md URL
  7. Agent authenticates: API key → JWT (15-min TTL)

Beta (agent self-registration):
  1. Agent calls POST /api/register
  2. Gets apiKey + ownerPrivateKey immediately
  3. Agent shares claim URL with human
  4. Default: $10 allowance, $5 max tx
```

### Sponge

```
Agent-First Mode:
  1. Agent calls POST /api/agents/register { agentFirst: true }
  2. Gets apiKey immediately + claimUrl
  3. Agent operates immediately with default limits
  4. Human approves later via claim URL

Device Flow (human-initiated):
  1. SDK displays verification URL + code
  2. Human visits URL, enters code, approves
  3. SDK polls for approval, gets API key
  4. Credentials stored locally

Master Key (programmatic):
  1. Admin creates agent via SpongeAdmin.createAgent()
  2. Gets apiKey + wallet addresses
  3. No human interaction needed
```

### Comparison

| Aspect | Locus | Sponge |
|--------|-------|--------|
| **Agent self-registration** | Beta only | Yes (agent-first mode) |
| **API key format** | `claw_dev_*` → JWT (15-min TTL) | `sponge_test_*` / `sponge_live_*` (persistent) |
| **Auth mechanism** | API key exchanged for short-lived JWT | API key used directly (Bearer token) |
| **Key rotation** | Not supported | `admin.rotateAgentKey()` |
| **Scoped permissions** | No (all-or-nothing) | Yes (10 granular scopes like `wallet:read`, `transaction:sign`) |
| **Programmatic agent creation** | No (beta has simple register) | Yes (Master Keys with full lifecycle control) |
| **Test/live separation** | Separate beta environment (same chain, real USDC) | Separate key prefixes (`test_` vs `live_`), different chains |
| **Versioned API** | No | Yes (`Sponge-Version: 0.2.1` header required) |

---

## 5. AI Agent Integration

### Locus: SKILL.md Reference Document

Locus uses a static markdown file (SKILL.md) as the canonical API reference for agents. The agent reads the file, learns the endpoints, and makes REST calls directly.

```
Agent                           Locus API
  |                                 |
  |-- Read SKILL.md (static doc) -->|
  |   (learns endpoints, params)   |
  |                                 |
  |-- POST /wrapped/openai/chat -->|
  |   (direct REST call)           |
  |                                 |
  |<-- JSON response --------------|
```

- No SDK — agents call the REST API directly
- No MCP server
- No tool definitions for Claude/LLM frameworks
- Agent must parse SKILL.md and construct requests manually

### Sponge: First-Class Claude Integration

Sponge provides a TypeScript SDK with native MCP and Anthropic tool support.

```
Agent                           Sponge SDK                  Claude API
  |                                 |                          |
  |-- wallet.mcp() --------------->|                          |
  |   (returns MCP server config)  |                          |
  |                                 |                          |
  |-- anthropic.messages.create -->|                          |
  |   { mcp_servers: { wallet } } |-- Tool calls ----------->|
  |                                 |<-- Tool results ---------|
  |<-- Natural language response --|                          |

OR:

  |-- wallet.tools() ------------->|                          |
  |   (returns tool definitions)   |                          |
  |                                 |                          |
  |-- tools.execute(name, input) ->|                          |
  |<-- Result ---------------------|                          |
```

**Three integration methods:**
1. **MCP Server** (`@spongewallet/mcp`) — for Claude Desktop, Claude Code
2. **Direct Tools** (`wallet.tools()`) — Anthropic tool definitions for custom apps
3. **REST API** — direct HTTP calls (documented in agent guide / SKILL.md)

### Comparison

| Aspect | Locus | Sponge |
|--------|-------|--------|
| **SDK** | None (REST only) | `@spongewallet/sdk` (TypeScript) |
| **MCP Server** | No | Yes (`@spongewallet/mcp`) |
| **Tool definitions** | No | Yes (`wallet.tools()` for Anthropic API) |
| **Agent discovery** | Read SKILL.md | Read SKILL.md or use SDK |
| **Tool filtering** | N/A | `include`/`exclude` arrays |
| **Before-execute hooks** | N/A | `beforeExecute` callback (approval flows) |
| **Debug mode** | N/A | `SPONGE_DEBUG=true` |
| **Claude Desktop support** | No | Yes (MCP config) |

---

## 6. Feature Coverage

### Locus Has, Sponge Doesn't

| Feature | Details |
|---------|---------|
| **Build (PaaS)** | Deploy containers to AWS ECS/Fargate. Project → Environment → Service hierarchy. GitHub, Docker image, or git push sources. PostgreSQL and Redis addons. Custom domains. |
| **Tasks (Human Labor)** | Marketplace to hire human workers for graphic design, content creation, etc. Category + timeline + price tier selection. |
| **Checkout System** | Merchant-facing payment acceptance. React SDK (`@withlocus/checkout-react`). Three display modes (embedded, popup, redirect). HMAC webhook verification. Payment Router smart contract. |
| **Email Transfers** | Send USDC to email addresses via subwallet escrow. OTP-based claiming. Time-limited with `disburseBefore` deadline. |
| **On-chain Smart Wallets** | ERC-4337 account abstraction with dual-key security. Verifiable on-chain. |
| **Approval Threshold** | Async human-in-the-loop for high-value transactions (HTTP 202 flow). |
| **Gasless Transactions** | All transactions sponsored by Locus paymaster. |

### Sponge Has, Locus Doesn't

| Feature | Details |
|---------|---------|
| **Multi-Chain** | Ethereum + Base + Solana (vs Base only). Cross-chain bridges via deBridge. USDC consolidation tool. |
| **Token Swaps** | Jupiter (Solana) and 0x (Base) DEX aggregators. Quote-then-execute flow. |
| **Polymarket Trading** | Search markets, place/cancel orders, manage positions, deposit/withdraw. |
| **Hyperliquid Trading** | Perpetual futures and spot trading. Leverage, orderbooks, funding rates. |
| **Amazon Checkout** | Search + purchase products on Amazon. Credit card storage (encrypted). Dry-run mode. |
| **TypeScript SDK** | `@spongewallet/sdk` with typed methods for all operations. |
| **MCP Server** | `@spongewallet/mcp` for Claude Desktop / Claude Code integration. |
| **Anthropic Tool Definitions** | `wallet.tools()` returns Claude-compatible tool schemas. |
| **Master Keys** | Programmatic multi-agent management. Create, pause, resume, delete agents via API. |
| **Address Allowlists** | Restrict transfers to pre-approved addresses only. |
| **Granular Spending Limits** | per_minute, hourly, daily, weekly, monthly (vs total allowance only). |
| **API Key Scopes** | 10 granular permission scopes (vs all-or-nothing). |
| **Key Rotation** | `admin.rotateAgentKey()` with old key revocation. |
| **Dynamic Service Discovery** | `GET /api/discover` with query/category filtering. |
| **Payment Links** | Reusable x402 links for receiving payments. |
| **SIWE Auth** | Sign-In with Ethereum (EIP-4361) signature generation. |
| **Versioned API** | `Sponge-Version` header for API compatibility. |

### Both Have

| Feature | Locus | Sponge |
|---------|-------|--------|
| **Agent wallets** | ERC-4337 smart wallets on Base | Managed EVM + Solana wallets |
| **USDC transfers** | `transferUSDC()` on Base | `POST /api/transfers/evm` or `/solana` |
| **Prepaid Visa cards** | Via Laso Finance (x402) | Via Laso Finance (MCP tools) |
| **Venmo/PayPal** | Via Laso Finance (x402) | Not documented (Laso cards only?) |
| **Third-party API access** | Wrapped APIs (static config) | x402 services (dynamic discovery) |
| **Spending controls** | Allowance + max tx + approval threshold | Per-tx + time-based + allowlists |
| **Agent self-registration** | Beta only | Agent-first mode (production) |
| **Human dashboard** | app.paywithlocus.com | wallet.paysponge.com |
| **Transaction history** | Dashboard + API | Dashboard + API + admin audit logs |

---

## 7. On-Chain Architecture

### Locus: Deep On-Chain

Locus has significant on-chain infrastructure:

```
LocusFactory (ERC4337Factory)
     |
     | CREATE2
     v
Locus Smart Wallet (ERC1967 Proxy → Solady ERC4337)
     |
     +-- Owner Key (client-side)
     +-- Permissioned Key (AWS KMS)
     +-- Subwallets (escrow, up to 100)
     +-- USDC transfers
     |
Payment Router Contract
     |
     +-- pay(sessionId, recipient, amount)
     +-- payWithPermit(...) (EIP-2612)
     +-- sessionPaid(sessionId) view
     |
Paymaster (gasless transactions)
     |
Base Blockchain (USDC settlement)
```

**Contract addresses:**
- Payment Router: `0x34184b7bCB4E6519C392467402DB8a853EF57806` (Base mainnet)
- USDC: `0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913` (Base mainnet)

### Sponge: Thin On-Chain

Sponge uses standard on-chain primitives — no custom smart contracts:

```
Sponge Server (holds private keys)
     |
     +-- EVM Wallet (standard EOA)
     |     +-- Ethereum (ETH, USDC)
     |     +-- Base (ETH, USDC)
     |
     +-- Solana Wallet (standard keypair)
           +-- SOL, USDC, SPL tokens
           +-- Jupiter swaps
     |
     +-- deBridge (cross-chain bridges)
     +-- Polymarket (prediction trading)
     +-- Hyperliquid (perps/spot)
```

No factory contracts, no payment router, no paymaster, no on-chain wallet logic. All intelligence lives in the API layer.

### Comparison

| Aspect | Locus | Sponge |
|--------|-------|--------|
| **Custom smart contracts** | Yes (wallet, factory, router, subwallets) | No |
| **Account abstraction** | ERC-4337 (UserOps via EntryPoint) | No (standard transactions) |
| **On-chain verification** | Full (wallet logic verifiable) | Minimal (standard token transfers) |
| **Gas abstraction** | Paymaster sponsors all gas | No paymaster, standard gas |
| **Settlement chain(s)** | Base only | Ethereum, Base, Solana |
| **On-chain escrow** | Subwallets with time-lock | No |

---

## 8. Pricing & Cost Model

### Locus

| Service | Cost |
|---------|------|
| **Wrapped API calls** | Upstream cost + $0.003/call flat (most) OR 15% markup (OpenAI/Gemini) |
| **Laso Finance auth** | $0.001 USDC |
| **Laso Finance cards/payments** | Face value ($5-$1,000) |
| **USDC transfers** | Free (gasless) |
| **Build deployments** | Not documented (likely usage-based) |
| **Checkout** | Not documented |
| **Tasks** | Tier-based (budget/mid/premium) |

### Sponge

| Service | Cost |
|---------|------|
| **x402 service calls** | Per-service pricing (discovered via catalog) |
| **Prepaid cards** | Face value ($5-$1,000) |
| **Transfers** | Gas costs (auto-estimated) |
| **Swaps** | DEX fees + gas |
| **Bridges** | deBridge fees + gas |
| **Polymarket/Hyperliquid** | Trading fees per exchange |
| **Amazon Checkout** | Product price |

### Key Difference

Locus absorbs gas costs via its paymaster and charges a flat per-call platform fee. Sponge passes through gas/exchange fees and charges via x402 micropayments. Locus's model is simpler for agents (everything is USDC, no gas), while Sponge's is more transparent (actual costs passed through).

---

## 9. Design Philosophy Summary

| Dimension | Locus | Sponge |
|-----------|-------|--------|
| **Core metaphor** | "One USDC balance for everything" | "Give your agent a bank account" |
| **Chain philosophy** | Single-chain maximalist (Base) | Multi-chain pragmatist |
| **Custody model** | Non-custodial (user holds keys) | Custodial (platform holds keys) |
| **API routing** | Configurable proxy (open, manual) | Curated marketplace (closed, automatic) |
| **Governance model** | Approval-based (ask permission) | Constraint-based (enforce limits) |
| **Integration approach** | SKILL.md (read the docs) | SDK + MCP (native tooling) |
| **Feature breadth** | Narrower but deeper (PaaS, labor, checkout) | Wider (trading, multi-chain, bridges) |
| **On-chain depth** | Deep (custom contracts, AA, paymaster) | Thin (standard accounts, no custom contracts) |
| **Target user** | Agent builders who want on-chain guarantees | Agent builders who want multi-chain flexibility |

---

## 10. When to Use Which

**Choose Locus when:**
- You need **non-custodial** wallets with on-chain verifiability
- You're building on **Base** and want a single-chain, USDC-only simplicity
- You need **gasless transactions** (zero gas complexity for agents)
- You want a **merchant checkout system** (React SDK, webhooks, payment router)
- You need **PaaS deployments** (containers on AWS) as part of the agent workflow
- You need agents to **hire human workers** for design/content tasks
- You want **human-in-the-loop approval** for high-value transactions
- **On-chain guarantees** matter (immutable wallets, verifiable transactions)

**Choose Sponge when:**
- You need **multi-chain** operations (Ethereum + Base + Solana)
- You want agents to **trade** on Polymarket or Hyperliquid
- You need **token swaps** and **cross-chain bridges**
- You want a **TypeScript SDK** and **MCP server** for Claude integration
- You need **programmatic multi-agent management** (master keys, fleet control)
- You want **dynamic service discovery** (no manual endpoint configuration)
- You need **address allowlists** and **granular time-based spending limits**
- You want agents to **shop on Amazon**
- Multi-token support matters (ETH, SOL, SPL tokens, not just USDC)
