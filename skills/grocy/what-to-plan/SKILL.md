---
name: what-to-plan
description: Suggests what to cook for a specific meal (lunch, dinner, etc.) by combining what's expiring in stock with what's still needed nutritionally that day. Reads both pantry state and today's nutrition progress, then recommends recipes that use up expiring products AND fit the remaining calorie/macro budget. Use when the user asks "що приготувати на вечерю", "що мені з'їсти щоб вкластись в калорії", "підбери вечерю по залишках і нормах", "what should I cook for dinner", "suggest a meal that fits my macros", or wants recipe suggestions that are both stock-driven and nutrition-aware.
---

# What to Plan

Find the best meal option for a specific slot — one that uses up expiring products AND fits
what's nutritionally missing for the rest of the day. The union of recipe-from-stock and
nutrition-today: don't just cook what's expiring, cook what the body needs right now.

## Prerequisites

Depends on the **`grocy-nutrients` MCP server**. Tools used:

- `mcp__grocy-nutrients__get_all_stock` — full pantry with expiry data
- `mcp__grocy-nutrients__get_day` — today's consumed + planned totals
- `mcp__grocy-nutrients__get_nutrition_targets` — daily targets
- `mcp__grocy-nutrients__search_recipe` — find matching local Grocy recipes
- `mcp__grocy-nutrients__get_recipe_detail` — macros per serving for candidate recipes
- `mcp__grocy-nutrients__add_recipe_to_meal_plan` — add chosen recipe to the plan

If these tools are not available, stop and tell the user to configure the MCP server.

## Workflow

### 1. Parse the request

Identify:
- **Meal slot**: lunch, dinner, snack, breakfast — or "something" if unspecified.
- **Day**: today by default. Map natural language if specified.
- **Constraints**: "без м'яса", "швидко", "без молочного" — note and apply to filtering.

### 2. Fetch data (parallel, 3 calls)

- `get_all_stock()` — full pantry
- `get_day("today")` — consumed + planned totals (or the target day if not today)
- `get_nutrition_targets("today")` — daily targets

### 3. Compute what's nutritionally needed

For each nutrient:
- **remaining** = target − consumed − planned (what still needs to come in today)

Identify the **primary gap**: whichever macronutrient is furthest below target in percentage
terms. This becomes the filter: prefer recipes that are strong in that nutrient.

> Example: if protein is 40% of target but calories are 80%, lean toward high-protein options.

If no targets are set, skip this step and fall back to pure stock-driven suggestion.

### 4. Pick expiry drivers

From the stock, choose the top 3–5 most urgent items (soonest expiry, non-trivial amount).
Skip drinks, supplements, spices, items expired by many months.

### 5. Find local recipe candidates

Run 2–3 parallel `search_recipe` calls using names built from the expiry drivers. For each
promising match, call `get_recipe_detail` to get per-serving macros. Score each candidate:

- **+2** for each expiry driver ingredient it uses
- **+1** if it's strong in the primary gap nutrient (≥ 30% of remaining in one serving)
- **−1** if it would push any nutrient significantly over target

Pick the **top 2** local candidates.

### 6. Suggest internet recipes

Web-search for recipes matching the expiry drivers AND the nutritional gap. Search query should
include the main driver ingredient and the gap nutrient (e.g. "рецепт курятина високий білок").
Pick **2** internet options, extract ingredients + steps + URL. Always cite the real source URL.

### 7. Present

Lead with context:

> "Залишок на сьогодні: ~400 ккал · 45 г білка · 30 г вуглеводів. В холодильнику закінчується
> куряча грудка (2 дні) і сметана (3 дні)."

Then show options ranked by score, local first:

For each option: name, how it uses the expiry drivers, per-serving macros vs what's needed,
and (for internet recipes) a source link with numbered steps.

### 8. Offer to add to the plan

> "Додати «Курячий стейк зі сметанним соусом» у план на вечерю сьогодні?"

If the user confirms: `add_recipe_to_meal_plan(recipe_id, servings=1, date="today", section="Вечеря")`.
For internet recipes with no local id: offer to search for the closest Grocy match first.

## Output rules

- If nutrition data is unavailable for the target day (future date), skip step 3 and fall back
  to pure stock-driven suggestion like recipe-from-stock.
- If stock is empty or no expiry drivers exist, suggest recipes based on any in-stock staples.
- Apply user constraints (no meat, no dairy, quick, etc.) to both local search and web search.
- Never claim an ingredient is in stock if it is not in the `get_all_stock` result.
- Reply in the language the user wrote in.
