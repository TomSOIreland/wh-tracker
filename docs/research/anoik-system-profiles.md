# Anoik System profile data access and license

Research for WH Tracker enrichment: how to obtain Anoik **System profile** facts (J-space class, statics, effects, names), and what attribution / license constraints apply when caching them in an alliance-private Dokploy deployment.

**Bottom line:** Obtain System profiles from Anoik’s public static dump [`https://anoik.is/static/static.json`](https://anoik.is/static/static.json) — there is **no** System-profile API. Anoik is a **static J-space reference** site, **not** a live connection book. Retain CCP’s limited-permission notice; Anoik itself publishes **no** redistributable-data license for that dump.

## What Anoik is

[Anoik.is](https://anoik.is/) is Eric Wastl’s browser app described on his projects page as a “[Database of wormhole information from EVE Online](https://was.tl/projects/)”. The site’s About page identifies the author and contact channels ([About route in `/static/controller.js`](https://anoik.is/static/controller.js), mounted at `/about`).

The client is a single-page app: HTML shells for routes like `/`, `/systems`, `/wormholes`, and `/about` are identical; content is filled from JS after load ([homepage HTML](https://anoik.is/), [`/static/controller.js`](https://anoik.is/static/controller.js)).

## What System profile data exists

All System-profile (and related wormhole-type) reference data ships in one JSON file:

| Item | Source observation |
| --- | --- |
| URL | [`GET /static/static.json`](https://anoik.is/static/static.json) (client requests `?version=<n>` matching an expected schema version) |
| Size / freshness (observed 2026-10-30) | ~1.97 MB; `Last-Modified: Sun, 05 Jan 2025 10:21:29 GMT`; top-level `"version": 11` |
| Top-level keys | `version`, `systems`, `wormholes`, `wormholeclasses`, `effects`, `regions`, `constellations`, `celestialtypes`, `roman` |

### Per-system fields (`systems`)

Observed coverage: **2604** systems (J-named wormhole systems plus `Thera`). Each system object includes:

| Field | Role for WH Tracker “System profile” |
| --- | --- |
| `solarSystemName` | Display / lookup key (also the object key) |
| `solarSystemID` | EVE ID |
| `wormholeClass` | Class key (`c1`…`c6`, `c13`, drifter keys, `thera`, …) |
| `statics` | Array of **wormhole type names** that are that system’s statics (e.g. `["E175"]`) — not live links |
| `effectName` | System effect (or empty / absent effect for many systems) |
| `regionID` / `constellationID` | Geography ids resolved via `regions` / `constellations` |
| `cels` | Packed celestial layout (stars/planets/moons) for Anoik’s map UI |

Class titles and effect scaling live in companion tables (`wormholeclasses`, `effects`). Wormhole **type** properties (lifetime, mass, `dest`/`src` class lists, whether the type can be a static) live under `wormholes`.

### What the dump does **not** contain

- No scanned signatures, no open wormhole instances, no chain / adjacency graph between systems.
- No Member locations or kill-derived “what’s connected right now”.
- `statics` are **catalog properties of a system** (which static WH *types* belong there). On a system page the UI also lists **wanderers** by looking up wormhole types whose `src` includes that system’s class and `static === false` — still type catalog data, not live holes ([system route in `controller.js`](https://anoik.is/static/controller.js)).

## How the app obtains that data (API vs dump)

### System profiles → static dump only

[`controller.js`](https://anoik.is/static/controller.js) loads System profiles exclusively through its `Static` module:

1. Prefer `localStorage` cache keyed off the dump’s `version`.
2. Else `GET /static/static.json?version=<static_expected_version>` (expected version **11** in the observed build).
3. Expose lookups (`systems.byname`, `effects.lookup`, wormhole indexes, search).

There is **no** `/api/systems` (or similar) endpoint for profiles: `GET /api/systems` returns **404** (observed 2026-10-30).

### Live `/api/*` endpoints are unrelated to System profiles

The same client only calls Anoik `/api` for **SKIN claimables** and EVE SSO session plumbing, e.g. `/api/claimables/available`, `/api/claimables/claimed`, `/api/auth/eve`, `/api/logout`. Kill activity is fetched from **zKillboard**, not Anoik (`https://zkillboard.com/api/kills/w-space/solarSystemID/...`). A comment in the client notes a **todo** to show Thera adjacency via [eve-scout.com](https://www.eve-scout.com/) — it is not implemented as Anoik System-profile data.

**Implication for WH Tracker:** treat Anoik as a **vendored/cached static enrichment source**. Do not design around a System-profile REST API that does not exist.

## Anoik is not live connections

Keep this boundary sharp for the product+tech spec and glossary:

| Source | Role |
| --- | --- |
| Anoik System profiles | Static J-space reference (class, static **types**, effects, names, celestials) |
| Member **Intel** | Only live edge source for scanned wormholes / Chain |
| Eve-scout / public Thera books | Explicitly out of scope for this map’s destination (do not re-host) |

Anoik’s own UI reinforces the split: system pages show statics + wanderer **types**, SKIN claimables, and optional zKill activity — not a live mapper book.

## License and attribution

### What Anoik publishes

The About page states only the CCP limited-permission notice (verbatim from the client):

> All data and material from or related to EVE Online is used with limited permission of CCP Games hf and is © CCP hf. All rights reserved. "EVE", "EVE Online", "CCP", and all related logos and images are trademarks or registered trademarks of CCP hf. No official affiliation or endorsement by CCP Games hf is stated or implied.

Sources: [Anoik About](https://anoik.is/about) (rendered from [`controller.js`](https://anoik.is/static/controller.js)); author credit via [was.tl/projects](https://was.tl/projects/) and About links to [was.tl](https://was.tl/), Bluesky, Mastodon, [contact](https://was.tl/contact/).

**No** separate open-source license, terms of use, or “you may redistribute `static.json`” grant was found on Anoik.is, was.tl project listing, or a public Anoik.is source repository (site source is not published under Eric Wastl’s GitHub [`topaz`](https://github.com/topaz) as an Anoik repo; unrelated “Anoikis” mappers exist elsewhere and are not this site).

So for licensing, split two layers:

1. **Underlying EVE Game Data** — © CCP; Anoik claims limited permission. Third-party apps that use CCP tools / Game Data are expected to operate under CCP’s [Developer License Agreement](https://developers.eveonline.com/docs/resources/license/) ([agreement text](https://developers.eveonline.com/license-agreement)): Purpose is enhancing players’ enjoyment of EVE on a **non-commercial / non-profit** basis (as defined there). An alliance-private intel board fits that Purpose better than a public paid SaaS; still require CCP attribution / Marks rules wherever Game Data appears.
2. **Anoik’s compiled dump** (especially curated `statics` / packaging) — publicly downloadable for the website, but **not** explicitly licensed for republication. Caching inside an alliance-private Dokploy instance for enrichment is the practical path other community tools already take informally; it is **not** a published grant. Safer practice: keep the cache private (aligns with this map’s “no public re-host of live books” out-of-scope stance), attribute **Anoik.is / Eric Wastl**, retain the CCP notice, and optionally ask Eric via [contact](https://was.tl/contact/) if the operator wants an explicit OK.

### Recommended attribution block (app About / docs)

Use wording at least as strong as Anoik’s CCP paragraph, plus a short credit such as: System profile reference data derived from [Anoik.is](https://anoik.is/) (Eric Wastl).

## Recommendation for WH Tracker

1. **Ingest path:** Periodically fetch and vendor [`https://anoik.is/static/static.json`](https://anoik.is/static/static.json) (pin/check `version`; observed schema version **11**). Prefer build-time or operator refresh over hitting Anoik on every page view.
2. **Map into glossary “System profile”:** `wormholeClass`, `statics` → type metadata via `wormholes`, `effectName` → scaled bonuses via `effects` + class `effectPower`, names/IDs/region.
3. **Do not** treat Anoik APIs, zKill, or eve-scout todos as System-profile or live Chain sources.
4. **Compliance:** alliance-private cache + CCP notice + Anoik credit; no public redistribution of the dump; escalate to author contact if legal comfort requires an explicit license.

## Sources

- [Anoik.is](https://anoik.is/)
- [Anoik About](https://anoik.is/about) (content from [controller.js](https://anoik.is/static/controller.js))
- [Anoik static dump](https://anoik.is/static/static.json)
- [was.tl projects — Anoik.is blurb](https://was.tl/projects/)
- [was.tl contact](https://was.tl/contact/)
- [CCP Developer License docs](https://developers.eveonline.com/docs/resources/license/)
- [CCP Developer License Agreement](https://developers.eveonline.com/license-agreement)
- WH Tracker `GLOSSARY.md` term **System profile**
