# Illinois Compiled Statutes by citation, plus topic search

Keyless. `720 ILCS 5/9-1` is first degree murder.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1686+ live data sources.

## The encoding is derivable

```
720 ILCS 5/9-1  ->  ?DocName=072000050K9-1
                             ^^^^chapter ^^^^act 0 K+section
```

Chapter and act zero-padded to four. The site navigates by internal ChapterID
and ActID, but none of that is needed — the citation carries everything.

## Why the live path was rejected twice first

Earlier probes used the OLD path (`fulltext.asp`, now 404 after a site
restructure) and a needle that does not appear in the statute. Two independent
mistakes producing the same "no", which read as "Illinois is unreachable"
rather than "you asked wrong".

## Coverage (fleet #2505)

The live site's `FullText` endpoint is unreachable from a Cloudflare Worker for
a different, unfixable reason: ilga.gov serves an incomplete TLS certificate
chain (its own leaf certificate with no intermediate). `curl` on a machine
whose OS trust store already has the missing Sectigo intermediate succeeds;
`fetch()` — Cloudflare's or plain Node's, both strict — fails outright (HTTP
526 at the gateway; `fetch failed` from Node directly). No retry fixes a
certificate chain, which is why `illinois-code` read `upstream_down` every day
for 14 straight days before this shipped.

ILGA's own "Access Denied" page for automated traffic to ilga.gov names the fix:
a public, nightly-synced file repository at `ftp.ilga.gov` explicitly meant for
bulk/automated consumption ("Automated systems should retrieve files from the
repository rather than scraping ILGA.gov"). `scripts/ingest-il-statutes.mjs`
walks it — outside Workers, where `curl` still works — and writes one JSON
bundle per CHAPTER (68 objects, not 72,000 — one per act or per section would
cost the same per-write and there is no reason to pay for it) into the shared
`pipeworx-datasets` R2 bucket at `statutes/il/<chapter>.json`.

`il_compiled_statute` tries the local bundle first. If the chapter/act has
been crawled and the section simply is not in it, that is authoritative — no
point retrying a live path that currently fails for every citation regardless
of validity — and the tool says `section_not_found` directly. If the
chapter/act has not been crawled at all, it falls back to the live fetch.

`data_as_of` on every response says whether it came from the pre-fetched text (the
crawl's `ingested_at`) or a live fetch (now).

## Topic search

`il_statute_search` answers questions no citation lookup can — "currency
exchange license fee", "concealed carry reciprocity" — via FTS5 in the shared
SEARCH_SHARD Durable Object (`workers/gateway/src/search-shard.ts`, shard name
`il-statutes`; reused, not a new Cloudflare binding — the gateway sits at its
text-binding ceiling). The FTS body is the section's heading + full text, not
just the heading: the DO's contentless design exists to avoid duplicating
opinion text at case-law's 46 GB scale, which does not apply at this corpus's
size, and a topic query's words are far more often in a section's body than in
its short heading. Hit ids resolve back to citations via
`statutes/il/_index.json`. Search hits carry a citation and heading; call
`il_compiled_statute` with the chapter/act/section from a result for the full
text.

## Miss behaviour

Live fallback: 200 with no `Sec.` heading — detected structurally, unchanged
from the original pack. Pre-fetched: the chapter/act bundle exists but the
section is not in its `sections` array.

## Refresh

`node scripts/ingest-il-statutes.mjs` — re-run periodically (Illinois's
legislative session runs annually; there is no daily-freshness need here). No
schedule is wired up: dispatch it by hand, or via `gh workflow run` if/when a
workflow is added — GitHub's own `schedule:` trigger has been unreliable since
~2026-09-09 (see CLAUDE.md), so a recurring refresh should follow the
monitor-dispatch pattern rather than a bare cron trigger.

## Data source

Illinois General Assembly (`ftp.ilga.gov` public file repository, and
`ilga.gov` for the live fallback). Illinois statutes are public record; no
reuse restriction was found on the bulk file repository.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "illinois-code": {
      "url": "https://gateway.pipeworx.io/illinois-code/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/illinois-code/mcp` returns the tools in the table
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

Both URLs reach the same gateway and the same 1686+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/il_compiled_statute \
  -H 'Content-Type: application/json' \
  -d '{"chapter":"720","act":"5","section":"9-1"}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/il_compiled_statute`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Standalone (no gateway account)

This package also runs as a local stdio MCP server — no Pipeworx account, no
gateway round-trip:

```json
{
  "mcpServers": {
    "illinois-code": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-illinois-code"]
    }
  }
}
```

Or run it directly to confirm it starts:

```bash
npx -y @pipeworx/mcp-illinois-code
```

It speaks MCP over stdin/stdout and answers `initialize`/`tools/list`/`tools/call`
for **only** this pack's tools — none of the shared meta-tools the gateway
connection above adds. Same source, same tools, no ask_pipeworx routing.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Illinois Code data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
