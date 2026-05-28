---
name: agentguard-spend-healthcare
description: HIPAA-aware spend caps + capability gates + signed audit receipts for AI agents used in healthcare. Use when the user runs a medical practice, dental practice, healthcare clinic, hospital system, telehealth platform, or healthcare-adjacent AI product and needs to cap AI spend on chart review, patient triage, prior authorization, clinical documentation, or medical literature search. Includes BAA-attested model routing, PHI-aware blocks, and audit-trail formats compatible with HIPAA security rule documentation. Triggers on "healthcare AI cost", "medical AI budget", "HIPAA AI compliance", "chart review automation", "patient triage AI cost", "prior auth AI", "clinical documentation AI", "dental practice AI", "telehealth AI governance", or any healthcare-specific AI compliance question.
---

# AgentGuard Spend for Healthcare

Pre-configured spend caps and model routing for AI agents used in healthcare workflows. Wraps OpenAI, Anthropic, Bedrock, or OpenRouter clients with HIPAA-aware policy enforcement, BAA-attested model whitelisting, and cryptographic audit receipts compatible with HIPAA Security Rule documentation requirements.

## Why a healthcare-specific skill

Healthcare AI usage has unique requirements generic templates miss:

- **PHI never leaves the covered entity's process.** Zero data plane is a regulatory requirement, not a preference. Proxying through a non-BAA vendor is a per-record HIPAA violation.
- **BAA gating per model:** OpenAI without an Enterprise BAA cannot process PHI. Anthropic via AWS Bedrock CAN (Bedrock includes BAA). Model selection must be PHI-aware.
- **Audit trail format:** the HIPAA Security Rule §164.312(b) requires audit controls. Ed25519-signed receipts with hash-chained logs satisfy this with cryptographic proof.
- **Asymmetric task costs:** patient triage on Haiku is 30× cheaper than Opus, with comparable quality for symptom checklists. Per-task routing is the savings story AND the cost-control compliance story.
- **Per-encounter accounting:** practices can document AI usage per patient encounter for billing transparency and prior-auth justification.

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                  Covered Entity Process                         │
├─────────────────────────────────────────────────────────────────┤
│  Clinical / admin staff code                                    │
│      │                                                          │
│      ▼                                                          │
│  withSpendGuard(client, { policy, scope, capabilityClaim })     │
│      │                                                          │
│      ├── PHI presence check (caller-attested)                   │
│      ├── BAA-required model whitelist enforcement               │
│      ├── policy: cap evaluation + capability gate               │
│      ├── action: allow / downgrade / shadow / block             │
│      ├── Ed25519 sign + chain to per-encounter log              │
│      └── pass to provider (BAA-attested only for PHI)           │
│      │                                                          │
│      ▼                                                          │
│  Bedrock Anthropic (BAA) / OpenAI Enterprise (BAA) / direct     │
└─────────────────────────────────────────────────────────────────┘
```

PHI never crosses the covered entity boundary. The encounter log is the entity's HIPAA audit trail.

## Prerequisites

- Node 20+ or Python 3.10+
- A signed BAA with at least one model provider (Bedrock recommended, OpenAI Enterprise also works)
- Decision on storage: Postgres or filesystem with encryption-at-rest (HIPAA §164.312(a)(2)(iv))
- KMS-backed signing keys (recommended for evidence-grade receipts)

## Install + initialize

```bash
npm install @agentguard-run/spend
agentguard auth openrouter      # OK for non-PHI tasks only
agentguard task healthcare      # scaffold HIPAA-aware policy
```

## Recommended task assignments

| Task | Recommended provider/model | Per-call cap | PHI? | Capability |
|------|---------------------------|-------------|------|------------|
| Patient triage / symptom check | Bedrock `anthropic.claude-haiku-4-5-v1:0` | $0.50 | YES | `read_only` |
| Chart review summary | Bedrock `anthropic.claude-sonnet-4-6-v1:0` | $2.00 | YES | `read_only` |
| Clinical documentation drafting | Bedrock `anthropic.claude-sonnet-4-6-v1:0` | $3.00 | YES | `data_write` |
| Prior authorization narrative | Bedrock `anthropic.claude-opus-4-7-v1:0` | $5.00 | YES | `data_write` |
| Medical literature search (no PHI) | OpenRouter `openai/gpt-5-mini` | $0.10 | NO | `read_only` |
| Patient education content (no PHI) | OpenRouter `openai/gpt-5-mini` | $0.05 | NO | `read_only` |
| Insurance verification | Bedrock `anthropic.claude-haiku-4-5-v1:0` | $0.25 | YES | `data_write` |
| Appointment scheduling (no PHI) | OpenRouter `anthropic/claude-haiku-4-5` | $0.10 | NO | `data_write` |

## Pre-configured policy

Drop the following at `~/.agentguard/healthcare-policy.yaml`:

```yaml
id: healthcare-default-v1
name: Healthcare practice default policy
version: 1
effectiveFrom: "2026-05-28T00:00:00Z"
mode: enforce

