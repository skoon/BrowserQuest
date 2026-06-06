# BrowserQuest — Project Milestones

A prioritized roadmap of 10 milestones to fix, modernize, and improve BrowserQuest.

---

## M1: Reproducible Builds & Dependency Modernization

**Goal:** Make the project buildable on modern Node.js (18+) with locked, known-good dependencies.

### Steps
1. Replace `package.json`'s catch-all `">0"` version ranges with exact, known-compatible versions.
2. Pin all server dependencies:
   - Replace abandoned `websocket-server` (miksago) with `ws` — the de facto standard WebSocket library.
   - Replace abandoned `sanitizer` with `sanitize-html` or `DOMPurify`.
   - Replace `log` with `winston` or `pino` for structured logging.
   - Replace `memcache` with `memcached` (maintained fork) or remove if unused.
   - Remove `bison` (dead code — `useBison` is hardcoded `false`).
   - Keep `underscore` (lightweight, no breaking changes needed).
3. Generate `package-lock.json` and add a `.nvmrc` (e.g., `18`).
4. Add a `.gitignore` entry for `node_modules/` and `client-build/` if missing.
5. Verify the server starts with `npm install && node server/js/main.js`.
6. Verify `bin/build.sh` still produces a valid client build, or replace with a webpack/vite step (see M5).

**Why:** Without pinned deps, `npm install` today installs versions the original authors never tested against. Several packages are abandoned and may have CVEs or simply fail to install on modern Node.

---

## M2: Server Crash Resilience & Graceful Shutdown

**Goal:** Eliminate the dangerous `process.on('uncaughtException')` blanket and add proper error boundaries.

### Steps
1. Replace `server/js/main.js:97` `process.on('uncaughtException')` with a handler that logs the error **and exits** with non-zero code, plus a process manager recommendation (e.g., `--watch` flag on Node 18+).
2. Add `process.on('unhandledRejection')` to catch promise rejections (not currently used, but future-proof).
3. Add `SIGTERM`/`SIGINT` handlers in `server/js/main.js`:
   - Close all WebSocket connections gracefully.
   - Save any in-memory player state.
   - Close the HTTP server.
   - Exit cleanly.
4. Add try/catch around the main per-tick logic in `worldserver.js` to prevent one bad entity from crashing the whole loop.

**Why:** The current catch-all hides crashes and leaves the server in an inconsistent state. A crash-restart cycle is vastly preferable.

---

## M3: Client Build Pipeline Modernization

**Goal:** Replace the antiquated RequireJS optimizer with a modern bundler for faster builds, smaller bundles, and better DX.

### Steps
1. Replace the AMD/RequireJS module system with ES modules (import/export).
2. Set up a modern bundler (Vite is ideal for this project — simple config, fast HMR, native ESM in dev).
3. Configure the bundler to:
   - Bundle and minify JS (Terser).
   - Copy static assets (sprites, audio, images, fonts, CSS).
   - Output to `client-build/`.
4. Migrate the existing `client/js/lib/` third-party libs:
   - Replace jQuery 1.7.1 with minimal vanilla DOM helpers (or a modern jQuery slim if absolutely needed).
   - Replace bundled `require-jquery.js` with standard jQuery + Vite plugin.
   - Keep Modernizr and Underscore but load them via npm or ES module copies.
5. Add `dev` and `build` scripts to `package.json`.
6. Remove `bin/r.js` (the bundled RequireJS optimizer).
7. Update `bin/build.sh` to delegate to the new bundler.

**Why:** The RequireJS optimizer is unmaintained, produces large bundles, and has no HMR. Vite (or even webpack/esbuild) gives instant dev feedback and smaller production builds.

---

## M4: Linting, Formatting & Code Quality

**Goal:** Enforce a consistent code style and catch bugs statically.

### Steps
1. Add ESLint with a baseline config:
   - Use `eslint:recommended` initially.
   - Set `"env": { "node": true, "browser": true, "jquery": true }`.
   - Add `"extends"` for any framework-specific rules (none needed for vanilla JS).
2. Add Prettier for formatting:
   - Single config across all JS/CSS/JSON.
   - Run as an ESLint plugin to show formatting issues as lint errors.
3. Fix all lint errors in:
   - `server/js/` (Node side).
   - `shared/js/` (shared code).
   - `client/js/` (browser side — be careful with globals).
