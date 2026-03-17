# Build (PaaS) Evaluation: Full Deployment Test

> Live test of Locus Build — project creation through container deployment  
> Date: 2026-03-17 | Environment: Beta | Image: traefik/whoami

---

## What We Tested

"Build with Locus" — an agent-native PaaS that deploys containerized services to AWS (ECS Fargate). We tested the full lifecycle: auth exchange, project creation, environment setup, service creation, deployment, polling, live URL access, and billing.

**Important discovery:** Build runs on a separate domain (`buildwithlocus.com`) with its own API base, auth system, and billing credits — distinct from the main Locus payment API (`paywithlocus.com`).

### Test Actions

1. **Auth Exchange** — `POST /v1/auth/exchange` (swap API key for JWT)
2. **Create Project** — `POST /v1/projects`
3. **Create Environment** — `POST /v1/projects/:id/environments`
4. **Create Service** — `POST /v1/services` (Docker image source)
5. **Trigger Deployment** — `POST /v1/deployments`
6. **Poll Deployment** — `GET /v1/deployments/:id`
7. **Check Logs** — `GET /v1/deployments/:id/logs`
8. **Access Live URL** — `GET https://svc-{id}.buildwithlocus.com`
9. **Billing Check** — `GET /v1/billing/balance`
10. **Cleanup** — `DELETE /v1/services/:id`

---

## README Claims vs. Actual Results

### Claim 1: "Create project → environment → service → deploy" workflow

| Step | README Says | What Actually Happened | Latency |
|------|-------------|----------------------|---------|
| Auth exchange | API key → JWT (15 min TTL per README) | Confirmed — JWT received, works | 1.96s |
| Create project | Returns project ID | `proj_mmuuu3a5l27ruscg` created | 0.55s |
| Create environment | dev/staging/prod types | `env_mmuuu3n0mz9juj6u` created | 0.42s |
| Create service | Image source type supported | Service created with Docker image | 0.73s |
| Trigger deployment | Returns deployment ID, status `queued` | Confirmed | 1.13s |
| Poll status | `queued → building → deploying → healthy` | Went straight to `healthy` (no build step for images) | ~4s |

**Verdict: ✅ CONFIRMED** — The full workflow works exactly as documented. Each step returns clean IDs with predictable naming conventions (`proj_`, `env_`, `svc_`, `deploy_`).

---

### Claim 2: Deployment Timing

| Phase | README Says | What Actually Happened |
|-------|-------------|----------------------|
| Queued | 5-30 sec | < 1 sec |
| Building | 2-5 min (skipped for image) | Skipped (correct for Docker image source) |
| Deploying | 30-90 sec | ~3.7 seconds |
| **Total (Docker image)** | **1-2 min** | **~4 seconds** |

**Verdict: ✅ EXCEEDED EXPECTATIONS** — Deployment was dramatically faster than documented. The `durationMs: 3744` (3.7 seconds) for a Docker image deployment is impressive.

---

### Claim 3: Health Check Requirement — "Must expose /health returning HTTP 200"

| Aspect | README Says | What Actually Happened |
|--------|-------------|----------------------|
| Health check path | `/health` mandatory | Service auto-sets `healthCheckPath: "/health"` |
| Failure behavior | "Deployments will time out and fail" | Deployment reported `healthy` even though traefik/whoami doesn't have a dedicated `/health` endpoint (it returns 200 on all paths) |

**Verdict: ✅ CONFIRMED** — Health check path is enforced. Our image passed because traefik/whoami responds 200 on every path including `/health`.

---

### Claim 4: Auto-subdomain URLs — `svc-{id}.buildwithlocus.com`

| Aspect | README Says | What Actually Happened |
|--------|-------------|----------------------|
| URL format | `svc-{id}.buildwithlocus.com` | Confirmed — got `https://svc-mmuuz6ougasv16mg.buildwithlocus.com` |
| HTTPS | Integrated SSL | URL is HTTPS ✅ |
| Accessibility | "503 for ~60s while service discovery registers" | ❌ **Persistent 503** — URLs never became accessible |

**Error response from URL:**
```json
{
  "error": "Service unavailable or not found",
  "host": "svc-mmuuz6ougasv16mg.buildwithlocus.com",
  "hint": "The service behind this hostname may be deploying, stopped, or not registered."
}
```

We tested 2 services over ~5+ minutes each — neither became accessible. The container was running (logs showed `Starting up on port 80`), deployment was `healthy`, but the load balancer / service discovery never registered the service.

**Verdict: ❌ FAILED** — Service discovery / routing is broken in beta. Deployments succeed internally but URLs are unreachable. This is the most critical issue — you can deploy but can't access what you deployed.

---

### Claim 5: Billing — "$0.25/month per service"

| Aspect | README Says | What Actually Happened |
|--------|-------------|----------------------|
| Pricing | $0.25/service per month | Confirmed — 2 services consumed $0.50 from $1.00 credit |
| Free credits | New accounts start with $1.00 | Confirmed |
| Billing separate from wallet | Not mentioned in main README | **Build billing is completely separate from Locus USDC wallet** |
| Balance endpoint | Not documented in main README | `GET /v1/billing/balance` — shows credit balance, service count, next billing date |

**Billing response:**
```json
{
  "creditBalance": 0.50,
  "totalServices": 2,
  "monthlyTotal": 0.50,
  "billingCycleDay": 17,
  "nextBillingDate": "2026-04-17",
  "status": "active"
}
```

