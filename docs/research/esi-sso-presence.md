# Research: ESI SSO, alliance gate, and Presence constraints

**Ticket:** [#2](https://github.com/TomSOIreland/wh-tracker/issues/2)  
**Fetched:** 2026-10-30  
**Primary sources:** [SSO docs](https://developers.eveonline.com/docs/services/sso/), [ESI OpenAPI](https://esi.evetech.net/meta/openapi.json), live `esi.evetech.net`, [Developer License Agreement](https://developers.eveonline.com/license-agreement), ESI rate-limit / best-practice docs, CCP developer blog posts.

## Verdict for the product+tech spec

Use **OAuth 2.0 Authorization Code** (confidential client: server-held `client_secret`) for a Dokploy-hosted webapp. Gate Members via public ESI affiliation/`alliance_id` against a configurable allowlist that includes **Goonswarm Federation `1354830081`**. Presence polls `GET /characters/{id}/location` (and optionally ship/online) under the shared **`char-location`** bucket (**`1200` tokens / `15m`** per app+character). Cosmic signatures, wormhole destinations, and bookmarks are **not** available via ESI — Intel stays Member-contributed. Consent and sharing of location among alliance Members must be explicit (SSO scopes + app UX) under the Developer License Agreement §2.3(c).

---

## 1. SSO: Authorization Code (confidential client)

[SSO documentation](https://developers.eveonline.com/docs/services/sso/) states:

- Register the app on the EVE Developers Portal → **client ID + secret**. Client ID is public; **secret must stay private**.
- **Authorization Code** is the flow for web apps that can store the secret server-side (vs Authorization Code + PKCE for clients that cannot).
- Player must **explicitly consent** to every requested scope (or cancel). App may only request scopes assigned at registration.
- Redirect URI must be **pre-registered**; others are rejected.
- Use a random **`state`** and verify it on return (CSRF).
- Token exchange: `POST` to the token endpoint with `grant_type=authorization_code` and **HTTP Basic** (`client_id:client_secret`). Returns access + refresh tokens.
- Discover endpoints from `https://login.eveonline.com/.well-known/oauth-authorization-server` (cache reasonably). Live metadata (2026-10-30) includes:
  - `authorization_endpoint`: `https://login.eveonline.com/v2/oauth/authorize`
  - `token_endpoint`: `https://login.eveonline.com/v2/oauth/token`
  - `jwks_uri`: `https://login.eveonline.com/oauth/jwks`
  - `code_challenge_methods_supported`: `S256` (PKCE available; not required for confidential Authorization Code)
- Access token is a JWT: verify signature via JWKS; accept issuers `https://login.eveonline.com` / `login.eveonline.com`; `aud` must include your `client_id` and `"EVE Online"`; `sub` is `CHARACTER:EVE:{character_id}`; `scp` lists granted scopes.
- Refresh tokens are long-lived until the player revokes access; **store them securely** (they mint new access tokens).

**Spec must-haves:** Dokploy app keeps `client_secret` and refresh tokens only on the server; registered HTTPS callback; Authorization Code (not implicit); JWT validation before trusting character id / scopes.

---

## 2. Presence scopes and endpoints

From the live OpenAPI at [`https://esi.evetech.net/meta/openapi.json`](https://esi.evetech.net/meta/openapi.json) (OpenAPI 3.1, title “EVE SKINR Ingenuity (ESI) - tranquility”, 2026-10-30):

| Purpose | Method / path | Scope | Cache (`x-cache-age` / client TTL) | Response (required fields) |
| --- | --- | --- | --- | --- |
| Solar-system Presence | `GET /characters/{character_id}/location` | `esi-location.read_location.v1` | **5 s** | `solar_system_id`; optional `station_id`, `structure_id` |
| Online flag | `GET /characters/{character_id}/online` | `esi-location.read_online.v1` | **60 s** | `online`; optional login stats |
| Current ship | `GET /characters/{character_id}/ship` | `esi-location.read_ship_type.v1` | **5 s** | `ship_type_id`, `ship_item_id`, `ship_name` |

OpenAPI `security` on each route is OAuth2 Authorization Code with exactly that scope. Base host in practice: `https://esi.evetech.net/latest/...` (also versioned `/vN/...`; prefer `X-Compatibility-Date` per [ESI overview](https://developers.eveonline.com/docs/services/esi/overview/)).

**Minimal Presence surface:** `esi-location.read_location.v1` alone yields `solar_system_id`. Ship/online are optional product extras, each adding a scope on the consent screen and sharing the same rate-limit group (below).

Unauthenticated probes to location/online/ship return `401` with live headers confirming rate-limit group `char-location` (2026-10-30).

---

## 3. Alliance allowlist / Goonswarm Federation

**GSF alliance id `1354830081` is correct.** Live `GET /alliances/1354830081` returned:

```json
{"name":"Goonswarm Federation","ticker":"CONDI", ...}
```

(2026-10-30; public, ~1 h cache per OpenAPI `x-cache-age: 3600`.)

### How to verify a signed-in character is in an allowed alliance

No special “membership” scope is required for the gate — affiliation is **public**:

1. After SSO, take `character_id` from JWT `sub`.
2. Resolve alliance via either:
   - `GET /characters/{character_id}` → optional `alliance_id` (+ required `corporation_id`), or
   - `POST /characters/affiliation` with a JSON array of character ids → `{character_id, corporation_id, alliance_id?}` (OpenAPI: bulk affiliation; cache age 3600 s).
3. Allow only if `alliance_id` ∈ configured allowlist (default includes `1354830081`).
4. Optional sanity: `GET /corporations/{corporation_id}` also exposes `alliance_id`; `GET /alliances/{alliance_id}/corporations` lists member corps (GSF returned **810** corp ids on 2026-10-30).

Re-check affiliation on a cadence (corp/alliance moves are public game state); do not treat SSO alone as proof of ongoing alliance membership.

---

## 4. Presence polling: rate limits and cache

### Shared bucket: `char-location`

OpenAPI `x-rate-limit` on location, online, and ship:

```json
{"group": "char-location", "max-tokens": 1200, "window-size": "15m"}
```

Live `401` responses agreed: `X-Ratelimit-Group: char-location`, `X-Ratelimit-Limit: 1200/15m` (2026-10-30).

Per [rate limiting docs](https://developers.eveonline.com/docs/services/esi/rate-limiting/):

- Bucket is per **rate-limit group × (applicationID + characterID)** for authenticated routes.
- Token costs: **2XX → 2**, **3XX → 1**, **4XX → 5** (not 429), **5XX → 0**.
- Over limit → **429** + `Retry-After` (seconds).
- Floating window: consumed tokens return after the window (~15 m).
- Do not run at the limit; stagger periodic jobs; slow down as `X-Ratelimit-Remaining` approaches 0.

[Hold your horses](https://developers.eveonline.com/blog/hold-your-horses-introducing-rate-limiting-to-esi) (updated 2025-12-01) notes `/location`, `/online`, `/ship` share this group; cache times match OpenAPI (two at 5 s, online at 1 m); limits were aimed between the 99th and 99.9th percentile of historical usage (≈1000–1800 tokens / 15 m). Live cap observed: **1200 / 15 m**.

**Rough budget (all 2XX):** 1200 ÷ 2 = **600 successful calls / 15 m / character** across location+online+ship combined. Polling location once per **≥5 s** cache uses ≤180 calls / 15 m (360 tokens) if every call is a fresh 200 — comfortable headroom if the app respects `Expires` / ETag.

### Cache discipline ([best practices](https://developers.eveonline.com/docs/services/esi/best-practices/))

- Honor `Expires` / `Cache-Control`; **do not poll faster than cache age** — circumventing ESI cache can get the app banned.
- Prefer `If-None-Match` / `If-Modified-Since` → **304** costs **1** token instead of 2.
- Still track the older **error limit**: ≤100 non-2xx/3xx per minute → then **420** on all routes (`X-ESI-Error-Limit-Remain` / `Reset`). Mutually exclusive with new rate-limit headers when the new limiter applies.
- Send a proper User-Agent / `X-User-Agent` with contact email and app name+version.

**Spec guidance for opt-in Presence:** poll only Members who opted in; default interval ≥ max(cache TTL, product need) — e.g. location every 5–30+ s, online ≤ every 60 s; stop when offline or opt-out; backoff on 429; never burst all characters on the same clock tick.

---

## 5. Consent, ToS, and sharing location among Members

Must sign / comply with the [Developer License Agreement](https://developers.eveonline.com/license-agreement) before creating apps ([license index](https://developers.eveonline.com/docs/resources/license/)).

Material constraints for this product:

| Clause | Implication |
| --- | --- |
| §1.13 Purpose | App must enhance EVE enjoyment; **non-commercial / non-profit** with limited monetization exceptions in §4.4 (ISK access fees, voluntary cost-offset donations, non-interfering ads). |
| §2.3(c) | Must not track Player information/activity **without express knowledge and consent**. SSO scope grant is necessary but the app should also make Presence sharing (who sees your system) unmistakable in UX. |
| §2.3(d)–(e) | No scams / theft of items or info; no DoS against EVE/ESI. Reinforced in [Reminder: No Scamming](https://developers.eveonline.com/blog/reminder-to-developers-no-scamming). |
| §2.5 | CCP may impose call-frequency limits; abuse → ban. |
| §9.5 | Comply with applicable **privacy / data-protection** law for stored Presence and Intel. |
| §7.1 | Retain CCP proprietary notice on Licensed Materials / marks as specified. |

**Sharing Presence with other alliance Members:** ESI returns location only to the app holding that character’s token. Redistributing that Game Data inside the alliance-private app is a product/policy choice that still requires the tracked Player’s **express knowledge and consent** (§2.3(c)) — implement as opt-in Presence, clear copy that alliance Members can see your system, and easy revoke (SSO revocation + in-app opt-out). Restrict visibility to authenticated Members who pass the alliance allowlist.

---

## 6. What ESI does **not** provide (Intel remains Member-contributed)

### No cosmic signatures / wormhole destinations

The live OpenAPI path set (**182** paths) contains **no** routes whose paths mention `signature`, `wormhole`, `anomaly`, or `cosmic` (verified 2026-10-30 against [`/meta/openapi.json`](https://esi.evetech.net/meta/openapi.json)). Location yields only the character’s current `solar_system_id` (and dock target) — not scan results or WH jump destinations.

**Spec assumption:** Signature/Wormhole/Chain Intel is **Member-contributed** (and/or other non-ESI sources such as Anoik for static system profile). Do not design as if ESI can enumerate signatures.

### Bookmarks removed

- [Deprecation blog (2025-02-03)](https://developers.eveonline.com/blog/deprecation-and-impending-removal-of-esi-opportunities-and-esi-bookmarks): bookmark endpoints scheduled for removal (`/characters/.../bookmarks/`, folders, corp equivalents).
- [Return of the Developers blog](https://developers.eveonline.com/blog/the-return-of-the-developers-blog): ACL/shareable bookmarks are not supported by ESI; endpoint had returned empty lists; **no plans to restore**.
- Live check 2026-10-30: `GET /latest/characters/1/bookmarks/` → **404**; OpenAPI has **no** bookmark paths.

Do not plan WH destination recovery via ESI bookmarks.

---

## 7. Concrete checklist for the product+tech spec

1. **Auth:** Authorization Code confidential client; server-only secret + refresh tokens; registered Dokploy callback; JWT validation; Login with EVE button art per SSO docs.
2. **Scopes at registration / consent:** at least `esi-location.read_location.v1`; add `esi-location.read_ship_type.v1` / `esi-location.read_online.v1` only if the product needs them.
3. **Alliance gate:** compare public `alliance_id` to allowlist including **`1354830081` (CONDI / Goonswarm Federation)**; re-validate periodically.
4. **Presence:** opt-in; poll `GET .../location` no faster than **5 s** cache; share `char-location` **1200/15m** budget across location/ship/online; respect 304/429/error-limit/User-Agent.
5. **Policy:** DLA signed; explicit consent UX for tracking + sharing Presence with alliance Members; non-commercial constraints; privacy compliance for stored Game Data.
6. **Non-goals via ESI:** cosmic signatures, WH destinations, bookmarks — Intel is Member-contributed.

---

## Sources

- [Single Sign-On (SSO)](https://developers.eveonline.com/docs/services/sso/)
- [OAuth authorization server metadata](https://login.eveonline.com/.well-known/oauth-authorization-server)
- [ESI OpenAPI](https://esi.evetech.net/meta/openapi.json) (and [esi-swagger-specs README](https://github.com/esi/esi-swagger-specs) pointing here as current)
- [ESI overview](https://developers.eveonline.com/docs/services/esi/overview/)
- [ESI rate limiting](https://developers.eveonline.com/docs/services/esi/rate-limiting/)
- [ESI best practices](https://developers.eveonline.com/docs/services/esi/best-practices/)
- [Hold your horses: introducing rate limiting to ESI](https://developers.eveonline.com/blog/hold-your-horses-introducing-rate-limiting-to-esi)
- [Developer License Agreement](https://developers.eveonline.com/license-agreement)
- [License index](https://developers.eveonline.com/docs/resources/license/)
- [Reminder to Developers: No Scamming](https://developers.eveonline.com/blog/reminder-to-developers-no-scamming)
- [Deprecation/removal of esi-bookmarks](https://developers.eveonline.com/blog/deprecation-and-impending-removal-of-esi-opportunities-and-esi-bookmarks)
- [Temporary blackout / no restore of bookmarks](https://developers.eveonline.com/blog/the-return-of-the-developers-blog)
- Live ESI: `GET /alliances/1354830081`, `GET /alliances/1354830081/corporations/`, unauthenticated location/online/ship/bookmarks probes (2026-10-30)
