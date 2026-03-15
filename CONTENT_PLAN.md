# Valence documentation content plan

This document defines what to write in this module, where it lives, and how it connects. Use it to prioritize and implement SDK docs, API keys/setup, component explanations, and POST links.

## 1. What we're documenting

- **Audience**: Traders, operators, and developers integrating with Valence (terminal, API, or both).
- **Format**: Mintlify MDX for guides; OpenAPI (or Mintlify API playground) for the trading API so POST links and request/response are explicit.
- **Scope**: Public-facing only. No internal admin, Pulumi secrets layout, or repo-internal crate boundaries unless they affect integration.

## 2. Components and confusing areas

Document these so users understand what they're talking to and what depends on what:

| Area | What to document | Where (section) |
|------|------------------|-----------------|
| **Terminal vs API** | Terminal is the Next.js UI; it talks to the Valence backend (tradegui) via REST + WebSocket. The "API" we document is the tradegui REST/WS surface that the terminal (and external integrators) use. | Getting started, Platform |
| **Backend (tradegui)** | Single Rust service: REST + WebSocket. No separate "SDK" binary; the TypeScript frontend is the reference client. Document base URL, auth (Bearer token from `/api/auth/token`), and that orders/books/inventory go through this service. | API overview, Authentication |
| **Venue abstraction** | Orders and books use a unified symbol format (`kalshi:TICKER`, `polymarket:CONDITION_ID`). One place order API; backend routes to Kalshi or Polymarket. Clarify in "How orders work" and in API reference. | Trading operations, API reference |
| **Proxy vs direct** | In production the frontend calls Next.js API routes (`/api/backend/*`), which proxy to tradegui. Document that the *documented* base URL is the tradegui base (e.g. `http://localhost:8080` for local), and that in hosted setups the base URL may be the proxy. | Getting started, API overview |
| **WebSocket** | Single `/api/ws` endpoint: order books, fills, and other streaming state. Document subscription model and message shapes so SDK-style integrators know what to subscribe to. | API reference (WebSocket), Platform |
| **Quoters / systematic** | High-level feature: quoters and tickers endpoints. Document what they control and that they are optional for basic order flow. | Platform, API reference |

Keep a short "Architecture" or "How the pieces fit" page that links Terminal → Backend → Exchanges and calls out the single API surface and the proxy.

## 3. SDK documentation and POST links

- **No separate SDK package today**: The "SDK" is the tradegui HTTP/WebSocket API. Document it as a REST + WS API with clear endpoint list and request/response bodies.
- **Use API reference with real endpoints**:
  - **POST links**: Document these in the API reference with exact path, method, and body:
    - `POST /api/orders` — place order (single venue by symbol)
    - `POST /api/orders/consolidated` — place order (unified qkey, optional ask split)
    - `POST /api/order/cancel` and `POST /api/order/cancel/consolidated`
    - `POST /api/auth/token` — get Bearer token for API calls
    - `POST /api/fair` — submit fair value (if applicable)
  - For each POST: method, path, request body schema (e.g. `m_symbol`, `m_side`, `m_action`, `m_price`, `m_quantity`, `m_post_only`, `m_ioc`), response schema, and a minimal example (curl or JSON).
- **OpenAPI**: Replace or extend the starter `api-reference/openapi.json` with Valence endpoints (auth, orders, books, inventory, trades, cancel, consolidated order, etc.) so the Mintlify API playground shows try-it-now and links. Keep auth as Bearer token from `POST /api/auth/token`.
- **Code examples**: Add a "Quick integration" or "Code examples" page with copy-paste snippets: get token, place order, cancel, subscribe to book via WebSocket. Link from API overview and from getting started.

## 4. API keys and authentication

Document two separate concepts so they're not confused:

### 4.1 Valence API (tradegui) — for Terminal and integrators

- **What**: Bearer token from `POST /api/auth/token`. No API key per se; the backend issues a short-lived token. No scopes in current implementation.
- **How**: Call `POST /api/auth/token` (no body); use `Authorization: Bearer <token>` on subsequent requests. Token expiry is returned in the response; client should refresh before expiry.
- **Where to document**: "Authentication" under API reference, plus Getting started "First API call".
- **Security note**: In production, the Valence UI goes through the Next.js proxy; the proxy may enforce NextAuth/session. Document that direct access to tradegui (e.g. local `:8080`) is for development or trusted backends; for browser/integrations, use the configured base URL (proxy or direct) and the token flow.

