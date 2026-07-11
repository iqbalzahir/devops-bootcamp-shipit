# Board MVP — Design (Milestone 2)

**Date:** 2026-07-12
**Status:** Approved — proceeding to plan
**Scope:** The `board/` MVP only (spec §5, §11 milestone 2). This resolves the open decisions
left by the architecture spec and details the MVP surface. It is a teaching prop: **pedagogy-first,
sensible defaults, no over-engineering.**

**Reads with:** `docs/specs/2026-07-11-ship-it-architecture-design.md` (§5 board, §11 milestones),
the pinned event contract in `CLAUDE.md`, GitHub issue #2, and the pattern to clone at
`~/repo/devops-bootcamp-game/server/`.

---

## 1. What this milestone delivers

The **board** — Mission Control — is the one long-running server in the prop. It ingests pipeline
events over authed HTTP, keeps an in-memory roster keyed by GitHub username, and broadcasts it to
Three.js spectators over WebSocket. It is dual-role (one codebase): the instructor's shared projected
orbit, **and** the artifact each learner builds + deploys to their own EC2 in the S4 capstone.

**This MVP = a correct, plain, honest board.** Ships render in the right place for their pipeline
state, an orbiting ship links to its live site, the scene disposes cleanly, and a no-WebGL /
reduced-motion fallback works. The *spectacle* (animated countdown/liftoff, blueprint aesthetic,
projector-legibility tuning) is **Milestone 4**; wiring the contract end-to-end with a real workflow
is **Milestone 3**; the multi-arch GHCR publish is **Milestone 6**.

## 2. Resolved decisions

