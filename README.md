# llm_skills

Personal agent skills for AI coding assistants — focused on Grocy meal planning and recipe discovery.

## Installation

Install on any supported agent (Claude Code, Cursor, Windsurf, OpenCode, and 70+ others):

```bash
npx skills@latest add yura1106/llm_skills
```

Or, for manual linking to Claude Code and compatible harnesses:

```bash
git clone https://github.com/yura1106/llm_skills ~/.llm_skills
bash ~/.llm_skills/scripts/link-skills.sh
```

## Skills

### `plan-meal`

Adds products and recipes to a [Grocy](https://grocy.info) meal plan via the `grocy-nutrients` MCP server.

**Triggers:** "додай X в меню", "заплануй на завтра", "add X to my meal plan", "plan Y for tomorrow"

**Requires:** `grocy-nutrients` MCP server configured in your agent.

### `recipe-from-stock`

Suggests what to cook from your current Grocy pantry, prioritising items expiring soon. Returns 2 local Grocy recipes + 3 fresh internet recipes.

**Triggers:** "що приготувати", "знайди рецепт по залишках", "what can I cook", "recipe from my stock"

**Requires:** `grocy-nutrients` MCP server configured in your agent.

## MCP Server Setup

Both skills require the `grocy-nutrients` MCP server. Add it to your Claude Code config (`~/.claude/claude_desktop_config.json` or `settings.json`):

```json
{
  "mcpServers": {
    "grocy-nutrients": {
      "command": "npx",
      "args": ["grocy-nutrients-mcp"],
      "env": {
        "GROCY_URL": "http://your-grocy-instance",
        "GROCY_API_KEY": "your-api-key"
      }
    }
  }
}
```

## Structure

```
skills/
└── grocy/
    ├── plan-meal/
    │   └── SKILL.md
    └── recipe-from-stock/
        └── SKILL.md
```

## Updating a skill

Skills in this repo are authored locally at `~/.claude/skills/<skill-name>/SKILL.md` and
then published here. When a skill's instructions change, sync the file and push:

```bash
# 1. Clone the repo if you don't have it locally yet
git clone git@github.com:yura1106/llm_skills.git /tmp/llm_skills

# 2. Copy the updated SKILL.md from your local skill directory
cp ~/.claude/skills/plan-meal/SKILL.md /tmp/llm_skills/skills/grocy/plan-meal/SKILL.md
# or for recipe-from-stock:
cp ~/.claude/skills/recipe-from-stock/SKILL.md /tmp/llm_skills/skills/grocy/recipe-from-stock/SKILL.md

# 3. Commit and push
cd /tmp/llm_skills
git add skills/
git commit -m "update plan-meal skill"
git push
```

Users who installed via `npx skills@latest add` can get the latest version by re-running
the same command. Users who installed manually via `git clone` just need to `git pull`
inside their cloned directory — the symlinks created by `link-skills.sh` already point
into that directory, so no relinking is needed.

### Adding a new skill

1. Create the skill directory and file:
   ```bash
   mkdir -p /tmp/llm_skills/skills/<category>/<skill-name>
   cp ~/.claude/skills/<skill-name>/SKILL.md /tmp/llm_skills/skills/<category>/<skill-name>/SKILL.md
   ```
2. Make sure `SKILL.md` starts with the required frontmatter:
   ```yaml
   ---
   name: skill-name
   description: One-line description of when and why to invoke this skill.
   ---
   ```
3. Commit and push. The `skills` CLI will discover any `SKILL.md` found under `skills/`
   automatically — no registration step needed.
