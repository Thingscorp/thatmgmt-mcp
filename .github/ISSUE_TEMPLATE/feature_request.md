---
name: Feature request
about: A new tool, field, or doc improvement
title: "[feature] "
labels: enhancement
---

## What you want

## Why

Which workflow does this unblock (availability check, quote, plan, portfolio…)?

## Constraints

- New tools must map to a real route in the [ThatMgmt OpenAPI spec](https://thatmgmt.com/openapi.json) — no invented endpoints.
- Spend-effect behavior needs the `src/approval.js` gate: quote id passed back plus explicit `approved: true`.
- Every new user-facing claim needs a mocked test in `test/` (see README's Verified claims).
