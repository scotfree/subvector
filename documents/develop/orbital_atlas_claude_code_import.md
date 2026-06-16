# Orbital Atlas — Claude Code Import Doc (MVP scaffold)

**Goal for this build:** scaffold a *separate* repo that implements the turn-based space combat game
described in `orbital_atlas_spec_v1.md`, **importing the Stewardship Atlas as a package dependency**
rather than living in its repo. Build the game logic as **infra-agnostic plain functions** (GeoJSON in /
GeoJSON out) that run locally with zero cloud dependencies, then expose a thin, **isolated** serverless
deployment target that can be wired up later without ever gating the local build.

> Read `orbital_atlas_spec_v1.md` first — it is the source of truth for game rules. This doc is *how to
> structure and build it*. Where they disagree, the spec wins on rules, this doc wins on layout.

---

## 0. Three architectural commitments (do not violate)

1. **Atlas is a dependency, not a host.** The game imports Atlas primitives (dataswale / version /
   eddy / outlet / delta) through a thin **adapter layer** (`orbital_atlas.atlas.ports`). Game logic
   never imports Atlas internals directly. This both protects the Atlas repo and proves the
   platform-as-library claim.

2. **Eddies are infra-agnostic.** Physics / detection / combat are pure functions over GeoJSON. No I/O,
   no cloud, no framework inside them. They run identically on a laptop, a container, or a Lambda.

3. **Serverless is isolated and one-directional.** Nothing under `src/orbital_atlas/{models,geometry,
   eddies,pipeline,outlets}` may import from `serverless/` or from any cloud SDK. The dependency arrow
   points *into* serverless only. This guarantees a wrong-infra guess costs deploy glue, never game code.

---

## 1. Repo layout

```
orbital-atlas/
  pyproject.toml
  README.md
  src/orbital_atlas/
    __init__.py
    models.py              # Pydantic schemas: GameConfig, Ship, Weapon, Sensor, Detection, Delta
    geometry.py            # shapely helpers: point-in-poly, translate, clamp, spatial join
    eddies/
      physics.py           # apply_physics(ships, dt) -> ships
      detection.py         # compute_detections(ships, sensors) -> detections
      combat.py            # resolve_combat(ships, weapons, detections, rng) -> ships
    pipeline.py            # run_turn(...) orchestrates eddies; infra-agnostic
    outlets/
      player_view.py       # filtered per-player view (objective truth masked to detected+owned)
    atlas/                 # ADAPTER LAYER — the only thing that knows Atlas exists
      ports.py             # Protocols: VersionStore, DeltaIngest, OutletWriter, ConfigStore
      local.py             # in-memory / file-backed impl for local dev + tests (default)
      atlas_backed.py      # real impl importing the Atlas package (TODO: wire to real API)
    api/
      app.py               # FastAPI: /status /view /input /advance  (local server)
      auth.py              # trust-auth (username) now; OAuth stub for later
  serverless/              # ISOLATED. Nothing in src/ imports from here.
    handler.py             # lambda entrypoints wrapping pipeline + api
    infra.md               # IaC sketch (notes only for MVP)
  client/
    index.html             # blank-basemap canvas, poll + render + click-to-set-vector
    app.js
    style.css
  scripts/
    seed_game.py           # author atlas_config + tiny 2v2 dataset (MADE-UP balance numbers)
    play_local.py          # CLI: advance turns with no server, for fast iteration
  tests/
    test_physics.py
    test_detection.py
    test_combat.py
    test_pipeline.py
```

---

## 2. Dependencies (`pyproject.toml`)

- Runtime: `pydantic>=2`, `shapely>=2`, `fastapi`, `uvicorn`. Optional: `duckdb` (only if a join is
  large enough to want it — for a 2v2 MVP, shapely loops are fine; keep DuckDB behind the adapter).
- The Atlas package: declare it as a dependency, but **the default wiring uses `atlas/local.py`**, so
  the build and tests pass even before the Atlas package is importable. Wire `atlas_backed.py` once the
  real import surface is confirmed (see §8 verifications).
