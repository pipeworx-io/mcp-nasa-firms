# @pipeworx/nasa-firms

Satellite active-fire detections from NASA FIRMS — every pixel MODIS or VIIRS
measured as actively burning, published within ~3 hours of satellite overpass.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1877+ live data sources. This is an independent, unofficial integration — not affiliated with, endorsed by, or published by the upstream provider.

## Tools

- `firms_hotspots_area(bbox, source?, day_range?, date?, limit?)` — detections inside a
  bounding box. `bbox` is `west,south,east,north` in degrees, or `"world"`.
- `firms_hotspots_near(latitude, longitude, radius_km?, source?, day_range?, date?)` —
  detections around a point, sorted nearest first with `distance_km`.
- `firms_data_availability(source?)` — first and last date each sensor product covers.
  Call it before a historical query so an empty result can be read as "outside the
  archive" rather than "no fires".

## Auth

BYO only. FIRMS requires a MAP_KEY and publishes no keyless or demo path —
verified 2026-09-17: every route answers `Invalid MAP_KEY.` (HTTP 400/401)
without one. Pass it as `_apiKey`; the platform fallback env var is
`PLATFORM_NASA_FIRMS_KEY`.

The key is free and self-service at
<https://firms.modaps.eosdis.nasa.gov/api/map_key/> — an email address, no
approval step. Limit is 5,000 transactions per 10 minutes, and a multi-day
query counts as several transactions.

## Data sources

- <https://firms.modaps.eosdis.nasa.gov/api/area/csv/{KEY}/{SOURCE}/{BBOX}/{DAYS}[/{DATE}]> —
  hotspot detections. The key goes in the PATH, not a header or a query param.
- <https://firms.modaps.eosdis.nasa.gov/api/data_availability/csv/{KEY}/{SENSOR}> — archive coverage.

### Things the next person would otherwise rediscover

- **`acq_time` is HHMM with leading zeros dropped.** `"52"` means 00:52Z. A naive
  two-character slice reports it as 52:00 and gets the hour wrong for every
  pre-10:00 overpass.
- **Brightness column names differ by sensor.** VIIRS reports `bright_ti4` /
  `bright_ti5`; MODIS reports `brightness` / `bright_t31`. Both are mapped to
  `brightness_k` / `bright_t31_k`.
- **The `/api/country/` and `/api/countries/` routes are dead.** FIRMS' own API
  index lists them as "Feature currently not available" and they answer
  `Invalid API call.` (HTTP 400). `firms_hotspots_country` was therefore NOT
  built. Do not emulate it with a country bounding box — a box around Chile
  contains Argentina, and the caller would get neighbouring fires labelled as
  the country they asked for. `firms_hotspots_near` is the honest shape.
- **`day_range` is capped at 5** by the upstream, and is counted FORWARD from
  `date` when one is given, backward from today when it is not.
- **Errors arrive as a bare English sentence**, not JSON, at HTTP 400/401/403 —
  so the response body is the diagnosis and is surfaced verbatim.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "nasa-firms": {
      "url": "https://gateway.pipeworx.io/nasa-firms/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/nasa-firms/mcp` returns the tools in the table
above **plus the shared Pipeworx meta-tools** — `ask_pipeworx`,
`discover_tools`, `search_within`, `remember`/`recall` and the rest of the
gateway-wide set. So the tool count you see is larger than this table: a
single-pack endpoint currently lists roughly 30 shared tools alongside the
pack's own. The connection's `initialize` response states its exact scope, and
is the authoritative answer for a given day.

This is deliberate, not multiplexing by accident. The meta-tools are what let a
scoped connection answer a question this pack does not cover — via
`ask_pipeworx`, which routes across the whole catalog — without you adding a
second MCP server. There is currently no way to mount a pack endpoint without
them; if the extra schemas cost you more context than the routing is worth,
connect to the full gateway once rather than to several pack endpoints.

Or connect to the full Pipeworx gateway to get every pack's tools listed
directly, instead of just this one's:

```json
{
  "mcpServers": {
    "pipeworx": {
      "url": "https://gateway.pipeworx.io/mcp"
    }
  }
}
```

Both URLs reach the same gateway and the same 1877+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/firms_hotspots_area \
  -H 'Content-Type: application/json' \
  -d '{"bbox":"-125,32,-114,42","source":"VIIRS_SNPP_NRT","day_range":1}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/firms_hotspots_area`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Standalone (no gateway account)

This package also runs as a local stdio MCP server — no Pipeworx account, no
gateway round-trip:

```json
{
  "mcpServers": {
    "nasa-firms": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-nasa-firms"]
    }
  }
}
```

Or run it directly to confirm it starts:

```bash
npx -y @pipeworx/mcp-nasa-firms
```

It speaks MCP over stdin/stdout and answers `initialize`/`tools/list`/`tools/call`
for **only** this pack's tools — none of the shared meta-tools the gateway
connection above adds. Same source, same tools, no ask_pipeworx routing.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Nasa Firms data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
