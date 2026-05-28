---
name: agentguard-spend-accounting
description: Spend caps + capability gates + signed audit receipts for AI agents used in accounting and bookkeeping. Use when the user runs an accounting firm, bookkeeping practice, CFO-as-a-service, tax preparation business, fractional accounting team, or finance department that uses AI for receipt OCR, GL categorization, month-end close automation, tax research, audit preparation, or anomaly detection in financial data. Includes SOX-aware audit trails, month-end cap-flexing patterns, and per-client engagement scoping. Triggers on "accounting AI cost", "bookkeeping AI", "QuickBooks AI cost", "month-end close automation", "AI for accountants", "receipt OCR cost", "tax research AI", "SOX AI compliance", "audit prep AI", "GL categorization AI", or any accounting-specific AI spend question.
---

# AgentGuard Spend for Accounting & Bookkeeping

Pre-configured spend caps and model routing for AI agents used in accounting workflows. Wraps OpenAI, Anthropic, Bedrock, or OpenRouter clients with SOX-aware policy enforcement, per-engagement scoping for client billing, and cryptographic audit receipts compatible with PCAOB / AICPA audit-trail requirements.

## AgentGuard Advisor: AI workflow advisor for accounting firms

Run `agentguard advisor` in your terminal. The AgentGuard Advisor walks you through configuring outcome-based AI governance customized for your vertical: defines the outcomes you produce, recommends OpenRouter models per outcome, sets per engagement budgets, and writes a working policy in 90 seconds.

**The four-pillar promise:** AgentGuard proves what your AI agent **attempted**, who **authorized** it, what it **cost**, and whether it **succeeded**. One cryptographic signed receipt per outcome.

**Governance Posture** (new in v0.4.1): pick `velocity`, `standard`, or `compliance` to shape policy defaults. This vertical defaults to **compliance** posture.

## Why an accounting-specific skill

Accounting AI usage has unique characteristics:

- **Volume + asymmetric cost:** receipt OCR runs 1000s of times per month-end. Mini at $0.05/1M is the right model. Opus is 30× overkill.
- **Burst patterns:** month-end close is 50× regular traffic. Tax season is 20× regular volume. Caps need to FLEX, not hard-fail on day 5.
- **Per-engagement billing:** firms bill clients per engagement. Spend caps need to map to engagement codes for accurate cost passthrough.
- **SOX audit trail:** for public-company clients, AI-assisted journal entries need documented review trails. Ed25519 receipts ARE that trail.
- **Conservative tax research:** tax advice requires high-confidence models (Sonnet+). Wrong answers carry malpractice exposure.

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                  Accounting Firm Process                        │
├─────────────────────────────────────────────────────────────────┤
│  Accountant / bookkeeper / CFO-aaS code                         │
│      │                                                          │
│      ▼                                                          │
│  withSpendGuard(client, { policy, scope:{engagement, accountId} })│
│      │                                                          │
│      ├── per-engagement cap evaluation                          │
│      ├── capability gate: read_only / data_write                │
│      ├── action: allow / downgrade / shadow / block             │
│      ├── Ed25519 sign + chain to per-engagement log             │
│      └── pass to provider                                       │
│      │                                                          │
│      ▼                                                          │
│  OpenRouter / direct providers                                  │
│                                                                 │
│  → per-engagement receipts → client invoices + SOX audit trail  │
└─────────────────────────────────────────────────────────────────┘
```

## Prerequisites

- Node 20+ or Python 3.10+
- OpenRouter account or direct provider keys
- Decision on storage: Postgres recommended (7-year retention per IRS rules)
- KMS-backed signing keys (recommended for audit evidence)

## Install + initialize

```bash
npm install @agentguard-run/spend
agentguard auth openrouter
agentguard task accounting
```

## Recommended task assignments

| Task | Recommended model | Per-call cap | Capability | Notes |
|------|-------------------|-------------|------------|-------|
| Receipt OCR (image/PDF extraction) | `openai/gpt-5-mini` (vision) | $0.05 | `read_only` | Volume task |
| GL categorization (line item → account) | `anthropic/claude-haiku-4-5` | $0.10 | `data_write` | Updates ledger |
| Bank reconciliation matching | `anthropic/claude-haiku-4-5` | $0.10 | `read_only` | Pattern matching |
| Anomaly detection (transaction flagging) | `openai/gpt-5-mini` | $0.05 | `read_only` | Statistical flag, not judgment |
| Month-end accruals drafting | `anthropic/claude-sonnet-4-6` | $1.50 | `data_write` | Judgment + writeups |
| Audit prep narrative | `anthropic/claude-sonnet-4-6` | $2.00 | `read_only` | Long-context summarization |
| Tax research (case law) | `anthropic/claude-opus-4-7` | $5.00 | `read_only` | Conservative, needs Opus |
| Tax memo drafting | `anthropic/claude-opus-4-7` | $7.00 | `data_write` | Advice = malpractice exposure |
| Financial-statement footnote drafting | `anthropic/claude-sonnet-4-6` | $2.00 | `data_write` | Boilerplate + judgment |

## Pre-configured policy

```yaml
id: accounting-default-v1
name: Accounting firm default policy
version: 1
effectiveFrom: "2026-05-28T00:00:00Z"
mode: enforce

