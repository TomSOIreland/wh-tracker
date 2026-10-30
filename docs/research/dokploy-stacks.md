# Boring stacks that fit Dokploy + ESI webapps

Research for [Research: Boring stacks that fit Dokploy + ESI webapps](https://github.com/TomSOIreland/wh-tracker/issues/5). Short comparison for a later stack pick — **not** a final choice.

## Requirements this comparison uses

From the map destination and this ticket:

- Host on **Dokploy on-prem** (operator-managed).
- **ESI SSO** as a confidential-client webapp (server holds `client_secret`; Authorization Code, not PKCE-only public clients).
- **Postgres**-friendly persistence (Members, Intel, Presence, Claims, Chain).
- **Live-ish** shared updates for Presence / Intel / Claims (WebSockets, SSE, or polling are all in scope).

## Dokploy deployment model (shared constraint)

Dokploy is a self-hostable Docker + Traefik platform ([Welcome](https://docs.dokploy.com/)). Two primary deploy shapes matter for this app ([Features](https://docs.dokploy.com/docs/core/features)):

| Shape | What it is | Fit for WH Tracker |
| --- | --- | --- |
| **Application** | One service/container; build via Nixpacks (default), Railpack, Dockerfile, Buildpacks, or Static ([Build Type](https://docs.dokploy.com/docs/core/applications/build-type)) | Fine for a single web process; pair with a separate Dokploy **Postgres** database service ([Databases](https://docs.dokploy.com/docs/core/databases)) |
| **Docker Compose** | Multi-service `docker-compose.yml`; Git or raw compose; env via `.env` / UI ([Docker Compose](https://docs.dokploy.com/docs/core/docker-compose)) | Better when the app needs web + worker + Redis (or similar) in one unit; join `dokploy-network`, avoid `container_name` ([Compose example](https://docs.dokploy.com/docs/core/docker-compose/example)) |

Postgres is a first-class Dokploy database type with backups ([Databases](https://docs.dokploy.com/docs/core/databases)). Named volumes (or `../files` bind mounts) persist data across deploys ([Docker Compose volumes](https://docs.dokploy.com/docs/core/docker-compose)).

Dokploy’s own Compose tutorial uses a **Next.js** app as the worked example ([Compose example](https://docs.dokploy.com/docs/core/docker-compose/example)). Railpack explicitly lists **Node.js, Python, Go** among supported languages ([Build Type](https://docs.dokploy.com/docs/core/applications/build-type)). For production, Dokploy recommends building a Dockerfile image off-box (registry pull) rather than heavy on-server Nixpacks builds ([Going Production](https://docs.dokploy.com/docs/core/applications/going-production)).

**Implication for every stack:** anything that runs as a long-lived HTTP(S) process in Docker deploys cleanly. Stacks that need a second process (Action Cable / Channels worker, Sidekiq, Celery) lean toward Compose or two Dokploy Applications sharing one Postgres.

## ESI confidential-client OAuth (shared constraint)

EVE SSO is OAuth 2.0 ([SSO docs](https://developers.eveonline.com/docs/services/sso/)). Registration yields a **client ID and secret**; the secret must stay private.

For webapps that can store a secret server-side, CCP documents the **Authorization Code** flow: redirect to authorize → receive `code` → `POST` to the token endpoint with **HTTP Basic** (`client_id:client_secret`) and `grant_type=authorization_code` ([Authorization Code](https://developers.eveonline.com/docs/services/sso/)). PKCE is documented for clients that **cannot** securely store a secret (mobile/desktop), not as a substitute for confidential webapps ([Authorization Code with PKCE](https://developers.eveonline.com/docs/services/sso/)).

Refresh tokens are long-lived and must be stored securely; access tokens are JWTs validated via JWKS from the SSO metadata URL ([Validating JWT Tokens](https://developers.eveonline.com/docs/services/sso/)).

**Implication:** every stack needs a **server-side** token exchange and secret/refresh-token storage. SPA-only / browser-held secret patterns are out. Session cookies or server sessions after SSO are the boring pattern.

---

## Stack comparison

Axes: Dokploy fit · confidential ESI OAuth · live updates · Postgres · “boring” assembly cost.

### 1. Next.js (App Router)

| Axis | Notes |
| --- | --- |
| **Dokploy** | Strong. Official Dokploy Compose example is a Next app ([Compose example](https://docs.dokploy.com/docs/core/docker-compose/example)). Next documents Node server and Docker deploys with full feature support ([Deploying](https://nextjs.org/docs/app/getting-started/deploying)). Nixpacks/Railpack Node path applies ([Build Type](https://docs.dokploy.com/docs/core/applications/build-type)). |
| **ESI OAuth** | Fits if token exchange and secret live in Server Actions / Route Handlers / a data-access layer — Next’s auth guide centers server-side sessions and warns that Server Actions are public POST endpoints needing auth checks ([Authentication](https://nextjs.org/docs/app/guides/authentication), [Data security](https://nextjs.org/docs/app/guides/data-security)). No first-party “ESI plugin”; implement Authorization Code against CCP’s endpoints. |
| **Live updates** | Route Handlers can stream via Web Streams; docs call out Server-Sent Events as a use case ([Streaming](https://nextjs.org/docs/app/guides/streaming)). Next’s BFF guide notes WebSockets are a poor fit for Route Handlers on hosts that treat them like short-lived functions ([Backend for frontend](https://nextjs.org/docs/app/guides/backend-for-frontend)) — on a long-lived Node container on Dokploy that constraint is softer, but **first-party WS is still not a built-in product feature**; SSE or polling is the boring path, or a separate WS process. |
| **Postgres** | Via app libraries (e.g. driver + query layer); not baked into the framework. Works with Dokploy’s Postgres service over `DATABASE_URL`. |
| **Boring?** | Fast UI iteration; more glue for OAuth + realtime + schema than Rails/Django. |

### 2. Ruby on Rails

| Axis | Notes |
| --- | --- |
| **Dokploy** | Deploy as Dockerfile Application or Compose (web + optional jobs). Rails is a conventional single (or dual) long-running process behind Traefik; no Dokploy-specific blocker. Prefer Dockerfile for production per Dokploy’s guide ([Going Production](https://docs.dokploy.com/docs/core/applications/going-production)). |
| **ESI OAuth** | Classic confidential-client home: server sessions, secrets in credentials/env. Rails security guide covers session cookies and HTTPS expectations ([Securing Rails Applications](https://guides.rubyonrails.org/security.html)). Wire Authorization Code to CCP’s token endpoint in a controller/service (any HTTP client); OmniAuth-style providers are common in the ecosystem but ESI itself is app-specific. |
| **Live updates** | **Action Cable** is first-party WebSockets with a **PostgreSQL adapter** using the app’s `database.yml` / Active Record pool ([Action Cable Overview](https://guides.rubyonrails.org/action_cable_overview.html)). Auth over cookies on connect is documented. `NOTIFY` payloads capped at 8000 bytes — fine for small Presence/Claim events, not huge blobs. Polling remains available as a fallback. |
| **Postgres** | First-class: `rails new --database=postgresql`, `pg` gem, Active Record PostgreSQL guide ([Command Line](https://guides.rubyonrails.org/command_line.html), [Active Record and PostgreSQL](https://guides.rubyonrails.org/active_record_postgresql.html)). |
| **Boring?** | Highest “batteries included” score for OAuth session app + Postgres + realtime in one framework. |

### 3. Python — Django (and FastAPI as a thinner alternative)

Treat as one family: **Django** for full-stack boring; **FastAPI** if the UI is a separate SPA and the API is the product surface.

| Axis | Notes |
| --- | --- |
| **Dokploy** | Railpack lists Python ([Build Type](https://docs.dokploy.com/docs/core/applications/build-type)). Django ASGI deploy is documented ([How to deploy with ASGI](https://docs.djangoproject.com/en/6.1/howto/deployment/asgi/)); FastAPI typically runs under Uvicorn in Docker. Channels/realtime often wants Redis → Compose is the natural Dokploy shape. |
| **ESI OAuth** | Django: custom auth backend / views holding `client_secret`, sessions after code exchange ([Customizing authentication](https://docs.djangoproject.com/en/6.1/topics/auth/customizing/)). FastAPI: OAuth2/JWT patterns are first-class in the security tutorial (adapt token URL/issuer to EVE SSO) ([OAuth2 with Password (and hashing), Bearer with JWT tokens](https://fastapi.tiangolo.com/tutorial/security/oauth2-jwt/)) — for ESI you use Authorization Code + Basic client auth per CCP, not the password flow in that tutorial’s example. |
| **Live updates** | Django **Channels** extends Django to WebSockets on ASGI; optional Redis channel layer; deploy docs cover protocol servers ([Channels](https://channels.readthedocs.io/en/latest/)). FastAPI has built-in `@app.websocket` and `StreamingResponse` for stream/SSE-style responses ([WebSockets](https://fastapi.tiangolo.com/advanced/websockets/), [Custom Response / StreamingResponse](https://fastapi.tiangolo.com/advanced/custom-response/)). FastAPI’s in-memory broadcast example is single-process — multi-replica needs Redis/Postgres pubsub (docs point at broadcaster-style libs). |
| **Postgres** | Django officially supports PostgreSQL ([Databases](https://docs.djangoproject.com/en/6.1/ref/databases/)). FastAPI pairs with SQLAlchemy/async drivers against the same Dokploy Postgres. |
| **Boring?** | Django ≈ Rails-shaped. FastAPI ≈ explicit API + more frontend/ops assembly. |

### 4. Go (`net/http` + `database/sql`)

| Axis | Notes |
| --- | --- |
| **Dokploy** | Railpack supports Go ([Build Type](https://docs.dokploy.com/docs/core/applications/build-type)). Single static binary in a Dockerfile is a textbook Application deploy; low memory vs Node/Ruby/Python runtimes. |
| **ESI OAuth** | [`golang.org/x/oauth2`](https://pkg.go.dev/golang.org/x/oauth2) is the standard library-adjacent OAuth2 client for Authorization Code + client secret. Combine with CCP’s authorize/token/JWKS endpoints. You write the session/cookie layer yourself. |
| **Live updates** | Long-lived connections are natural for Go servers. Stdlib focuses on HTTP; WebSockets typically use a widely used third-party upgrader. SSE is straightforward by flushing an `http.ResponseWriter`. Polling is trivial. |
| **Postgres** | Stdlib [`database/sql`](https://pkg.go.dev/database/sql) + a Postgres driver; Go’s data-access docs cover relational DBs and connection pooling ([Accessing relational databases](https://go.dev/doc/database/)). |
| **Boring?** | Excellent ops fit; **highest application assembly cost** (ORM/migrations/auth/UI/realtime all chosen à la carte). |

---

## Side-by-side

| | Next.js | Rails | Django / FastAPI | Go |
| --- | --- | --- | --- | --- |
| Dokploy path | App or Compose; documented Next example | Dockerfile App or Compose | App or Compose (+ Redis if Channels) | App (binary) or Compose |
| Confidential ESI | Server Actions / Route Handlers | Controllers + session | Views/backends or FastAPI deps | `x/oauth2` + own session |
| Live-ish (boring) | SSE / polling (WS DIY) | Action Cable + Postgres adapter | Channels or FastAPI WS/SSE | SSE/WS DIY |
| Postgres | External libs | Active Record first-class | ORM first-class (Django) | `database/sql` |
| Glue you write | Medium–high | Low | Low (Django) / Medium (FastAPI) | High |

## What this does **not** decide

No option clearly dominates for WH Tracker:

- **Rails** and **Django** dominate on “boring full-stack + Postgres + first-party realtime,” with nearly identical Dokploy shapes.
- **Next.js** dominates on Dokploy familiarity (their Compose tutorial) and modern ops-board UI velocity, with a weaker built-in WS story (SSE/polling still fine for live-*ish*).
- **Go** dominates on deploy simplicity and resource use, not on time-to-working ESI+Chain product.

A later stack ticket should weigh operator language comfort, whether the UI is React-first, and whether Action Cable / Channels / SSE is the preferred live mechanism — using this file as the options set.

## Sources

- [Dokploy Welcome](https://docs.dokploy.com/)
- [Dokploy Features](https://docs.dokploy.com/docs/core/features)
- [Dokploy Applications](https://docs.dokploy.com/docs/core/applications)
- [Dokploy Build Type](https://docs.dokploy.com/docs/core/applications/build-type)
- [Dokploy Going Production](https://docs.dokploy.com/docs/core/applications/going-production)
- [Dokploy Databases](https://docs.dokploy.com/docs/core/databases)
- [Dokploy Docker Compose](https://docs.dokploy.com/docs/core/docker-compose)
- [Dokploy Docker Compose example](https://docs.dokploy.com/docs/core/docker-compose/example)
- [EVE SSO](https://developers.eveonline.com/docs/services/sso/)
- [Next.js Deploying](https://nextjs.org/docs/app/getting-started/deploying)
- [Next.js Authentication](https://nextjs.org/docs/app/guides/authentication)
- [Next.js Data security](https://nextjs.org/docs/app/guides/data-security)
- [Next.js Streaming](https://nextjs.org/docs/app/guides/streaming)
- [Next.js Backend for frontend](https://nextjs.org/docs/app/guides/backend-for-frontend)
- [Rails Action Cable Overview](https://guides.rubyonrails.org/action_cable_overview.html)
- [Rails Securing Applications](https://guides.rubyonrails.org/security.html)
- [Rails Command Line (PostgreSQL)](https://guides.rubyonrails.org/command_line.html)
- [Active Record and PostgreSQL](https://guides.rubyonrails.org/active_record_postgresql.html)
- [Django ASGI deployment](https://docs.djangoproject.com/en/6.1/howto/deployment/asgi/)
- [Django Databases](https://docs.djangoproject.com/en/6.1/ref/databases/)
- [Django custom auth](https://docs.djangoproject.com/en/6.1/topics/auth/customizing/)
- [Django Channels](https://channels.readthedocs.io/en/latest/)
- [FastAPI WebSockets](https://fastapi.tiangolo.com/advanced/websockets/)
- [FastAPI Custom Response / StreamingResponse](https://fastapi.tiangolo.com/advanced/custom-response/)
- [FastAPI OAuth2 JWT tutorial](https://fastapi.tiangolo.com/tutorial/security/oauth2-jwt/)
- [Go database/sql](https://pkg.go.dev/database/sql)
- [Go accessing relational databases](https://go.dev/doc/database/)
- [golang.org/x/oauth2](https://pkg.go.dev/golang.org/x/oauth2)
