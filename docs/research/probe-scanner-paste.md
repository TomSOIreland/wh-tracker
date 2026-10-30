# Probe-scanner paste formats and parsers

Research for [Research: Probe-scanner paste formats and parsers](https://github.com/TomSOIreland/wh-tracker/issues/4) (map: [WH Tracker: way to the product+tech spec](https://github.com/TomSOIreland/wh-tracker/issues/1)).

## Verdict

EVE explorers’ primary Intel paste is the **probe scanner clipboard**: select-all + copy yields **tab-separated rows with six fields**. Open WH mappers (Pathfinder, Wanderer, Tripwire, WormholeSystems, and several smaller parsers) all target that shape. **v1 should require English six-column Cosmic Signature rows keyed by `ABC-123`, with reconcile (add / update / missing) semantics.** Localization, Cosmic Anomalies, lazy-delete, D-Scan, and non-signature scan groups are nice-to-have.

## In-game clipboard format

### How explorers produce it

Community mapper docs describe the same workflow: open the Probe Scanner, `Ctrl+A`, `Ctrl+C`, paste into the signatures panel (`Ctrl+V`).

- [WormholeSystems — Pasting scan results](https://wormhole.systems/documentation/signatures/pasting-scan-results)
- [EVE University — EVE UNI Mapper (Wanderer)](https://wiki.eveuniversity.org/EVE_UNI_Mapper)

### Column layout (authoritative for parsers)

Every serious open parser treats a row as **exactly six tab-separated fields**, in this order:

| # | Field | Example (scanned WH) | Example (bare sig) |
|---|---|---|---|
| 1 | Signature ID | `QLM-802` | `ASD-123` |
| 2 | Scan group | `Cosmic Signature` | `Cosmic Signature` |
| 3 | Group | `Wormhole` | *(empty)* |
| 4 | Type / name | `Unstable Wormhole` | *(empty)* |
| 5 | Signal strength | `100.0%` | `0.0%` |
| 6 | Distance | `3.72 AU` | `21.66 AU` |

Documented with example rows in [mogglemoss/vagari `vagari/parsers/scanner.py`](https://raw.githubusercontent.com/mogglemoss/vagari/main/vagari/parsers/scanner.py) and [zanreal-labs/darkbeacon `signature-paste.ts`](https://raw.githubusercontent.com/zanreal-labs/darkbeacon/main/packages/modules/mapping/src/domain/signature-paste.ts). Pathfinder’s parser requires `rowData.length === 6` after splitting on tab (or four spaces) and validates field 2 against known scan-group labels ([`parseSignatureString` in `system_signature.js`](https://github.com/exodus4d/pathfinder/blob/master/js/app/ui/module/system_signature.js)). Fixture text in [th-lange/eve-online-tooling](https://github.com/th-lange/eve-online-tooling/blob/main/src-tauri/src/modules/wormholes/commands.rs) matches the same six columns:

```text
ABC-123<TAB>Cosmic Signature<TAB>Wormhole<TAB>Unstable Wormhole<TAB>100.0%<TAB>1.2 AU
```

**Partial scan behaviour:** Group and Type stay empty until the signature is probed far enough; ID + scan group + signal + distance still paste. Vagari and Darkbeacon both state this explicitly.

**Wormhole type codes (e.g. `K162`, `H296`) do not come from a normal probe-scanner paste.** The Type column for a fully scanned hole is the generic label `Unstable Wormhole` (localized). Pathfinder skips type extraction when group is wormhole; WormholeSystems documents that wormhole catalog types never come from a paste ([`paste.rs`](https://raw.githubusercontent.com/wormholesystems/wormholesystems/main/src/maps/signatures/paste.rs)).

### Signature ID convention

Canonical display form: **three letters, hyphen, three digits** — `ABC-123`.

- Pathfinder validates `^[a-zA-Z]{3}-\d{3}$` ([commit closing #876](https://github.com/mrurb/pathfinder/commit/9ecbe33f5ea07a3b19ea569a55a54f5f415e5b06); current [`system_signature.js`](https://github.com/exodus4d/pathfinder/blob/master/js/app/ui/module/system_signature.js)).
- Tripwire stores lowercase **without** the dash (`abc123`) and formats for display ([`eve-wormhole-primer.md`](https://raw.githubusercontent.com/apettey/tripwire/main/docs/eve-wormhole-primer.md)).
- Server-emulator comments also describe the scanner ID as `xxx-nnn` ([evemuproject/evemu_server](https://github.com/evemuproject/evemu_server) `POD_containers.h` / scan code).

### English group vocabulary (column 3)

Shared across Pathfinder, Wanderer, Tripwire, Darkbeacon, Vagari:

| Paste label | Typical Intel category |
|---|---|
| `Wormhole` | Wormhole |
| `Combat Site` | Combat |
| `Relic Site` | Relic |
| `Data Site` | Data |
| `Gas Site` | Gas |
| `Ore Site` | Ore |

Pathfinder also had a `ghost` / Ghost Site entry historically ([DE/FR localization commit](https://github.com/Timestorm/pathfinder/commit/8e84da07573bd548b5ba63a6298ac40f69384685)).

### Scan groups (column 2) beyond Cosmic Signature

Probe scanner can list more than exploration signatures (ships, structures, deployables, drones, starbases, celestials). Darkbeacon keeps those as `other` kind ([`signature-paste.ts`](https://raw.githubusercontent.com/zanreal-labs/darkbeacon/main/packages/modules/mapping/src/domain/signature-paste.ts)). Wanderer exposes filter flags for Cosmic Anomaly / Ship / Structure / etc. ([`constants.ts`](https://raw.githubusercontent.com/wanderer-industries/wanderer/main/assets/js/hooks/Mapper/components/mapInterface/widgets/SystemSignatures/constants.ts)).

English Cosmic Anomaly label: `Cosmic Anomaly` (Pathfinder / Tripwire / Darkbeacon).

### Localization

Paste labels are **client-language dependent**. Mature tools ship synonym tables:

| Client | Cosmic Signature | Wormhole (group) | Sources |
|---|---|---|---|
| English | `Cosmic Signature` | `Wormhole` | Pathfinder, Tripwire, Wanderer, Darkbeacon |
| German | `Kosmische Signatur` (+ optional `(typ)`) | `Wurmloch` | Pathfinder, Tripwire, Darkbeacon (strips `(typ)`/`(gruppe)`) |
| French | `Signature cosmique` (+ optional `(type)`) | `Trou de ver` | Pathfinder, Tripwire, Darkbeacon |
| Russian | `Скрытый сигнал` | `Червоточина` | Pathfinder, Tripwire, Darkbeacon, Wanderer |
| Japanese | `宇宙のシグネチャ` | `ワームホール` | Pathfinder, Tripwire |
| Chinese | `空间信号` | *(varies by tool)* | Pathfinder |
| Korean | `코즈믹 시그니처` | `웜홀` | Tripwire |

WormholeSystems claims EN/DE/FR/ES/RU/JA/KO/ZH matching and mixed-language pastes ([docs](https://wormhole.systems/documentation/signatures/pasting-scan-results)).

### Related but different: D-Scan paste

Directional scan is a **separate** three-field TSV (`name`, `type`, `distance` with `km`/`m`/`AU` or `-`) — see [skyride/capritools `parseDscan`](https://github.com/skyride/capritools/blob/master/dscan/functions.php). Wanderer’s UNI guide uses D-Scan paste for **structures**, not signatures ([EVE UNI Mapper](https://wiki.eveuniversity.org/EVE_UNI_Mapper)). Do not conflate with probe-scanner paste.

Inventory / multibuy parsers ([isk-insider/eve-paste](https://github.com/isk-insider/eve-paste), evepraisal/evepaste) are unrelated to signature Intel.

## Open-source parsers and conventions

| Project | What to copy | Notes |
|---|---|---|
| [exodus4d/pathfinder](https://github.com/exodus4d/pathfinder) `parseSignatureString` | Strict 6 columns; localize scan group + group; map site type from column 4; **ignore WH type from paste**; preview add/change/delete ([issue #839](https://github.com/exodus4d/pathfinder/issues/839)) |
| [wanderer-industries/wanderer](https://github.com/wanderer-industries/wanderer) `parseSignatures` + paste hook | Same 6-field rows; EN/RU/FR/DE maps; lazy delete; unsupported-language heuristic when ID lines >> parsed rows |
| [apettey/tripwire](https://github.com/apettey/tripwire) `paste.js` | Tokenize fields; multi-language scan groups/types; skip signal % and distance; optional “full paste” delete of missing IDs |
| [wormholesystems/wormholesystems](https://github.com/wormholesystems/wormholesystems) `paste.rs` | Server upsert merge rules; client parses TSV; WH types never from paste; `signatures_pasted_at` only advances on paste |
| [zanreal-labs/darkbeacon](https://github.com/zanreal-labs/darkbeacon) `signature-paste.ts` | Clean reference implementation: languages table, qualifier stripping, ID regex, skip list |
| [mogglemoss/vagari](https://github.com/mogglemoss/vagari) `scanner.py` | Compact Python: require 6 tabs, keep signal + distance |
| [th-lange/eve-online-tooling](https://github.com/th-lange/eve-online-tooling) | Dedupe by ID, SIG_CAP, destructive-paste confirmation |
| [calli-eve/ava-mapper](https://github.com/calli-eve/ava-mapper) | Functional but README format is incomplete; parser remaps Cosmic Signature rows — prefer Pathfinder/Darkbeacon/Vagari as references |

### Shared product conventions (not just parse)

1. **Reconcile, don’t blind-replace** — preview new / updated / missing ([WormholeSystems](https://wormhole.systems/documentation/signatures/pasting-scan-results), Pathfinder signature reader, Wanderer).
2. **ID-only rows are valid Intel** — type can be filled later by a fuller paste ([WormholeSystems](https://wormhole.systems/documentation/signatures/pasting-scan-results)).
3. **Lazy delete is opt-in** — missing IDs stay until the Member confirms (Wanderer, Tripwire “full paste”, WormholeSystems confirm-to-delete).
4. **Deduplicate by signature ID** within one paste (th-lange, Wanderer, Darkbeacon).
5. **Generic WH name is not a destination type** — treat `Unstable Wormhole` / localized equivalents as “no extra type info” (Darkbeacon `genericNames`).

## Recommendation for WH Tracker v1

### Minimal parse surface (require in the product+tech spec)

1. **Accept** probe-scanner clipboard: newline-separated rows, **tab**-separated fields (also tolerate `\r\n` and Pathfinder’s `\s{4}` fallback if cheap).
2. **Require** field 1 matching `^[A-Za-z]{3}-\d{3}$`; normalize display to `ABC-123` uppercase.
3. **Require** English field 2 ∈ {`Cosmic Signature`} for Intel rows (optionally accept `Cosmic Anomaly` but store as anomaly / out-of-focus unless product expands).
4. **Parse** field 3 when present into Signature/Wormhole categories: Wormhole, Combat/Relic/Data/Gas/Ore Site; empty → unknown / unscanned.
5. **Parse** field 4 as free-text site name when non-empty and not a generic WH label (`Unstable Wormhole`); do **not** invent WH type codes from paste.
6. **Optionally store** signal % and distance (fields 5–6) if cheap — useful for “how far scanned” UX, not required for Chain.
7. **UX**: paste accelerator upserts into the current system’s Intel list with **preview of added / updated / missing**; missing deletion is confirm/lazy, not default wipe.
8. **Skip** unrecognised lines; report count of skipped vs parsed.

This matches how explorers already feed Pathfinder/Wanderer/Tripwire and is enough for Member-contributed scanned-WH Intel without D-Scan or localization debt.

### Nice-to-have (defer unless cheap)

- Localized labels (at least DE/FR/RU; then JA/KO/ZH/ES) using Tripwire/Pathfinder/Darkbeacon tables; strip French/German `(type)`/`(typ)` qualifiers.
- Cosmic Anomalies and non-signature scan groups (ships/structures) as filterable noise.
- D-Scan structure import (separate parser).
- Destructive-paste guard when paste would drop >50% of existing IDs (th-lange pattern).
- Multi-language mixed pastes (WormholeSystems).
- Factional Warfare / event free-text site names (WormholeSystems edge cases).
- Ghost sites.

### Explicit non-goals for the paste accelerator

- Inferring wormhole **type codes**, mass, or EOL from scanner paste alone.
- Treating D-Scan or inventory paste as signature Intel.
- Relying on ava-mapper’s abbreviated README format (`ID Group Type` without Cosmic Signature) as the contract — real client output is the six-column TSV.

## Sources

- [WormholeSystems — Pasting scan results](https://wormhole.systems/documentation/signatures/pasting-scan-results)
- [EVE University — EVE UNI Mapper](https://wiki.eveuniversity.org/EVE_UNI_Mapper)
- [exodus4d/pathfinder `system_signature.js` (parseSignatureString, validSignatureNames)](https://github.com/exodus4d/pathfinder/blob/master/js/app/ui/module/system_signature.js)
- [Pathfinder issue #839 — signature reader preview](https://github.com/exodus4d/pathfinder/issues/839)
- [Pathfinder ID validation commit](https://github.com/mrurb/pathfinder/commit/9ecbe33f5ea07a3b19ea569a55a54f5f415e5b06)
- [Pathfinder DE/FR localization commit](https://github.com/Timestorm/pathfinder/commit/8e84da07573bd548b5ba63a6298ac40f69384685)
- [wanderer-industries/wanderer `parseSignatures.ts`](https://raw.githubusercontent.com/wanderer-industries/wanderer/main/assets/js/hooks/Mapper/helpers/parseSignatures.ts)
- [wanderer-industries/wanderer SystemSignatures `constants.ts`](https://raw.githubusercontent.com/wanderer-industries/wanderer/main/assets/js/hooks/Mapper/components/mapInterface/widgets/SystemSignatures/constants.ts)
- [wanderer-industries/wanderer `useSystemSignaturesData.ts` (paste / lazy delete)](https://raw.githubusercontent.com/wanderer-industries/wanderer/main/assets/js/hooks/Mapper/components/mapInterface/widgets/SystemSignatures/hooks/useSystemSignaturesData.ts)
- [apettey/tripwire `app/js/tripwire/paste.js`](https://raw.githubusercontent.com/apettey/tripwire/main/app/js/tripwire/paste.js)
- [apettey/tripwire `docs/eve-wormhole-primer.md`](https://raw.githubusercontent.com/apettey/tripwire/main/docs/eve-wormhole-primer.md)
- [wormholesystems/wormholesystems `src/maps/signatures/paste.rs`](https://raw.githubusercontent.com/wormholesystems/wormholesystems/main/src/maps/signatures/paste.rs)
- [zanreal-labs/darkbeacon `signature-paste.ts`](https://raw.githubusercontent.com/zanreal-labs/darkbeacon/main/packages/modules/mapping/src/domain/signature-paste.ts)
- [mogglemoss/vagari `vagari/parsers/scanner.py`](https://raw.githubusercontent.com/mogglemoss/vagari/main/vagari/parsers/scanner.py)
- [th-lange/eve-online-tooling wormhole `parse_signatures`](https://github.com/th-lange/eve-online-tooling/blob/main/src-tauri/src/modules/wormholes/commands.rs)
- [calli-eve/ava-mapper README + `signatureParser.js`](https://github.com/calli-eve/ava-mapper)
- [skyride/capritools D-Scan parser](https://github.com/skyride/capritools/blob/master/dscan/functions.php)
- [isk-insider/eve-paste](https://github.com/isk-insider/eve-paste) (inventory paste — out of scope)
- Odyssey-era scanner column discussion: [eve-search thread](https://eve-search.com/thread/233600-1/author/Kai%20Pirinha) (UI columns: ID, Scan Group, Group, Type, Signal strength, Distance)