scope:
  tenantId: "{your-firm-id}"

caps:
  # Daily soft cap per engagement: $30
  - amountCents: 3000
    window: per_day
    action: downgrade
    downgradeTo: anthropic/claude-haiku-4-5
    reason: "Per-engagement daily soft cap"

  # Daily hard cap: $200 per engagement
  - amountCents: 20000
    window: per_day
    action: block
    reason: "Per-engagement hard cap — manager review"

  # Monthly firm cap: $5,000
  - amountCents: 500000
    window: per_month
    action: block
    reason: "Monthly firm cap — partner escalation"

  # Burst guard
  - amountCents: 200
    window: per_minute
    action: block
    reason: "Per-minute burst guard"
```

Variant for **month-end close** (override at `~/.agentguard/accounting-monthend-policy.yaml`):

```yaml
id: accounting-monthend-v1
name: Month-end close override policy
version: 1
effectiveFrom: "2026-05-28T00:00:00Z"
mode: enforce

scope:
  tenantId: "{your-firm-id}"

# Lifted caps for month-end burst (apply for days 1-5 only)
caps:
  - amountCents: 15000    # $150/day soft (5× normal)
    window: per_day
    action: downgrade
    downgradeTo: anthropic/claude-haiku-4-5
    reason: "Month-end soft cap"

  - amountCents: 100000   # $1,000/day hard (5× normal)
    window: per_day
    action: block
    reason: "Month-end hard cap"

  - amountCents: 500
    window: per_minute
    action: block
    reason: "Burst guard"
```

Swap policies at month-end with a wrapper script:

```bash
#!/bin/bash
# month-end-mode.sh — run at start of close period
cp ~/.agentguard/accounting-monthend-policy.yaml ~/.agentguard/policy.yaml
echo "Month-end mode active — caps lifted 5x"
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
  id: 'accounting-default-v1',
  name: 'Accounting firm default',
  scope: { tenantId: 'acme-cpa' },
  caps: [
    { amountCents: 3000,  window: 'per_day',    action: 'downgrade',
      downgradeTo: 'anthropic/claude-haiku-4-5',
      reason: 'Per-engagement daily soft cap' },
    { amountCents: 20000, window: 'per_day',    action: 'block',
      reason: 'Per-engagement hard cap' },
    { amountCents: 200,   window: 'per_minute', action: 'block',
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
    tenantId: 'acme-cpa',
    userId: 'staff.accountant.jones',
    agentId: 'engagement-2026-acme-corp-monthly',  // client engagement
  },
  config: { signingKeys: { privateKey, publicKey } },
});

