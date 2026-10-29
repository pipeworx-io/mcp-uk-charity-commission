# @pipeworx/uk-charity-commission

The UK Register of Charities from the Charity Commission for England and Wales —
a registered charity's status, objects, areas of operation, reported income and
expenditure year by year, and its named trustees.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1857+ live data sources. This is an independent, unofficial integration — not affiliated with, endorsed by, or published by the upstream provider.

## Tools

- `ukcharity_search(query, limit?, registered_only?, _apiKey)` — find charities by name.
- `ukcharity_details(charity_number, subsidiary_number?, _apiKey)` — the full register entry.
- `ukcharity_financials(charity_number, subsidiary_number?, _apiKey)` — filed income/expenditure history.
- `ukcharity_trustees(charity_number, subsidiary_number?, _apiKey)` — the trustee board.

## Auth

BYO key, with `PLATFORM_CHARITY_COMMISSION_KEY` as the platform fallback once
one is provisioned. Until then the pack is `byoExpected` in
`workers/gateway/src/pack-manifest.json` and every call needs a caller-supplied
`_apiKey`; the refusal says so in those words.

Registration is free but **not instant-self-serve in one step**: sign up at
<https://api-portal.charitycommission.gov.uk/>, then subscribe to the "Charity
Commission Register of Charities" product to be issued an
`Ocp-Apim-Subscription-Key`. There is no keyless or demo path — every operation
answers `401 Access Denied` without the header.

## Data sources

- <https://api.charitycommission.gov.uk/register/api/searchCharityName/{text}>
- <https://api.charitycommission.gov.uk/register/api/allcharitydetailsV2/{number}/{subsidiary}>
- <https://api.charitycommission.gov.uk/register/api/charityfinancialhistory/{number}/{subsidiary}>
- <https://api.charitycommission.gov.uk/register/api/charitytrusteeinformation/{number}/{subsidiary}>

Things worth knowing:

- **Azure API Management 401s, never 403s**, on a missing or wrong key, and says
  so in a `WWW-Authenticate: AzureApiManagementKey` header — so a key problem is
  unambiguous and is not confusable with an entitlement lag.
- **A wrong path 404s BEFORE the key is checked**, which is how the operation
  names above were confirmed without holding a key. `charitytrustees`,
  `charityclassification`, `charitypublishedreport`, `charityotheraddresses` and
  `charityeventhistory` all 404 — they do not exist. The real names are
  `charitytrusteeinformation`, `charityareaofoperation`, `charitygoverningdocument`
  and `charityotherregulators`, all of which 401 (i.e. exist).
- **Every charity-scoped operation takes TWO path segments**: the registered
  charity number and a subsidiary number. The main charity is `0`; omitting the
  second segment 404s.
- `register-of-charities.charitycommission.gov.uk/api/search` — the public
  website's own endpoint — was returning `502 Bad Gateway` from the Azure
  Application Gateway when this pack was built, so it is not a keyless fallback.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "uk-charity-commission": {
      "url": "https://gateway.pipeworx.io/uk-charity-commission/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/uk-charity-commission/mcp` returns the tools in the table
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

Both URLs reach the same gateway and the same 1857+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/ukcharity_search \
  -H 'Content-Type: application/json' \
  -d '{"query":"oxfam","limit":5}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/ukcharity_search`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Run it from npm

`@pipeworx/mcp-uk-charity-commission` is a local stdio bridge to the Pipeworx gateway: your MCP client
starts it with `npx`, and it forwards every call to the same
`https://gateway.pipeworx.io/uk-charity-commission/mcp` endpoint shown above. It needs network access;
there is no offline mode.

```json
{
  "mcpServers": {
    "uk-charity-commission": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-uk-charity-commission"]
    }
  }
}
```

Without a key it calls anonymously, capped at 50 calls a day. A free account
raises that to 200 a day: get a key at https://pipeworx.io/signup and add
`"env": { "PIPEWORX_API_KEY": "<your key>" }` to the server entry.
The key is sent as a Bearer token, the same way a hosted connection sends it,
so calls are metered and billed exactly as they are there.

It lists **only** this pack's tools by default. Set `PIPEWORX_META_TOOLS=1` to
also list `ask_pipeworx` and the other gateway-wide tools.

Behind an HTTPS proxy: Node's built-in `fetch` does not read `HTTPS_PROXY` by
default. Newer Node releases honour it when `NODE_USE_ENV_PROXY=1` is set; check
your Node version's documentation.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Uk Charity Commission data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
