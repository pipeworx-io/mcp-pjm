# @pipeworx/pjm

PJM Interconnection market data via Data Miner 2 — real-time LMPs, generation by
fuel type and the seven-day load forecast for the largest US grid operator
(13 states + DC, ~65 million people, Illinois to New Jersey).

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1683+ live data sources.

## Tools

- `pjm_feeds(feed?, row_count?, start_row?)` — list the Data Miner 2 feed catalog,
  or fetch ANY feed by exact name (`da_hrl_lmps`, `inst_load`, …). The escape
  hatch for the hundreds of PJM feeds without a dedicated tool here.
- `pjm_lmp_realtime(pnode_id?, pnode_name?, datetime_beginning_ept?, row_count?)`
  — five-minute real-time LMP with congestion and loss components.
- `pjm_load_forecast(forecast_area?, row_count?)` — seven-day hourly load forecast
  by transmission zone and RTO total.
- `pjm_generation_fuel_mix(fuel_type?, datetime_beginning_ept?, row_count?)` — MW
  and percentage of total by fuel type.

## Auth

**BYO only today.** `Ocp-Apim-Subscription-Key` header (Azure API Management).
Platform fallback env var — **not provisioned**: `PLATFORM_PJM_KEY`.

A key is free but requires registering an account at <https://apiportal.pjm.com/>
**and** subscribing that account to the Data Miner 2 product. Those are two steps:
a registered key that is not subscribed to the feed is refused identically to no
key at all, which is why the pack's 401 message names the subscription
explicitly rather than just saying "bad key".

Until the key lands, every tool refuses with a message containing
`requires an API key`, and the pack is marked `byoExpected: true` in
`workers/gateway/src/pack-manifest.json` so the vendor-key ratchet does not fail
the deploy. **Drop `byoExpected` the day `PLATFORM_PJM_KEY` is provisioned.**

## Data sources

- <https://dataminer2.pjm.com/> — the browser UI over the same feeds.
- <https://api.pjm.com/api/v1/> — the API. Documented envelope:
  `{ "totalRows": N, "items": [...], "links": [...] }`.

## Traps

- **Response shapes here are UNVERIFIED.** Holding no key, no PJM response body
  in this pack has been observed live. The pack normalizes the documented
  `{totalRows, items, links}` envelope and returns `items` **as published**,
  without renaming or reshaping individual fields. That is deliberate: a field
  mapping invented from documentation we could not exercise is a silent-wrong-
  answer waiting to happen, and returning upstream rows untouched is honest about
  what we actually know. Tighten this once a key exists and the real payload can
  be read.
- **A 401 here tells you almost nothing.** Verified 2026-09-17: `api.pjm.com`
  answers 401 with an **empty body** and **no `WWW-Authenticate` header**, both
  with no key and with a syntactically valid but unregistered one. So 401 cannot
  distinguish "no key" from "wrong key" from "key not subscribed to this feed".
  The pack says so in the error rather than guessing at a cause.
- **`format=json` is not the default** — Data Miner 2 defaults to another
  representation. This pack always sets it.
- **Times are Eastern Prevailing Time** (EPT — shifts with DST), and the range
  filter uses Data Miner's own `"M/D/YYYY HH:MM to M/D/YYYY HH:MM"` syntax, not
  ISO-8601.
- **A 200 with 0 rows is treated as an error**, not an empty success: it almost
  always means the date filter fell outside the feed's published range.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "pjm": {
      "url": "https://gateway.pipeworx.io/pjm/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/pjm/mcp` returns the tools in the table
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

Both URLs reach the same gateway and the same 1683+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/pjm_feeds \
  -H 'Content-Type: application/json' \
  -d '{"row_count":50}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/pjm_feeds`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Standalone (no gateway account)

This package also runs as a local stdio MCP server — no Pipeworx account, no
gateway round-trip:

```json
{
  "mcpServers": {
    "pjm": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-pjm"]
    }
  }
}
```

Or run it directly to confirm it starts:

```bash
npx -y @pipeworx/mcp-pjm
```

It speaks MCP over stdin/stdout and answers `initialize`/`tools/list`/`tools/call`
for **only** this pack's tools — none of the shared meta-tools the gateway
connection above adds. Same source, same tools, no ask_pipeworx routing.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Pjm data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
