# MovieBox Express API

A full-featured Express proxy for the MovieBox API — exposes search, trending, movie details, series/season/episode navigation, stream URLs, and subtitle links through clean REST endpoints.

## Run & Operate

- `pnpm --filter @workspace/api-server run dev` — run the API server (port 8080, proxied at `/api`)
- `pnpm run typecheck` — full typecheck across all packages
- `pnpm run build` — typecheck + build all packages

## Stack

- pnpm workspaces, Node.js 24, TypeScript 5.9
- API: Express 5
- Build: esbuild (CJS bundle)
- No database required — pure proxy

## Where things live

- `artifacts/api-server/src/lib/moviebox.ts` — MovieBox API client (V1/V2/V3 layers + HMAC-MD5 signing)
- `artifacts/api-server/src/routes/` — Express route handlers (search, trending, details, docs)
- `lib/api-spec/openapi.yaml` — OpenAPI spec (health only; MovieBox routes are proxy-only)

## Architecture decisions

- **Three API layers**: V1 (`h5.aoneroom.com`) for cookie-auth popular searches, V2 (`h5-api.aoneroom.com`) for homepage/details GET endpoints, V3 (`api6.aoneroom.com`) for signed HMAC-MD5 search, subjects, play info, and resources.
- **V3 HMAC-MD5 request signing**: `X-Client-Token = {ts},{md5(reverse(ts))}` and `x-tr-signature = {ts}|2|{base64(hmac-md5(canonical, key))}` — implemented in TypeScript using Node.js `crypto`.
- **V3 host pool fallback**: tries api6 → api5 → api4 → api4sg → api3 on 5xx/429/403 responses.
- **Cookie caching**: V1 `account` cookie is fetched once from app-info endpoint and cached for 23 hours.
- **Runtime token refresh**: absorbs `x-user` response headers from V3 to refresh the bearer token automatically.

## Product

Users can call this API to:
- Search movies, series, anime, music by keyword with filtering and pagination
- Browse trending/homepage content and hot/popular searches
- Fetch full movie or series details with subjectId
- Get season episode lists for TV series
- Retrieve stream URLs (multiple resolutions) and subtitle files for any episode or movie
- Autocomplete with search suggestions

## API Endpoints

```
GET /api/                        — Full API docs with workflow
GET /api/healthz                 — Health check

GET /api/homepage                — Homepage banners + category rows
GET /api/trending                — Trending content (?page=1&tab=0)
GET /api/trending/movies         — Trending movies tab
GET /api/trending/series         — Trending series tab
GET /api/hot                     — Hot ranked content
GET /api/popular-searches        — Popular search terms

GET /api/search?q=Avatar         — Search (?page=1&size=20&type=MOVIES|TV_SERIES|ANIME)
GET /api/search/suggest?q=Av     — Autocomplete suggestions

GET /api/movie/details?id=...    — Movie info + stream links
GET /api/episode/resource?id=... — Download/stream URLs (?season=0&episode=0)
GET /api/episode/play?id=...     — Play info MPD/HLS (?season=1&episode=3)

GET /api/series/details?id=...   — Series info + episode list (?season=1)
GET /api/season?id=...           — Season metadata + episode list (?season=1)

GET /api/details?id=...          — Raw detail by subjectId or ?path=detailPath
```

## Upstream APIs

| Layer | Base URL | Auth | Used for |
|-------|----------|------|----------|
| V1 | `https://h5.aoneroom.com` | `account` cookie | Hot content, popular searches |
| V2 | `https://h5-api.aoneroom.com` | None | Homepage, suggest, detailPath lookup |
| V3 | `https://api6.aoneroom.com` | HMAC-MD5 signed headers | Search, subjects, seasons, play info, resources |

## User preferences

_Populate as you build — explicit user instructions worth remembering across sessions._

## Gotchas

- V3 signing: always sign with the timestamp at request time — stale timestamps are rejected.
- V1 cookie expires server-side after 30 days; client caches for 23h and auto-refreshes.
- V3 host pool: api6 is default; on 5xx/429/403 the next host is tried automatically.
- Run `pnpm --filter @workspace/api-server run dev` (not root `pnpm dev`) — the root has no dev script.

## Pointers

- See the `pnpm-workspace` skill for workspace structure, TypeScript setup, and package details
