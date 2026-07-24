---
name: blackbird-member-passport
description: Read a signed-in Blackbird member's identity and dining history via the Flynet API — profile, status, wallets, and check-ins — using an OAuth bearer token.
api: openapi/blackbird-flynet-openapi-original.yml
operations: [getMe, getMyStatus, listMyWallets, listMyCheckIns, listMemberships]
method: generated
source: openapi/blackbird-flynet-openapi-original.yml + conventions/blackbird-conventions.yml
---

# Build a member passport on Flynet

Read the authenticated Blackbird member's profile, standing, wallets, and visit
history. All routes here are member routes: authenticate with an **OAuth bearer
JWT** (`Authorization: Bearer <token>`), never the `X-API-Key`.

## Auth

- Obtain an access token via OAuth 2.0 + PKCE (see `scopes/blackbird-scopes.yml`).
- Required scopes: `read:profile` (profile + status), `read:wallets`,
  `read:user_checkins`.
- A token missing a scope returns **403 insufficient_scope** (empty body, reason
  in the `WWW-Authenticate` header) — not a 404. A missing/expired token returns
  an **empty-body 401**.

## Steps

1. `getMe` — `GET /users/me`. The subject is resolved from the token's `sub`
   claim; there is no `/users/{id}` form.
2. `getMyStatus` — `GET /users/me/status`. If no active status exists the route
   returns **404 resource_not_found** (branch on status, not an empty object).
3. `listMyWallets` — `GET /users/me/wallets`. A member usually has two wallets:
   `MEMBERSHIP` and `SPENDING` (auto-minted on first OAuth completion).
4. `listMyCheckIns` — `GET /users/me/check_ins`. Paginated (`page` zero-indexed,
   `page_size` default 50); read `pagination.next_page` (null on last page).
5. `listMemberships` — `GET /users/me/memberships`. Per-restaurant membership
   records; accepts a `restaurant` filter (unknown filter params are ignored).

## Rules

- Money on wallets uses the `{ value, currency }` shape — `value` is a
  stringified integer in FLY wei. Don't hand-roll decimal math.
- Keep the access token in memory only; refresh via the backend when it expires
  (60-minute TTL).
