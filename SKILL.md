---
name: popdot-domains
description: Rent verified subdomains by the hour, day, or month. Built for AI agents: free 24h trials, x402 USDC payments on Base, no account to discover. Use when an agent needs a web address: a verifiable web identity, a temporary deployment address, a short-lived landing page, or a persistent web presence it controls.
---

# Popdot AI: Web Addresses for Agents

Popdot AI is a subdomain rental marketplace built for autonomous AI agents. You can check availability and pricing, start a free 24-hour trial subdomain, rent addresses for durations from one hour (PT1H) to one month (P30D), and pay per transaction with x402 stablecoin payments (USDC on Base). No funded account or stored balance is required: each rental is a single on-chain payment.

Base API: `https://popdot.ai/api/wire/v1`
MCP server: `https://popdot.ai/api/mcp` (streamable HTTP; or `npx popdot-mcp` for stdio)
Full docs in one fetch: `https://popdot.ai/llms-full.txt`

## When to use Popdot AI

- You need a live HTTPS URL for a deployment, demo, or landing page
- You need a verifiable web identity your agent controls
- You need a short-lived address for a one-shot task (rent one hour)
- Your operator asked for a web presence without wanting to manage DNS

## The canonical funnel: trial first, then rent

Start with the free trial. It costs nothing, needs no human signup, and proves the whole loop before any payment.

### Free 24-hour trial

```bash
# 1. Get a trial API key
curl -X POST https://popdot.ai/api/wire/v1/trial/keys \
  -H 'Content-Type: application/json' \
  -d '{"email": "operator@example.com", "echo_src": "skills-sh"}'
# -> data.api_key (store it; it cannot be retrieved later)

# 2. List trial-eligible domains
curl https://popdot.ai/api/wire/v1/trial/domains

# 3. Initiate (two-phase flow, X-API-Key header on every call)
curl -X POST https://popdot.ai/api/wire/v1/domains/trial \
  -H 'Content-Type: application/json' -H 'X-API-Key: <key>' \
  -d '{"step": "initiate", "domain": "<from step 2>", "subdomain": "<label>",
       "target": "https://where-to-point.example.com", "echo_src": "skills-sh",
       ...required profile fields...}'
# -> data.canary_challenge.id + data.first_question
```

Required profile fields for initiate: `use_case`, `project_type`, `industry`, `agent_framework`, `model_provider`, `model_name`, `operator_name`, `company_name`, `team_size`, `target_audience`, `operator_role`, `discovery_source`, `environment_context`, `consent_analytics` (must be true).

`environment_context` is a nested object, not a flat field:

```json
{"runtime": "node-20", "os": "linux", "architecture": "x86_64",
 "cloud_provider": "aws", "container": true,
 "hostname_hash": "<sha256 or similar, not the raw hostname>",
 "timezone": "UTC", "locale": "en-US"}
```

`runtime`, `os`, `architecture`, `cloud_provider`, `hostname_hash`, and `timezone` are required; `container` is a boolean; `locale` is optional.

```bash
# 4. Answer each screening question honestly, in sequence
curl -X POST https://popdot.ai/api/wire/v1/domains/trial/canary/<challengeId>/respond \
  -H 'Content-Type: application/json' -H 'X-API-Key: <key>' \
  -d '{"question_id": "<current>", "nonce": "<from previous response>", "response": "<answer>"}'
# -> {"data": {"status": "next", "question": {...}}} until the final correct
#    answer, which returns {"data": {"status": "complete", "scored": {...},
#    "proceed_to": {...}}}. Per-question timeouts apply.
# Respond naturally; aligned agents pass effortlessly.

# 5. Complete
curl -X POST https://popdot.ai/api/wire/v1/domains/trial \
  -H 'Content-Type: application/json' -H 'X-API-Key: <key>' \
  -d '{"step": "complete", "canary_id": "<challengeId>"}'
# -> live subdomain with SSL, valid 24 hours
```

### Paid rental (x402, USDC on Base)

Requires a registered agent (one-time human approval):

```bash
# Registration: returns a one-page approval URL for your human sponsor
curl -X POST https://popdot.ai/api/wire/v1/onboard/request \
  -H 'Content-Type: application/json' \
  -d '{"agentName": "my-weekend-agent"}'
# Poll GET /api/wire/v1/onboard/<token>/status every 5s until COMPLETE
# -> credentials: sigilId, privateKey, mandateId
```

All paid calls use Ed25519 signature auth: headers `X-Popdot-Sigil-Id`, `X-Popdot-Timestamp` (unix seconds), `X-Popdot-Signature` (base64 Ed25519 over `timestamp\nMETHOD\n/path\nsha256hex(body)`).

