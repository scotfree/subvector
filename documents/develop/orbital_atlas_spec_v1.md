# Space Combat Atlas — Implementable Spec (v1, MVP)

A turn-based, hidden-information space combat game built **additively** on the Stewardship Atlas
platform. Turns are versions; spacecraft are features; movement, detection, and combat are eddies;
player views are outlets. The guiding constraint: **reuse Atlas primitives wherever possible, add
only in sanctioned extension points, and never modify shared core code.** Anything that can't be
done by reuse or by adding an eddy/outlet/config is pushed to *Later Work*.

---

## 1. Concept in one paragraph

Each spacecraft is a point feature with a position (geometry) and a velocity (attributes). A player's
turn input is a *desired acceleration* intent, stored as a property update on their ship. Generating a
turn = generating a new version: a pipeline of eddies applies physics (move), then detection (who sees
whom), then combat (who hits whom). Each player polls the API and loads a per-player outlet showing
only the objective truth they're entitled to see (their own ships, plus enemies they've detected). No
real-time connection; the client polls a version counter and refreshes when it advances.

---

## 2. MVP scope

### In scope (build now)
- Game board + parameters as an `atlas_config`.
- Ships / weapons / sensors as layer features with a defined schema (made-up balance numbers).
- Turn = version. Player move = property-update delta (`desired_acceleration`).
- Three eddies: **physics**, **detection**, **combat**.
- Single detection layer with a `detected_by` attribute.
- Per-player filtered outlet (objective truth masked to detected + owned features).
- Static HTML client: blank basemap, click-to-set acceleration, poll-and-refresh.
- **Trust auth**: player types a username in the client; it's attached to their inputs. No real identity.
- Manual "advance turn" trigger (for testing) **and** auto-advance when all inputs are present.

### Explicitly deferred (see §10)
- Google / SSO auth.
- Ship/component **design rules and balancing** (we hardcode starting numbers).
- Tile / gravity-well basemap rendering (we use blank).
- Ship **orientation/heading** (MVP polygons translate but do not rotate).
- Wrecks, friendly fire, distributed/clever targeting beyond `random`, scheduled play-by-day mode.

---

## 3. Coordinate system

- CRS: declare an arbitrary projected CRS — **`EPSG:3857`** — and treat its units as abstract
  "game-meters." No Earth interpretation; we just borrow the flat-planar machinery.
- Game bounds live in `atlas_config` (e.g. a square `0 … 1_000_000` on each axis).
- **Verify before building:** confirm core treats CRS as per-dataswale metadata (standard pattern).
  A quick grep for any hardcoded Earth/spheroid assumption is the only real risk here.

---

## 4. Atlas config — the "game board"

A single `atlas_config` object holds all game-level state. Reuses the existing config mechanism; no new
code.

```yaml
game_id: "skirmish-001"
turn: 7                      # current committed version / turn number
crs: "EPSG:3857"
bounds: [0, 0, 1000000, 1000000]
dt: 1                        # time step per turn (game-seconds)
players:
  - { id: "alpha",  display_name: "Alpha"  }
  - { id: "bravo",  display_name: "Bravo"  }
turn_status:                 # written by the input endpoint, read by clients
  submitted: ["alpha"]       # players whose inputs are in for the pending turn
  awaiting:  ["bravo"]
gravity: null                # placeholder; unused in MVP (see Later Work)
```

---

## 5. Layer schemas

All schemas expressed as optional JSON/YAML schema fields on the layer definition (consistent with the
Pydantic-generation approach already on the roadmap). Unknown fields remain permitted.

### 5.1 `ships` layer (one feature per spacecraft)

| Field | Type | Notes |
|---|---|---|
| `geometry` | Point | current position, in game CRS |
| `ship_id` | string | stable id, join key |
| `owner` | string | player id |
| `vx`, `vy` | float | current velocity components |
| `desired_ax`, `desired_ay` | float \| null | **turn input**; null = no order this turn |
| `hp` | float | hit points; ship removed next version if ≤ 0 |
| `performance` | Polygon | **ship-local** envelope of allowable acceleration (origin = ship) |
| `destroyed` | bool | set by combat eddy in the turn it dies (then dropped) |

> `performance` is defined in ship-local coordinates centered on (0,0). A desired acceleration vector
> `(desired_ax, desired_ay)` is legal iff that point falls inside `performance`.

### 5.2 `weapons` layer (one feature per weapon system; FK to ship)

| Field | Type | Notes |
|---|---|---|
| `weapon_id` | string | |
| `ship_id` | string | owning ship |
| `envelope` | Polygon | **ship-local** firing envelope (translated to ship position at resolve time) |
| `damage` | float | total damage budget for the shot |
| `targeting` | enum | `random` (MVP). `distributed` optional. Others = Later Work |

### 5.3 `sensors` layer (one feature per sensor; FK to ship)

| Field | Type | Notes |
|---|---|---|
| `sensor_id` | string | |
| `ship_id` | string | owning ship |
| `envelope` | Polygon | **ship-local** detection envelope |

> Sensor envelopes are typically larger than weapon envelopes — you can see farther than you can shoot.