4. Add `lint` script to `package.json`.
5. Add a pre-commit hook (husky or simple `.githooks/`) that runs lint.
6. For the existing codebase, add `/* eslint-disable */` or inline disable comments only where absolutely necessary (e.g., in the old 3rd-party libs that will be replaced in M5).
7. Add JSDoc type annotations to critical functions (or switch to TypeScript in M8).

**Why:** The codebase has inconsistent indentation, loose equality, missing semicolons, and commented-out code. This makes maintenance harder and increases the chance of bugs.

---

## M5: Server Input Validation & Anti-Abuse

**Goal:** Harden the server against malicious clients.

### Steps
1. Add rate limiting:
   - Chat messages: max 3 per second per player.
   - Movement commands: max 10 per second per player.
   - Attack commands: rate-limited by weapon speed (server-authoritative).
   - General message throttle: buffer floods and disconnect repeat offenders.
2. Expand message format validation in `format.js`:
   - Bounds-check coordinates (must be within map dimensions).
   - Validate entity IDs exist in the current world state.
   - Validate item types against known item definitions.
   - Validate player names: length (3-15 chars), alphanumeric only, no reserved names.
3. Add entity-action validation:
   - Verify attacker is within range of target.
   - Verify attacking own faction/self is not allowed.
   - Verify loot target is actually dead.
4. Sanitize chat input with the new library from M1 (e.g., `sanitize-html`).
5. Add player session timeout enforcement (reduce from 15 min to a configurable duration, e.g., 5 min).

**Why:** The server currently trusts client messages almost entirely. A malicious client can teleport, attack through walls, crash the server, or inject XSS via chat.

---

## M6: Testing Infrastructure

**Goal:** Add automated tests to prevent regressions.

### Steps
1. Choose a test runner: Vitest (modern, fast, compatible with Vite from M3).
2. Add unit tests for shared logic:
   - `shared/js/gametypes.js` — all type-checking functions.
   - `server/js/formulas.js` — damage calculation, HP formulas.
   - `server/js/utils.js` — sanitization, random, math.
   - `server/js/format.js` — message validation.
3. Add server integration tests:
   - Connect a mock WebSocket client.
   - Send valid/invalid messages.
   - Verify server responds correctly.
4. Add a simple client rendering test:
   - Verify game initializes without crashing.
   - Verify entities render in expected positions (snapshot test of canvas).
5. Add `test` script to `package.json`.
6. Set up test fixtures (mock player, mock mob, mock map).
7. Add coverage reporting.

**Why:** Zero tests means every change risks regression. Starting with the shared and server-side logic gives the highest ROI.

---

## M7: Extract Hardcoded Map Coordinates Into Data

**Goal:** Decouple game logic from the specific map layout so the map can be edited without touching code.

### Steps
1. Audit all hardcoded coordinate references in `client/js/game.js`:
   - Achievement trigger zones (lines 867-884: `player.gridX <= 85 && player.gridY <= 179`, etc.).
   - Any hardcoded spawn or camera positions.
2. Add a "trigger zones" section to the map JSON format:
   ```json
   {
     "triggerZones": [
       { "id": "death_achieve_1", "type": "achievement", "x": 85, "y": 179, "w": 5, "h": 5, "achievement": "death" }
     ]
   }
   ```
3. Load trigger zones from map data at game start.
4. Replace the `if (player.gridX <= 85 && ...)` chains with data-driven checks.
5. Update `tools/maps/export.py` to preserve trigger zone data from Tiled custom properties if possible.

**Why:** The achievement system is tightly coupled to a specific map. If anyone wants to create a custom map, they have to modify source code. This is the main barrier to modding.

---

## M8: TypeScript Migration (Optional — High Impact)

**Goal:** Add type safety across the entire codebase to catch bugs at compile time.

### Steps
1. Add TypeScript to the project (`typescript` and necessary `@types/*` packages).
2. Start with `shared/js/gametypes.js` — convert to `.ts` first since it has no dependencies.
3. Convert `server/js/formulas.js` and `server/js/utils.js`.
4. Convert the server-side entity classes: `Entity`, `Character`, `Player`, `Mob`, `Item`, `Chest`, `NPC`.
5. Convert `server/js/worldserver.js` and `server/js/message.js`.
6. For the client, convert `game.js`, `renderer.js`, `gameclient.js`.
7. Define key interfaces:
   - `EntityState`, `PlayerState`, `MobState`
   - `Message` (discriminated union per message type)
   - `MapData`, `TileData`
   - `GameConfig`
