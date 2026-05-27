# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a demo of AI-powered shopping using the **Agentic Commerce Protocol (ACP)** and **Stripe Shared Payment Tokens (SPT)**. Users chat with an AI assistant that handles the full checkout lifecycle: product discovery → checkout creation → address collection → payment authorization → order completion.

## Services

Three services run locally:

| Service | Port | Directory | Purpose |
|---------|------|-----------|---------|
| Frontend | 3000 | `frontend/` | Next.js 14 chat UI + payment setup |
| Agent | 3001 | `agent-service/` | Express.js AI orchestrator |
| Merchant | 4000 | `merchant-service/` | Express.js ACP checkout backend |

Optional AWS Lambda services in `agent-service/ai-service/` and `agent-service/stripe-service/` (SAM deployable).

## Development Commands

### Start All Services (recommended)
```bash
./dev.sh           # Mac/Linux — installs deps, creates .env files, starts all services
./dev.sh --setup   # Re-run interactive .env configuration
.\dev.ps1          # Windows PowerShell equivalent
```

### Individual Services
```bash
# Frontend
cd frontend && npm run dev      # start dev server
cd frontend && npm run build    # production build
cd frontend && npm run lint     # ESLint

# Agent service
cd agent-service && npm run dev   # nodemon watch mode
cd agent-service && npm start     # production

# Merchant service
cd merchant-service && npm run dev
cd merchant-service && npm start
```

No test runner is configured — there are no test commands.

## Architecture

```
Frontend (Next.js) → Agent Service → Lambda AI (OpenAI function calling)
                                   → Merchant Service (ACP checkout)
                                   → Stripe Proxy (SPT creation)
```

**Chat flow**: The frontend POSTs messages to `agent-service POST /api/chat`. The agent calls the Lambda AI endpoint which uses OpenAI function calling. When the AI wants to act (e.g., create a checkout), it returns tool calls. The agent executes them against the merchant service and Stripe proxy, then feeds results back to the AI for a final response.

**ACP functions** available to the AI:
- `create_checkout`, `update_checkout`, `complete_checkout`, `cancel_checkout`
- `request_payment_method`, `set_user_email`, `get_checkout`

**Shared Payment Tokens**: When completing a checkout, the agent creates an SPT from the customer's saved Stripe payment method (with usage limits: max amount, currency, expiration) and sends it to the merchant. The merchant charges via SPT — card details never pass through the merchant.

## Key Files

- `agent-service/routes/chat.js` — core AI function-calling loop
- `agent-service/routes/checkout.js` — ACP checkout proxy endpoints
- `agent-service/routes/payment.js` — SPT creation and payment method management
- `agent-service/lib/openai.js` — Lambda AI service integration
- `merchant-service/routes/checkouts.js` — ACP checkout implementation (status machine)
- `merchant-service/lib/*.json` — product catalogs (books, coffee, skis, vinyl)
- `frontend/components/ChatInterface.tsx` — chat UI + client-side function call handling
- `frontend/components/PaymentSetup.tsx` — Stripe Elements card collection

## Environment Configuration

Copy `agent-service/.env.example` to `agent-service/.env` and `merchant-service/.env.example` to `merchant-service/.env`.

Critical agent-service variables:
```
LAMBDA_ENDPOINT     # AWS Lambda URL for OpenAI completions
STRIPE_SECRET_KEY   # Stripe secret key (sk_test_...)
STRIPE_PROXY_URL    # Stripe Lambda proxy (default: http://localhost:3002)
MERCHANT_API_URL    # Merchant backend (default: http://localhost:4000)
WORKSHOP_SECRET     # Shared secret for workshop auth
SPT_SIMULATION_MODE # true = same Stripe account (demo mode)
```

## Data Persistence

No database — all state is in-memory:
- **Checkouts**: stored in a `Map` in `merchant-service/routes/checkouts.js`
- **User profiles**: stored in a `Map` in `agent-service/routes/profile.js`
- Data is lost on service restart

## Product Catalogs

Products live in `merchant-service/lib/*.json`. Each catalog is served at `/api/{filename-without-extension}` (e.g., `GET /api/skis`). To add a new catalog, create a new JSON file — it's loaded dynamically.

## ACP Checkout Status Flow

`not_ready_for_payment` → (add address + shipping) → `ready_for_payment` → (complete with SPT) → `completed`

## Knowledge Graph

A graphify knowledge graph of this codebase lives in `graphify-out/`:

- `graphify-out/graph.json` — full node/edge graph (587 nodes, 892 edges across 45 communities)
- `graphify-out/GRAPH_REPORT.md` — god nodes, surprising connections, community map
- `graphify-out/graph.html` — interactive visualization

**Before answering architecture questions**, check `graphify-out/GRAPH_REPORT.md` first — it contains pre-computed community structure, god nodes, and cross-cutting relationships.

**Key god nodes** (highest connectivity — touch these carefully):
- `getConfig()` (29 edges) — all frontend components depend on this
- `loggedFetch()` (18 edges) — all API calls route through this for ACP logging
- `Chat Route / chat.js` (13 edges) — the AI function-calling orchestration hub
- `Shared Payment Token / SPT` (11 edges) — central to the payment security model

**To query the graph** for a specific question: `/graphify query "<question>"`
**To update after code changes**: `/graphify . --update`