- Dev: `pytest`, `ruff`.

---

## 3. Data representation passed between eddies

The unit of exchange is a **GeoJSON FeatureCollection (FC)** per layer, represented as plain dicts (or
thin Pydantic models that round-trip to GeoJSON). Eddies take FCs and return FCs. No hidden state.

`models.py` defines validated schemas matching spec §5. Notable fields:

- **Ship:** `ship_id, owner, geometry(Point), vx, vy, desired_ax|None, desired_ay|None, hp,
  performance(Polygon, ship-local), destroyed(bool)`
- **Weapon:** `weapon_id, ship_id, envelope(Polygon, ship-local), damage, targeting('random'|'distributed')`
- **Sensor:** `sensor_id, ship_id, envelope(Polygon, ship-local)`
- **Detection:** `geometry(Point), target_ship_id, detected_by(list[str])`
- **GameConfig:** `game_id, turn, crs='EPSG:3857', bounds, dt, players[], turn_status, gravity=None`
- **Delta:** `{op:'update', key:'ship_id', match:<id>, set:{desired_ax, desired_ay}}` — the existing
  update-via-join shape, *not* a new delta type.

All polygons are **ship-local** (origin = ship). They are translated to world coordinates at resolve
time. **No rotation in MVP** (orientation is Later Work).

---

## 4. Geometry helpers (`geometry.py`)

Keep these tiny and pure; they are the math the eddies lean on.

```python
def translate_polygon(poly, dx, dy):        # ship-local -> world: shift by ship position
def point_in_polygon(pt, poly) -> bool:
def clamp_to_polygon(vx, vy, poly):         # if (vx,vy) outside poly, scale along its direction
                                            # to the boundary; else return unchanged
def enemies_in_world_polygon(world_poly, ships, owner) -> list[str]:
    # spatial join: ship points of OTHER owners inside world_poly -> their ship_ids
```

`clamp_to_polygon` implements the physics legality rule. **v0.0 fallback allowed:** if implementing the
boundary clamp is fiddly, ship "reject → (0,0)" first and leave a `# TODO: clamp` — note which you
shipped in the README. (Spec §7.1.)

---

## 5. Eddies (pure functions — the heart of the build)

### `eddies/physics.py`
```python
def apply_physics(ships: FC, dt: float) -> FC:
    # per ship:
    #   ax, ay = (desired_ax, desired_ay) or (0, 0)
    #   ax, ay = clamp_to_polygon(ax, ay, ship.performance)   # ship-local check
    #   vx += ax; vy += ay                                    # semi-implicit Euler
    #   x  += vx*dt; y += vy*dt
    #   clear desired_ax/desired_ay -> None
    # return updated ships FC
```

### `eddies/detection.py`
```python
def compute_detections(ships: FC, sensors: FC) -> FC:
    # for each sensor: world_env = translate_polygon(sensor.envelope, ship.x, ship.y)
    #   detected = enemies_in_world_polygon(world_env, ships, owner)
    # aggregate to ONE Detection per detected ship with detected_by = [players who saw it]
    # return detections FC   (single layer; spec §5.4)
```

### `eddies/combat.py`
```python
def resolve_combat(ships: FC, weapons: FC, detections: FC, rng) -> FC:
    # detected_by_owner: {player -> set(ship_ids they detected)}
    # for each weapon (once per turn):
    #   world_env = translate_polygon(weapon.envelope, ship.x, ship.y)
    #   candidates = enemies_in_world_polygon(world_env, ships, owner)
    #              & detected_by_owner[owner]            # detection-gated
    #   if candidates:
    #       if targeting == 'random':      apply full damage to rng.choice(candidates)
    #       elif targeting == 'distributed': split damage across candidates  # optional
    # after all weapons: hp<=0 -> destroyed=True
    # drop destroyed ships from returned FC (spec §7.3; wrecks = Later Work)
    # NOTE: pass rng in for determinism in tests (seedable)
```

---

## 6. Pipeline (`pipeline.py`) — infra-agnostic orchestration