8. Add a `typecheck` script to `package.json`.
9. Run typecheck in CI.
10. Keep `jsconfig.json` in the interim for better editor support on `.js` files.

**Why:** The largest source of bugs in a dynamic MMO is type confusion — treating an ID as a coordinate, or a string as a number. TypeScript catches these at compile time. This is a big investment but pays off immediately for any feature work.

**Cost:** High. Estimate 2-4 weeks for a full migration. Skip if the project will remain in maintenance mode.

---

## M9: Security Hardening

**Goal:** Eliminate XSS, injection, and network sniffing vulnerabilities.

### Steps
1. **WebSocket TLS:**
   - Add `wss://` support with configurable SSL cert/key paths.
   - Update `client/js/config.js` to support `wss://` endpoints.
   - Auto-detect secure context (if page is served over HTTPS, force WSS).
2. **Input sanitization overhaul:**
   - Replace the old `sanitizer` with `sanitize-html` (or `DOMPurify` on the client side).
   - Strip all HTML tags from chat except basic formatting (bold, italic) if desired.
   - Validate player names with a strict regex: `/^[a-zA-Z0-9_-]{3,16}$/`.
3. **localStorage integrity:**
   - Add a HMAC signature to player data stored in `client/js/storage.js` to detect tampering.
   - Or treat localStorage as untrusted and validate all fields on load.
4. **Entity ID entropy:**
   - Replace the weak `_createId`  in `server/js/ws.js` with `crypto.randomUUID()` (Node 18+).
5. **Add a Content-Security-Policy header:**
   - In the HTTP server, serve a restrictive CSP.
   - Whitelist only the WebSocket origin and self.

**Why:** The game transmits all data in cleartext, accepts arbitrary HTML in chat, and stores player data without integrity checks. For a tech demo this is acceptable; for any real deployment it's not.

---

## M10: Developer Experience & Observability

**Goal:** Make the project easy to develop, debug, and operate.

### Steps
1. **Structured logging:**
   - Replace `console.log` / `log()` calls with structured JSON logging using `pino`.
   - Log levels: `info` (connections, disconnections), `warn` (suspicious activity), `error` (exceptions, invalid states), `debug` (entity state, messages).
   - Add request/connection IDs to correlate log lines.
2. **Metrics dashboard (optional):**
   - Expose Prometheus metrics: connected players, entities per map, messages per second, tick duration.
   - Or restore the memcached metrics path with a simple Grafana dashboard.
3. **Hot-reload:**
   - If using Vite (M3), the client already has HMR.
   - For the server, add `nodemon` or Node `--watch` to auto-restart on file changes.
4. **Environment-based configuration:**
   - Use `.env` files (via `dotenv`) for port, host, TLS, metrics toggles.
   - Provide `.env.example` with defaults.
5. **Docker support:**
   - Add `Dockerfile` for the server (multi-stage: build deps, then run).
   - Add `docker-compose.yml` with server + optional memcached if metrics are used.
6. **Debug tools:**
   - Reinstate the commented-out debug keys (Space for pathing grid, F for debug info, A for test hit) behind a `?debug=true` flag or config toggle.
   - Add a `/debug` WebSocket command (admin only) to inspect world state.

**Why:** The current DX is poor — no logs, no metrics, no hot-reload, no Docker. The commented-out debug tools show the original developers wanted them but never finished the feature.

---

## Suggested Ordering

| Phase | Milestones | Theme |
|-------|-----------|-------|
| **Phase 1** | M1, M2 | Foundation — make it build and not crash |
| **Phase 2** | M3, M4 | Modernization — bundler and code quality |
| **Phase 3** | M5, M6 | Robustness — security and testing |
| **Phase 4** | M7, M9 | Correctness — data-driven map, security hardening |
| **Phase 5** | M8, M10 | Polish — TypeScript and DX |

Each phase builds on the previous one. Phase 1 is essential for any further work. Phases 2-4 can be parallelized. Phase 5 is optional depending on project goals.
