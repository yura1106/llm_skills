---
name: shopping-from-recipe
description: Adds a recipe's missing ingredients to the user's Grocy shopping list — the products the recipe needs that aren't in stock — via the grocy-nutrients MCP server. This writes to the shopping list; it does NOT consume stock or plan meals. Use when the user says "додай продукти для борщу в список", "що докупити для рецепта X", "add ingredients for recipe Y to my shopping list", "what do I need to buy to cook Z", or wants the shortfall for a recipe added to their list.
---

# Shopping From Recipe

Given a recipe, work out which of its ingredients the user is missing (needs minus stock) and add
those to the shopping list. Orchestrates existing tools. It writes to the **default shopping
list**; it does not plan meals or consume stock.

## Prerequisites

Depends on the **`grocy-nutrients` MCP server**. Tools used:

- `mcp__grocy-nutrients__search_recipe` — fuzzy-find the recipe → local id
- `mcp__grocy-nutrients__get_recipe_detail` — the recipe's `ingredients` (required_amount + in_stock per ingredient)
- `mcp__grocy-nutrients__add_to_shopping_list` — add a missing product to the list

If these tools are not available, stop and tell the user to configure the MCP server.

## Workflow

### Step 1 — Resolve the recipe

Call `search_recipe` with the name. If multiple plausible matches, **show them and ask** — don't
guess. Use the chosen local id.

### Step 2 — Get ingredient requirements

Call `get_recipe_detail(id)`. Read the `ingredients` list: each has `product_id`, `product_name`,
`required_amount`, `amount_in_stock`, and `in_stock`. If `ingredients` is empty, Grocy is
unreachable — tell the user the shortfall can't be computed right now and stop (don't invent it).

### Step 3 — Compute the shortfall

The missing items are the ingredients with `in_stock: false`. Optionally compute the gap
(`required_amount − amount_in_stock`) as the amount to buy; if the gap is unclear, default to the
full `required_amount`.

### Step 4 — Confirm

Show the user the missing items (name + amount to buy) and **ask to confirm** before writing.

### Step 5 — Add to the list

On confirmation, for each missing item call `add_to_shopping_list(product_id, amount, note?)`
(e.g. note "для рецепта <recipe name>").
- Handle `needs_unit` / `needs_units` exactly like `/shopping-list` (ask / report and retry).
- Report each `{"status": "added"}`.

### Step 6 — Summarise

List what was added to the shopping list. If everything was already in stock, say the user has all
the ingredients and added nothing.

## Rules

- **Confirm before writing.** Don't bulk-add without showing the shortfall first.
- Only add items flagged `in_stock: false` — never re-buy things already in stock.
- **Never** plan meals or consume stock — this only touches the shopping list.
- Never invent ingredients, amounts, or ids. Reply in the user's language; quote tool errors
  verbatim.
