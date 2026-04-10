# Integration Guide

This guide explains how to integrate the n8n Production Readiness skill with your existing n8n skills.

---

## Overview

There are two components:

1. **New Skill:** `n8n-production-readiness/` — Contains AI interaction instructions and tier definitions
2. **Patches:** Additions to existing skills with implementation patterns

---

## Step 1: Install the New Skill

Copy the `n8n-production-readiness` folder to your skills directory:

```bash
cp -r skills/n8n-production-readiness /mnt/skills/user/
```

This gives you:
- `SKILL.md` — Full AI instructions, tier system, checklists
- `README.md` — Skill overview

---

## Step 2: Apply Patches to Existing Skills

The patch files contain content to add to your existing n8n skills. Each patch file explains exactly where to add the content.

### Patch Files

| Patch File | Target | What It Adds |
|------------|--------|--------------|
| `n8n-workflow-patterns/webhook_processing.patch.md` | `webhook_processing.md` | Silent failure problem, validation checklist, external logging, status codes |
| `n8n-workflow-patterns/SKILL.patch.md` | `SKILL.md` | 80/20 rule, tiered deployment checklists |
| `n8n-code-javascript/COMMON_PATTERNS.patch.md` | `COMMON_PATTERNS.md` | Validation templates, logging patterns |
| `n8n-code-javascript/ERROR_PATTERNS.patch.md` | `ERROR_PATTERNS.md` | Null vs empty string deep dive |

### How to Apply Patches

Each patch file contains sections like:

```markdown
## Patch 1: Add X Section

**Find:**
```markdown
### Existing Header
```

**Replace with:** (or **Add after:**)
```markdown
### New Content
...
```
```

You can either:

**Option A: Manual Integration**
1. Open the patch file
2. Open your existing skill file
3. Find the location described
4. Copy/paste the new content

**Option B: Use Claude Code**
Feed Claude Code this prompt:
```
Read the patch file at [path] and apply the changes to [target file].
```

**Option C: Use the Implementation Prompt**
Use the full implementation prompt in `docs/IMPLEMENTATION_PROMPT.md` to have Claude Code apply all patches at once.

---

## Step 3: Verify Integration

After applying patches, verify:

1. **New skill is accessible:**
   ```
   ls /mnt/skills/user/n8n-production-readiness/
   # Should show: SKILL.md, README.md
   ```

2. **Cross-references work:**
   - `webhook_processing.md` should reference `n8n-production-readiness`
   - `SKILL.md` (workflow-patterns) should link to production readiness skill

3. **No duplicate content:**
   - Validation patterns should be in `COMMON_PATTERNS.md` only
   - `webhook_processing.md` should reference them, not duplicate

---

## File Structure After Integration

```
/mnt/skills/user/
├── n8n-production-readiness/     # NEW
│   ├── SKILL.md
│   └── README.md
├── n8n-workflow-patterns/
│   ├── SKILL.md                  # PATCHED: 80/20 rule, tiered checklists
│   ├── webhook_processing.md     # PATCHED: validation, logging, status codes
│   └── ...
├── n8n-code-javascript/
│   ├── COMMON_PATTERNS.md        # PATCHED: validation & logging templates
│   ├── ERROR_PATTERNS.md         # PATCHED: null handling deep dive
│   └── ...
└── ...
```

---

## Standalone Usage (No Patches)

If you don't want to modify existing skills, you can use `n8n-production-readiness` standalone:

1. Copy just the new skill folder
2. All patterns are also included in the main `SKILL.md` (with more detail in `docs/PATTERNS.md`)
3. The skill will still work, just without deep integration into other skills

---

## Updating

When updating the skill:

1. Replace `n8n-production-readiness/` with the new version
2. Check patch files for any new additions to existing skills
3. Re-apply patches if needed

The patch files are designed to be additive — they shouldn't conflict with existing content.