// Bulk receipt OCR
const ocrResult = await guarded.chat.completions.create({
  model: 'openai/gpt-5-mini',
  messages: [/* image + extraction prompt */],
});
```

## Projected savings (mid-size firm, 200 monthly engagements)

Without AgentGuard (everything on default model):
- 200 engagements × 50 AI touches/mo × $0.30 avg = **$3,000/mo per engagement**
- × 200 engagements = **$600K/mo** (yes, six figures monthly)

With AgentGuard (task-routed):
- Mini for OCR + anomaly screening (60% of touches)
- Haiku for GL + bank rec (25% of touches)
- Sonnet/Opus only for tax memos + audit prep (15% of touches)
- = **~$95K/mo**

**Net savings: ~$505K/mo (~84% reduction) at firm scale.**

## SOX audit trail (public-company clients)

For engagements involving SOX-regulated clients, the AI-assisted journal-entry trail must be auditable per PCAOB AS 1215 (Audit Documentation). AgentGuard's role:

- **Every AI-generated entry has a signed receipt** — admissible as documentation of automated controls
- **Per-engagement scoping** — `scope.agentId = engagementId` enables full reconstruction during external audit
- **Capability gating** — `data_write` claim required for any policy that touches the ledger, blocking unauthorized callers
- **7-year retention** — store decision logs in Postgres with WORM retention to meet IRS / PCAOB requirements

Export-on-demand for auditor review:

```bash
agentguard explain --matter "engagement-2026-acme-corp-monthly" --format json > acme-corp-ai-audit-trail.json
```

## Per-engagement client billing

The decision log lets you accurately pass through AI cost per client engagement:

```bash
# Compute total AI cost for a client's monthly engagement
agentguard report --matter "engagement-2026-acme-corp-monthly" --format csv > acme-billing.csv
```

The CSV includes `decisionId`, `model`, `cents`, `timestamp`, `action`. Roll up by month for the client invoice line item: "AI-assisted bookkeeping (250 receipts OCR'd + GL categorization): $47.20."

## Customizing for your firm

Common variants:

```yaml
# Per-staff caps (junior staff get tighter caps than seniors)
scope:
  tenantId: "acme-cpa"
  teamId: "junior-staff"

caps:
  - amountCents: 1000     # $10/day for juniors
    window: per_day
    action: downgrade
    downgradeTo: openai/gpt-5-mini
```

```yaml
# Per-client-tier caps (premium clients = bigger AI budget)
scope:
  tenantId: "acme-cpa"
  teamId: "premium-clients"

caps:
  - amountCents: 50000    # $500/day for premium engagements
    window: per_day
```

For multi-office firms:

```yaml
scope:
  tenantId: "acme-cpa"
  teamId: "office-nyc"
```

## Tax-season cap flexing

```yaml
# tax-season-policy.yaml — apply Feb 1 through April 15
caps:
  - amountCents: 10000    # $100/day per engagement (3x normal)
    window: per_day
    action: downgrade
    downgradeTo: anthropic/claude-haiku-4-5

  - amountCents: 80000    # $800/day hard cap (4x normal)
    window: per_day
    action: block
