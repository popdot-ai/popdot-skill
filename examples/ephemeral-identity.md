# Ephemeral Identity

A one-hour rental (PT1H) for a one-shot task. Cheapest possible web presence.

## When this fits

- A verification flow needs a URL you control for the next few minutes
- A demo link that should die after the meeting
- A short-lived webhook receiver or redirect with a clean domain

## Flow

```bash
# Quote the hour first (prices are per-domain)
POST https://popdot.ai/api/mcp
{"jsonrpc": "2.0", "id": 1, "method": "tools/call",
 "params": {"name": "get_price_quote",
   "arguments": {"domainId": "dom_...", "duration": "PT1H"}}}

# Challenge for one hour. Every challenge declares the paying wallet, so
# payerAddress + payerProof + payerProofIssuedAt are always required
# (sign the canonical message with the paying wallet; see SKILL.md)
POST /api/wire/v1/lease/snap/x402
{"intentMandateId": "im_...", "domainId": "dom_...",
 "subdomain": "verify-8f3a", "duration": "PT1H", "echo_src": "skills-sh",
 "payerAddress": "0x...", "payerProof": "0x...", "payerProofIssuedAt": "2026-07-13T00:00:00.000Z"}

# Pay the exact USDC amount on Base from the same wallet that signed
# payerProof, settle with the tx hash
POST /api/wire/v1/lease/snap/x402/settle
{"paymentId": "pay_...", "txHash": "0x...", "fromAddress": "0x..."}

# Deploy a redirect or a single HTML file, use the URL, walk away
```

## Notes for short rentals

- The clock starts at first content deploy, so deploy when you are ready to use it, not at checkout.
- Pick a random-suffixed subdomain (`verify-8f3a`) to avoid `SUBDOMAIN_TAKEN` collisions and cooldowns on reuse.
- The Web Identity Credential (JWT) issued at settle proves your agent controls the address for the rental window. Attach it where the counterparty needs proof.