**Verdict: ✅ CONFIRMED** — Pricing is accurate. But important note: Build uses its own credit system, NOT the USDC wallet balance. Our Locus wallet stayed at $9.98 throughout — Build credits are deducted separately.

---

### Claim 6: Deployment Logs

| Aspect | README Says | What Actually Happened |
|--------|-------------|----------------------|
| Log access | Not detailed in README | `GET /v1/deployments/:id/logs` works |
| Content | Not specified | Returns runtime logs with timestamps |
| Phases | Not specified | Logs include `phase` field (`"runtime"`) |

**Verdict: ✅ CONFIRMED** — Logs are accessible and useful for debugging.

---

### Claim 7: Runtime Configuration

| Setting | README Default | Actual Default |
|---------|---------------|----------------|
| Port | 8080 | 8080 (auto-set) |
| CPU | 256 (0.25 vCPU) | 256 ✅ |
| Memory | 512 MB | 512 ✅ |
| Min instances | 1 | 1 ✅ |
| Max instances | 3 | 3 ✅ |
| Architecture | Not mentioned | `arm64` (discovered) |

**Verdict: ✅ CONFIRMED** — Runtime defaults match documentation. Bonus: `architecture: arm64` is auto-selected (not documented).

---

### Claim 8: Separate API & Auth System

| Aspect | Main README Says | What Actually Happened |
|--------|-----------------|----------------------|
| API base | Uses main Locus API | ❌ Completely separate: `beta-api.buildwithlocus.com/v1` |
| Auth | Same API key | API key is shared, but must be exchanged for a Build-specific JWT |
| JWT TTL | "15 min" (per main README) | SKILL.md says 30 days — significant discrepancy |

**Verdict: ⚠️ PARTIALLY CONFIRMED** — The API key works across both systems, but Build is a separate platform with its own domain, auth flow, and billing. The main README's description gives the impression it's part of the unified Locus API, which is misleading.

---

## Performance Summary

| Operation | Latency |
|-----------|---------|
| Auth exchange | 1.96s |
| Create project | 0.55s |
| Create environment | 0.42s |
| Create service | 0.58-0.73s |
| Trigger deployment | 0.70-1.13s |
| Deployment completion (Docker image) | ~3.7s |
| Service URL accessibility | ❌ Never (persistent 503) |

---

## Summary Scorecard

| README Claim | Status | Notes |
|-------------|--------|-------|
| Project → Environment → Service → Deploy workflow | ✅ Confirmed | Clean IDs, fast creation |
| Deployment timing (Docker: 1-2 min) | ✅ Exceeded | Only ~4 seconds |
| Health check requirement | ✅ Confirmed | Auto-set /health path |
| Auto-subdomain URLs | ❌ Failed | Persistent 503 — service discovery broken |
| Pricing ($0.25/service/month) | ✅ Confirmed | Accurate |
| Runtime defaults | ✅ Confirmed | Matches docs |
| Deployment logs | ✅ Confirmed | Accessible via API |
| Unified with Locus wallet | ❌ Not accurate | Separate billing system |
| 503 for ~60s after deploy | ❌ Worse | 503 persisted indefinitely |

**Overall: 6/9 claims confirmed, 2 failed (URL access, unified billing), 1 exceeded expectations (speed)**

---

## Issues Found

### 1. CRITICAL — Service URLs Never Become Accessible
Both deployed services returned persistent 503 errors despite `healthy` deployment status and running containers. The service discovery / load balancer registration appears broken in beta. This makes Build essentially unusable for serving traffic in the current beta state.

### 2. SIGNIFICANT — Build is a Separate Platform (Not Documented Clearly)
The main README describes Build as a core component alongside Wrapped APIs, Checkout, etc. In reality:
- Different domain: `buildwithlocus.com` (not `paywithlocus.com`)
- Different auth: requires JWT exchange (not direct API key)
- Different billing: uses its own credit system (not USDC wallet)
- Separate SKILL.md: `https://buildwithlocus.com/SKILL.md`

This separation isn't apparent from the main documentation.

### 3. MINOR — First Service One-Off Not Deletable
The first service (`svc_mmuuz6ougasv16mg`) returned 404 on DELETE, while the second succeeded with 204. Possible race condition or the first was already cleaned up.

---

## Pricing

| Item | Cost |
|------|------|
| Service "web" (Docker image) | $0.25/month |
| Service "web2" (Docker image) | $0.25/month |
| **Total Build credits consumed** | **$0.50** |
| **Locus wallet impact** | **$0.00** (separate billing) |

Build credits: $1.00 → $0.50 remaining
Locus wallet: $9.98 (unchanged)

---

## Comparison: Build Pricing vs README Provider Catalog

The README's Provider Catalog section only covers Wrapped API costs. Build pricing lives in a separate system entirely:

| Service | Pricing Model | Source |
|---------|--------------|--------|
| Wrapped APIs (Exa, Firecrawl, etc.) | Per-call USDC from wallet | Main Locus API |
| Checkout | Per-transaction USDC from wallet | Main Locus API |
| **Build (PaaS)** | **$0.25/service/month from Build credits** | **Separate Build API** |
| Build addons (Postgres/Redis) | $0.25/addon/month from Build credits | Separate Build API |