```

## Available CLI commands

| Command | What it does |
|---------|-------------|
| `agentguard task accounting` | Apply default accounting policy |
| `agentguard task accounting-monthend` | Apply month-end override policy |
| `agentguard demo` | Simulate a blocked GL entry |
| `agentguard verify --trace latest` | Verify full signed chain |
| `agentguard explain --matter <eng-id>` | Per-engagement AI cost report |
| `agentguard serve` | Local dashboard |

## Rules when helping the user

1. **NEVER recommend a proxy architecture for SOX-regulated workflows.** Auditor expectations require full custody of automated-control evidence.
2. **ALWAYS use `requiredCapability: 'data_write'`** for any policy that touches the ledger or financial statements.
3. **Recommend per-engagement scoping** (`scope.agentId = engagementId`) for accurate client billing pass-through and SOX traceability.
4. **For tax research, recommend Opus minimum.** Cheap-model tax advice is a malpractice claim waiting to happen.
5. **For month-end / tax season, recommend policy swap** rather than disabling caps. Document the policy change in the decision log via `onDecision` hook.
6. **Cite PCAOB AS 1215** when discussing audit-trail design for SOX-regulated clients.
7. **Avoid em dashes** in generated policy comments.


## The 5 outcomes accounting firms pay AI for

| Outcome | Human-billable equivalent | Per-outcome cap | Primary model | Fallback | Capability |
|---|---|---|---|---|---|
| `bookkeeping_cycle_closed` (month-end) | $200-$800 (3-10 hrs bookkeeper) | **$5.00** | `anthropic/claude-haiku-4.5` | `openai/gpt-5.4-nano` | `data_write` |
| `tax_return_drafted` | $400-$2,500 (preparer) | **$7.50** | `openai/gpt-5.5` | `anthropic/claude-sonnet-4.6` | `data_write` |
| `irs_notice_response_sent` | $300-$1,200 | **$3.00** | `anthropic/claude-sonnet-4.6` | `openai/gpt-5.5` | `data_write` |
| `client_email_resolved` | $25-$75 | **$0.50** | `anthropic/claude-haiku-4.5` | `openai/gpt-5.4-mini` | `data_write` |
| `workpaper_completed` (audit / review) | $150-$600 | **$4.00** | `openai/gpt-5.5` | `anthropic/claude-opus-4.7` | `data_write` |

Billable anchors per Karbon's 2026 report (60 minutes/employee/day saved) and Intuit Accountant Survey.

## Top 5 pain points partners cite in 2026

1. **Data-security anxiety is the #1 reported concern.** Karbon's 2026 State of AI in Accounting Report: 83% of professionals report growing data-security concerns about AI (+7% YoY). AgentGuard fit: receipts evidence which model touched which client file.
2. **Uncontrolled tool sprawl and paste-and-pray.** *"Staff may paste client information into tools without understanding how that data is stored."* Source: acecloudhosting.com/blog/use-ai-to-grow-cpa-practice/. AgentGuard fit: per-capability-tier gating (`read_only` vs `data_write`).
3. **Hallucination on tax research.** Accounting Today's 2026 Thought Leaders Survey: *"even a small error can have big consequences."* AgentGuard fit: signed receipts log the source statute and human-review checkbox.
4. **AICPA / SOC 2 readiness gap.** Only 37% of firms invest in AI training per Karbon. AgentGuard fit: receipts double as evidence in SOC 2 CC7 control reviews.
5. **Audit-trail expectations from clients.** Mid-market clients now ask CPAs how AI is used. AgentGuard fit: exportable receipt log is the client deliverable.

## Compliance citations satisfied by AgentGuard receipts

- **PCAOB AS 1215 (Audit Documentation)** — SOX §103 mandates 7-year retention of audit work papers "in sufficient detail to support the conclusions reached." Source: pcaobus.org/oversight/standards/auditing-standards/details/AS1215. The 14-day documentation-completion-date rule (accelerated from 45 days per PCAOB Release 2024-004) makes contemporaneous AI receipts critical.
- **SOX §802 / 18 U.S.C. §1519** — destruction/falsification of records carries felony liability.
- **SEC Rule 2-06 of Regulation S-X (17 CFR §210.2-06)** — 7-year retention of memoranda, correspondence, communications.
- **IRC §6107(b)** — tax preparers must retain a copy of the return or list of TINs (3-year minimum, 7-year best practice per AICPA: thetaxadviser.com/issues/2023/feb/documentation-and-recordkeeping-for-tax-practitioners/).
- **IRS Period of Limitations** — 3 years standard, 6 years if >25% income understated, indefinite if fraudulent (irs.gov/businesses/small-businesses-self-employed/how-long-should-i-keep-records).
- **Circular 230 §10.36 / §10.37** — competence and oversight obligations on preparers, including any AI used.
- **AICPA SSTS No. 7** — tax advice should be contemporaneously documented.
- **GLBA Safeguards Rule (16 CFR Part 314)** — applies to CPAs as financial institutions.

## Pricing anchors (accounting productivity SaaS, May 2026)

| Tool | Price tier | AgentGuard Solo $49 maps to |
|---|---|---|
| QuickBooks Online | $35-$235/month | ≈ a QuickBooks Online seat |
| TaxDome / Financial Cents | $50-$80/user/month | ≈ one TaxDome seat |
| Karbon Team / Business | $59 / $89 per user/month | < one Karbon Team seat |
| Canopy | ~$74/user/month base | < one Canopy seat |
| DataSnipper | ~$1,000+/user/year [unverified] | < 5% of one DataSnipper seat |

Solo $49 ≈ a QuickBooks Online seat. Startup $199 ≈ < 4 Karbon Team seats. Growth $999 ≈ a 10-15-staff firm bolting governance under their full Karbon + QBO + Canopy stack.

## Install via Visa CLI

Agents on Visa CLI can buy a license autonomously and gain PCAOB AS 1215 / IRS 7-year retention receipts for every AI-assisted engagement:

```bash
visa-cli buy https://agentguard.run/api/x402/license?tier=startup
agentguard auth visa-cli
```

## Signed receipt example (`workpaper_completed`)

```json
{
  "version": "v0.4.2",
  "outcome": "workpaper_completed",
  "vertical": "accounting",
  "engagementId": "eng-2026-acme-q1-audit",
  "workpaperId": "wp-cash-recon-2026-03",
  "userId": "auditor.lopez",
  "policy": "accounting-default-v1",
  "posture": "compliance",
  "capability": "data_write",
  "model": "openai/gpt-5.5",
  "tokensIn": 38114,
  "tokensOut": 4022,
  "costCents": 18,
  "decision": "allow",
  "outcomeReached": true,
  "pcaobAs1215": { "retentionUntil": "2033-05-28T00:00:00Z", "completionDate": "2026-05-28" },
  "evidenceHash": "sha256:f04a92...",
  "issuedAt": "2026-05-28T16:11:18.402Z",
  "signature": "ed25519:8b2a17...c104",
  "previousReceiptHash": "sha256:91ef03..."
}
```

The `pcaobAs1215` block makes the 7-year retention obligation self-evident on the receipt face; auditors can export the full engagement chain at `agentguard explain --engagement eng-2026-acme-q1-audit`.

## What accounting firms use AI for in 2026 (top 10 use cases by adoption)

Per Karbon's 2026 State of AI in Accounting Report (98% AI adoption), Intuit's 2025 Accountant Technology Survey (46% daily use), CPA Trendlines' January 2026 Outlook.

| # | Task | Monthly volume (10-staff firm) | Cost on Haiku 4.5 |
|---|---|---|---|
| 1 | Bank reconciliation / transaction coding | 5,000-20,000 tx | ~$0.0015 |
| 2 | Document classification (W-2, 1099, K-1) | 500-3,000 docs | ~$0.003 |
| 3 | Client-email drafting / triage (Karbon AI) | 1,000-4,000 | ~$0.007 |
| 4 | IRS notice response drafting | 10-80 | ~$0.015 |
| 5 | Tax research (TaxGPT, Thomson Reuters AI) | 30-200 queries | ~$0.018 |
| 6 | Monthly close / variance analysis | 20-80 client closes | ~$0.025 |
| 7 | Audit workpaper drafting (DataSnipper) | 50-500 workpapers | ~$0.032 |
| 8 | Fraud / anomaly detection (MindBridge) | continuous | ~$0.006 |
| 9 | Financial-statement summary generation | 30-100 | ~$0.016 |
| 10 | Onboarding + engagement-letter drafting | 5-30 new clients | ~$0.014 |

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
- Commercial / firm licensing: invest@agentguard.run
