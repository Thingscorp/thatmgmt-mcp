# thatmgmt-mcp

[![license](https://img.shields.io/badge/license-MIT-blue)](LICENSE)
[![npm](https://img.shields.io/npm/v/@thatmgmt/mcp)](https://www.npmjs.com/package/@thatmgmt/mcp)

An MCP (Model Context Protocol) server that wraps the ThatMgmt domain API. **Check availability, get locked price quotes, and prepare registrations from any MCP client — with zero signup for the public tools.** Tenant tools (portfolio, suggestions, dry-run planning) unlock with an API key. Nothing executes blindly: the API exposes no execute endpoints, so this server plans and prices, never purchases.

Base API: `https://api.thatmgmt.com`
Spec: [openapi.json](https://thatmgmt.com/openapi.json)
Machine docs: [llms-full.txt](https://thatmgmt.com/llms-full.txt)

## Verified claims

Every claim below is backed by a test in `test/` — fully mocked, no network, no key, no charges. Run them yourself: `npm test` (47/47 pass).

| Claim | Backing data |
|---|---|
| 13 MCP tools, exactly 4 of them public | `test/tools.test.js` "exposes the expected tool set" asserts all 13 names; "exactly the zero-signup tools are marked public" pins `tmgmt_health`, `tmgmt_capabilities`, `domains_check_availability`, `domains_get_quote` |
| Tenant tools fail closed without an API key | `test/server.test.js` "refuses tenant tools without an API key": `isError` true, message names `TMGMT_API_KEY` |
| Public tools send no Authorization header | `test/server.test.js` "public reads work with no API key and send no Authorization header" asserts the header is absent on all 4 public routes |
| The API key never leaks into error messages | `test/thatmgmt.test.js` "never leaks the API key in error messages": key replaced with `[REDACTED]` |
| Exactly one retry on retryable GET / 429 / 5xx; none on 400 or non-retryable 503 | `test/thatmgmt.test.js`: 2 fetch calls after a 429; 1 call for a 400 and for a 503 marked non-retryable |
| The approval gate refuses without a quote id, without `approved: true`, and on a mismatched id | `test/tools.test.js` approval-gate cases |
| Invalid domains are rejected before any network call | `test/tools.test.js` "callTool rejects an invalid domain before any network call": 0 fetch calls |
| A quote for an unavailable domain says so plainly instead of returning bare `{available:false}` | `test/tools.test.js` quote-shaping cases |
| Transfer dry-runs pass `authCodePresent`, never the raw auth code | `test/tools.test.js` "passes authCodePresent (never the raw code)" |

## Quickstart

```bash
git clone https://github.com/thingscorp/thatmgmt-mcp.git && cd thatmgmt-mcp
npm install
node demo-agent-run.mjs   # real server over stdio, stub API, zero signup
```

The demo drives the real server through `tmgmt_health` → availability → locked quote → prepare-registration, with no network and no key. Live mode (public tools only, read-only): `node demo-agent-run.mjs --live`.

## Install

From npm (no clone needed):

```bash
npx -y @thatmgmt/mcp
```

From source (Node 18+):

```bash
git clone https://github.com/thingscorp/thatmgmt-mcp.git thatmgmt-mcp
cd thatmgmt-mcp
npm install
npm test
```

Add to your MCP client (Claude Code `.mcp.json`, Cursor, Replit):

```json
{
  "mcpServers": {
    "thatmgmt": {
      "command": "npx",
      "args": ["-y", "@thatmgmt/mcp"],
      "env": { "TMGMT_API_KEY": "your-thatmgmt-api-key" }
    }
  }
}
```

The key is only needed for tenant tools; the public reads work without it.

Releases are automated: pushing a version tag (e.g. `v0.2.2`) runs the **Publish to npm** workflow (`npm publish` via the `NPM_TOKEN` secret) and the **Publish to MCP Registry** workflow (validates `server.json`, publishes `io.github.thingscorp/thatmgmt-mcp` via GitHub OIDC). The tag must match `version` in `package.json` and `server.json`.

## Usage

The two-step purchase flow is written into every tool description so agents show the human the price first:

1. **Quote.** Call `domains_get_quote`. It returns the locked price: `wholesaleCents`, the itemized 1% cut (`platformCutCents`, `platformCutBasisPoints`), and `totalCents`. No key needed. Show this to the human.
2. **Plan.** Call `domains_prepare_registration` (needs `TMGMT_API_KEY`). It returns the safety-checked plan. It never executes anything.

A quote carries wholesale, the flat 1% platform cut, and the total — no subscription, no tiers. The shape, from the mocked test fixture: `wholesaleCents: 1299`, `platformCutCents: 13`, `platformCutBasisPoints: 100`, `totalCents: 1312`.

### What this server cannot do

The ThatMgmt API exposes **no execute endpoints**. Purchase, renewal, transfer, and DNS changes are never executed by the API; the API returns validated plans and preflights instead. This server therefore cannot register, renew, or transfer a domain, and it will never claim it did. `src/approval.js` holds the approval gate that future execute tools will use (quote id passed back plus an explicit `approved: true` flag, or the call is refused) — implemented and tested now so the safety design is ready the day execute routes exist.

Current release: 0.2.2 (read-only public tools plus validated plans). Execute tools are planned for the 0.3.0 release.

## Configuration

Env vars only — no flags, no config file:

| Variable | Required | Default | What it does |
|---|---|---|---|
| `TMGMT_API_KEY` | Only for tenant tools | — | Tenant API key, sent as a Bearer token. Never logged; redacted from errors. |
| `TMGMT_BASE_URL` | No | `https://api.thatmgmt.com` | API base URL override. |
| `TMGMT_TIMEOUT_MS` | No | `30000` | Per-request timeout in milliseconds. |

Transient failures (network errors, timeouts, 429s, retryable 5xx) are retried once on side-effect-free calls; the server never retries anything that could move money.

## API reference

The 13 tools the server exposes over MCP stdio, exactly as defined in `src/tools.js`. Each maps to a real route in the [live OpenAPI spec](https://thatmgmt.com/openapi.json); no invented endpoints.

| Tool | What it does | API route | Key needed |
|---|---|---|---|
| `tmgmt_health` | Liveness check | `GET /health/live` | No |
| `tmgmt_capabilities` | Discover the public no-auth surface | `GET /v1/public/capabilities` | No |
| `domains_check_availability` | Check if a domain is available | `GET /v1/public/availability` | No |
| `domains_get_quote` | Locked price quote: wholesale + itemized 1% cut + total (step 1) | `GET /v1/public/quote` | No |
| `domains_suggest` | Suggest alternative names | `GET /v1/domains/suggestions` | Yes |
| `domains_prepare_registration` | Safety-checked plan, never executes (step 2) | `GET /v1/domains/prepare-registration` | Yes |
| `domains_list` | List portfolio domains | `GET /v1/domains` | Yes |
| `domains_dns` | Inspect DNS records for a domain | `GET /v1/domains/{resourceId}/dns` | Yes |
| `portfolio_health` | Portfolio health summary | `GET /v1/portfolio/health` | Yes |
| `portfolio_renewal_risk` | Renewal-risk view | `GET /v1/portfolio/renewal-risk` | Yes |
| `portfolio_exceptions` | Prioritized exception queue | `GET /v1/portfolio/exceptions` | Yes |
| `orders_dry_run` | Full order plan with itemized pricing, never moves money | `POST /v1/orders/dry-run` | Yes |
| `offerings` | Offering coverage matrix | `GET /v1/offerings` | Yes |

Public tools never send an Authorization header and never ask for a key. Tenant tools return a 401-style refusal without a key; the server tells you to set `TMGMT_API_KEY`.

## For agents

`skills/thatmgmt/SKILL.md` is the agent skill for this server: when to use it, the tool list, and the two-step purchase flow. Point your agent at it for the fastest start.

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md).

## License

MIT. See [LICENSE](LICENSE).
