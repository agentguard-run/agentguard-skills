# AgentGuard Skills

A collection of [Agent Skills](https://agentskills.io/home) for [AgentGuard Spend](https://agentguard.run) — local-runtime spend caps, capability-gated model routing, and Ed25519-signed audit receipts for AI agents.

## Installing

These skills work with any agent that supports the Agent Skills standard, including Claude Code, Cursor, OpenCode, OpenAI Codex, Gemini CLI, Windsurf, and Pi.

### Claude Code

```
/plugin marketplace add agentguard-run/agentguard-skills
/plugin install agentguard@agentguard
```

### Cursor

Add via **Settings → Rules → Add Rule → Remote Rule (GitHub)** with `agentguard-run/agentguard-skills`.

### OpenCode

```bash
git clone https://github.com/agentguard-run/agentguard-skills.git /tmp/agentguard-skills
cp -r /tmp/agentguard-skills/skills/* ~/.config/opencode/skills/
rm -rf /tmp/agentguard-skills
```

### OpenAI Codex / Codex CLI

Codex CLI auto-discovers Agent Skills from any GitHub repo. Two install paths:

**Option A — via GitHub CLI extension (recommended)**

```bash
# Install the gh-skill extension once (if not already installed)
gh extension install github/gh-skill

# Then install the marketplace
gh skill install agentguard-run/agentguard-skills
```

**Option B — direct copy (no extensions needed)**

```bash
git clone https://github.com/agentguard-run/agentguard-skills.git /tmp/agentguard-skills
mkdir -p ~/.codex/skills
cp -r /tmp/agentguard-skills/skills/* ~/.codex/skills/
rm -rf /tmp/agentguard-skills
```

Verify with `codex skills list`.

### Gemini CLI / Windsurf / Pi

Same `gh skill install` flow as Codex above (or direct copy to the tool's skills directory: `~/.gemini-cli/skills/`, `~/.codeium/windsurf/skills/`, etc.).

### GitHub CLI (`gh skill`) — universal

Works with Claude Code, Cursor, OpenCode, Codex, Gemini CLI, Windsurf, and any tool that supports the Agent Skills standard. Requires [GitHub CLI](https://cli.github.com/) v2.90.0+ and the `gh-skill` extension.

```bash
# One-time extension install
gh extension install github/gh-skill

# Install the full marketplace
gh skill install agentguard-run/agentguard-skills
```

#### Installing a single vertical skill

```bash
gh skill install agentguard-run/agentguard-skills/agentguard-spend-law-firm
```

## Skills

### `agentguard-spend` (general purpose)

The horizontal skill. Wraps OpenAI / Anthropic / Bedrock / OpenRouter clients with spend caps + signed audit receipts. Use when you don't fit a specific vertical yet, or when building a custom policy from scratch.

[Read the skill →](skills/agentguard-spend/SKILL.md)

### Vertical skills

Pre-configured for specific industries with vertical-aware task templates, compliance posture, and cap structures. Each one is **editable** — install it, then customize the policy.yaml to match your actual business.

| Skill | For | Compliance focus |
|-------|-----|-----------------|
| [`agentguard-spend-law-firm`](skills/agentguard-spend-law-firm/SKILL.md) | Law firms, solo practices, in-house counsel | Attorney-client privilege, ABA Rule 1.1 / 5.3, per-matter accounting |
| [`agentguard-spend-healthcare`](skills/agentguard-spend-healthcare/SKILL.md) | Medical practices, dental, telehealth, healthcare platforms | HIPAA Security Rule §164.312, BAA-required model whitelisting |
| [`agentguard-spend-ecommerce`](skills/agentguard-spend-ecommerce/SKILL.md) | Shopify stores, DTC brands, marketplaces, headless commerce | PCI-DSS, Visa CE 3.0 chargeback evidence |
| [`agentguard-spend-accounting`](skills/agentguard-spend-accounting/SKILL.md) | CPA firms, bookkeeping practices, fractional CFO, finance teams | SOX, PCAOB AS 1215, 7-year retention |
| [`agentguard-spend-real-estate`](skills/agentguard-spend-real-estate/SKILL.md) | Brokerages, mortgage brokers, property management, title companies | Fair Housing Act, state real-estate commission audits |

Each vertical skill includes:

- **Pre-mapped task → model recommendations** (e.g., haiku for contract redlining, opus for brief drafting)
- **Pre-configured cap structure** with sensible defaults for the vertical
- **Compliance-aware capability gates** (PHI requires `data_write`, payment actions require `payment_initiate`, etc.)
- **Per-transaction / per-matter / per-engagement scoping** for client billing and audit trails
- **Customization instructions** — fork your local copy, edit cap values to match your real business, AgentGuard reads from YOUR file
- **Projected savings math** specific to that vertical's typical usage patterns

## How vertical skills work

1. **Install** the marketplace via `/plugin marketplace add agentguard-run/agentguard-skills`
2. **Pick the skill** matching your business via `/plugin install agentguard@agentguard`
3. **The skill walks you through** init: cap values, model assignments, capability tiers customized for your vertical
4. **The skill writes** a `policy.yaml` to `~/.agentguard/` that you can edit
5. **You customize** the policy.yaml to match your actual business — change cap values, swap models, add scopes
6. **AgentGuard reads YOUR customized policy** on every wrapped client call

Skills are **editable**. They're a starting point shaped by industry knowledge, not a black box.

## Customizing a vertical skill for your specific business

After install, the skill files are at:
- Claude Code: `~/.claude/skills/agentguard-run/agentguard-skills/`
- OpenCode: `~/.config/opencode/skills/`
- Cursor: managed via the Remote Rule UI

Edit the `policy.yaml` template the skill ships with. Common customizations:

```yaml
# Lower the daily soft cap for your size of business
caps:
  - amountCents: 1500     # $15/day instead of $30
    window: per_day
    action: downgrade
    downgradeTo: anthropic/claude-haiku-4-5

# Add per-staff caps on top of the per-transaction ones
  - amountCents: 200000   # $2,000/month per staff member
    window: per_month
    action: block
```

Your edits stay. The skill's recommendations are guidance, not enforcement.

## Don't see your vertical?

The general [`agentguard-spend`](skills/agentguard-spend/SKILL.md) skill works for any business. Build a custom policy from there.

Or open an issue requesting a vertical: [github.com/agentguard-run/agentguard-skills/issues](https://github.com/agentguard-run/agentguard-skills/issues).

Verticals on the roadmap (request via issue):
- Insurance (claims processing, underwriting assistance)
- Marketing agencies (content generation at scale, brand-voice gates)
- Software dev teams (PR review, code search, test generation)
- Education / tutoring (adaptive content, plagiarism detection)
- Local services (barbershops, salons, gyms, dental — appointment-based businesses)
- Restaurants (menu engineering, customer service, supplier comms)

## About AgentGuard

AgentGuard Spend is a zero-data-plane SDK that lets enterprises enforce per-task spend caps, capability gates, and cryptographically signed audit receipts on every AI agent call. Customers' prompts, API keys, and signing keys never leave their runtime.

- Website: [https://agentguard.run](https://agentguard.run)
- OpenRouter quickstart: [https://agentguard.run/docs/openrouter-quickstart](https://agentguard.run/docs/openrouter-quickstart)
- Receipt verifier (any browser): [https://agentguard.run/verify](https://agentguard.run/verify)
- Pricing: [https://agentguard.run/pricing](https://agentguard.run/pricing)
- npm: [@agentguard-run/spend](https://www.npmjs.com/package/@agentguard-run/spend)
- PyPI: [agentguard-spend](https://pypi.org/project/agentguard-spend/)

Powered by [@OpenRouter](https://openrouter.ai) for unified model access across 400+ models and 60+ providers.

## License

Skills published here are licensed Apache 2.0 for skill content. The `@agentguard-run/spend` SDK itself ships under an alpha license: free for production deployments under 10,000 enforcement calls per calendar month, commercial license required above. See [agentguard.run/pricing](https://agentguard.run/pricing) for details.
