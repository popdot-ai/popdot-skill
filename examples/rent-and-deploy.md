# Rent and Deploy

Search, rent, deploy content, verify SSL. The full paid loop.

## 1. Find a domain

Via MCP (`search_domains`) or ask the marketplace what fits the budget:

```json
{"jsonrpc": "2.0", "id": 1, "method": "tools/call",
 "params": {"name": "search_domains",
   "arguments": {"query": "dev", "maxPrice": 5, "limit": 10}}}
```

Quote it before committing:

```json
{"jsonrpc": "2.0", "id": 2, "method": "tools/call",
 "params": {"name": "get_price_quote",
   "arguments": {"domainId": "dom_...", "duration": "P1D"}}}
```

## 2. Rent (x402)

`payerAddress`, `payerProof`, and `payerProofIssuedAt` are required on every challenge; a wallet registered on your agent is metadata only and never substitutes for the proof (sign the canonical message with the wallet that will pay; see SKILL.md for the code sample):

```bash
POST /api/wire/v1/lease/snap/x402
{"intentMandateId": "im_...", "domainId": "dom_...",
 "subdomain": "my-service", "duration": "P1D", "echo_src": "skills-sh",
 "payerAddress": "0x...", "payerProof": "0x...", "payerProofIssuedAt": "2026-07-13T00:00:00.000Z"}
# 402 -> pay exact USDC amount on Base, from the same wallet that signed
# payerProof -> settle with tx hash
POST /api/wire/v1/lease/snap/x402/settle
{"paymentId": "pay_...", "txHash": "0x...", "fromAddress": "0x..."}
```

## 3. Deploy content

```bash
curl -X POST https://popdot.ai/api/wire/drop/files \
  -H 'X-Popdot-Sigil-Id: ...' -H 'X-Popdot-Timestamp: ...' -H 'X-Popdot-Signature: ...' \
  -F 'rentalId=rent_...' -F 'file=@index.html'
```

Content is scanned before going live (see trust rules). The rental clock starts at first deploy, not checkout: an undeployed rental does not burn time.

## 4. Verify SSL before announcing the URL

```bash
GET /api/wire/v1/rental/rent_.../status
Authorization: Bearer <Web Identity Credential JWT from the settle response>
# Poll every few seconds until data.ssl.ready is true (30-60s typical)
```

Handle `SUBDOMAIN_TAKEN` by picking another name; handle `CONCURRENT_CONFLICT` by retrying the identical body with a fresh signature.
