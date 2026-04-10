# n8n Production Readiness Skill

Dynamic tier system for right-sizing n8n workflow hardening.

## When to Use

Use this skill on ANY n8n workflow request. It provides:
- Tier selection (ask user or infer automatically)
- Guidance on when to recommend tier changes (up or down)
- Pre-deployment checklists by tier
- Links to implementation patterns in other skills

## Tiers

| Tier | Use Case | Build Time Impact |
|------|----------|-------------------|
| **Tier 1** | Internal/Prototype | +10% |
| **Tier 2** | Production/Client-facing | +80% |
| **Tier 3** | Mission-Critical/High-volume | +200% |

## Key Behaviors

1. **Ask once** after initial prompt (or infer silently in autopilot)
2. **Escalate** when debugging goes in circles or deployment is mentioned
3. **De-escalate** when over-engineering a prototype
4. **Respect user choice** but flag serious concerns

## Key Files

- `SKILL.md` — Full AI instructions and tier definitions

## Related Skills

- `n8n-workflow-patterns` — Core workflow architecture
- `n8n-code-javascript` — Code patterns including validation/logging templates