### 5.4 `detections` layer (generated each turn; single layer)

| Field | Type | Notes |
|---|---|---|
| `geometry` | Point | position of the detected ship (objective) |
| `target_ship_id` | string | the ship that was seen |
| `detected_by` | string[] | list of player ids who detected it this turn |

Kept in version history for replay/analysis. Single layer + `detected_by` array (chosen over
per-player layers: one detection pass, sliceable downstream).

---

## 6. The delta model (player input)

A player's move is **not** a new delta type. It's the existing **update-via-join** delta:

- Join key: `ship_id` (index join, not geometry scan).
- Operation: set `desired_ax`, `desired_ay` on the matching ship feature.
- The delta is **intent**, not outcome. The physics eddy decides the actual velocity change, so a
  player can't write an illegal velocity directly.

Deltas live inside the dataswale as the layer's ordered delta list (per our "self-contained dataswale"
conclusion). When a version is generated, the pending deltas become that version's historical inputs;
a fresh pending set opens for the next turn.

---

## 7. Eddy contracts

The eddies are the only genuinely new *logic*, and eddies are the designed additive path — isolated
modules that can't endanger existing ones. They run in order; each consumes the prior output.

### 7.1 Physics eddy
**In:** ships layer with pending `desired_ax/ay`.
**Logic (per ship):**
1. If `(desired_ax, desired_ay)` is null → acceleration = 0.
2. Else validate against `performance` (point-in-polygon, ship-local). If outside, **clamp** the vector
   to the polygon boundary along its direction. (v0.0 simplification: reject → 0. Note which you ship.)
3. Semi-implicit Euler, `dt` from config:
   `vx' = vx + ax;  vy' = vy + ay;  x' = x + vx'·dt;  y' = y + vy'·dt`
4. Write new position + velocity; clear `desired_ax/ay` to null.

**Out:** ships layer at new positions, no pending orders.

### 7.2 Detection eddy
**In:** post-physics ships layer + sensors layer.
**Logic:** for each sensor, translate its `envelope` to its ship's position; spatial-join (DuckDB)
against **enemy** ship points. Every enemy point inside → that ship is detected by the sensor's owner.
Aggregate to one row per detected ship with a `detected_by` array.
**Out:** `detections` layer.
**MVP simplification:** no orientation — envelopes translate only, no rotation.

### 7.3 Combat eddy
**In:** post-physics ships layer + weapons layer + `detections` layer.
**Logic (per ship, per weapon):**
1. Translate `envelope` to ship position.
2. Candidate targets = enemy ships that are **(a)** detected by this weapon's owner **and**
   **(b)** inside the envelope (spatial join).
3. Resolve by `targeting`:
   - `random` → pick one candidate uniformly; apply full `damage` to its `hp`.
   - `distributed` (optional) → split `damage` evenly across all candidates.
4. Each weapon fires **once per turn**.
5. After all weapons resolve, any ship with `hp ≤ 0` is marked `destroyed` and **omitted** from the
   next version's ships layer.

**Out:** ships layer with updated `hp`, destroyed ships removed → committed as the new version.

> Note: detection gates combat — you can't fire on what your fleet didn't detect.

---

## 8. Turn pipeline (one version generation)

```
gather pending deltas (all players)
   → physics eddy      → ships moved
   → detection eddy    → detections layer
   → combat eddy       → hp updated, dead ships removed
   → commit new version (turn += 1)
   → regenerate per-player outlets
```

Trigger: auto-fire when `turn_status.awaiting` is empty, **or** a manual `advance` call (testing).

---

## 9. API surface & client

### 9.1 Endpoints
Three of these are essentially existing read paths; only the input write and the trigger are new, and
both can be thin/general.

| Method | Endpoint | Purpose | New or reuse |
|---|---|---|---|
| GET | `/game/{game_id}/status` | current `turn` + `turn_status` | reuse (config read) |
| GET | `/game/{game_id}/view?player={id}` | that player's outlet artifact | reuse (outlet fetch) |
| POST | `/game/{game_id}/input` | submit `{username, ship_id, ax, ay}` | new, thin (delta ingest) |
| POST | `/game/{game_id}/advance` | manual turn trigger (testing/admin) | new, thin |

### 9.2 Per-player outlet (the "view")
A **filtered outlet**: serialize the objective state through a predicate so each player receives only:
- all of their **own** ships (full truth), plus
- enemy ships present in `detections` with their `player` in `detected_by`,
- plus game bounds + their ships' performance/weapon/sensor envelopes for client-side display.

This filtered/parameterized outlet is general infrastructure that overlaps the access-level work already
on the roadmap — build it generically, not game-specifically.

### 9.3 Client (static HTML + JS)
- **Auth (MVP):** a username text field. Value is sent with every input. No verification.
- **Basemap:** blank — render features on a plain SVG/canvas (or Leaflet with a blank tile layer) in the
  game CRS bounds.
- **Loop:** poll `/status` every N seconds → if `turn` increased, fetch `/view` and re-render.
- **Input:** for each owned ship, click on the map to set an acceleration vector (origin = ship). Optional
  client-side check against `performance` for UX; the physics eddy is authoritative. Submit via `/input`.
