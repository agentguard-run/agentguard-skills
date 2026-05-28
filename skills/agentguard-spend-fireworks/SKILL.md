---
name: agentguard-spend-fireworks
description: Wrap Fireworks OpenAI-compatible inference endpoints with AgentGuard Spend. Use for hosted OSS alternatives and dual-provider receipt attribution.
---

# AgentGuard Spend: Fireworks Integration

Use this skill when a customer wants to route some outcomes through Fireworks while keeping AgentGuard Spend enforcement local. AgentGuard never proxies prompts, completions, provider keys, policies, or signing keys.

## TypeScript

```ts
import OpenAI from 'openai';
import { withSpendGuard } from '@agentguard-run/spend';

const client = new OpenAI({
  apiKey: process.env.FIREWORKS_API_KEY,
  baseURL: process.env.FIREWORKS_BASE_URL,
});

const guarded = withSpendGuard(client, { policy, scope });
await guarded.chat.completions.create({
  model: 'accounts/fireworks/models/llama-v3p1-70b-instruct',
  messages: [{ role: 'user', content: 'Run this governed outcome.' }],
});
```

## Python

```py
from openai import OpenAI
from agentguard_spend import with_spend_guard

client = OpenAI(
    api_key=os.environ["FIREWORKS_API_KEY"],
    base_url=os.environ["FIREWORKS_BASE_URL"],
)

guarded = with_spend_guard(client, policy=policy, scope=scope)
guarded.chat.completions.create(
    model="accounts/fireworks/models/llama-v3p1-70b-instruct",
    messages=[{"role": "user", "content": "Run this governed outcome."}],
)
```

## Receipt attribution sample

```json
{
  "decision": {
    "provider": "openai-compatible",
    "modelRequested": "accounts/fireworks/models/llama-v3p1-70b-instruct",
    "modelResolved": "accounts/fireworks/models/llama-v3p1-70b-instruct",
    "reasons": ["Hosted OSS route selected by local policy"]
  },
  "metadata": {
    "inferenceProvider": "fireworks",
    "agentguardDataPlane": "none"
  }
}
```

## Vertical defaults

- Velocity posture: consider accounts/fireworks/models/llama-v3p1-70b-instruct as primary for read-only and data-write drafts.
- Standard posture: use accounts/fireworks/models/llama-v3p1-70b-instruct as hosted OSS fallback behind a frontier model.
- Compliance posture: use a BAA-covered route first where required, then keep accounts/fireworks/models/llama-v3p1-70b-instruct only for non-sensitive drafts.
