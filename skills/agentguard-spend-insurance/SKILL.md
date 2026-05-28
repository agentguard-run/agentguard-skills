---
name: agentguard-spend-insurance
description: Configure AgentGuard Spend for insurance agencies and brokerages. Use for quote letters, policy comparisons, certificates of insurance, renewals, inbound lead qualification, E&O-sensitive workflows, NAIC Model Bulletin references, and insurance producer AI governance.
---

# AgentGuard Spend: Insurance Agencies

Use this skill when an insurance agency, producer, broker, or agency principal wants local AI spend caps, capability gates, and signed receipts for insurance workflows. AgentGuard stays in the customer runtime. Prompts, policyholder data, API keys, and signing keys never leave the customer process.

## 1. Buyer And Operating Model

Primary buyer: agency principal or operations lead at a 1 to 50 producer insurance agency.

Common systems: Applied Epic, HawkSoft, AMS360, EZLynx, NowCerts, AgencyZoom, Formstack, DocuSign, Postmark, OpenRouter.

Pricing anchor: Solo $49 is about half of one HawkSoft seat at $94 per user. The paid tier should feel cheaper than one agency-management seat while producing a cryptographic receipt for each AI-assisted insurance outcome.

Default Governance Posture: compliance.

## 2. Five Insurance Outcomes

| Outcome | Capability | Per-outcome cap | Recommended primary | Reviewer Cascade |
|---|---:|---:|---|---|
| `quote_letter_drafted` | `data_write` | $0.50 | `anthropic/claude-sonnet-4-6` | always |
| `policy_comparison_generated` | `data_write` | $1.50 | `anthropic/claude-opus-4-7` | always |
| `certificate_of_insurance_issued` | `payment_initiate` | $0.25 | `openai/gpt-5-mini` | risk-triggered |
| `renewal_outreach_completed` | `data_write` | $0.15 | `anthropic/claude-haiku-4-5` | risk-triggered |
| `inbound_lead_qualified` | `read_only` | $0.30 | `google/gemini-3-flash-preview` | risk-triggered |

Top E&O-risk outcomes: `policy_comparison_generated`, `quote_letter_drafted`, and `claims_correspondence_drafted`. These should always run Reviewer Cascade before the final receipt is issued.

## 3. Policy Defaults

```yaml
id: insurance-agency-v1
name: Insurance agency AI outcomes
scope:
  tenantId: agency
mode: enforce
version: 1
effectiveFrom: 2026-05-28T00:00:00Z
requiredCapability: data_write
governancePosture:
  posture: compliance
  auditRetentionDays: 2555
models:
  cloudFrontierPrimary: anthropic/claude-opus-4-7
  hostedOssFallback: baseten/llama-4
  baaCoveredOption: aws-bedrock/anthropic.claude-sonnet-4-6
caps:
  - window: per_call
    amountCents: 150
    action: block
    reason: policy comparison cap. E&O-sensitive workflows should not silently downgrade.
  - window: per_day
    amountCents: 5000
    action: block
    reason: agency daily ceiling.
outcomes:
  quote_letter_drafted:
    cap: 0.50
    capability: data_write
    drafter:
      model: anthropic/claude-sonnet-4-6
      maxCostCents: 35
    reviewer:
      model: anthropic/claude-opus-4-7
      maxCostCents: 15
      trigger:
        - keyword_watchlist: insurance-defaults
        - high_risk_classifier_score_above: 0.55
  policy_comparison_generated:
    cap: 1.50
    capability: data_write
    drafter:
      model: anthropic/claude-sonnet-4-6
      maxCostCents: 90
    reviewer:
      model: anthropic/claude-opus-4-7
      maxCostCents: 60
      trigger:
        - keyword_watchlist: insurance-defaults
        - high_risk_classifier_score_above: 0.55
```

Why the caps: quote letters are short but E&O-sensitive, policy comparisons need larger context and review, COIs are templated, renewals are high-volume, and inbound qualification can route to lower-cost models.

## 4. Reviewer Cascade Rules

Always trigger Reviewer Cascade for:

- Policy comparisons with exclusions, endorsements, limits, or deductibles.
- Quote letters that mention coverage sufficiency, replacement cost, business interruption, cyber, or professional liability.
- Claims correspondence or any draft likely to affect claim handling.

Default triggers:

```yaml
trigger:
  - keyword_watchlist: insurance-defaults
  - output_contains_any:
      - exclusion
      - endorsement
      - deductible
      - business interruption
      - cyber liability
      - professional liability
      - claims denial
  - high_risk_classifier_score_above: 0.55
```

## 5. Compliance Citations For Receipts

Use these citations as audit context, not as a claim that the software satisfies a regulation:

- NAIC Model Bulletin on the Use of Artificial Intelligence Systems by Insurers.
- 11 NYCRR §215.13 for unfair claims settlement and documentation expectations.
- Verisk CG 40 47 and CG 40 48 AI-related exclusions and endorsements.
- NYDFS Circular Letter 2024-7 for insurer AI governance expectations.

## 6. System Instructions

```text
You are assisting a licensed insurance producer. Draft only in a descriptive and review-ready form. Do not state that coverage is guaranteed. Do not imply a policy comparison is exhaustive. Flag exclusions, endorsements, limits, deductibles, carrier-specific forms, and claims-sensitive language for human review. Preserve policyholder confidentiality. AgentGuard signed receipts prove what the AI workflow attempted, what it cost, and which capability tier authorized the call.
```

## 7. Implementation Checklist

1. Run `agentguard auth openrouter` and keep the key local.
2. Run `agentguard advisor --posture compliance` for the agency profile.
3. Add `insurance-defaults` keyword watchlist triggers for E&O-sensitive outcomes.
4. Use Reviewer Cascade always for policy comparisons and quote letters.
5. Verify a demo receipt at `https://agentguard.run/verify` before showing the workflow to an agency principal.
