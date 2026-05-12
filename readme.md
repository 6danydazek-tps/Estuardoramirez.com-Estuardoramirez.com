# Bankless Content MCP

Plug Bankless articles, podcast transcripts, airdrops, and profiles into any MCP-aware AI client (Claude Desktop, Cursor, Continue, ChatGPT desktop apps, custom agents).

A single Bearer token gives your AI read-only access to years of Bankless content, gated to active premium subscribers. JSON-RPC 2.0 over HTTP, nine tools, zero SDK required.

- **Read more**: https://www.bankless.com/docs/mcp
- **Endpoint**: `POST https://www.bankless.com/mcp`
- **Generate a key**: https://www.bankless.com/account#generateApiKey
- **Become a Premium member**: https://www.bankless.com/join

The Bankless Content MCP is only available to Premium members.

## Table of contents

1. [Quick start](#quick-start)
2. [Authentication](#authentication)
3. [Premium gating](#premium-gating)
4. [Endpoint and envelope](#endpoint-and-envelope)
5. [Tools](#tools)
6. [Errors](#errors)
7. [Rate limits](#rate-limits)
8. [Logging and privacy](#logging-and-privacy)
9. [Versioning](#versioning)
10. [Support](#support)

---

## Quick start

### 1. Get a key

Sign in to https://www.bankless.com/account and click **Generate** under "MCP API Key". You must be on an active premium plan. We show the token once. Store it somewhere safe; if you lose it, regenerate (which immediately revokes the old one).

A token looks like:

```
bls_mcp_AKx7...sLk
```

### 2. Wire it into your client

You can often just ask your client to install an MCP, paste them this doc, and they'll do it for you. If that's not possible, there's written docs below:

**Claude Desktop**

Claude Desktop's `claude_desktop_config.json` validator only accepts **stdio** server configs, so we wrap our HTTP endpoint with `mcp-remote` (community stdio bridge, needs Node 18+). Add to `~/Library/Application Support/Claude/claude_desktop_config.json` on macOS (or the Windows / Linux equivalent). Restart Claude Desktop afterwards.

```json
{
  "mcpServers": {
    "bankless": {
      "command": "npx",
      "args": [
        "-y",
        "mcp-remote",
        "https://www.bankless.com/mcp",
        "--header",
        "Authorization: Bearer bls_mcp_YOUR_TOKEN"
      ]
    }
  }
}
```

If the file already has other top-level keys (`preferences`, etc.), add `mcpServers` alongside them, separated by a comma. The file fails to parse silently if a comma is missing.

First launch downloads `mcp-remote`; subsequent launches are instant.

> **Newer Claude Desktop beta that also accepts `url`?** Some recent builds accept this simpler form:
>
> ```json
> {
>   "mcpServers": {
>     "bankless": {
>       "url": "https://www.bankless.com/mcp",
>       "headers": { "Authorization": "Bearer bls_mcp_YOUR_TOKEN" }
>     }
>   }
> }
> ```
>
> If Claude Desktop pops a "*not a valid MCP server configuration*" warning after restart, your build does not accept this shape; fall back to the `mcp-remote` config above.

> **Custom Connector UI**: Claude Desktop's *Settings → Connectors → Add custom connector* dialog only accepts URL + OAuth (Client ID and Secret). Bankless MCP uses static Bearer tokens, so that path is not supported today. If we add OAuth in a future release, this section will get updated.

**Cursor** (Settings → MCP → Edit Config)

The same JSON also works for **Continue**, **Windsurf**, and other HTTP-aware MCP clients (paths differ; the JSON shape does not).

```json
{
  "mcpServers": {
    "bankless": {
      "url": "https://www.bankless.com/mcp",
      "headers": { "Authorization": "Bearer bls_mcp_YOUR_TOKEN" }
    }
  }
}
```

**OpenAI Codex CLI** (`~/.codex/config.toml`)

Wraps the HTTP server with `mcp-remote` via stdio (needs Node 18+):

```toml
[mcp_servers.bankless]
command = "npx"
args = ["-y", "mcp-remote", "https://www.bankless.com/mcp", "--header", "Authorization: Bearer bls_mcp_YOUR_TOKEN"]
```

> **ChatGPT Connectors / Custom GPTs**: ChatGPT's Connectors feature (Plus, Pro, Enterprise) requires OAuth, the same limitation as Claude Desktop's *Add custom connector* dialog. It does not yet work for Bankless static Bearer tokens.

**Raw cURL** smoke test:

```bash
curl -X POST https://www.bankless.com/mcp \
  -H 'Authorization: Bearer bls_mcp_YOUR_TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}'
```

### 3. Ask anything

Restart your client. The nine tools register automatically. Try:

- *"Find the latest Bankless newsletter on restaking and summarise it."*
- *"Pull the moment Vitalik talks about rollups on the Bankless podcast."*
- *"What airdrops is Bankless tracking on Arbitrum right now?"*

---

## Authentication

Every authenticated method requires an `Authorization: Bearer <token>` header. Premium members can get their token from bankless.com/account

### Public vs authenticated methods

These four MCP protocol methods do not require auth (so your client can finish its handshake before the premium check kicks in):

- `initialize`
- `ping`
- `notifications/initialized`
- `notifications/cancelled`

Everything else (`tools/list`, `tools/call`) requires a valid Bearer + active premium.

### What fails when

| Condition | JSON-RPC code | Message |
|---|---|---|
| No `Authorization` header | `-32001` | `Missing Bearer token` |
| Hash does not match any active row | `-32001` | `Invalid API key` |
| Token's `permissions` lacks `CONTENT_MCP` | `-32001` | `Token lacks CONTENT_MCP permission` |
| Linked account currently not premium | `-32001` | `Bankless premium subscription required` |
| Daily MCP quota hit | `-32002` | `Daily MCP rate limit exceeded` |

All auth failures are surfaced as JSON-RPC error objects with **HTTP 200**, so MCP clients render them as tool errors rather than transport errors.

---


## Endpoint and envelope

```
POST https://www.bankless.com/mcp
Content-Type: application/json
Authorization: Bearer bls_mcp_<token>
```

Body: a JSON-RPC 2.0 request.

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "search_posts",
    "arguments": { "query": "restaking", "limit": 5 }
  }
}
```

Successful response:

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "content": [
      { "type": "text", "text": "{\"results\":[...],\"count\":5}" }
    ],
    "structuredContent": { "results": [ /* ... */ ], "count": 5 },
    "isError": false
  }
}
```

`result.structuredContent` is the canonical payload. `result.content[0].text` mirrors the same data as a stringified JSON blob, which is what MCP clients display in their tool-call disclosures.

---

## Tools

Nine read-only tools. The live catalog is queryable via:

```bash
curl -X POST https://www.bankless.com/mcp \
  -H 'Authorization: Bearer bls_mcp_YOUR_TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}'
```

### `search_posts`

Search Bankless articles and podcast episodes by keyword. Matches across `title`, `excerpt`, `searchTerms`, and the post body, ranked by where the match appears (title hits weighted highest, body lowest). Multi-word queries are AND'd across columns.

**Input schema**

```json
{
  "type": "object",
  "properties": {
    "query": { "type": "string", "minLength": 1, "maxLength": 200 },
    "tag":   { "type": "string", "enum": ["newsletter", "podcast", "news", "brief"] },
    "limit": { "type": "integer", "minimum": 1, "maximum": 25 }
  },
  "required": ["query"]
}
```

**Example call**

```json
{ "jsonrpc": "2.0", "id": 1, "method": "tools/call",
  "params": {
    "name": "search_posts",
    "arguments": { "query": "Ethereum staking", "tag": "newsletter", "limit": 3 }
  }
}
```

**Example `structuredContent`**

```json
{
  "results": [
    {
      "id": 12834,
      "title": "The Restaking Renaissance",
      "slug": "the-restaking-renaissance",
      "excerpt": "Why EigenLayer is the most important Ethereum primitive of 2024.",
      "type": "article",
      "featuredImage": "https://cdn.bankless.com/posts/restaking-cover.webp",
      "publishedOn": "2024-12-01 14:23:11",
      "readingTimeMinutes": 8,
      "tags": ["Ethereum", "Restaking", "Newsletter"],
      "interactionScore": 156,
      "link": "https://www.bankless.com/the-restaking-renaissance",
      "isPremiumOnly": false
    }
  ],
  "count": 1
}
```

### `get_post`

Fetch a single post by slug or id. Returns full HTML body.

**Input schema**

```json
{
  "type": "object",
  "properties": {
    "slug": { "type": "string" },
    "id":   { "type": "integer", "minimum": 1 }
  },
  "oneOf": [ { "required": ["slug"] }, { "required": ["id"] } ]
}
```

**Example call**

```json
{ "jsonrpc": "2.0", "id": 2, "method": "tools/call",
  "params": {
    "name": "get_post",
    "arguments": { "slug": "the-restaking-renaissance" }
  }
}
```

**Example `structuredContent`**

```json
{
  "id": 12834,
  "title": "The Restaking Renaissance",
  "slug": "the-restaking-renaissance",
  "excerpt": "...",
  "type": "article",
  "featuredImage": "...",
  "publishedOn": "2024-12-01 14:23:11",
  "readingTimeMinutes": 8,
  "tags": ["Ethereum", "Restaking"],
  "interactionScore": 156,
  "link": "https://www.bankless.com/the-restaking-renaissance",
  "isPremiumOnly": false,
  "html": "<p>EigenLayer's TVL surge is driving a second wave of AVS launches...</p>",
  "authors": [
    {
      "name": "David Hoffman",
      "bio": "Co-founder of Bankless.",
      "link": "https://www.bankless.com/author/david-hoffman",
      "socials": {
        "x": "TrustlessState",
        "discord": "",
        "linkedin": ""
      }
    }
  ]
}
```

### `list_recent_posts`

The newest posts, optionally filtered by tag. Metadata only (no body). Call `get_post` afterwards if you need the full text.

**Input schema**

```json
{
  "type": "object",
  "properties": {
    "tag":   { "type": "string", "enum": ["newsletter", "podcast", "news", "brief"] },
    "limit": { "type": "integer", "minimum": 1, "maximum": 25 }
  }
}
```

**Example call**

```json
{ "jsonrpc": "2.0", "id": 3, "method": "tools/call",
  "params": {
    "name": "list_recent_posts",
    "arguments": { "tag": "podcast", "limit": 5 }
  }
}
```

Response shape is identical to `search_posts`.

### `get_podcast_transcript`

Fetch the transcript for a podcast episode by slug or id. Returns the structured transcript JSON (with speaker maps and timestamps when available) plus a plaintext fallback and the AI-generated summary.

**Input schema**

```json
{
  "type": "object",
  "properties": {
    "slug": { "type": "string" },
    "id":   { "type": "integer", "minimum": 1 }
  },
  "oneOf": [ { "required": ["slug"] }, { "required": ["id"] } ]
}
```

**Example call**

```json
{ "jsonrpc": "2.0", "id": 4, "method": "tools/call",
  "params": {
    "name": "get_podcast_transcript",
    "arguments": { "slug": "vitalik-on-ethereums-endgame" }
  }
}
```

**Example `structuredContent`**

```json
{
  "postId": 2866,
  "title": "Restaking Alignment with Vitalik, Sreeram, Tim Beiko, Justin Drake, Dankrad & Jessy",
  "slug": "restaking-alignment-with-vitalik-sreeram-tim-beiko-justin-drake-dankrad-jessy",
  "available": true,
  "source": "podcastTranscripts",
  "transcription": {
    "turns": [
      { "speaker": "SPEAKER_00", "start": 0.0,    "end": 4.3,    "text": "Welcome back to Bankless..." },
      { "speaker": "SPEAKER_01", "start": 4.3,    "end": 12.4,   "text": "Today we have a very special panel on restaking..." },
      { "speaker": "SPEAKER_02", "start": 12.4,   "end": 28.1,   "text": "Thanks for having us." }
    ],
    "identification": [
      { "speaker": "SPEAKER_00", "name": "Ryan",    "confidence": 0.98 },
      { "speaker": "SPEAKER_01", "name": "David",   "confidence": 0.96 },
      { "speaker": "SPEAKER_02", "name": "Vitalik", "confidence": 0.94 }
    ]
  },
  "transcriptText": "Welcome back to Bankless... Today we have a very special panel on restaking... Thanks for having us...",
  "aiSummary": null,
  "speakerMap": "{\"SPEAKER_00\":\"Ryan\",\"SPEAKER_01\":\"David\",\"SPEAKER_02\":\"Vitalik\"}"
}
```

- `transcription.turns` is the time-sliced segment list (each turn = one continuous speaker).
- `transcription.identification` maps each `SPEAKER_XX` label to a human name with a confidence score.
- `transcriptText` is the concatenated plain-text version, easier for AI summarization.
- `speakerMap` is a redundant JSON string mapping (kept for backwards compatibility).

The fallback `source: "youtubeVideos"` returns a different shape (varies; usually just a `text` field plus an upstream provider error if the auto-caption pipeline failed). Always check `source` before assuming a structure.

When no transcript is on file at all, the response shape is:

```json
{
  "postId": 9242,
  "title": "...",
  "slug": "...",
  "available": false,
  "message": "No transcript available for this episode."
}
```

> ### ⚠ Response size warning
>
> Podcast transcripts are large. A typical 60-minute episode runs **150KB to 300KB of JSON**. Some MCP clients (Claude Desktop, ChatGPT) have per-response size caps and will truncate or refuse responses above a threshold (~200-250KB observed in testing). If you're building an agent on top of this, prefer chaining `search_transcripts` first (which returns ~200-char snippets) and only call `get_podcast_transcript` when you genuinely need the full transcript for the chosen episode.

### `search_transcripts`

Search Bankless podcast transcripts for a phrase. Returns episodes that contain every token in the query, with a short snippet of surrounding context around the first match. Useful when you want to find *when* a topic comes up without paying for fetching whole multi-megabyte transcripts.

**Input schema**

```json
{
  "type": "object",
  "properties": {
    "query": { "type": "string", "minLength": 1, "maxLength": 200 },
    "limit": { "type": "integer", "minimum": 1, "maximum": 25 }
  },
  "required": ["query"]
}
```

**Example call**

```json
{ "jsonrpc": "2.0", "id": 5, "method": "tools/call",
  "params": {
    "name": "search_transcripts",
    "arguments": { "query": "EigenLayer slashing risk", "limit": 5 }
  }
}
```

**Example `structuredContent`**

```json
{
  "results": [
    {
      "postId": 14210,
      "title": "Vitalik on Ethereum's Endgame",
      "slug": "vitalik-on-ethereums-endgame",
      "publishedOn": "2024-08-12 09:00:00",
      "link": "https://www.bankless.com/vitalik-on-ethereums-endgame",
      "snippet": "... and the concern there is operator slashing risk on long-tail AVSs. EigenLayer's design tries to bound that by requiring operators to ..."
    }
  ],
  "count": 1
}
```

### `list_airdrops`

Active airdrop opportunities from Bankless Airdrop Hunter, optionally filtered by category and network.

**Input schema**

```json
{
  "type": "object",
  "properties": {
    "category": { "type": "string", "maxLength": 64 },
    "network":  { "type": "string", "maxLength": 64 },
    "limit":    { "type": "integer", "minimum": 1, "maximum": 50 }
  }
}
```

**Example call**

```json
{ "jsonrpc": "2.0", "id": 5, "method": "tools/call",
  "params": {
    "name": "list_airdrops",
    "arguments": { "network": "arbitrum", "limit": 5 }
  }
}
```

**Example `structuredContent`**

```json
{
  "results": [
    {
      "id": 142,
      "title": "EigenLayer",
      "slug": "eigenlayer",
      "link": "https://www.bankless.com/airdrop-hunter/eigenlayer",
      "imageURL": "https://www.bankless.com/assets/cdn/airdrops/eigenlayer.png",
      "shortDescription": "Restaking marketplace for Ethereum's economic security.",
      "category": "Restaking",
      "likelihood": "High",
      "website": "https://eigenlayer.xyz",
      "twitter": "https://twitter.com/eigenlayer",
      "telegram": "",
      "sponsored": "0",
      "assetsNeeded": ["ETH", "stETH"],
      "networks": ["ethereum", "arbitrum"],
      "premium": true,
      "questsCount": 5
    }
  ],
  "count": 1
}
```

### `get_airdrop`

Single airdrop plus its quests (steps, on-chain criteria, CTA links).

**Input schema**

```json
{
  "type": "object",
  "properties": {
    "slug": { "type": "string" },
    "id":   { "type": "integer", "minimum": 1 }
  },
  "oneOf": [ { "required": ["slug"] }, { "required": ["id"] } ]
}
```

**Example `structuredContent`**

```json
{
  "id": 142,
  "title": "EigenLayer",
  "slug": "eigenlayer",
  "link": "https://www.bankless.com/airdrop-hunter/eigenlayer",
  "imageURL": "...",
  "shortDescription": "...",
  "category": "Restaking",
  "likelihood": "High",
  "website": "https://eigenlayer.xyz",
  "twitter": "https://twitter.com/eigenlayer",
  "telegram": "",
  "sponsored": "0",
  "assetsNeeded": ["ETH", "stETH"],
  "networks": ["ethereum"],
  "premium": true,
  "questsCount": 5,
  "quests": [
    {
      "id": 511,
      "title": "Deposit stETH to EigenLayer",
      "type": "Hunt",
      "difficulty": "Easy",
      "minsToComplete": 5,
      "network": "ethereum",
      "rankPoints": 50,
      "apy": null,
      "ctaTitle": "Deposit now",
      "ctaLink": "https://app.eigenlayer.xyz/restake",
      "description": "Stake stETH to start earning restaking points.",
      "assetsNeeded": [ { "symbol": "stETH", "min": "0.1" } ],
      "steps": [
        { "n": 1, "text": "Visit app.eigenlayer.xyz/restake" },
        { "n": 2, "text": "Connect your wallet" },
        { "n": 3, "text": "Deposit at least 0.1 stETH" }
      ],
      "bridge": "0",
      "onchainCompletionCriteria": [
        {
          "action": "deposit",
          "contract": "0x858646372cc42e1a627fce94aa7a7033e7cf075a",
          "network": "ethereum",
          "methodID": "0xf6fb059b",
          "tokenID": null,
          "asset": "stETH"
        }
      ]
    }
  ]
}
```

### `search_profiles`

Search Bankless profiles (people, projects, tokens) by keyword across `title`, `snippet`, `description`, and `ticker`. Optional `type` filter.

**Input schema**

```json
{
  "type": "object",
  "properties": {
    "query": { "type": "string", "minLength": 1, "maxLength": 200 },
    "type":  { "type": "string", "enum": ["project", "person", "token"] },
    "limit": { "type": "integer", "minimum": 1, "maximum": 25 }
  },
  "required": ["query"]
}
```

**Example `structuredContent`**

```json
{
  "results": [
    {
      "id": 88,
      "title": "Vitalik Buterin",
      "slug": "vitalik-buterin",
      "snippet": "Co-founder of Ethereum.",
      "type": "person",
      "iconImg": "https://cdn.bankless.com/stubs/vitalik-icon.png",
      "blockchain": null,
      "ticker": null,
      "link": "https://www.bankless.com/vitalik-buterin"
    }
  ],
  "count": 1
}
```

### `get_profile`

Full profile by slug or id. Includes related profiles.

**Input schema**

```json
{
  "type": "object",
  "properties": {
    "slug": { "type": "string" },
    "id":   { "type": "integer", "minimum": 1 }
  },
  "oneOf": [ { "required": ["slug"] }, { "required": ["id"] } ]
}
```

**Example `structuredContent`**

```json
{
  "id": 88,
  "title": "Vitalik Buterin",
  "slug": "vitalik-buterin",
  "snippet": "Co-founder of Ethereum.",
  "description": "Vitalik Buterin is the co-founder of Ethereum and the public face of its research direction...",
  "type": "person",
  "category": "Founder",
  "iconImg": "https://cdn.bankless.com/stubs/vitalik-icon.png",
  "coverImg": "https://cdn.bankless.com/stubs/vitalik-cover.png",
  "website": "https://vitalik.eth.limo",
  "twitter": "https://twitter.com/VitalikButerin",
  "discord": "",
  "ticker": null,
  "blockchain": null,
  "link": "https://www.bankless.com/vitalik-buterin",
  "relatedStubs": [
    {
      "id": 17,
      "title": "Ethereum Foundation",
      "slug": "ethereum-foundation",
      "snippet": "Non-profit stewarding Ethereum protocol research.",
      "type": "project",
      "iconImg": "...",
      "link": "https://www.bankless.com/ethereum-foundation"
    }
  ]
}
```

---

## Errors

All errors come back over **HTTP 200** inside the standard JSON-RPC `error` object so MCP clients surface them as tool errors, not transport errors.

| Code | Meaning | Typical cause |
|------|---------|---------------|
| `-32700` | Parse error | Request body was not valid JSON. |
| `-32600` | Invalid request | Missing `jsonrpc` / `method`, or wrong HTTP verb. |
| `-32601` | Method not found | Unknown JSON-RPC method or tool name. |
| `-32602` | Invalid params | Tool arguments failed schema validation. |
| `-32603` | Internal error | Retry once; report if persistent. |
| `-32001` | Auth failure | Missing / invalid Bearer, wrong scope, or premium subscription inactive. |
| `-32002` | Rate limit exceeded | Daily MCP quota hit. Resets at midnight UTC. |

Example error response:

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "error": {
    "code": -32001,
    "message": "Bankless premium subscription required",
    "data": { "httpStatus": 403 }
  }
}
```

Per-tool argument validation errors come back inside the `tools/call` result envelope with `isError: true`:

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "content": [{ "type": "text", "text": "{\"error\":\"Invalid tag: foo\"}" }],
    "structuredContent": { "error": "Invalid tag: foo" },
    "isError": true
  }
}
```