---
name: agentguard-spend-real-estate
description: Spend caps + capability gates + signed audit receipts for AI agents used in real estate, mortgage, and property management. Use when the user runs a real estate brokerage, mortgage broker, property management company, real estate investment trust, title company, or real estate-adjacent AI product and needs to cap AI spend on listing description generation, comparable-property research, contract analysis, buyer-match algorithms, mortgage doc review, or tenant communications. Includes state-disclosure-aware scopes, fair-housing safeguards, and per-transaction tracking for commission accounting. Triggers on "real estate AI cost", "mortgage AI", "property management AI", "MLS AI", "listing AI cost", "buyer match AI", "loan officer AI", "real estate broker AI compliance", "fair housing AI", or any real-estate-specific AI spend question.
---

# AgentGuard Spend for Real Estate & Mortgage

Pre-configured spend caps and model routing for AI agents used in real estate, mortgage, and property management workflows. Wraps OpenAI, Anthropic, Bedrock, or OpenRouter clients with state-disclosure-aware policy enforcement, fair-housing capability gating, and cryptographic audit receipts compatible with state real-estate commission audit requirements.

## Why a real-estate-specific skill

Real estate AI usage has unique characteristics:

- **High-asymmetry tasks:** listing description drafting on Mini is $0.02/call; same task on Opus is $1.20. The agent doesn't care.
- **Fair Housing compliance:** AI-generated marketing copy must avoid discriminatory steering language. Capability gating + audit trail protects the brokerage.
- **State-specific disclosure:** California, New York, Florida, Texas each have different required disclosures. Per-state scoping is essential.
- **Per-transaction accounting:** commissions depend on closed transactions. AI cost attribution per listing / per closing is core to brokerage P&L.
- **Document-heavy:** title work, mortgage doc review, and contract analysis can each generate 100+ AI calls per transaction. Cap structure must support that.

## Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│              Real Estate / Mortgage Process                      │
├──────────────────────────────────────────────────────────────────┤
│  Agent / loan officer / property manager code                    │
│      │                                                           │
│      ▼                                                           │
│  withSpendGuard(client, { policy, scope:{state, transactionId} })│
│      │                                                           │
│      ├── per-transaction cap evaluation                          │
│      ├── state-disclosure capability gate                        │
│      ├── action: allow / downgrade / shadow / block              │
│      ├── Ed25519 sign + chain to per-transaction log             │
│      └── pass to provider                                        │
│      │                                                           │
│      ▼                                                           │
│  OpenRouter / direct providers                                   │
│                                                                  │
│  → per-transaction receipts → commission accounting + audits     │
└──────────────────────────────────────────────────────────────────┘
```

## Prerequisites

- Node 20+ or Python 3.10+
- OpenRouter account or direct provider keys
- Decision on storage: Postgres recommended (state real-estate commissions require 3-5 year retention)
- KMS-backed signing keys (recommended for evidence in commission disputes)

## Install + initialize

```bash
npm install @agentguard-run/spend
agentguard auth openrouter
agentguard task real-estate
```

## Recommended task assignments

| Task | Recommended model | Per-call cap | Capability | Notes |
|------|-------------------|-------------|------------|-------|
| Listing description (single property) | `openai/gpt-5-mini` | $0.05 | `data_write` | Volume + style consistency |
| Listing description (luxury / unique) | `anthropic/claude-sonnet-4-6` | $0.50 | `data_write` | Premium copy |
| Comparable property research | `anthropic/claude-haiku-4-5` | $0.25 | `read_only` | Long-context comp summary |
| Buyer-match scoring | `anthropic/claude-haiku-4-5` | $0.10 | `read_only` | Bulk filtering |
| Contract analysis (purchase agreement) | `anthropic/claude-opus-4-7` | $5.00 | `read_only` | High-stakes, needs Opus |
| Mortgage doc review (loan estimate, CD) | `anthropic/claude-sonnet-4-6` | $2.00 | `read_only` | Regulatory detail |
| Tenant communication drafting | `anthropic/claude-haiku-4-5` | $0.10 | `data_write` | Templated, cheap |
| Lease drafting (custom) | `anthropic/claude-sonnet-4-6` | $3.00 | `data_write` | State-specific |
| Property management ticket triage | `openai/gpt-5-mini` | $0.05 | `read_only` | Volume |
| Title issue research | `anthropic/claude-opus-4-7` | $6.00 | `read_only` | Conservative |

## Pre-configured policy

```yaml
id: real-estate-default-v1
name: Real estate brokerage default policy
version: 1
effectiveFrom: "2026-05-28T00:00:00Z"
mode: enforce