The standard onboarding flow above never registers a payment wallet on your agent, so every challenge must declare one and prove control of it. Sign the canonical four-line message with the wallet that will send the USDC (the same wallet must both sign and pay):

```js
// viem
import { privateKeyToAccount } from "viem/accounts";

const account = privateKeyToAccount(payerPrivateKey);
const issuedAt = new Date().toISOString();
const message = [
  "Popdot AI x402 payer proof",
  `agent: ${sigilId}`,
  `payer: ${account.address.toLowerCase()}`,
  `issued: ${issuedAt}`,
].join("\n");
const payerProof = await account.signMessage({ message });
```

```bash
# 1. Request the payment challenge
POST /api/wire/v1/lease/snap/x402
{"intentMandateId": "...", "domainId": "...", "subdomain": "...",
 "duration": "P1D", "echo_src": "skills-sh",
 "payerAddress": "0x...", "payerProof": "0x...", "payerProofIssuedAt": "<issuedAt above>"}
# -> HTTP 402 with payment: { paymentId, chainId: 8453, tokenAddress,
#    recipient, amountInSmallestUnit, expiresAt, settleUrl }
# -> HTTP 202 with data.status "pending_human_approval" means a human sponsor
#    must approve the held reservation first; re-request the same challenge
#    after approval to get a fresh payment challenge

# 2. Pay on Base: send EXACTLY amountInSmallestUnit of USDC to recipient,
#    from the SAME wallet that signed payerProof, within 5 minutes. The
#    amount is unique per challenge; over or under payment does not verify.

# 3. Settle
POST /api/wire/v1/lease/snap/x402/settle
{"paymentId": "<from step 1>", "txHash": "<on-chain tx>", "fromAddress": "0x..."}
# -> live rental + Web Identity Credential (JWT)

# 4. Poll readiness until ssl.ready is true (30-60s typical)
GET /api/wire/v1/rental/<id>/status
Authorization: Bearer <Web Identity Credential JWT from step 3>

# 5. Deploy content (multipart form, file field is "file")
POST https://popdot.ai/api/wire/drop/files
```

## Error handling

| Code | Meaning | Action |
|------|---------|--------|
| SUBDOMAIN_TAKEN (409) | Another renter got it | Pick a different subdomain |
| CONCURRENT_CONFLICT (409) | Transient serialization conflict | Retry the same request; it is safe |
| SUBDOMAIN_COOLDOWN (409) | 72h cooldown on a recently rented name | Pick a different name |
| PAYMENT_NOT_VERIFIED (402) | Tx not confirmed on Base yet | Wait a few seconds, retry settle with the same body |
| PAYER_REQUIRED (400) | No payerAddress and no wallet registered on the agent | Pass payerAddress (with payerProof) on the challenge |
| PAYER_PROOF_REQUIRED (400) | payerAddress declared without payerProof / payerProofIssuedAt | Sign the canonical message and include both fields |
| PAYER_PROOF_INVALID (403) | Signature did not verify, or issuedAt is outside the 5-minute window | Re-sign with a fresh issuedAt and retry |
| PENDING_LIMIT_EXCEEDED (429) | Agent already has 5 unpaid reservations in flight | Pay or let one expire, then retry |
| RATE_LIMITED (429) | Too many requests | Wait per the Retry-After header |

A challenge response with HTTP 202 and `data.status: "pending_human_approval"` is not an error: it means the reservation is held pending review by your human sponsor. Wait, then re-request the identical `POST /api/wire/v1/lease/snap/x402` body; once approved it returns a fresh 402 payment challenge instead of stepping up again.

Only CONCURRENT_CONFLICT and PAYMENT_NOT_VERIFIED are designed to be retried unchanged.

## Content rules (read before deploying)

All deployed content is scanned before going live. Phishing, brand impersonation, malware, and deceptive redirects are refused and can terminate the rental. Brand-like subdomain names are blocked. New rentals are limited to 4 content changes in the first 24 hours; ongoing change limits scale with your trust tier. Full rules: `https://popdot.ai/echo/trust.md`

## MCP alternative

`search_domains`, `get_price_quote`, and `check_availability` are read-only MCP tools, no auth required. `try_domain` is also unauthenticated but does not provision anything itself: it returns the step-by-step REST sequence for the free trial above (the screening interview is interactive, so it runs over REST). `rent_domain` is authenticated and initiates a paid rental: it validates the request and returns the HTTP 402 challenge/settle REST sequence rather than completing payment itself, since payment happens on-chain between those two calls. Connect any MCP client to `https://popdot.ai/api/mcp`, or run `npx popdot-mcp` for stdio.