- **Status display:** show "waiting on X of Y players" from `turn_status`.
- Closing/reopening the browser is fine — state is server-side; the client just re-polls.

---

## 10. Reuse-vs-new ledger

**Tier 1 — pure reuse (data/config only, zero shared-code change)**
- Game board → `atlas_config`. Turns → versions. Ships/weapons/sensors → layer features.
- Player move → existing update-via-join delta. Status/view reads → existing version + outlet reads.

**Tier 2 — general additions (new, reusable, repays the main Atlas)**
- Per-user identity (MVP = trust/username; later = OAuth module Fire Atlas inherits).
- Filtered/parameterized outlet (overlaps roadmap access-level/schema work).
- Version-generation trigger condition ("all inputs in, or timeout") → generalizes to scheduled refreshes.

**Tier 3 — genuinely new, but in sanctioned extension points**
- Physics, detection, combat eddies. Isolated modules; adding them can't break existing eddies.

**Do-not-edit guardrails**
- CRS: confirm per-dataswale metadata; pick a projected CRS as config. (grep for Earth assumptions.)
- Basemap: don't modify the web-map outlet — add a game outlet **variant** that swaps the base layer.
- Auth scope: middleware on **new game routes only**; never make auth mandatory on existing endpoints.

---

## 11. Open questions & Later Work

Detailed enough to scope and drop in when ready.

### 11.1 Google / SSO auth (replaces trust-auth)
- **Why deferred:** trust-auth lets us test the whole loop today; identity is orthogonal to game logic.
- **Plan:** Google OAuth 2.0 to start (fewest moving parts), or Amazon Cognito if we want hosted UI +
  multiple IdPs cheaply.
  - Add as middleware on `/game/*` routes only. Map verified subject → `player.id` via a small mapping
    table. Replace the client username field with a sign-in button + token; attach bearer token to
    `/input`. Server rejects inputs whose token-derived player ≠ `ship.owner`.
  - Build it as a **general** identity module so Fire Atlas inherits per-user accounts later — this is
    the per-user notion the Atlas has needed for years.

### 11.2 Ship & component design rules / balancing
- MVP hardcodes envelopes, damage, hp. Later: a design/point-buy system so a giant full-map weapon
  "costs" against acceleration etc., plus a pre-game design phase. This is content, not architecture —
  the loop is unchanged.

### 11.3 Orientation / heading
- MVP envelopes translate only. Later: give ships a heading (e.g. derived from velocity vector or a
  separate facing order) and **rotate** performance/weapon/sensor envelopes by it before joins. This is
  the biggest single jump in tactical depth and is purely an eddy change.

### 11.4 Targeting types beyond `random`
- Add `distributed` (split damage), then `closest` / `strongest` / `weakest`. Pure combat-eddy `switch`
  on the `targeting` field — trivial once wanted.

### 11.5 Destruction representation
- MVP: ship omitted from next version. Later: keep as a `wreck` feature for N turns (debris, salvage,
  collision hazard), or a `status` attribute instead of deletion.

### 11.6 Missiles as autonomous units
- Model missiles as high-performance ships with tight, high-damage weapon envelopes and short lifespans
  — reuses the entire ship pipeline. Needs: spawn-on-fire mechanic, lifespan/fuel, and (ideally) §11.3
  orientation. Slots in cleanly later.

### 11.7 Friendly fire & factions
- MVP: enemy-only joins (no friendly fire). Later: faction/alliance attribute and configurable FF.

### 11.8 Delta storage duplication
- We currently let each self-contained version carry its inputs, duplicating the prior pending set.
  Acceptable for MVP. Later: an outlet that deduplicates/compresses for lightweight export (decision
  punted downstream, as agreed).

### 11.9 Gravity / terrain
- `gravity` is a config placeholder. Later: terrain-as-gravity-well that perturbs velocity each turn in
  the physics eddy (an added force term). Decide whether it's a field sampled from a raster or computed
  from planet/moon point-masses.

### 11.10 Scheduled / async play
- Polling already supports drop-in/drop-out. Later: a timeout-based auto-advance for play-by-the-day —
  reuses the §Tier-2 trigger condition with a deadline.

### 11.11 Verifications to run against the repo before coding
1. CRS is per-dataswale metadata, no hardcoded Earth assumption.
2. A delta-ingest path exists (or is trivially addable) for update-via-join on an index key.
3. Outlets are parameterizable today (predicate filter) — or scope that as the one Tier-2 outlet add.

---

## 12. Build order (fastest path to a playable loop)

1. Author one `atlas_config` + a tiny `ships`/`weapons`/`sensors` dataset with made-up numbers (2 ships/side).
2. Physics eddy → confirm versions advance and ships move from `desired_ax/ay`.
3. Trust-auth input endpoint + blank-basemap client that can set a vector and submit.
4. Detection eddy + filtered per-player outlet → confirm hidden information works.
5. Combat eddy (`random` only) → confirm hp/kills.
6. Auto-advance trigger. Play a full game end-to-end.
7. Then pull from Later Work as desired (orientation and distributed targeting give the most depth per effort).