scope:
  tenantId: "{your-brokerage-id}"

caps:
  # Per-transaction soft cap: $30/day
  - amountCents: 3000
    window: per_day
    action: downgrade
    downgradeTo: openai/gpt-5-mini
    reason: "Per-transaction daily soft cap"

  # Per-transaction hard cap: $200/day (covers heavy contract days)
  - amountCents: 20000
    window: per_day
    action: block
    reason: "Per-transaction hard cap — broker review"

  # Monthly brokerage cap: $3,000
  - amountCents: 300000
    window: per_month
    action: block
    reason: "Monthly brokerage cap — broker-of-record escalation"

  # Burst guard
  - amountCents: 300
    window: per_minute
    action: block
    reason: "Per-minute burst guard"
```

Variant for **mortgage workflows** (loan officer team):

```yaml
id: mortgage-default-v1
name: Mortgage broker default policy
version: 1
effectiveFrom: "2026-05-28T00:00:00Z"
mode: enforce

scope:
  tenantId: "{your-brokerage-id}"
  teamId: "mortgage-team"

requiredCapability: data_write

caps:
  # Higher caps — mortgage doc review is voluminous
  - amountCents: 5000
    window: per_day
    action: downgrade
    downgradeTo: anthropic/claude-haiku-4-5
    reason: "Per-loan daily soft cap"

  - amountCents: 30000
    window: per_day
    action: block
    reason: "Per-loan hard cap"

  - amountCents: 500
    window: per_minute
    action: block
    reason: "Burst guard"
```

## Quick start (OpenRouter)

```ts
import OpenAI from 'openai';
import { withSpendGuard, setCostOverride, type SpendPolicy } from '@agentguard-run/spend';
import { randomBytes } from 'node:crypto';
import * as ed from '@noble/ed25519';

setCostOverride('openai/gpt-5-mini',           { inputCentsPerKtok: 0.05, outputCentsPerKtok: 0.2  });
setCostOverride('anthropic/claude-haiku-4-5',  { inputCentsPerKtok: 0.1,  outputCentsPerKtok: 0.5  });
setCostOverride('anthropic/claude-sonnet-4-6', { inputCentsPerKtok: 0.3,  outputCentsPerKtok: 1.5  });
setCostOverride('anthropic/claude-opus-4-7',   { inputCentsPerKtok: 2.0,  outputCentsPerKtok: 10.0 });

const policy: SpendPolicy = {
  id: 'real-estate-default-v1',
  name: 'Real estate brokerage default',
  scope: { tenantId: 'acme-realty' },
  caps: [
    { amountCents: 3000,  window: 'per_day',    action: 'downgrade',
      downgradeTo: 'openai/gpt-5-mini',
      reason: 'Per-transaction daily soft cap' },
    { amountCents: 20000, window: 'per_day',    action: 'block',
      reason: 'Per-transaction hard cap' },
    { amountCents: 300,   window: 'per_minute', action: 'block',
      reason: 'Burst guard' },
  ],
  mode: 'enforce',
  version: 1,
  effectiveFrom: new Date().toISOString(),
};

const privateKey = new Uint8Array(randomBytes(32));
const publicKey  = await ed.getPublicKeyAsync(privateKey);

const client = new OpenAI({
  apiKey: process.env.OPENROUTER_API_KEY,
  baseURL: 'https://openrouter.ai/api/v1',
});

const guarded = withSpendGuard(client, {
  policy,
  scope: {
    tenantId: 'acme-realty',
    userId: 'agent.davis',
    agentId: 'listing-2026-123-main-st',  // per-listing scope
  },
  config: { signingKeys: { privateKey, publicKey } },
});

