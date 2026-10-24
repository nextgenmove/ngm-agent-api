# ngm-agent-api

A public API and MCP server that lets AI agents get an indicative moving price from **Next Generation Movers** and send a free booking request.

- **Coverage:** nationwide interstate and long-distance household moves between any two US states, plus local moves in DC, Maryland and Virginia.
- **Base URL:** `https://ngm-agent-api.nextgenmovers.workers.dev`
- **MCP endpoint (streamable HTTP):** `https://ngm-agent-api.nextgenmovers.workers.dev/mcp`
- **Agent card (A2A):** `/.well-known/agent.json`
- **OpenAPI 3.1:** `/openapi.json`  ·  **llms.txt:** `/llms.txt`  ·  **Company info:** `/`
- **MCP registry:** `dev.workers.nextgenmovers.ngm-agent-api/movers`
- **Contact:** mover@mail.grokbot.com · (240) 422-2243

## Tools

| Tool | REST | Price | What it does |
| --- | --- | --- | --- |
| `get_quote` | `POST /quote` | $0.01 USDC per quote (x402 on Base) | Indicative price per our published tariff |
| `request_booking` | `POST /book-deposit` | Free | Date hold request; our team sets the deposit and sends a signed estimate |
| `company_info` | `GET /` | Free | Coverage, USDOT/MC numbers, contact |

Quotes are indicative, from the tariff. The final price comes from the signed estimate.

## Get a quote with curl

```bash
curl -i -X POST https://ngm-agent-api.nextgenmovers.workers.dev/quote \
  -H 'content-type: application/json' \
  -d '{"from_zip":"20850","to_zip":"33602","move_date":"2026-11-20","home_size":"2br","packing":"none","stairs_flights":1}'
```

Without payment the server answers **HTTP 402 Payment Required** with x402 v2 payment requirements
(network `eip155:8453`, asset USDC, amount `10000` = $0.01). An x402 client signs the payment and retries
with the payment header; the server settles and returns the quote.

```ts
import { x402Client, x402HTTPClient } from "@x402/core/client";
import { registerExactEvmScheme } from "@x402/evm/exact/client";
// register your signer with registerExactEvmScheme(client, { signer }), then use x402HTTPClient
// to POST /quote: it handles the 402, pays, and retries automatically.
```

## Use it over MCP

```json
{
  "mcpServers": {
    "next-generation-movers": {
      "url": "https://ngm-agent-api.nextgenmovers.workers.dev/mcp"
    }
  }
}
```

List tools:

```bash
curl -s -X POST https://ngm-agent-api.nextgenmovers.workers.dev/mcp \
  -H 'content-type: application/json' -H 'accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}'
```

`get_quote` over MCP is paid the same way (x402 MCP payment, $0.01 USDC on Base). `request_booking` and `company_info` are free.

## Payment note

- x402 v2, exact scheme, Base mainnet (`eip155:8453`), USDC, $0.01 per quote.
- Settled through the Coinbase CDP facilitator. Never send more than the stated amount.
- Booking requests are free. Moving services are billed separately in USD, or in USDC on request.