```python
def run_turn(cfg, ships, weapons, sensors, deltas, rng) -> TurnResult:
    ships = apply_deltas(ships, deltas)        # update-via-join -> sets desired_ax/ay
    ships = apply_physics(ships, cfg.dt)
    detections = compute_detections(ships, sensors)
    ships = resolve_combat(ships, weapons, detections, rng)
    return TurnResult(turn=cfg.turn + 1, ships=ships, detections=detections)
```

`apply_deltas` is the update-via-join: join on `ship_id`, write `desired_ax/ay`. This is the only place
deltas touch state; everything downstream is pure eddies. `run_turn` does **no I/O** — persistence and
outlet generation are the adapter's job (§7).

---

## 7. Atlas adapter (`atlas/ports.py`, `local.py`, `atlas_backed.py`)

Define Protocols so game logic depends on *interfaces*, not on Atlas or on cloud:

```python
class ConfigStore(Protocol):   def get(game_id)->GameConfig; def put(cfg)->None
class VersionStore(Protocol):  def latest(game_id)->Version; def commit(game_id, turn_result)->Version
class DeltaIngest(Protocol):   def pending(game_id)->list[Delta]; def submit(game_id, delta)->None
class OutletWriter(Protocol):  def write_player_view(game_id, player, fc)->uri; def read(...)->fc
```

- `local.py`: in-memory + JSON-on-disk implementation. **This is the default for dev/tests/PoC.** It
  lets the entire game run on a laptop with no Atlas package and no cloud.
- `atlas_backed.py`: implements the same Protocols against the real Atlas primitives (dataswale store,
  version mechanism, delta list, outlet generator). Fill in once §8 verifications pass. Swapping from
  `local` to `atlas_backed` should be a one-line wiring change — no game-logic edits.

This adapter is what makes "70% reuse" concrete at the *abstraction* level while keeping local-first
development unblocked.

---

## 8. Per-player outlet (`outlets/player_view.py`)

```python
def player_view(player, ships, detections) -> FC:
    # include: all ships where owner == player (full truth)
    #        + ships present in detections with player in detected_by
    #        + (client also needs that player's own performance/weapon/sensor envelopes
    #           and game bounds — bundle or fetch separately)
```

This is the **filtered outlet** — general infrastructure that overlaps the roadmap access-level work.
Build it generic. It is what makes hidden information real; under trust-auth it's honor-system only
(see Later Work in the spec — real auth + coarse artifact gating is the one access piece not fully
deferrable if competitive integrity matters).

---

## 9. API (`api/app.py`) + trust-auth (`api/auth.py`)

FastAPI, runs locally with uvicorn. Endpoints mirror spec §9.1:

| Method | Path | Body / params | Notes |
|---|---|---|---|
| GET | `/game/{id}/status` | — | `{turn, turn_status}` from ConfigStore |
| GET | `/game/{id}/view` | `?player=` | returns that player's outlet FC |
| POST | `/game/{id}/input` | `{username, ship_id, ax, ay}` | trust-auth; build Delta; DeltaIngest.submit |
| POST | `/game/{id}/advance` | — | manual trigger (testing) |

Auto-advance: when `turn_status.awaiting` empties, call `run_turn`, commit a version, regenerate
outlets. Keep the manual `/advance` for testing regardless.

