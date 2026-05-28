---
name: agentguard-spend-baseten
description: Wrap Baseten OpenAI-compatible inference endpoints with AgentGuard Spend. Use for hosted OSS fallback routes and cost-best vertical policies.
---

# AgentGuard Spend: Baseten Integration

Use this skill when a customer wants to route some outcomes through Baseten while keeping AgentGuard Spend enforcement local. AgentGuard never proxies prompts, completions, provider keys, policies, or signing keys.

## TypeScript

```ts
import OpenAI from 'openai';
import { withSpendGuard } from '@agentguard-run/spend';

const client = new OpenAI({
  apiKey: process.env.BASETEN_API_KEY,
  baseURL: process.env.BASETEN_BASE_URL,
});

const guarded = withSpendGuard(client, { policy, scope });
await guarded.chat.completions.create({
  model: 'baseten/llama-4',
  messages: [{ role: 'user', content: 'Run this governed outcome.' }],
});
```

## Python

```py
from openai import OpenAI
from agentguard_spend import with_spend_guard

client = OpenAI(
    api_key=os.environ["BASETEN_API_KEY"],
    base_url=os.environ["BASETEN_BASE_URL"],
)

guarded = with_spend_guard(client, policy=policy, scope=scope)
guarded.chat.completions.create(
    model="baseten/llama-4",
    messages=[{"role": "user", "content": "Run this governed outcome."}],
)
```

## Receipt attribution sample

```json
{
  "decision": {
    "provider": "openai-compatible",
    "modelRequested": "baseten/llama-4",
    "modelResolved": "baseten/llama-4",
    "reasons": ["Hosted OSS route selected by local policy"]
  },
  "metadata": {
    "inferenceProvider": "baseten",
    "agentguardDataPlane": "none"
  }
}
```

## Vertical defaults

- Velocity posture: consider baseten/llama-4 as primary for read-only and data-write drafts.
- Standard posture: use baseten/llama-4 as hosted OSS fallback behind a frontier model.
- Compliance posture: use a BAA-covered route first where required, then keep baseten/llama-4 only for non-sensitive drafts.
