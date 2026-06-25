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
