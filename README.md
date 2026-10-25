# Next Generation Movers: agent API (v0.4)

Nationwide interstate and long-distance household moves between any two US states, plus local moves in the DC/Maryland/Virginia area. Licensed carrier USDOT 3764697, MC 1341637.
Lets AI agents get an **indicative price per tariff** (not a legal estimate) and send a **free booking request**. Runs on Cloudflare Workers (free tier). `/quote` is paid ($0.01) via x402 with the Coinbase (CDP) payment service and listed in the x402 Bazaar.

## Routes
| Route | Price | What it does |
|---|---|---|
| GET / , /health , /llms.txt , /agents.json , /openapi.json , /icon.png | free | Facts, health check, docs, PNG logo |
| POST /quote | $0.01 | Any two US ZIPs. Over ~50 road miles: long distance per cubic foot. Shorter and both ends in DC/MD/VA: local hourly. Shorter elsewhere: `local_not_offered`, free, no price |
| POST /book-deposit | free | Booking request. Saves it in KV, sends an alert, returns `date_hold_pending`: "the owners set the deposit and send the signed estimate". Never names a deposit amount |

Every quote has `price_type: "indicative price per tariff"`, one total (no ranges), and a `handoff` to a free in-home or video survey and a written estimate.

## Pricing (src/config/rates.ts; source: 2026 tariff + NGM adjustment log)
Long distance: billed cf (350 cf minimum; weight converts at 6.5 lb/cf) x band tariff rate, then the published -20% adjustment = line haul. Plus fuel 15% and origin/destination 7% of line haul.

Band table (tariff $/cf, net after -20% in brackets). FOUNDER REVIEW:

| Size \ distance | up to 500 mi | 501-1,000 mi | 1,001-1,500 mi | 1,501+ mi |
|---|---|---|---|---|
| 350-600 cf | 6.75 (5.40) | 7.50 (6.00) | 8.25 (6.60) | 8.75 (7.00) |
| 601-1,200 cf | 6.25 (5.00) | 7.00 (5.60) | 7.75 (6.20) | 8.25 (6.60) |
| 1,201+ cf | 5.75 (4.60) | 6.50 (5.20) | 7.25 (5.80) | 7.75 (6.20) |

Extras: stairs $75/flight, long carry $100 per 75 ft, shuttle $1.25/cf ($700 min), extra stop $150, safe $500, storage $50 per 100 cf/month, full pack or unpack $1.25/cf, specialty items per tariff Addendum B, guaranteed delivery 42% of line haul (option). Basic valuation $0.60/lb included.
Local (DC/MD/VA only): 2 movers $229/hr, 3 movers $259/hr, 4 movers $299/hr, 3-hour minimum, 1 travel hour, fuel on labor + travel (`LOCAL.fuelRate`, FOUNDER REVIEW).
`RATES_APPROVED = false` makes quotes say `rates_status: "pending_publication"`. Set it to true once Garcia publishes the adjustment log with an effective date (NGM_Tariff_Adjustment_Log.pdf).

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

## Go-live checklist (in order)

F = founder does it (accounts, keys, money). A = agent does it (code and commands, after F gives values).

1. **F** Review every `FOUNDER REVIEW` in `src/config/rates.ts` and send NGM_Tariff_Adjustment_Log.pdf to Garcia. Once it is published with an effective date, **A** sets `RATES_APPROVED = true` and runs `npm test`.
2. **F** Create a Cloudflare account and run `npx wrangler login` on the machine that deploys.
3. **F** Create a CDP account at portal.cdp.coinbase.com and a Secret API key. Keep the ID and secret private.
4. **F** Choose the receiving address on Base. Give **A** the public address only.
5. **A** Make the KV namespace and paste its id into `wrangler.toml`:
   `npx wrangler kv namespace create HOLDS`
6. **A** Set `X402_PAY_TO` in `wrangler.toml`. Keep `X402_NETWORK = "eip155:84532"` (Base Sepolia) for the test phase.
7. **F** Add the secrets (you paste them; the agent never sees them):
   `npx wrangler secret put CDP_API_KEY_ID`
   `npx wrangler secret put CDP_API_KEY_SECRET`
   Optional: `npx wrangler secret put ALERT_WEBHOOK_URL`
8. **A** Run the checks and deploy:
   `npm test && npx wrangler deploy --dry-run --outdir dist && npx wrangler deploy`
9. **A** Set `PUBLIC_BASE_URL` to the deployed URL in `wrangler.toml`, then run `npx wrangler deploy` again so `iconUrl` is published.
10. **A** Smoke test. Free: `curl -s $URL/health`. Unpaid quote must return 402 with a `PAYMENT-REQUIRED` header:
    `curl -si -X POST $URL/quote -H 'content-type: application/json' -d '{"from_zip":"10001","to_zip":"29401","move_date":"2026-11-10","home_size":"2br"}' | head -20`
11. **A** Validate Bazaar config (read-only, no key, no payment):
    `curl -s -X POST https://api.cdp.coinbase.com/platform/v2/x402/validate -H 'content-type: application/json' -d '{"resource":"'$URL'/quote","method":"POST"}'`
12. **F** Make one paid test call on Sepolia with test funds. Check that the `EXTENSION-RESPONSES` header shows bazaar accepted.
13. **F** Approve the switch to mainnet. **A** sets `X402_NETWORK = "eip155:8453"` and runs `npx wrangler deploy`.
14. **F** Make one real $0.01 paid `/quote` call. The listing appears in Bazaar within about 15 minutes. **A** confirms it with the Coinbase connector, read-only.
15. **A** Re-run step 11 against mainnet. Then monitor `/health` (curation needs 99% uptime).
