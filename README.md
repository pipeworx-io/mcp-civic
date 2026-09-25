# @pipeworx/civic

Expert-curated clinical interpretation of cancer variants from CIViC — which
somatic alterations are clinically actionable, in which disease, with which
therapy, and the graded published evidence behind each claim.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1683+ live data sources.

## Tools

- `civic_search_genes(query, limit?)` — cancer genes and fusions by name, with
  variant / evidence / assertion counts and the CIViC feature id.
- `civic_gene_variants(gene, variant_name?, limit?)` — the curated variants of a
  gene, with associated diseases and therapies.
- `civic_variant_evidence(molecular_profile | variant_id, evidence_type?, disease?, therapy?, limit?)`
  — individual evidence items with type, level A-E, direction, significance,
  1-5 star rating and the PubMed/ASCO source.
- `civic_assertions(molecular_profile? | disease? | therapy?, limit?)` — the
  AMP/ASCO/CAP-tiered rollups, with FDA companion-test and regulatory-approval
  flags, ACMG codes and NCCN guideline.

## Auth

Keyless. Public GraphQL endpoint; CIViC's content is CC0.

## Data sources

- <https://civicdb.org/api/graphql> — POST, `{query, variables}`.

Things that will otherwise cost you an afternoon:

- **An explicitly null variable is a FILTER, not an absent filter — and it
  returns HTTP 200.** Measured 2026-09-17: `evidenceItems(molecularProfileName:
  "BRAF V600E")` reports 249 matches; adding `variantId: null` reports **0**,
  `diseaseName: null` reports 245, `therapyName: null` reports 313. Unset
  filters must be omitted from the variables map entirely. `omitNulls()` in
  `src/index.ts` exists for exactly this and removing it produces a silent zero.
- **`browseVariants.totalCount` and `browseFeatures.totalCount` ignore their own
  filter arguments.** `browseVariants(featureName: "BRAF")` and an unfiltered
  `browseVariants` both report 5029, the whole catalogue. `evidenceItems` and
  `assertions` DO filter their totalCount. This pack only reports the count for
  the two that mean it.
- **`description` on the `BrowseFeature` type 500s the entire query.** It is in
  the schema and it is not callable. The gene summary lives on the `Gene` type,
  which is why `civic_search_genes` makes a second call.
- **`genes` takes `entrezSymbols`/`entrezIds`, not `name`.** Free-text gene
  search is `browseFeatures(featureName:)` (substring, unranked — this pack
  re-ranks exact matches first) or `featureTypeahead(queryTerm:)`.
- Evidence and assertion `status` can be `SUBMITTED` as well as `ACCEPTED`;
  a submitted item has not been through CIViC review yet.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "civic": {
      "url": "https://gateway.pipeworx.io/civic/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/civic/mcp` returns the tools in the table
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
curl -X POST https://gateway.pipeworx.io/v1/tools/civic_search_genes \
  -H 'Content-Type: application/json' \
  -d '{"query":"BRAF","limit":5}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/civic_search_genes`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Standalone (no gateway account)

This package also runs as a local stdio MCP server — no Pipeworx account, no
gateway round-trip:

```json
{
  "mcpServers": {
    "civic": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-civic"]
    }
  }
}
```

Or run it directly to confirm it starts:

```bash
npx -y @pipeworx/mcp-civic
```

It speaks MCP over stdin/stdout and answers `initialize`/`tools/list`/`tools/call`
for **only** this pack's tools — none of the shared meta-tools the gateway
connection above adds. Same source, same tools, no ask_pipeworx routing.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Civic data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
