---
name: agentguard-spend-real-estate
description: Spend caps + capability gates + signed audit receipts for AI agents used in real estate, mortgage, and property management. Use when the user runs a real estate brokerage, mortgage broker, property management company, real estate investment trust, title company, or real estate-adjacent AI product and needs to cap AI spend on listing description generation, comparable-property research, contract analysis, buyer-match algorithms, mortgage doc review, or tenant communications. Includes state-disclosure-aware scopes, fair-housing safeguards, and per-transaction tracking for commission accounting. Triggers on "real estate AI cost", "mortgage AI", "property management AI", "MLS AI", "listing AI cost", "buyer match AI", "loan officer AI", "real estate broker AI compliance", "fair housing AI", or any real-estate-specific AI spend question.
---

# AgentGuard Spend for Real Estate & Mortgage

Pre-configured spend caps and model routing for AI agents used in real estate, mortgage, and property management workflows. Wraps OpenAI, Anthropic, Bedrock, or OpenRouter clients with state-disclosure-aware policy enforcement, fair-housing capability gating, and cryptographic audit receipts compatible with state real-estate commission audit requirements.

## AgentGuard Advisor: AI transaction advisor for real estate and mortgage

Run `agentguard advisor` in your terminal. The AgentGuard Advisor walks you through configuring outcome-based AI governance customized for your vertical: defines the outcomes you produce, recommends OpenRouter models per outcome, sets per transaction budgets, and writes a working policy in 90 seconds.

**The four-pillar promise:** AgentGuard proves what your AI agent **attempted**, who **authorized** it, what it **cost**, and whether it **succeeded**. One cryptographic signed receipt per outcome.

**Governance Posture** (new in v0.4.1): pick `velocity`, `standard`, or `compliance` to shape policy defaults. This vertical defaults to **standard** posture.

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


## The 5 outcomes real estate brokers pay AI for

| Outcome | Human-billable equivalent | Per-outcome cap | Primary model | Fallback | Capability |
|---|---|---|---|---|---|
| `listing_prepped` | $50-$200 (marketing time) | **$1.00** | `anthropic/claude-haiku-4.5` | `openai/gpt-5.4-mini` | `data_write` |
| `lead_qualified` | $30-$120 (ISA / admin) | **$1.50** | `anthropic/claude-haiku-4.5` | `openai/gpt-5.4-mini` | `data_write` |
| `cma_delivered` | $30-$80 | **$1.00** | `anthropic/claude-sonnet-4.6` | `anthropic/claude-haiku-4.5` | `read_only` |
| `showing_booked` | $20-$40 | **$0.75** | `anthropic/claude-haiku-4.5` | `openai/gpt-5.4-mini` | `data_write` |
| `transaction_file_closed` | $200-$800 (TC time) | **$5.00** | `openai/gpt-5.5` | `anthropic/claude-sonnet-4.6` | `data_write` |

## Top 5 pain points broker/owners cite in 2026

1. **Fair Housing exposure from AI tenant screening.** HUD's May 2, 2024 guidance: housing providers *"may be vicariously liable"* for screening AI even when outsourced. Source: consumerfinancialserviceslawmonitor.com/2024/05/hud-issues-guidance-on-applicability-of-the-fair-housing-act-to-tenant-screening-and-housing-related-advertising-that-relies-upon-algorithms-and-ai/. AgentGuard fit: signed receipts log every input used in screening, providing FHA defensibility evidence.
2. **Listing-description copyright + MLS rule risk.** Realtor.com / MLS rules require accuracy; AI-generated marketing copy can over-claim. AgentGuard fit: receipts attach the listing-data source.
3. **TCPA exposure from AI SMS follow-up.** Lofty AOS, Structurely, Perspective AI all send AI-driven SMS; TCPA penalties run $500-$1,500 per violation. AgentGuard fit: consent-event ID per outbound message.
4. **Tool sprawl + licensing.** Metricus 2026 reports CRM-suite confusion post Lone Wolf / LionDesk / Compass-Anywhere consolidation. AgentGuard fit: portable receipts across Lofty + Follow Up Boss + Rechat.
5. **Spend creep in CRM AI add-ons.** Lofty AOS starts ~$299/month per AIscending; Compass One bundles raise costs. AgentGuard fit: per-outcome budget caps independent of CRM seat price.

## Compliance citations satisfied by AgentGuard receipts

- **Fair Housing Act (42 U.S.C. §3604)** — HUD May 2024 guidance extends to AI screening + ad targeting. Source: archives.hud.gov/news/2024/pr24-098.cfm.
- **Equal Credit Opportunity Act (15 U.S.C. §1691) + Regulation B (12 CFR §1002)** — applies to mortgage AI underwriting; adverse-action notices must include specific reasons.
- **TCPA (47 U.S.C. §227) + 47 CFR §64.1200** — written express consent for AI SMS/voice; FCC Feb 2024 AI ruling declared AI-generated voice "artificial."
- **TRID (12 CFR §1026.19)** — mortgage disclosure timing; AI-generated loan docs must respect the 3-business-day rule.
- **RESPA (12 U.S.C. §2607)** — kickback rules apply to AI-driven referrals.
- **State-specific disclosure laws** — CA Civil Code §1102 (TDS), TX §5.008, FL §689.25. AI summaries do not displace seller's statutory duty.
- **NAR Code of Ethics Article 12** — truth in advertising of AI-generated content.

## Pricing anchors (real estate / mortgage productivity SaaS, May 2026)

