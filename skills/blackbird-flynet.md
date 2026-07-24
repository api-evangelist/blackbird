---
name: flynet
description: Use when building a Flynet (Blackbird) integration with @flynetdev/core or @flynetdev/react — teaches the two-credential auth model, the verified API rules that trip up first integrations, and how to compose the SDK and its components correctly.
---

# Building on Flynet

Flynet is Blackbird's API for restaurants, locations, member identity, and FLY
payments. Build on it through the published SDK — don't hand-roll `fetch()` or
invent endpoints.

- **`@flynetdev/core`** — framework-agnostic TypeScript client. Two clients, OAuth, typed errors, money + pagination helpers.
- **`@flynetdev/react`** — `FlynetProvider` + TanStack-Query hooks + themeable components, built on `@flynetdev/core`.

When you need an exact field, status code, or scope, read the live docs — connect the **Docs MCP** (`https://flynet-dev-portal.mintlify.app/mcp`) or fetch `https://flynet-dev-portal.mintlify.app/llms-full.txt`. To call the API directly while reasoning, use the **API MCP** (`npx -y @flynetdev/mcp`).

## Two credentials, two route families (the #1 mistake)

The API has two auth schemes. They are **not interchangeable** — using the wrong header is the most common failure.

- **`X-API-Key`** → Discovery routes: `/restaurants*`, `/locations*`. Server-only.
- **`Authorization: Bearer <JWT>`** → member routes: `/users/me/*`, `/check_ins*`, `/memberships`, `/payment_intents*`.

`@flynetdev/core` enforces this with two clients, so a credential can't cross families:

```ts
import { FlynetDiscoveryClient, FlynetMemberClient } from "@flynetdev/core";

const discovery = new FlynetDiscoveryClient({ apiKey: process.env.API_KEY! }); // server-only
const member = new FlynetMemberClient({ accessToken });                        // OAuth JWT
```

## The rules to honor (ground every generated call on these)

1. **Route → auth gating.** API key for Discovery, Bearer JWT for member. Sending the wrong one fails.
2. **`/users/me/*` is canonical.** Member reads resolve the subject from the JWT. There is **no `/users/{id}/*`** form — legacy paths 404.
3. **PKCE is required on `/oauth/authorize`.** A plain authorize call 422s. Token exchange needs both `client_secret` and `code_verifier`. Use the SDK's `FlynetOAuth` / `createPkcePair` — they do this for you.
4. **OAuth 401s have an empty body.** The cause is in the `WWW-Authenticate` header, not JSON.
5. **Wrong scope → 403 `insufficient_scope`, not 404.** The route exists; the token isn't scoped. Scopes: `read:profile` (profile + status), `read:checkins`, `read:wallets`.
6. **Unknown filter params are silently ignored** (e.g. `cohort`/`cuisine` on restaurants; `restaurant`/`neighborhood`/`payments_enabled`/`is_club` on locations). Filter client-side if the server behavior isn't shipped.
7. **No payment webhooks in v1.** After confirming a payment intent, poll `GET /payment_intents/{id}` for terminal state. Don't generate webhook receivers.
8. **`payment0030` = insufficient FLY** on confirm. Pre-check `GET /users/me/wallets`.

Every error from a client call is normalized to a `FlynetError` — branch on `error.kind` / `error.code`, never on raw HTTP shape. Format FLY (18-decimal wei strings) and USD (integer cents) with `formatFly` / `formatUsdCents`; don't hand-roll decimal math.

## `@flynetdev/core` — the curated surface

```ts
// Discovery (API key)
discovery.restaurants.listRestaurants(req?)        // RestaurantList
discovery.restaurants.getRestaurant({ id })        // Restaurant
discovery.restaurants.listRestaurantLocations({ id })
discovery.locations.listLocations(req?)            // LocationList
discovery.locations.getLocation({ id })            // Location
discovery.locations.listLocationOpenHours({ id })  // OpenHoursList

// Member (Bearer JWT) — prefer these curated methods
member.getProfile()        // GET /users/me           · read:profile
member.getStatus()         // GET /users/me/status     · read:profile
member.listWallets()       // GET /users/me/wallets    · read:wallets
member.listCheckIns(req?)  // GET /users/me/check_ins  · read:checkins
member.listVenueCheckIns({ location, pageSize })  // GET /check_ins?location= (anonymized; location required)
member.getCheckIn({ id })  // GET /check_ins/{id}   · read:checkins
member.listMemberships(req?)
```

List responses are `{ <items>, pagination }` (e.g. `{ restaurants, pagination }`); `pagination.nextPage` is `null` on the last page. `getProfile`/`listWallets` are not paginated.

OAuth:

```ts
import { FlynetOAuth } from "@flynetdev/core";
const oauth = new FlynetOAuth({ clientId, clientSecret, redirectUri, audience, scopes: ["read:profile"] });
const { url, state, codeVerifier } = await oauth.getAuthorizeUrl(); // redirect to url; stash state + codeVerifier
const tokens = await oauth.exchangeCode({ code, codeVerifier });    // { access_token, ... }
```

## `@flynetdev/react` — components and provider

Wrap the tree in `FlynetProvider` (it owns the TanStack `QueryClient` — never construct your own). Pass constructed clients:

```tsx
import { FlynetProvider, RestaurantList, UserPassport, ConnectWithBlackbird } from "@flynetdev/react";

<FlynetProvider member={new FlynetMemberClient({ accessToken })}>
  <UserPassport />
</FlynetProvider>
```

Built components: `WalletBadge`, `CheckInFeed`, `RestaurantList`, `RestaurantCard`, `LocationCard`, `OpenHoursBadge`, `NearbyLocations`, `UserPassport`, `RecentVisits`, `ConnectWithBlackbird`. Hooks: `useWallet`, `useCheckIns`, `usePassport`, `useVenueCheckIns`. `RestaurantList` is presentational (pass `restaurants`); member components read from the provider.

**Do not reference as if they exist:** Pay-with-FLY, Tags Strip / TagGate, Payment Receipt, Location Map — not built.

## Never

- Put `API_KEY` or `client_secret` in browser code — both are server-only.
- Use `/users/{id}/*` paths.
- Expect a JSON body on OAuth 401s, or payment webhooks.
- Treat `cohort`/`cuisine`/`neighborhood`/`payments_enabled`/`is_club` as working server-side filters.
- Invent fields, endpoints, or components not in the OpenAPI spec / the lists above.
