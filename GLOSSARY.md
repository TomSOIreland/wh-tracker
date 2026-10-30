# WH Tracker

Alliance-private EVE Online exploration intel: scanned wormholes, explorer presence, and wormhole chains.

## Language

**Member**:
An EVE character signed in via ESI who belongs to an allowed alliance.
_Avoid_: User, account, player (when meaning the authenticated character)

**Presence**:
A Member's current solar-system location as known to the app.
_Avoid_: GPS, tracking ping, location ping

**Intel**:
Alliance-private information about a scanned signature or wormhole, contributed by Members.
_Avoid_: Report, pin, sighting (unless a sharper subtype is introduced later)

**Signature**:
A cosmic signature in a solar system that Members may scan and turn into Intel.
_Avoid_: Anomaly (combat/ore sites are out of this product's focus unless later expanded)

**Wormhole**:
A Signature (or known connection) that links two solar systems; the primary Intel the product exists to track.
_Avoid_: WH (in prose; fine in UI shorthand), gate

**Chain**:
A graph of Wormhole connections between solar systems, built from Member-contributed Intel.
_Avoid_: Map (ambiguous with the k-space geography map), path

**Claim**:
A temporary reservation of a Signature by a Member so others don't double-scan it.
_Avoid_: Lock, assignment, ticket

**System profile**:
Static reference facts about a solar system (especially J-space class, statics, effects), sourced from Anoik — not live Intel.
_Avoid_: Anoik row, encyclopedia entry