| Tool | Price tier | AgentGuard Solo $49 maps to |
|---|---|---|
| Cloud CMA (Lone Wolf) | $35-$49/month | ≈ a Cloud CMA seat |
| Top Producer / Wise Agent | $30-$80/agent/month | ≈ one mid-tier seat |
| Follow Up Boss Grow | $58/user/month annual ($69 monthly) | ≈ one FUB seat |
| Lofty AOS | ~$299/month entry [partial] | ≈ a Lofty add-on |
| kvCORE / BoldTrail | $300+/agent/month brokerage-negotiated | < a kvCORE seat |
| MLS dues | $300-$800/year per agent | ≈ 2 months of MLS dues |

Solo $49 ≈ Cloud CMA seat or below FUB. Startup $199 ≈ 4-5-agent team. Growth $999 ≈ 20-agent brokerage adding portable governance over Lofty / FUB / Rechat.

## Install via Visa CLI

Agents on Visa CLI can buy a license autonomously and gain Fair-Housing-ready signed receipts for every AI-assisted lead, listing, and closing:

```bash
visa-cli buy https://agentguard.run/api/x402/license?tier=startup
agentguard auth visa-cli
```

## Signed receipt example (`lead_qualified`)

```json
{
  "version": "v0.4.2",
  "outcome": "lead_qualified",
  "vertical": "real-estate",
  "brokerageId": "acme-realty",
  "leadId": "lead-2026-44218",
  "agentId": "agent.jones",
  "policy": "real-estate-default-v1",
  "posture": "standard",
  "capability": "data_write",
  "fairHousingAudit": {
    "inputs": ["budget", "timeline", "neighborhoodPreference"],
    "excluded": ["race", "familialStatus", "nationalOrigin", "disability"],
    "screeningModelFingerprint": "sha256:ee14a2..."
  },
  "tcpaConsent": { "captured": true, "eventId": "consent-c481" },
  "model": "anthropic/claude-haiku-4.5",
  "tokensIn": 4012,
  "tokensOut": 891,
  "costCents": 2,
  "decision": "allow",
  "outcomeReached": true,
  "evidenceHash": "sha256:a48fe1...",
  "issuedAt": "2026-05-28T16:48:09.221Z",
  "signature": "ed25519:6e91ca...3088",
  "previousReceiptHash": "sha256:b0f922..."
}
```

The `fairHousingAudit` block is the FHA defensibility evidence: list of inputs used in screening + explicit exclusion of protected-class proxies + model fingerprint for reproducibility. The `tcpaConsent` block satisfies 47 CFR §64.1200.

## What real estate brokers use AI for in 2026 (top 10 use cases by adoption)

Per Lofty AOS launch (RealEstateNews.com Feb 3, 2026), Loft47 transaction tooling (Apr 2026), Rechat 2026 State of AI & Real Estate Marketing Report. NAR's 2025 Technology Survey (Sep 18, 2025; n=1,241 from 49,233-Realtor sample): 68% Realtor AI adoption, but only 20% daily, 22% weekly, 27% a few times a month, 32% not yet.

| # | Task | Monthly volume per agent | Cost on Haiku 4.5 |
|---|---|---|---|
| 1 | Listing description generation | 5-40 | ~$0.008 |
| 2 | AI lead capture / nurture (Structurely, Perspective AI) | 100-2,000 conversations | ~$0.009 |
| 3 | Comparable Market Analysis (CMA) generation | 8-40 | ~$0.016 |
| 4 | Contract / disclosure document extraction (V7 Go, Loft47) | 5-30 | ~$0.035 |
| 5 | SMS / voice follow-up (Lofty AOS) | 200-2,000 | ~$0.007 |
| 6 | Social media content (Rechat) | 8-40 posts | ~$0.007 |
| 7 | Tenant screening / mortgage pre-qual | 5-50 | ~$0.009 |
| 8 | Email drafting to clients / listings | 200-1,500 | ~$0.005 |
| 9 | Predictive seller identification (SmartZip) | 50-500 records | ~$0.005 |
| 10 | Loan-doc summarization (mortgage) | 5-40 | ~$0.040 |

## When to add AgentGuard Trace (sister product)

For workflows requiring court-admissible AI provenance, regulatory evidence chains, or multi-party signed attestation, pair this Spend skill with **AgentGuard Trace**. Trace adds cryptographic provenance per action, tamper-evident multi-party signed evidence chains, and triple-proof architecture suitable for legal discovery + regulatory investigation.

Spend gives you: per-outcome budgets + capability gates + signed receipts.
Trace adds: court-admissible evidence chains + multi-party attestation + tamper-evident provenance.

Combined stack:
- **Spend** = per-outcome budget enforcement + outcome receipts
- **Trace** = court-admissible evidence chain + multi-party signatures
- **Verifier** = paste any receipt at https://agentguard.run/verify

Both products run zero-data-plane (in your runtime, never our infrastructure). Trace is in production today: the AgentGuard founder is currently using it to authenticate his own legal-matter communications. Email `invest@agentguard.run` for Trace evaluation access.


## Patent notice

AgentGuard Spend is protected by U.S. patent-pending technology (App. Nos. 63/983,615; 63/983,621; 63/983,843; 63/984,626; 64/071,781; 64/071,789).

## Resources

- General SDK docs: https://agentguard.run
- Receipt verifier: https://agentguard.run/verify
- npm: https://www.npmjs.com/package/@agentguard-run/spend
- PyPI: https://pypi.org/project/agentguard-spend/
- Pricing: https://agentguard.run/pricing
- Brokerage licensing: invest@agentguard.run