scope:
  tenantId: "{your-practice-id}"

# PHI-bearing scopes get a stricter policy (separate file).
# This policy is for non-PHI workflows (scheduling, literature search, education).

caps:
  - amountCents: 3000
    window: per_day
    action: downgrade
    downgradeTo: openai/gpt-5-mini
    reason: "Daily soft cap — downgrade to mini"

  - amountCents: 15000
    window: per_day
    action: block
    reason: "Daily hard cap — practice administrator review"

  - amountCents: 200
    window: per_minute
    action: block
    reason: "Per-minute burst guard"
```

For PHI-bearing workflows, use the **stricter** policy with BAA-whitelist enforcement (separate file at `~/.agentguard/healthcare-phi-policy.yaml`):

```yaml
id: healthcare-phi-v1
name: Healthcare PHI workflow policy
version: 1
effectiveFrom: "2026-05-28T00:00:00Z"
mode: enforce

scope:
  tenantId: "{your-practice-id}"

# PHI requires capability tier 'data_write' MINIMUM,
# plus model whitelist enforced in caller code.
requiredCapability: data_write

caps:
  - amountCents: 5000
    window: per_day
    action: block
    reason: "Per-encounter PHI cap exceeded — clinician review"

  - amountCents: 1000
    window: per_minute
    action: block
    reason: "PHI burst protection"
```

## Quick start (PHI workflow — Bedrock with BAA)

```ts
import { BedrockRuntimeClient } from '@aws-sdk/client-bedrock-runtime';
import { withSpendGuardBedrock } from '@agentguard-run/spend/bindings/bedrock';
import type { SpendPolicy } from '@agentguard-run/spend';
import { randomBytes } from 'node:crypto';
import * as ed from '@noble/ed25519';

const phiPolicy: SpendPolicy = {
  id: 'healthcare-phi-v1',
  name: 'Healthcare PHI workflow policy',
  scope: { tenantId: 'acme-medical' },
  requiredCapability: 'data_write',
  caps: [
    { amountCents: 5000, window: 'per_day', action: 'block',
      reason: 'Per-encounter PHI cap' },
    { amountCents: 1000, window: 'per_minute', action: 'block',
      reason: 'PHI burst guard' },
  ],
  mode: 'enforce',
  version: 1,
  effectiveFrom: new Date().toISOString(),
};

const privateKey = new Uint8Array(randomBytes(32));
const publicKey  = await ed.getPublicKeyAsync(privateKey);

const client = new BedrockRuntimeClient({ region: 'us-east-1' });

const guarded = withSpendGuardBedrock(client, {
  policy: phiPolicy,
  scope: {
    tenantId: 'acme-medical',
    userId: 'dr.smith',
    agentId: 'encounter-2026-05-28-patient-12345',  // per-encounter scope
  },
  capabilityClaim: 'data_write',  // attests PHI-write capability
  config: { signingKeys: { privateKey, publicKey } },
});