`auth.py` (MVP): trust the `username` field; map it to a `player.id`; reject inputs where the
username's player ≠ `ship.owner`. Leave a clearly marked `oauth.py`-shaped seam: a `verify(token) ->
player_id` function that's stubbed to read the username now, swappable for Google/Cognito later
(spec §11.1).

---

## 10. Client (`client/`)

Static HTML + vanilla JS, **blank basemap**:

- Render features on a plain `<canvas>` or inline SVG in game-CRS bounds (no tile layer).
- Username field at top (trust-auth). Stored in memory, sent with every input.
- Poll `GET /status` every ~2s; if `turn` increased, `GET /view?player=` and re-render.
- For each owned ship: click on the map to set an acceleration vector (origin = ship). Optional
  client-side `point_in_polygon` against `performance` for instant UX feedback; the physics eddy is
  authoritative. `POST /input`.
- Show "waiting on X of Y" from `turn_status`.
- **Defer** multi-ship selection UX polish — spec leaves §9.4 open; a simple click-the-ship-then-
  click-target cycle is enough for 2v2.

---

## 11. Seed + local play (`scripts/`)

- `seed_game.py`: author one `GameConfig` (bounds `0..1_000_000`, `dt=1`, two players) and a tiny 2v2
  dataset — 2 ships/side, each with a square-ish `performance` polygon, one `random` weapon with a
  modest `envelope` + `damage`, one larger `sensor` envelope, `hp=100`. **All numbers made up**;
  balancing is Later Work. Write via the `local` adapter.
- `play_local.py`: a CLI that loads the seed, accepts acceleration vectors (or random ones), calls
  `run_turn` in a loop with a seeded `rng`, and prints positions / detections / hp each turn. This is
  the fastest correctness-and-fun loop and needs no server or client.

---

## 12. Serverless target (`serverless/` — isolated, build last)

`infra.md` only for MVP — notes, not required to run:

- Each eddy run is milliseconds and bursty (turn boundaries only) → function-per-turn fits well; no
  long-running containers, no QGIS/GRASS/GDAL, single CRS, no reprojection.
- `handler.py`: thin Lambda wrappers that call the *same* `pipeline.run_turn` and the FastAPI app
  (via an ASGI adapter). Persistence/outlets go to S3 through an `atlas_backed`-style adapter impl.
- Sketch only: API Gateway → handlers; S3 layout for versions + per-player outlets; IaC (CDK/SAM/TF)
  TBD. Coarse artifact gating (player A may fetch artifact A) is the access piece to add alongside real
  auth.
- **Hard rule:** `serverless/` imports from `src/`, never the reverse. Verify with a lint/import check.

---

## 13. Tests (`tests/`)

- `test_physics.py`: legal accel moves as expected; illegal accel is clamped (or rejected per fallback);
  `desired_*` cleared after.
- `test_detection.py`: enemy inside sensor envelope is detected; outside isn't; own ships never self-
  detect into `detected_by` as enemies; `detected_by` aggregates multiple viewers.
- `test_combat.py`: detection-gated (undetected enemy in-envelope is NOT fired on); seeded `rng` makes
  `random` deterministic; `hp` decrement and destruction/removal correct.
- `test_pipeline.py`: full `run_turn` on the seed dataset over a few turns; golden-ish assertions on a
  known scenario.

Everything in tests runs against the `local` adapter with a seeded RNG — fast, deterministic, no cloud.

---

## 14. Build order (what to do, in order)

1. `models.py` + `geometry.py` + tests for the geometry helpers.
2. The three eddies + their unit tests (seeded rng). **Stop and confirm the loop is correct/fun via
   `play_local.py` before any server or cloud.**
3. `atlas/ports.py` + `atlas/local.py`; `pipeline.run_turn` wired through it.
4. `outlets/player_view.py` + test that hidden information masks correctly.
5. `api/app.py` + trust-auth + the static client. Play a full 2v2 game in the browser, two tabs.
6. Auto-advance trigger.
7. *Only then:* `atlas_backed.py` against the real Atlas package (after §15 verifications), and the
   isolated `serverless/` target as the infra pilot.

---

## 15. Verifications against the Atlas repo (before wiring `atlas_backed.py`)

1. Atlas primitives (dataswale / version / delta / outlet) are **importable as a library**, or surface
   the coupling that prevents it — decoupling that is the platform-making work.
2. CRS is **per-dataswale metadata** with no hardcoded Earth assumption (grep).
3. An **update-via-join delta-ingest** path exists for an index key (or is trivially addable).
4. Outlets are **parameterizable** (predicate filter) — or scope that one filtered-outlet add.

If any of 1–4 isn't ready, the `local` adapter keeps the whole MVP moving; these only gate the
`atlas_backed` swap and the serverless pilot, never the playable loop.
