---
name: product-info
description: Looks up one product or recipe in the user's Grocy catalog — nutrition per 100g / per serving, ingredient requirements, and when they last ate it — via the grocy-nutrients MCP server. Read-only lookup. Use when the user asks "скільки білка в X", "яка калорійність Y", "коли я останній раз їв Z", "що входить в рецепт борщу", "how much protein is in X", "calories in Y", "when did I last eat Z", "what's in recipe W", or wants details about a specific product or recipe.
---

# Product Info

Answer focused questions about a single product or recipe: its nutrition, its ingredients (for
recipes), and the user's own consumption history of it. Read-only.

## Prerequisites

Depends on the **`grocy-nutrients` MCP server**. Tools used:

- `mcp__grocy-nutrients__search_product` — fuzzy-find a product → local id + per-100g macros + `last_consumption`
- `mcp__grocy-nutrients__search_recipe` — fuzzy-find a recipe → local id + per-serving macros
- `mcp__grocy-nutrients__get_product_detail` — product nutrient-data history + consumption history
- `mcp__grocy-nutrients__get_recipe_detail` — recipe per-serving history, last consumed-products breakdown, and `ingredients` (required amount + in_stock per ingredient)

If these tools are not available, stop and tell the user to configure the MCP server.

## Workflow

### Step 1 — Product or recipe?

A single grocery item → product. A cooked dish / named recipe → recipe. If unsure, try
`search_recipe` for dish-like names, else `search_product`.

### Step 2 — Resolve the id

Call `search_product` / `search_recipe`. If multiple plausible matches, **show them and ask** —
don't guess. The search result already carries headline macros and (for products)
`last_consumption`, which may be enough to answer simple questions without a detail call.

### Step 3 — Answer with the right depth

- **Simple macro question** ("скільки білка в X", "calories in Y") → answer straight from the
  search result's per-100g (product) or per-serving (recipe) numbers.
- **"When did I last eat X"** → use the product's `last_consumption` (from search) or
  `get_product_detail` for the fuller consumption history.
- **Recipe ingredients / "what's in it" / "can I make it"** → call `get_recipe_detail` and use the
  `ingredients` list: each has `required_amount` and `in_stock`. Summarise what's needed and flag
  anything not in stock. (To actually add the missing items to the shopping list, hand off to
  `/shopping-from-recipe`.)
- **Deeper history / trends** → `get_product_detail` / `get_recipe_detail` for the full history.

### Step 4 — Present

Give the specific number(s) asked for, with units, plainly. Don't dump the whole detail object —
extract what answers the question.

## Rules

- **Read-only** — never plan, consume, or change anything.
- Only report macros the tool returned; never estimate Grocy nutrition yourself.
- If `ingredients` is empty (Grocy unreachable), say the ingredient breakdown isn't available right
  now rather than inventing one.
- Reply in the user's language; quote tool errors verbatim.
