# @pipeworx/allen-brain

Genes, the brain structure ontology, and in-situ hybridization expression
experiments from the Allen Institute's public Brain Atlas RMA API — where in
the brain a gene is expressed, and the canonical region hierarchy that
expression is annotated against.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1683+ live data sources.

## Tools

- `allen_search_genes(acronym?, name_contains?, entrez_id?, organism?, num_rows?, start_row?)`
  — gene symbol / name / Entrez ID → the Allen internal gene id and organism.
- `allen_structures(acronym?, name_contains?, structure_id?, graph_id?, num_rows?, start_row?)`
  — brain region name or acronym → structure id, parent, and the root-to-leaf
  `structure_id_path` that establishes containment.
- `allen_expression_datasets(gene_acronym, plane_of_section?, product_id?, include_failed?, num_rows?, start_row?)`
  — the ISH experiments (SectionDataSets) for a gene: atlas product, plane of
  section, section thickness, and the dataset id behind an Allen figure.

## Auth

Keyless.

## Data sources

- <https://api.brain-map.org/api/v2/data/query.json> — the RMA query endpoint;
  every tool here is one RMA expression against it (`model::Gene`,
  `model::Structure`, `model::SectionDataSet`).

## Traps

**Gene acronyms are case-distinct across species and that is a data fact, not a
formatting one.** `Gabra1` is the mouse gene, `GABRA1` the human one; they are
separate records with different experiments attached (14 vs 94 SectionDataSets
at time of writing). An exact-match query with the wrong casing returns a clean
empty array. Both gene tools retry case-insensitively and set `casing_note`
saying so, rather than reporting a silent zero.

**Over-filtering a SectionDataSet query empties it silently.** Adding
`products[id$eq1]` to a gene with no Mouse-Brain-ISH series returns
`total_rows: 0` with `success: true` — not an error. Product and plane filters
here are optional, and when a filtered query comes back empty the pack re-runs
it unfiltered and reports the unfiltered total, so a zero reads as "your filter
excluded everything" rather than "this gene has no expression data".

**Encode the whole `criteria` value.** RMA criteria contain `[`, `]`, `$` and
`'`. `encodeURIComponent` over the entire expression works; passing brackets
raw works from a browser but is eaten by a shell and by some HTTP clients,
which produces an empty result that looks like an API outage. (That is exactly
how this pack's first probe failed.)

**`success: true` with `msg: []` is the normal shape of "no rows".** A rejected
query is `success: false` with the error in `msg` — the pack raises on that,
so the two are never conflated.

**Structure graphs are per-species.** `graph_id` 1 is adult mouse, 10 human,
17 developing mouse. A human region is not in the mouse graph; the default is
1, and an empty result says which graph was searched.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "allen-brain": {
      "url": "https://gateway.pipeworx.io/allen-brain/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/allen-brain/mcp` returns the tools in the table
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
curl -X POST https://gateway.pipeworx.io/v1/tools/allen_search_genes \
  -H 'Content-Type: application/json' \
  -d '{"acronym":"Gabra1"}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/allen_search_genes`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Standalone (no gateway account)

This package also runs as a local stdio MCP server — no Pipeworx account, no
gateway round-trip:

```json
{
  "mcpServers": {
    "allen-brain": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-allen-brain"]
    }
  }
}
```

Or run it directly to confirm it starts:

```bash
npx -y @pipeworx/mcp-allen-brain
```

It speaks MCP over stdin/stdout and answers `initialize`/`tools/list`/`tools/call`
for **only** this pack's tools — none of the shared meta-tools the gateway
connection above adds. Same source, same tools, no ask_pipeworx routing.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Allen Brain data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
