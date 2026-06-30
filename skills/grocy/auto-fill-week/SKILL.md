---
name: auto-fill-week
description: Plans several days of meals at once by picking recipes that fit the user's nutrition targets and use up their stock, then writing them to the Grocy meal plan via the grocy-nutrients MCP server. This PLANS multiple days (writes meal-plan entries) — it does NOT consume stock. Use when the user says "сплануй мені тиждень", "заповни меню на 3 дні", "склади план харчування на тиждень", "plan my week", "auto-fill my meal plan", "fill the next N days", or wants a multi-day plan generated for them.
---

# Auto-Fill Week

Generate a multi-day meal plan: for each day, pick recipes that fit that day's nutrition targets
and lean on what's in stock, then write them to the plan. Orchestrates existing tools — no new
capability. It **plans**; it does not consume stock.

## Prerequisites

Depends on the **`grocy-nutrients` MCP server**. Tools used:

- `mcp__grocy-nutrients__get_meal_plan` — what's already planned (so we don't double-book)
- `mcp__grocy-nutrients__get_nutrition_targets` — each day's targets
- `mcp__grocy-nutrients__get_all_stock` — what's on hand / expiring (to prefer those recipes)
- `mcp__grocy-nutrients__search_recipe` + `mcp__grocy-nutrients__get_recipe_detail` — candidate recipes + per-serving macros + ingredients
- `mcp__grocy-nutrients__add_recipe_to_meal_plan` — write a recipe entry for a day

If these tools are not available, stop and tell the user to configure the MCP server.

## Workflow

### Step 1 — Parse the span and constraints

Determine the date range (e.g. "тиждень" → today..today+6; "3 дні" → today..today+2) and any
constraints (vegetarian, quick, meals per day — default to a sensible main meal per day unless the
user says otherwise).

### Step 2 — Gather context once

In parallel: `get_all_stock()` (for expiry drivers) and `get_meal_plan(start, end)` (to see which
days/sections are already filled — **never overwrite or duplicate** an existing line).

### Step 3 — Plan day by day

For each empty day/slot in the range:
1. `get_nutrition_targets(date)` — the day's budget.
2. Run `search_recipe` queries built from expiring stock drivers + the day's primary gap; call
   `get_recipe_detail` on candidates for per-serving macros and `ingredients` (prefer recipes whose
   ingredients are mostly `in_stock`).
3. Pick a recipe (and servings) that fits the remaining budget without blowing a target. Vary
   choices across days — don't plan the same dish every day.

### Step 4 — Confirm before writing

Show the proposed plan (day → recipe → servings, with a one-line macro fit per day) and **ask the
user to confirm** before writing anything. This skill writes several entries, so confirmation is
mandatory.

### Step 5 — Write

On confirmation, for each chosen recipe call `add_recipe_to_meal_plan(recipe_id, servings, date,
section?)`. Report each `{"status": "queued"}`. If any fail, say which and continue with the rest.

### Step 6 — Summarise

Report the full plan that was written, day by day.

## Rules

- **Always confirm before writing** — never auto-write a whole week unprompted.
- **Never duplicate** an existing planned line (check `get_meal_plan` first).
- **Never** consume stock — this only plans.
- Only use recipes that exist locally (`search_recipe`); don't invent recipes or ids.
- If targets are missing for a day, fall back to stock-driven choices and note that the plan isn't
  nutrition-optimised for that day.
- Reply in the user's language; quote tool errors verbatim.
