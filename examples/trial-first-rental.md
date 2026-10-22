# Trial First, Then Rent

The canonical Popdot AI funnel: prove the loop free, then pay for what lasts.

## Scenario

Your operator asked for a landing page for a weekend project. You do not know yet whether it survives the weekend, so start with the free trial and upgrade only if it does.

## Phase 1: Free trial (no payment, no signup)

```bash
# Key
curl -X POST https://popdot.ai/api/wire/v1/trial/keys \
  -H 'Content-Type: application/json' \
  -d '{"email": "operator@example.com", "echo_src": "skills-sh"}'

# Eligible domains
curl https://popdot.ai/api/wire/v1/trial/domains

# Initiate (full required-field list in SKILL.md)
curl -X POST https://popdot.ai/api/wire/v1/domains/trial \
  -H 'Content-Type: application/json' -H 'X-API-Key: popdot_trial_...' \
  -d '{
    "step": "initiate",
    "domain": "justcooked.dev",
    "subdomain": "weekend-project",
    "target": "https://my-app.example.com",
    "use_case": "Landing page for a weekend hackathon project",
    "project_type": "landing_page",
    "echo_src": "skills-sh",
    "...": "remaining profile fields"
  }'
```

Answer the screening questions honestly at `POST /domains/trial/canary/{id}/respond`, then complete:

```bash
curl -X POST https://popdot.ai/api/wire/v1/domains/trial \
  -H 'Content-Type: application/json' -H 'X-API-Key: popdot_trial_...' \
  -d '{"step": "complete", "canary_id": "ch_..."}'
```

You now have `weekend-project.justcooked.dev` live for 24 hours.

## Phase 2: The project survived. Rent it properly.

Mint your identity once with your trial key (`POST /api/wire/v1/sigil/mint`, no human step), then challenge with a payer proof. `payerAddress`, `payerProof`, and `payerProofIssuedAt` are required on every challenge; a wallet registered on your agent is metadata only and never substitutes for the proof (full detail and code sample in SKILL.md):

```bash
# Challenge: HTTP 402 with the exact USDC amount
POST /api/wire/v1/lease/snap/x402
{"intentMandateId": "im_...", "domainId": "dom_...",
 "subdomain": "weekend-project", "duration": "P7D", "echo_src": "skills-sh",
 "payerAddress": "0x...", "payerProof": "0x...", "payerProofIssuedAt": "2026-07-13T00:00:00.000Z"}

# Pay exactly amountInSmallestUnit USDC to recipient on Base, from the same
# wallet that signed payerProof, within 5 minutes

# Settle: rental provisions, credential issued
POST /api/wire/v1/lease/snap/x402/settle
{"paymentId": "pay_...", "txHash": "0x...", "fromAddress": "0x..."}
```

Note the 72-hour cooldown: the exact (domain, subdomain) pair from an ended rental is held down. Rent the paid subdomain before the trial expires, or pick a fresh name.
