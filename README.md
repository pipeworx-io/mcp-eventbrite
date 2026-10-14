# @pipeworx/eventbrite

Eventbrite event discovery by city/category, and single-event detail
(venue, organizer, ticket price range).

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1800+ live data sources. This is an independent, unofficial integration — not affiliated with, endorsed by, or published by the upstream provider.

## Tools

- `eventbrite_city_events(city, category?, keyword?, page?)` — upcoming
  events in a city, optionally filtered by category slug (e.g. `"music"`,
  `"food-and-drink"`) and/or a free-text keyword. Returns event name, time,
  venue, category tags and a ticket URL. An unrecognized category slug is
  silently ignored by Eventbrite (falls back to all categories) rather than
  erroring, so the response carries `category_applied` to say whether the
  filter actually took.
- `eventbrite_event(event_id)` — full detail for one numeric Eventbrite
  event id: description, start/end time, venue, organizer, and ticket
  price range/availability.

Neither tool returns attendee or guest lists — Eventbrite does not expose
those without authentication, and this pack would not surface them even if
it could.

## Auth

Keyless. Both surfaces are Eventbrite's own public pages/endpoint with no
login wall: a bad event id answers a clean 404 ("event does not exist"),
and a bad city answers a clean 404 from the discovery page — neither reads
as an auth failure.

Eventbrite retired its public v3 search API; it now answers 401 to a
missing CSRF token/Referer, which this pack does not forge. Both tools here
use surfaces that do not require that: the per-event detail endpoint, and
the server-rendered city/category discovery pages a browser gets.

## Data sources

- Eventbrite — event-discovery pages (city/category browse and keyword
  search) and single-event detail. Routes live only in `src/index.ts`.

## Notes

- `eventbrite_city_events` accepts free-text city names ("San Francisco,
  CA", "London", "Austin, TX") — Eventbrite's own place lookup resolves
  them; an unresolvable city answers a clean 404 rather than a silent empty
  result.
- Results page at 20 events/page (Eventbrite's own page size past page 1);
  `page` is 1-based.
- Eventbrite rate-limits these pages by IP. The pack does not retry on
  429 — a caller should back off and retry later rather than hammer it.
- `eventbrite_event`'s price fields reflect ticket tiers as currently
  configured and can be `0.00` for a genuinely free event.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "eventbrite": {
      "url": "https://gateway.pipeworx.io/eventbrite/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/eventbrite/mcp` returns the tools in the table
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

Both URLs reach the same gateway and the same 1800+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/eventbrite_city_events \
  -H 'Content-Type: application/json' \
  -d '{"city":"San Francisco, CA","category":"music"}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/eventbrite_city_events`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Standalone (no gateway account)

This package also runs as a local stdio MCP server — no Pipeworx account, no
gateway round-trip:

```json
{
  "mcpServers": {
    "eventbrite": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-eventbrite"]
    }
  }
}
```

Or run it directly to confirm it starts:

```bash
npx -y @pipeworx/mcp-eventbrite
```

It speaks MCP over stdin/stdout and answers `initialize`/`tools/list`/`tools/call`
for **only** this pack's tools — none of the shared meta-tools the gateway
connection above adds. Same source, same tools, no ask_pipeworx routing.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Eventbrite data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