| # | Decision | Resolution |
|---|---|---|
| Scene scope | M2 vs M4 boundary | **Correct state, plain art.** Ships placed by `(stage,status)`; click orbit ship → `siteUrl`; correct dispose on rebuild; fallback solid. Animation/aesthetic/projector tuning → M4. |
| Roster lifecycle | ships arrive via stateless POST (no ws lifecycle) | **Latest-wins per `callsign`, session-persistent.** Newest event replaces state; nothing auto-expires; process restart = clean slate (matches "ephemeral per cohort"). |
| Auth mode | how "reject unauthenticated in prod" toggles | **Token-set enforces, else warn.** `SHIPIT_TOKEN` set → Bearer required (401 otherwise); unset → open + loud startup warning, so local smoke/dev needs no token. |
| Client build | how the spectator is built/served | **Vite-built client** (consistent with `launchpad`'s three+vite toolchain), served static by the Node process; multi-stage Dockerfile. |

## 3. Repo layout (`board/`, mirrors `launchpad/`)

```
board/
  package.json          # dependencies: ws   ·   devDependencies: three, vite
  vite.config.js        # build client/ → board/dist, base: './'
  src/                  # the Node server — run directly, NO build step
    index.js            # entry: read PORT + SHIPIT_TOKEN → createServer, log the auth mode
    app.js              # http (serve dist/ + POST /api/event) + ws + dirty-tick broadcast
    room.js             # Roster (Map<callsign,entry>) + sanitizeEvent
    messages.js         # parse + rosterMsg helpers
  client/               # the Three.js spectator (the showpiece; Vite-built)
    index.html
    main.js             # bootstrap: fallback-detect → scene | plain roster; ws connect + reconnect
    scene.js            # Three.js: place ships by (stage,status); dispose/rebuild correct
    ship-mesh.js        # small procedural rocket (adapted from launchpad), tinted + labelled
    placement.js        # PURE (stage,status) → {zone, height} — unit-testable, no WebGL
    fallback.js         # no-WebGL / reduced-motion → live plain roster list
    style.css
  test/                 # dev-time node --test only
    room.test.js        # sanitizeEvent + Roster latest-wins lifecycle
    placement.test.js   # the (stage,status) → zone mapping
    server.test.js      # POST /api/event (authed / 401 / 400) → appears in ws roster  ← loop proof
  Dockerfile            # multi-stage: vite build → node:20-alpine runtime
  README.md
```

Node 20, ESM, no CDN (three is bundled by Vite). `ws` is the only runtime dependency; `three` and
`vite` are dev-only (build stage), keeping the runtime image lean.

## 4. Server — `src/app.js` (clones arena-server; two differences)

`createServer({ port = 3000, token = null, publicDir = <dist> })` → `{ port, roster, server, close() }`.

- **`POST /api/event`** — the ingest. Read the JSON body → **auth check** (§6) → `sanitizeEvent`
  (§5) → `roster.upsert(event)` → `dirty = true` → `202 { ok: true }`. Ingest is HTTP, **not** ws.
- **`GET *`** — serve the built client from `dist/`, with the game's path-traversal guard
  (`file.startsWith(publicDir)`); `/` → `index.html`. Unknown path → `404`.
- **WebSocket** — **spectators only.** On connect: immediately send the current roster, then add the
  socket to the broadcast set. Inbound ws messages are ignored (spectators are read-only). Drop on
  `close`/`error`.
- **Broadcast** — the game's `dirty`-flag + ~50 ms `setInterval` tick coalesces event bursts into one
  roster push to all sockets.
- **Robustness** — a malformed request or one misbehaving socket **never crashes the hub** (try/catch
  around body parsing and per-socket sends, per the game).

Message helpers (`src/messages.js`): `parse(raw)` (safe JSON→object|null) and
`rosterMsg(list)` → `JSON.stringify({ t: 'roster', ships: list })`.

## 5. Roster + `sanitizeEvent` — `src/room.js`

`Roster`: a `Map<callsign, entry>`.
- `upsert(event)` — `entries.set(event.callsign, event)` (latest-wins).
- `list()` — `[...entries.values()]`.
- No `leave`/TTL — cleared only by process restart (decision #2).

`sanitizeEvent(raw)` — **strict on identity/enums, lenient on cosmetics.** Returns a clean entry, or
`null` when the event is unusable (→ the server answers `400` with a short hint, so a learner
hand-authoring the POST sees *why*):

| Field | Rule | On violation |
|---|---|---|
| `callsign` | required; string, control-chars stripped, trimmed, ≤ 39 chars (GitHub max) | empty → **reject (null)** |
| `stage` | ∈ `{pad, build, test, clearance, liftoff}` | **reject** |
| `status` | ∈ `{running, passed, failed, aborted, shipped}` | **reject** |
| `color` | `/^#[0-9a-fA-F]{6}$/` | **default** to a neutral hex (cosmetic — don't reject) |
| `version` | optional; cleaned string, capped length | drop if absent/invalid |
| `siteUrl` | optional; must parse as an `http:`/`https:` URL | drop otherwise — blocks `javascript:` before it can be rendered as an orbit link |

Entry shape (matches the pinned contract): `{ callsign, stage, status, color, version?, siteUrl? }`.

## 6. Auth — `src/index.js` + `app.js` (decision #3)

At startup `index.js` reads `token = process.env.SHIPIT_TOKEN`:
- **set** → enforce. `POST /api/event` requires `Authorization: Bearer <token>`, compared with
  `crypto.timingSafeEqual`; missing or wrong → `401`. This is the S3 "no clearance → can't report"
  lesson.
- **unset** → open, with `console.warn('[board] SHIPIT_TOKEN unset — accepting UNAUTHENTICATED events (dev mode)')`
  at boot, so local smoke/dev runs tokenless.

## 7. State → placement — `client/placement.js` (pure, unit-tested)

The heart of "renders the roster by stage/status," kept as a pure function so it is testable without
WebGL:

- `status ∈ {failed, aborted}` → **grounded** on the pad (aborted marker) — regardless of stage.
- `stage === 'liftoff' && status ∈ {passed, shipped}` → **orbit** (slow ring; clickable → `siteUrl`).
- otherwise → **ascending**, height ∝ stage index (`pad 0 · build 1 · test 2 · clearance 3 · liftoff 4`).

MVP visuals are deliberately plain: dark background, a pad plane, an orbit ring, ships = small
procedural rockets tinted by `color` with a `callsign` text label. A single `Raycaster` handles clicks
— an orbit ship with a `siteUrl` opens it in a new tab. No countdown/liftoff animation, no blueprint
styling, no projector tuning (all M4).

## 8. Dispose correctness — the deferred M1 item (#1), now live

Ship objects are tracked in `Map<callsign, obj>` and **diffed** each roster update (add / reposition /
update-color-label / remove), the game's `sync()` shape — not a full teardown per tick. The scene's
`dispose()` cascades geometry + material + **textures** (the M1 fix, carried forward) and has **real
callers here**: page unload, and the reduced-motion media-query toggle switching scene → fallback.
(MVP ships are procedural, so the M1 in-flight-GLTF-load guard is not needed unless a `.glb` is later
introduced.)

## 9. Fallback — `client/fallback.js`

No-WebGL or `prefers-reduced-motion` → a live **plain roster list** (callsign · stage · status ·
colour chip), fed by the same ws stream. Text is escaped before insertion (carry the launchpad
`escapeHtml` habit). Projectors have WebGL, but the fallback keeps the board honest and testable.

## 10. Testing — light, per spec §8/§9

Dev-time `node --test` only (no framework, no Playwright, no jsdom):
- `room.test.js` — `sanitizeEvent` (valid; bad enum rejected; bad colour → default; `javascript:`
  siteUrl dropped; empty callsign rejected) and `Roster` latest-wins.
- `placement.test.js` — the `(stage,status)` → zone/height mapping across the state space.
- `server.test.js` — spins the server on port 0; **this is the end-to-end loop proof**: `POST
  /api/event` with a valid Bearer → `200`, ship appears in the ws roster; wrong/missing token (when
  enforcing) → `401`; malformed body → `400`.

`scene.js` stays **untested by design** (no WebGL/jsdom harness — the no-extra-framework rule);
verified by build + manually driving a `curl` POST against the running server. A `curl`-based
`scripts/smoke.sh` belongs to **Milestone 3** (wire the contract end-to-end), not here.

## 11. Dockerfile (multi-stage) + env

```dockerfile
# --- build the client ---
FROM node:20-alpine AS build
WORKDIR /app
COPY package.json package-lock.json* ./
RUN npm install
COPY vite.config.js ./
COPY client ./client
RUN npm run build                       # → /app/dist

# --- runtime ---
FROM node:20-alpine
WORKDIR /app
COPY package.json package-lock.json* ./
RUN npm install --omit=dev              # only ws
COPY src ./src
COPY --from=build /app/dist ./dist
EXPOSE 3000
ENV PORT=3000
CMD ["node", "src/index.js"]
```

Environment: `PORT` (default `3000`), `SHIPIT_TOKEN` (unset ⇒ dev/open mode). `BOARD_URL` is a
*workflow/client* concern (where learner pipelines POST), **not** consumed by the board process.

`package.json` scripts: `dev` (build client + start server), `build` (`vite build`),
`start` (`node src/index.js`), `test` (`node --test`).

## 12. Non-goals (M2 boundary)

Explicitly **out** of this milestone: countdown/liftoff animation, blueprint aesthetic, projector
legibility tuning (**M4**); a `curl` `scripts/smoke.sh` full-loop proof (**M3**); multi-arch GHCR
publish (**M6**); roster persistence, TTL eviction, admin-clear endpoint; any change to `launchpad`
or the pinned event contract.

## 13. Pedagogy notes

- The board is a **black box learners build and ship, never edit** — so internal complexity is fine,
  but the *Dockerfile* and *run story* stay clean because those are what S4 touches.
- Keep it **self-contained and env-configurable** (`PORT`, `SHIPIT_TOKEN`) — one image, two deploy
  contexts (shared instructor instance + each learner's EC2).
- Fail loud, no swallowed errors; strict on the enums a learner's payload must get right (helpful
  `400`s teach the contract), lenient on cosmetics.
