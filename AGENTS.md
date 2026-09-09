# AGENTS.md — Conventions for AI Coding Agents in This Repo

This repo distributes **one plugin**, `costory`, from a single marketplace using the canonical Anthropic per-plugin subdirectory layout (same pattern as [tsuga-dev/agent-plugins](https://github.com/tsuga-dev/agent-plugins)). Skills teach agents how to use Costory MCP tools for FinOps workflows.

## Layout

```
.claude-plugin/marketplace.json            ← marketplace listing; each plugin points to its own subdir via source
skills.json                                ← Costory MCP skillId → SKILL.md path index (for get_skill)
plugins/costory/
  .claude-plugin/plugin.json               ← per-plugin manifest
  skills/<skill-name>/SKILL.md             ← skills auto-discovered by Claude Code / Codex
```

Notes:

- **No top-level `skills/` directory.** Every skill lives under `plugins/costory/skills/`.
- **No `.codex-plugin/` directory.** Codex consumes the same `.claude-plugin/marketplace.json`.
- **No `skills[]` array in marketplace.json.** Skills are auto-discovered from the plugin's `skills/` directory.
- **`skills.json`** maps Costory MCP `skillId` values (`bigquery`, `virtual-dimensions`, `dashboards`, `reports`, `query`, `recipes`) to on-disk paths so the backend can load markdown without hardcoding content.

## Skill ID mapping (Costory MCP)

`skillId`, folder name, and frontmatter `name` are the **same kebab-case token**:

| MCP `skillId` |
|---------------|
| `bigquery` |
| `virtual-dimensions` |
| `dashboards` |
| `reports` |
| `query` |
| `recipes` |

When Costory `get_skill` is wired to this repo, resolve via `skills.json`. Return the **body** of `SKILL.md` (optionally strip YAML frontmatter).

## When changing a skill

1. Edit `plugins/costory/skills/<skill>/SKILL.md` (or files under its `references/`).
2. **Bump versions in lockstep.** Three places:
   - `.claude-plugin/marketplace.json` → `metadata.version`
   - `plugins/costory/.claude-plugin/plugin.json` → `version`
   - `skills.json` → `version`

   Semver:

   - Patch (`0.1.0 → 0.1.1`): content tweaks, doc fixes, clarifications.
   - Minor (`0.1.x → 0.2.0`): new skill, new reference doc, new behavior, packaging/install-surface changes.
   - Major (`0.x → 1.0`): breaking change to a SKILL.md contract or MCP skillId (rare).

## When adding a new skill

1. Create `plugins/costory/skills/<new-skill>/SKILL.md` with proper frontmatter (`name:`, `description:`).
2. Use kebab-case folder names matching frontmatter `name`.
3. If the skill is exposed via Costory MCP `get_skill`, add an entry to `skills.json` whose record key and `id` equal that same kebab-case name.
4. Minor version bump in all three version fields (see above).

No `skills[]` array to maintain — discovery is automatic for Claude/Codex plugins.

## Path placeholders

Inside `SKILL.md` or `references/*.md`, reference your own bundled files with:

```
${CLAUDE_PLUGIN_ROOT}/skills/<skill-name>/references/foo.md
```

`${CLAUDE_PLUGIN_ROOT}` resolves to `plugins/<plugin>/` at install time, so use paths relative to that root (`skills/...`, not `plugins/<plugin>/skills/...`).

**Never use `{{SKILLS_DIR}}/...`** — that's the `npx skills` install-time placeholder, NOT substituted by Claude Code or Codex.

## Cross-skill references

Skills can reference other skills by **name** (e.g. `dashboards`, `virtual-dimensions`). Claude Code and Codex resolve skill names across installed plugins.

Do **not** use relative paths for cross-skill handoffs. Use skill names.

Inside skill bodies that mention Costory MCP `get_skill`, use the same kebab-case `skillId` as the folder / frontmatter `name`.

## Validation

CI (`.github/workflows/lint.yml`) runs on every PR. Lint locally before opening a PR:

```bash
# JSON validity
jq empty .claude-plugin/marketplace.json
jq empty plugins/costory/.claude-plugin/plugin.json
jq empty skills.json

# Every skill folder has a SKILL.md
find plugins/costory/skills -mindepth 1 -maxdepth 1 -type d | \
  xargs -I{} sh -c '[ -f {}/SKILL.md ] || echo "missing: {}/SKILL.md"'

# Versions in lockstep
v_market=$(jq -r '.metadata.version' .claude-plugin/marketplace.json)
v_plugin=$(jq -r '.version' plugins/costory/.claude-plugin/plugin.json)
v_index=$(jq -r '.version' skills.json)
[ "$v_market" = "$v_plugin" ] && [ "$v_market" = "$v_index" ] || \
  echo "FAIL: version mismatch (marketplace=$v_market plugin=$v_plugin skills.json=$v_index)"

# skills.json paths resolve
jq -r '.skills[].path' skills.json | while read -r p; do [ -f "$p" ] || echo "missing: $p"; done

# No stray {{SKILLS_DIR}}
grep -rn '{{SKILLS_DIR}}' plugins/ && echo "FAIL: replace with \${CLAUDE_PLUGIN_ROOT}"

# Frontmatter descriptions approaching 1024 chars
for f in plugins/costory/skills/*/SKILL.md; do
  len=$(awk -F'description: ' '/^description:/{print length($2); exit}' "$f")
  [ "${len:-0}" -gt 900 ] && echo "WARN: $len chars, close to 1024: $f"
done

# No stray top-level skills/ directory
[ -d skills ] && echo "FAIL: skills/ should be empty/absent — move into plugins/costory/skills/"
```

## What NOT to do

- Don't put a `skills/` directory at the repo root. Skills live under `plugins/costory/skills/`.
- Don't change a skill without bumping version in marketplace, plugin, and `skills.json`.
- Don't restore `.codex-plugin/`. Codex reads the same marketplace.json.
- Don't add a `skills[]` array to marketplace.json. Skills auto-discover from the plugin's subdir.
- Don't hardcode skill markdown back into costory-app once `get_skill` loads from this repo.
- Don't rename a skill folder or MCP skillId without updating `skills.json` and grepping for cross-references.

## Skill authoring rules

- Keep each skill **self-contained** in `SKILL.md` when it is served via Costory MCP `get_skill` (agents may not load `references/`).
- Use literal Costory MCP tool names (`query`, `create_dashboard`, …), not TypeScript constants.
- Prefer `datePreset` over frozen dates for scheduled/recurring workflows.
- Mutation / delivery actions require explicit user confirmation (`publish_virtual_dimension`, report `NOW` / `SCHEDULED`).
- Inline the safety rules that matter for the skill in its own Safety section.
