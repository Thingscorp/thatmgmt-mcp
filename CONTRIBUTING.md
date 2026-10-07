# Contributing

## Ground rules

- This is a docs-and-safety surface over a live reseller API. **No protocol changes**: tool names, descriptions, and routes must keep matching `src/tools.js` and the [live OpenAPI spec](https://thatmgmt.com/openapi.json).
- **Additive only.** Never remove or rename an MCP tool, an env var, or a field in a tool description without a version bump and a changelog note in the PR.
- Tool descriptions must stay accurate: if a description promises a field or behavior, a test in `test/` must pin it.

## Workflow

1. Fork, branch from `main`.
2. Node 18+. `npm install`, then `npm test` — the suite is fully mocked (`node --test test/*.test.js`); no live calls, no API key needed. All 47 tests must pass.
3. Keep the README's Verified-claims table honest: every user-facing claim you add needs a mocked test backing it.
4. Open a PR against `main`. Describe what changed, what test covers it, and what you deliberately did not change.

## Releases

Maintainers only. Bump `version` in `package.json` **and** `server.json` (they must match), push to `main`, then tag `v<version>` — the tag fires the npm and MCP-registry publish workflows. Never publish from a local machine.
