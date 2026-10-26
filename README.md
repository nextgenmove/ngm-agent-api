# Next Generation Movers: agent API (v0.6.1)

Nationwide interstate and long-distance household moves between any two US states, plus local moves in the DC/Maryland/Virginia area. Licensed carrier USDOT 3764697, MC 1341637.
Lets AI agents get an **indicative price per tariff** (not a legal estimate) and send a **free booking request**. Runs on Cloudflare Workers (free tier). `/quote` is paid ($0.01) via x402 with the Coinbase (CDP) payment service and listed in the x402 Bazaar.

## Routes
| Route | Price | What it does |
|---|---|---|
| GET / , /health , /llms.txt , /agents.json , /openapi.json , /icon.png | free | Facts, health check, docs, PNG logo |
| POST /quote | $0.01 | Any two US ZIPs. Over ~50 road miles: long distance per cubic foot. Shorter and both ends in DC/MD/VA: local hourly. Shorter elsewhere: `local_not_offered`, free, no price |
| POST /book-deposit | free | Booking request. Saves it in KV, sends an alert, returns `date_hold_pending`: "the owners set the deposit and send the signed estimate". Never names a deposit amount |

Every quote has `price_type: "indicative price per tariff"`, one total (no ranges), and a `handoff` to a free in-home or video survey and a written estimate.

## Remote MCP server
Endpoint: `https://ngm-agent-api.nextgenmovers.workers.dev/mcp` (streamable HTTP). Typical flow: `company_info` (optional), then `get_quote_free`, then `request_booking`.

| Tool | Price | When to use it |
|---|---|---|
| `get_quote_free` | free | Use first for any US household move quote. Required: `from_zip`, `to_zip`, `move_date`; size with `cubic_feet`, `inventory`, `home_size` or `bedrooms`. Optional `contact_name` plus `contact_phone` or `contact_email` sends a callback lead to the owners; leave them out for a price only. 30 free quotes per hour per IP |
| `get_quote` | $0.01 (x402, USDC on Base) | Same price and inputs as `get_quote_free`. Use only when the free tool is rate-limited or your agent pays per call. No contact fields, never creates a lead |
| `request_booking` | free | After a quote, once a date is chosen. Required: `move_date` plus `contact_email` or `contact_phone`. Returns `date_hold_pending`; the owners make contact within 24 hours to schedule a free survey. No payment taken |
| `company_info` | free | Licensing (USDOT/MC), coverage, hours and contact. No inputs, no price |

## Pricing (src/config/rates.ts; source: 2026 tariff)
Long distance: billed cf (350 cf minimum; weight converts at 6.5 lb/cf) x band tariff rate, then the published -20% adjustment = line haul. Plus fuel 15% and origin/destination 7% of line haul.

Band table (tariff $/cf, net after -20% in brackets):

| Size \ distance | up to 500 mi | 501-1,000 mi | 1,001-1,500 mi | 1,501+ mi |
|---|---|---|---|---|
| 350-600 cf | 6.75 (5.40) | 7.50 (6.00) | 8.25 (6.60) | 8.75 (7.00) |
| 601-1,200 cf | 6.25 (5.00) | 7.00 (5.60) | 7.75 (6.20) | 8.25 (6.60) |
| 1,201+ cf | 5.75 (4.60) | 6.50 (5.20) | 7.25 (5.80) | 7.75 (6.20) |

Extras: stairs $75/flight, long carry $100 per 75 ft, shuttle $1.25/cf ($700 min), extra stop $150, safe $500, storage $50 per 100 cf/month, full pack or unpack $1.25/cf, specialty items per tariff Addendum B, guaranteed delivery 42% of line haul (option). Basic valuation $0.60/lb included.
Local (DC/MD/VA only): 2 movers $229/hr, 3 movers $259/hr, 4 movers $299/hr, 3-hour minimum, 1 travel hour, fuel on labor + travel (`LOCAL.fuelRate`).

## Structure
- `src/config/brand.ts`: brand facts (public name Next Generation Movers, USDOT/MC, contacts, tags, lanes).
- `src/config/rates.ts`: every price, band table, service area, road factor.

## Design notes (from CDP docs)
- Bazaar metadata per route: input schema + example, output example, description that says when to use it, serviceName, tags, PNG iconUrl, documented errors.
- Bad input returns 400 before payment; empty body returns 402 so Bazaar indexing works.
- Booking request is free (no x402). Send `idempotency_key`; a repeat returns the same request with 409.
- MCP (`npm run mcp`): `get_quote` is charged $0.01 over MCP with `@x402/mcp` when X402_PAY_TO and CDP keys are set, free in local dev. `request_booking` is free, returns `date_hold_pending` and points to /book-deposit.

## Commands
`npm test` · `npm run dev` (test network, no money) · `npm run mcp`