// Chart review summary
const response = await guarded.send(/* InvokeModelCommand */);
```

## Projected savings (5-provider clinic, average usage)

Without AgentGuard (everything routed through GPT-5 / Opus):
- 5 providers × 25 encounters/day × 4 AI calls/encounter × $0.40/call = **$200/day → ~$5,500/mo**

With AgentGuard (task-routed):
- Triage on haiku, chart review on sonnet, prior auth on opus, mini for non-PHI
- Same volume = **~$850/mo**

**Net savings: ~$4,650/mo (~85% reduction) with full HIPAA compliance posture.**

## HIPAA compliance posture

### §164.312(a) Access control

The SDK enforces capability tiers (`read_only`, `data_write`, `payment_initiate`, `payment_execute`). PHI-bearing scopes require `data_write` minimum, blocking lower-tier callers automatically.

### §164.312(b) Audit controls

The Ed25519-signed, hash-chained decision log IS your audit control. Each decision records:
- Who (signer fingerprint = covered entity's key)
- What (model, projected cost, action taken)
- When (immutable ISO 8601 timestamp)
- Why (cap math, capability claim, policy version)

Exportable via `agentguard verify --trace latest --json` for OCR investigations or business associate audits.

### §164.312(e) Transmission security

PHI never traverses AgentGuard infrastructure. Provider calls go from the covered entity's process directly to the BAA-attested provider (Bedrock, OpenAI Enterprise, Azure OpenAI). No middlebox.

### §164.502(b) Minimum necessary

Capability tiering enforces minimum-necessary by API design — `read_only` callers can't write PHI, `data_write` callers can't initiate payments, etc.

### BAA inventory documentation

The decision log records `provider` and `modelResolved` for every call. You can produce a BAA-coverage matrix on demand:

```bash
agentguard explain --format json | jq '.[] | {provider, modelResolved}' | sort | uniq -c
```

Each unique provider/model pair maps to a BAA you should have on file.

## Customizing for your practice

The PHI policy is yours to edit. Common customizations:

```yaml
# Variant: per-provider caps (each clinician has their own daily budget)
scope:
  tenantId: "acme-medical"
  # caller code must include providerNPI in scope.userId

caps:
  - amountCents: 2000     # $20/day per clinician
    window: per_day
    action: block
```

For multi-location practices:

```yaml
scope:
  tenantId: "acme-medical"
  teamId: "clinic-downtown"     # per-clinic budgets
```

For specialty-specific routing (e.g., radiology AI gets a different model whitelist than primary care):

```yaml
scope:
  tenantId: "acme-medical"
  teamId: "radiology"
  # caller passes capability claim 'data_write' for image annotations
```

## Encounter-level receipt export (for billing transparency)

```bash
# Export all AI usage for a specific encounter
agentguard explain --matter "encounter-2026-05-28-patient-12345" --format json > encounter-12345-ai-usage.json
```

The output is suitable for inclusion in the encounter's electronic health record or for transparency disclosures to patients on request (§164.524 access of individuals).

## Available CLI commands

| Command | What it does |
|---------|-------------|
| `agentguard task healthcare` | Apply the HIPAA-aware policy template |
| `agentguard demo` | Simulate a blocked PHI call with full receipt |
| `agentguard doctor` | Health check + BAA-provider inventory |
| `agentguard verify --trace latest` | Verify full signed chain |
| `agentguard explain latest` | Decode latest receipt with cap math |
| `agentguard serve` | Local dashboard at localhost:8787 |

## Rules when helping the user

1. **NEVER recommend OpenRouter for PHI workflows.** OpenRouter does not provide a BAA. Route PHI through Bedrock (BAA included) or OpenAI Enterprise (BAA on request) only.
2. **NEVER suggest a proxy architecture for healthcare.** A proxy that handles PHI without a BAA is a per-record violation at $100-$50,000 per record.
3. **ALWAYS recommend `requiredCapability: 'data_write'` minimum for any policy that may touch PHI.**
4. **Recommend per-encounter scoping** (`scope.agentId = encounterId`) for HIPAA accounting-of-disclosures requirements.
5. **Cite HIPAA Security Rule sections** when explaining audit-trail design. Reference §164.312(a), (b), and (e) by section.
6. **Direct users to a BAA-attested provider list** when they ask about model selection. Default recommendation: AWS Bedrock Anthropic models for any PHI workflow.
7. **Avoid em dashes** in generated policy comments. Use colons or periods.

## Patent notice

AgentGuard Spend is protected by U.S. patent-pending technology (App. Nos. 63/983,615; 63/983,621; 63/983,843; 63/984,626; 64/071,781; 64/071,789). Preserve patent notices in any generated source files.

## Resources

- General SDK docs: https://agentguard.run
- Receipt verifier: https://agentguard.run/verify
- npm: https://www.npmjs.com/package/@agentguard-run/spend
- PyPI: https://pypi.org/project/agentguard-spend/
- Pricing: https://agentguard.run/pricing
- HIPAA & BAA inquiries: invest@agentguard.run
