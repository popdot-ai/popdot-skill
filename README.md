# popdot-skill

Agent skill for [Popdot AI](https://popdot.ai): rent verified subdomains by the hour, day, or month. Free 24-hour trials, x402 USDC payments on Base, no account needed to discover.

Give your agent a web address it controls: a live HTTPS URL for deployments, demos, landing pages, or a persistent verifiable identity.

## Install

OpenClaw (ClawHub):

```
clawhub install popdot-domains
```

Skills.sh or any framework that installs from GitHub:

```
npx skills add popdot-ai/popdot-skill
```

Claude Code: copy `SKILL.md` into your project's `.claude/skills/popdot-domains/`.

## What the skill covers

- The free trial flow: API key to live subdomain in one sitting
- Paid rentals with x402: challenge, exact-amount USDC payment on Base, settle
- Error recovery: which codes retry safely and which need a different request
- Content rules, so deployments pass screening the first time

## Examples

- [Trial first, then rent](examples/trial-first-rental.md): the canonical funnel
- [Rent and deploy](examples/rent-and-deploy.md): search, rent, deploy content, verify SSL
- [Ephemeral identity](examples/ephemeral-identity.md): a one-hour rental for a one-shot task

## Docs

- Quickstart: https://popdot.ai/echo/quickstart.md
- Everything in one fetch: https://popdot.ai/llms-full.txt
- MCP server: https://popdot.ai/api/mcp (`npx popdot-mcp` for stdio)

## License

MIT
