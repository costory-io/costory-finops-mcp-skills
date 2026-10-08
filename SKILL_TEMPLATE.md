# Skill Authoring Template

Copy this file to `plugins/costory/skills/<skill-name>/SKILL.md` and fill in each section. Remove all `[placeholder]` markers before merging.

If the skill is exposed via Costory MCP `get_skill`, also add an entry to `skills.json`.

---

```
---
name: [skill-name]
description: [One sentence. What does this skill do? When should an agent trigger it? Be specific about trigger conditions. Mention get_skill skillId if applicable.]
---
```

---

# [Skill Name]

## When to Trigger

- [trigger example 1]
- [trigger example 2]

## Prerequisite

1. `get_skill` with `skillId: "[snake_case_id]"` (this guide) — if served via Costory MCP
2. `get_context` when org context is needed
   - **Several organizations:** if `get_context` returns "Multiple organizations" with a list of slugs, do not pick one. Ask the user which org to use (from that list or `list_organizations`), then pass that `slug` to `get_context` and the following tool calls. If the user already named the org (e.g. "org `acme`"), use that slug directly.

## Workflow

1. [Step 1: tool + what to extract]
2. [Step 2: depends on output of step 1]
3. [Step 3: ...]

## Supporting tools

| When | Tool |
|------|------|
| [when] | `[tool_name]` |

## Safety Rules / Anti-patterns

- Never [specific forbidden action]
- Always [required confirmation or check]

## Related Skills / Next Steps

- `<skill-name>` — <when to hand off>
