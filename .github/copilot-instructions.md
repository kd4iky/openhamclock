The following guidance is tuned for an AI coding agent (Copilot/GitHub Code Assistant) working on OpenHamClock. Keep instructions concise and actionable — focus on patterns, files, and examples the project actually uses.

1) High-level architecture (why and where)
- This is a single-repo full-stack Node + React app. Backend: `server.js` (Express) serves static build (`dist/`) and provides API proxy endpoints (NOAA, POTA, PSKReporter, TLE, DX cluster proxies). Frontend: Vite + React 18 in `src/` — entry `src/main.jsx` -> `src/App.jsx`.
- Major responsibilities:
  - `server.js`: API proxy, caching, config loading (.env/.env.example, config.json), visitor tracking, optional hybrid propagation via ITURHFPROP service.
  - `src/`: UI panels and map. Hooks in `src/hooks/` implement data fetching and realtime patterns.
  - `src/components/WorldMap.jsx`: central visualization; integrates plugin layers via `src/plugins/layerRegistry.js`.

2) Key files to read for context
- `server.js` — start here to understand how backend data sources, caching, and configuration works.
- `README.md` — contains runtime/dev workflows (dev server vs production build, `.env` usage, ITURHFProp notes).
- `src/App.jsx` and `src/main.jsx` — app startup, persisted localStorage patterns, and config merge order (localStorage > server config > defaults).
- `src/hooks/*` — each panel's data logic lives here; hooks use polling, WebSocket/MQTT, or fallbacks (see `usePSKReporter` in README for MQTT fallback behavior).
- `src/components/WorldMap.jsx` and `src/components/PluginLayer.jsx` — where map layers and plugin hooks are composed. The plugin registry is `src/plugins/layerRegistry.js`.

3) Project-specific conventions and patterns
- Config precedence: localStorage (frontend) overrides server-provided config, which itself prefers `.env` values when server starts. When editing station info prefer `.env` or `config.json` for server-side defaults.
- Map and panel state is persisted to localStorage under keys prefixed with `openhamclock_` (e.g., `openhamclock_mapSettings`, `openhamclock_dxFilters`). When changing UI behavior, search for those keys to keep persistence consistent.
- Hooks return lightweight objects with `.data` and `.error` or explicit event streams. Follow existing hook shapes when adding new ones. Inspect `src/hooks/index.js` for available hooks.
- Plugins: A plugin exports metadata and a `useLayer` hook (see `src/plugins/layers/*`). Register by adding the module to `src/plugins/layerRegistry.js` and exposing metadata with `id`, `name`, `defaultEnabled`.

4) Dev & run workflows (explicit commands)
- Local development (hot reload for frontend):
  - Start backend (serves API proxies and optional static build): `node server.js`
  - Start frontend dev server (Vite hot reload): `npm run dev` (default port 5173)
  - Open app: backend at http://localhost:3000 (if running `server.js`) or frontend dev at http://localhost:5173
- Production / preview:
  - Build frontend: `npm run build` (Vite)
  - Serve built app: `npm start` (runs `node server.js`, which serves `dist/`)
- Tests: `npm test` is a placeholder that currently exits 0. Use Playwright tests in `devDependencies` if adding E2E tests.

5) Integration points & external services
- DX Cluster: proxied via separate microservice (see `dxspider-proxy/` folder) and the server connects via persistent telnet or HTTP sources. Check `.env` variables and README sections for `DX_CLUSTER_SOURCE` options.
- PSKReporter: primary via MQTT WebSocket; the frontend hook falls back to `server.js` HTTP proxy when MQTT fails (see README). Look at `src/hooks/usePSKReporter.js` for reconnection/backoff patterns.
- Satellites: TLEs fetched in server and served under `/api/satellites/tle` with long cache (1h). Frontend uses `satellite.js` for SGP4 calculations.
- ITURHFProp: optional microservice in `iturhfprop-service/`. If `ITURHFPROP_URL` is present in `.env`, the server uses it for higher-fidelity propagation predictions.

6) Preferred change patterns
- When adding a new data panel:
  - Add a hook in `src/hooks/` following existing examples (polling interval, cache behavior, and rate-limit handling).
  - Add a component under `src/components/` and wire it in `src/App.jsx` or the dockable layout.
  - If it needs map overlays, implement a plugin-like `useLayer` hook under `src/plugins/layers/` and register it in `src/plugins/layerRegistry.js`.
- When changing server fetch behavior: respect the server's caching TTL patterns in `server.js` (look for `NOAA_CACHE_TTL` and `/api` cache durations) and avoid aggressive client polling.

7) Examples (copy-paste patterns)
- Persisting UI map settings: localStorage key `openhamclock_mapSettings` stored as JSON in `WorldMap.jsx`.
- Registering a plugin layer: export `metadata` and `useLayer` from `src/plugins/layers/<name>.js` and add import + entry in `src/plugins/layerRegistry.js`.
- Grid conversion: server's `gridToLatLon` in `server.js` shows how Maidenhead grid → lat/lon is computed; frontend uses `utils/geo.js` helpers (use the same convention).

8) Quick lint/test/build checks for pull requests
- Run `npm run build` to ensure Vite build passes. Then `node server.js` to smoke test static serving. There is no CI config in-repo; if you add tests, include Playwright or a simple `npm test` script.

If any part of this guidance is unclear or you'd like me to include more examples (hooks, a concrete plugin skeleton, or a short contributor checklist), tell me which sections to expand and I'll iterate.
