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

### GitHub CLI (`gh skill`)

Works with Claude Code, Cursor, OpenCode, Codex, Gemini CLI, Windsurf, and many more. Requires [GitHub CLI](https://cli.github.com/) v2.90.0 or later.

```bash
gh skill install agentguard-run/agentguard-skills
```

#### Installing a single skill

```bash
gh skill install agentguard-run/agentguard-skills/agentguard-spend
```

## Skills

### `agentguard-spend`

Add local-runtime spend caps, capability gates, and Ed25519-signed audit receipts to any AI agent. Wraps OpenAI, Anthropic, Bedrock, or OpenRouter clients. Zero data plane.

[Read the skill →](skills/agentguard-spend/SKILL.md)

## About AgentGuard

AgentGuard Spend is a zero-data-plane SDK that lets enterprises enforce per-task spend caps, capability gates, and cryptographically signed audit receipts on every AI agent call. Customers' prompts, API keys, and signing keys never leave their runtime.

- Website: https://agentguard.run
- npm: https://www.npmjs.com/package/@agentguard-run/spend
- PyPI: https://pypi.org/project/agentguard-spend/
- Source: https://github.com/MerchantGuardOps/agentguard-site
- Receipt verifier (any browser): https://agentguard.run/verify

## License

Skills published here are licensed Apache 2.0 for skill content. The `@agentguard-run/spend` SDK itself ships under an alpha license: free for production deployments under 10,000 enforcement calls per calendar month, commercial license required above. See https://agentguard.run for details.