### 4.2 Exchange credentials (Kalshi, Polymarket) — for backend only

- **What**: Used by the Valence backend to talk to exchanges. Not for end-users to call Valence API; needed so the backend can place/cancel orders and stream data.
- **Kalshi**: API key ID + RSA private key (PEM). Often stored in `~/.kalshi/` (e.g. env file with `KALSHI_API_KEY`, `KALSHI_SECRET_KEY_PATH` or inline key). Document that operators must obtain these from Kalshi and configure the backend; link to Kalshi’s official "API key" / "Generate API key" docs.
- **Polymarket**: API key, secret, passphrase, optional builder/relay keys; env vars like `POLYMARKET_API_KEY`, etc. Same idea: operators get from Polymarket and configure backend; link to Polymarket API docs.
- **Where to document**: "Get access" / "Account access": onboarding checklist, security requirements, and a subsection "Exchange API keys" that explains these are for backend configuration, with links to Kalshi/Polymarket instructions for generating keys. Optionally a small table: exchange, env vars, link to vendor’s key generation page.

## 5. End-to-end setup (how the whole setup works)

Provide a single "Setup" or "How setup works" flow that ties everything together:

1. **Request access** to Valence (intake form / invite).
2. **Get credentials**: Valence access (e.g. session/token flow) and, if you run/operate the backend, exchange API keys (Kalshi, Polymarket) from the respective vendor sites.
3. **Configure backend** (if applicable): env vars, `~/.kalshi/` or equivalent for Kalshi; Polymarket env vars. Mention `KALSHI_ENV` / prod vs test.
4. **Run or connect to backend**: Local: `make webserver` / `make webserverd`; frontend points at backend (or proxy). Hosted: use provided base URL.
5. **Get a Valence API token**: `POST /api/auth/token`, then use Bearer token for REST and (if supported) WebSocket.
6. **Place first order**: e.g. `POST /api/orders` with symbol, side, price, quantity; confirm in Terminal or via `GET /api/orders` and trades.
7. **Optional**: WebSocket for books/fills; quoters/tickers if using systematic features.

Keep this as a linear "Setup" page and cross-link from Getting started, Account access, and API overview so the same story is easy to find.

## 6. Suggested doc structure (navigation)

- **Getting started**
  - Overview (what Valence is, who it’s for)
  - **Setup** (end-to-end flow above; "how the whole setup works")
  - **Account access** (onboarding, roles, exchange API keys, links to generate keys)
- **Platform**
  - Terminal (main UI modules, how it uses the API)
  - Unified execution (venue abstraction, symbol format, order lifecycle)
  - Supported markets
- **Trading operations**
  - How orders work (including symbol format, POST vs cancel)
  - Risk / limits (as already planned)
- **API reference**
  - **Overview** (auth = Bearer from `/api/auth/token`, base URL, environments)
  - **Authentication** (token lifecycle, how to send Bearer, refresh)
  - **Endpoints** (from OpenAPI or MDX): auth, orders, orders/consolidated, cancel, books, inventory, trades, fair, quoters, tickers, WebSocket
  - **OpenAPI** (link to spec + playground)

Add a **Code examples** or **Quick integration** page under API reference or Getting started with minimal snippets and POST examples; link to it from Overview and Setup.

## 7. Terminology and style (for AGENTS.md)

- **Valence API** = tradegui REST + WebSocket (Bearer token).
- **Exchange API keys** = Kalshi/Polymarket keys used by the backend only; "generate API key" links go to vendor docs.
- **Terminal** = the Valence web UI (Next.js).
- **Backend** = tradegui (Rust service).
- **Symbol** = prefixed ticker, e.g. `kalshi:TICKER`, `polymarket:CONDITION_ID`.
- Use "you" and active voice; keep one idea per sentence; bold UI elements; code for paths, env vars, and JSON fields.

## 8. Out of scope (content boundaries)

- Internal repo layout (crates, Makefile targets) unless needed for local dev.
- Pulumi/iac secrets layout and per-tenant env vars.
- NextAuth whitelist or deployment-specific proxy details (only high-level "requests may go through a proxy").
- Polymarket/Kalshi internal APIs; only how operators get keys and where to set env vars.

---

Use this plan to:
- Add or rewrite pages under Getting started, Platform, Trading operations, and API reference.
- Replace the placeholder OpenAPI with Valence endpoints and ensure POST links work in the API playground.
- Update AGENTS.md with terminology and content boundaries from §7 and §8.
