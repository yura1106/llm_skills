---
name: recipe-from-stock
description: Suggests recipes based on the user's current Grocy stock, prioritising what is expiring. Reads the whole pantry via the grocy-nutrients MCP server, then returns 2 matching local Grocy recipes plus 3 fresh recipes found on the internet. Use when the user asks "what can I cook", "find a recipe from my stock", "recipe from my groceries", "що приготувати", "знайди рецепт по залишках", or mentions cooking from what they have / using up expiring food.
---

# Recipe From Stock

Suggest what to cook from the food the user already has, prioritising items that are expiring.
Combines the user's own Grocy recipes with fresh ideas from the web. Ends by offering to add the
chosen recipe directly to the meal plan.

## Prerequisites

This skill depends on the **`grocy-nutrients` MCP server** being configured in the host (Claude
Code / Desktop). It calls these MCP tools:

- `mcp__grocy-nutrients__get_all_stock` — the whole pantry (one line per product), sorted by urgency
- `mcp__grocy-nutrients__get_expiring_stock` — only items expiring soon / overdue / expired
- `mcp__grocy-nutrients__search_recipe` — fuzzy-search the user's Grocy recipes
- `mcp__grocy-nutrients__get_recipe_detail` — full ingredient/macro detail for one recipe

If these tools are not available, stop and tell the user to configure the `grocy-nutrients` MCP
server first (see [INSTALL.md](INSTALL.md)). Do **not** invent stock data.

## Workflow

### 1. Read stock

Call `get_all_stock` to see the whole pantry (each line has `days_until_expiry` /
`expiry_status`; the list is sorted by urgency). If it errors, report the error verbatim — do
not guess. A meal can use any in-stock product, not only expiring ones.

### 2. Pick drivers

From the results, choose the few items most worth cooking:
- Prefer items expiring soonest (smallest `days_until_expiry`) as the drivers to use up.
- Then fill out the dish with other in-stock staples so the recipe is actually complete.
- Skip non-cookable items: drinks, energy gels/isotonics, spices, supplements, capsules.
- Skip items already expired by many months unless the user asked to include everything.
- Group by what realistically combines into one dish (e.g. cheese + eggs + milk + a fruit).

### 3. Find 2 local recipes

Use `search_recipe` with names/themes built from the driver ingredients (search several queries in
parallel). For the best candidates, call `get_recipe_detail` to confirm the ingredient list and
show per-serving macros. Pick the **2** that best use the expiring drivers while drawing on what's
in stock.

### 4. Find 3 internet recipes

Web-search for recipes built around the same drivers (search in the user's language). Fetch the
top pages to extract real ingredient lists + steps (temperature, time). Pick **3** distinct dishes.
Always cite source URLs as markdown links. If a page returns 403/blocked, fall back to another
result rather than fabricating.

### 5. Present

Show local matches first (with a small table mapping driver → amount on hand → expiry → amount in
recipe), then the 3 internet recipes with ingredients + numbered steps + source link.

### 6. Offer to add to meal plan

After presenting all recipes, ask which one (if any) the user wants to add to today's meal plan:

> "Хочеш додати один із цих рецептів у план харчування? Якщо так — скажи який і на скільки порцій."

If the user confirms: call `mcp__grocy-nutrients__search_recipe` to get the local id, then
`mcp__grocy-nutrients__add_recipe_to_meal_plan`. For internet recipes (no local id), say that
only Grocy recipes can be added to the plan directly — offer to search for a matching local one
instead.

## Output rules

- Reply in the language the user wrote in.
- Be honest: if stock is mostly long-expired, say so and work from the freshest realistic items.
- Never claim an ingredient is in stock if it is not in the `get_all_stock` result.
- Keep macros only where the MCP returned them; do not estimate Grocy macros.