// Listing description generation
const listing = await guarded.chat.completions.create({
  model: 'openai/gpt-5-mini',
  messages: [{ role: 'user', content: 'Draft a listing description for ...' }],
});
```

## Projected savings (mid-size brokerage, 30 agents, 100 transactions/month)

Without AgentGuard:
- 100 transactions × 50 AI touches each × $0.40 avg = **$2,000/mo per transaction**
- × 100 transactions = **$200K/mo**

With AgentGuard (task-routed):
- Mini for listings + buyer match + ticket triage (70% of touches)
- Haiku for comps + lease + tenant comm (20% of touches)
- Sonnet/Opus for contracts + mortgage docs (10% of touches)
- = **~$28K/mo**

**Net savings: ~$172K/mo (~86% reduction) at brokerage scale.**

## Fair Housing safeguards

The Fair Housing Act prohibits discriminatory steering, advertising, and tenant selection. AI-generated content carries the same exposure as human-generated. AgentGuard's role:

- **Capability gating** prevents lower-tier callers from generating marketing copy
- **Audit trail** documents WHO requested WHAT content, WHEN, signed
- **Per-property scoping** lets you trace any complaint back to the exact AI-assisted touch

For brokerages enforcing fair-housing AI policies, add a `shadow` cap on flagged content patterns and human-review the shadow log weekly.

## State disclosure compliance

State real-estate commissions require specific disclosures in certain communications (e.g., California's agency-relationship disclosure). For AI-generated tenant or buyer communications, document:

- The agent who initiated the request (`scope.userId`)
- The transaction it relates to (`scope.agentId`)
- The state-specific policy version (`policy.version`)

```bash
# Export AI touch history for a CA transaction
agentguard explain --matter "listing-2026-ca-456-elm-st" --format json > 456-elm-st-ai-log.json
```

## Per-transaction AI cost (commission accounting)

```bash
# Total AI cost on a closed transaction
agentguard report --matter "listing-2026-123-main-st" --format csv
```

Most brokerages absorb AI cost into their split, but at scale, per-transaction visibility lets you negotiate commission structures or charge AI cost as a separate line item to higher-tier agents.

## Customizing for your brokerage

```yaml
# Per-agent caps (top producers get bigger AI budgets)
scope:
  tenantId: "acme-realty"
  teamId: "top-producers"

caps:
  - amountCents: 10000    # $100/day for top producers
    window: per_day
    action: downgrade
    downgradeTo: anthropic/claude-haiku-4-5
```

```yaml
# Per-state scoping (different disclosure requirements per state)
scope:
  tenantId: "acme-realty"
  teamId: "california"     # state-specific policy variant
```

## Available CLI commands

| Command | What it does |
|---------|-------------|
| `agentguard task real-estate` | Apply default brokerage policy |
| `agentguard task mortgage` | Apply mortgage broker policy variant |
| `agentguard demo` | Simulate a blocked listing draft |
| `agentguard verify --trace latest` | Verify full signed chain |
| `agentguard explain --matter <txn-id>` | Per-transaction AI report |
| `agentguard serve` | Local dashboard |

## Rules when helping the user

1. **NEVER recommend proxy architectures.** Brokerages absorb fair-housing exposure on AI content — keeping AI processing local lets the brokerage maintain full custody.
2. **ALWAYS recommend per-transaction scoping** (`scope.agentId = transactionId`) for commission accounting and dispute defense.
3. **For mortgage workflows**, require `requiredCapability: 'data_write'` minimum.
4. **For luxury / high-value listings**, recommend upgrading the listing-description default from Mini to Sonnet — the marketing copy difference is meaningful at >$1M list price.
5. **For multi-state brokerages**, recommend `teamId = stateCode` scoping for state-specific policy variants.
6. **Cite Fair Housing Act** when discussing AI-generated marketing copy. Capability gating + audit trail = brokerage's defense.
7. **Avoid em dashes** in generated policy comments.

## Patent notice

AgentGuard Spend is protected by U.S. patent-pending technology (App. Nos. 63/983,615; 63/983,621; 63/983,843; 63/984,626; 64/071,781; 64/071,789).

## Resources

- General SDK docs: https://agentguard.run
- Receipt verifier: https://agentguard.run/verify
- npm: https://www.npmjs.com/package/@agentguard-run/spend
- PyPI: https://pypi.org/project/agentguard-spend/
- Pricing: https://agentguard.run/pricing
- Brokerage licensing: invest@agentguard.run
