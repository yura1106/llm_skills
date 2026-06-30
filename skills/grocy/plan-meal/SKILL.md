---
name: plan-meal
description: Adds products and recipes to the user's Grocy meal plan for a given day via the grocy-nutrients MCP server. This PLANS meals (writes meal-plan entries to local DB + Grocy) — it does NOT log consumption or change stock. Use when the user says "додай X в меню", "заплануй на завтра", "додай рецепт борщу на сьогодні", "add X to my meal plan", "plan Y for tomorrow", or otherwise wants to put a product/recipe onto a day of their meal plan.
---

# Plan Meal

Add products and recipes to the user's meal plan for a day. This is **planning, not eating** —
it writes meal-plan entries to both the local DB and Grocy. It does **not** consume stock or log
nutrition; the user handles consuming the plan separately, themselves.

## Prerequisites

Depends on the **`grocy-nutrients` MCP server**. Tools used:

- `mcp__grocy-nutrients__get_day` — today's nutrition totals + targets (for context)
- `mcp__grocy-nutrients__search_product` — fuzzy-find a product, returns its local `id`
- `mcp__grocy-nutrients__search_recipe` — fuzzy-find a recipe, returns its local `id`
- `mcp__grocy-nutrients__get_recipe_detail` — macro totals per serving (for context)
- `mcp__grocy-nutrients__add_product_to_meal_plan` — write a product entry (local + Grocy)
- `mcp__grocy-nutrients__add_recipe_to_meal_plan` — write a recipe entry (local + Grocy)

If these tools are not available, stop and tell the user to configure the MCP server. Do **not**
invent ids or write anything blindly.

## Workflow

### Step 0 — Nutrition context (today only)

If the target day is **today**, call `get_day("today")` before writing anything. Use the result
to show a one-line context before the confirmation:

> "Сьогодні: 1 240 / 2 100 ккал · Б 68 / 130 г · Ж 52 / 70 г · В 140 / 240 г"

Then, after resolving what the user wants to add, estimate its contribution and warn if any
nutrient target would be exceeded:

> "⚠️ Борщ на 2 порції додасть ~580 ккал — перевищить денну ціль на 280 ккал. Додати?"

For recipes: use `get_recipe_detail` to get per-serving macros before writing.
For products: skip the estimate if no macro data is available (don't block the flow).
For **future days**: skip step 0 — nutrition context is irrelevant then.

### Step 1 — Parse the request

Parse each item into: name, amount, unit (if any), day, and meal section (if any).

- **Day:** map natural language to what the tool accepts — an ISO date (`YYYY-MM-DD`) or the
  literal aliases `today` / `tomorrow` / `yesterday`. "на завтра" → `tomorrow`, "на сьогодні"
  → `today`, "в п'ятницю" → resolve to the next Friday's ISO date.
- **Amount/unit:** "2 банки квасолі" → amount 2, unit "банка". "200 г сиру" → amount 200,
  unit "g". A recipe takes **servings** instead ("борщ на 2 порції" → servings 2).
- **Section:** "на сніданок" → section "Сніданок"; if none mentioned, omit it (defaults to
  the unsectioned slot).

### Step 2 — Decide product vs recipe

A cooked dish / named recipe → recipe. A single grocery item → product. If unsure, prefer
`search_recipe` first for dish-like names, else `search_product`.

### Step 3 — Resolve the id

Call `search_product` / `search_recipe` with the name. If there are multiple plausible matches
or the top match looks wrong, **show the user the options and ask** which one — do not guess.
Use the local `id` of the chosen match.

### Step 4 — Write the entry

- Recipe: `add_recipe_to_meal_plan(recipe_id, servings, date, section?)`.
- Product: `add_product_to_meal_plan(product_id, amount, date, unit?, section?)`.
- **CRITICAL — unit handling:** If the user mentioned any unit (гр, г, кг, мл, л, порція, банка,
  etc.), you MUST pass `unit`. Omitting it causes the tool to silently default to the product's
  stock unit, which is often wrong (e.g. "Порція" instead of "Грам"). Pass the unit string as the
  user said it — if the tool doesn't recognise it, it returns `needs_unit` with valid options
  (see step 5). Only omit `unit` when the user gave no unit at all (e.g. "додай 1 яблуко").

### Step 5 — Handle unit clarification (products only)

The tool may return:
- `{"status": "needs_unit", "available_units": [...]}` — the unit was unclear or not found.
  **Show the user the available unit names and ask which to use**, then call again with that
  `unit`. (E.g. "Я можу записати в банках або грамах — скільки саме?")
- `{"status": "needs_units", ...}` — no units are cached for that product. Tell the user to
  **open that product once in the app** to warm the cache, then retry.

### Step 6 — Confirm

On `{"status": "queued"}` the entry is enqueued (local row written; Grocy POST runs in the
background). Report what was added — item, amount/servings, unit, day, section — succinctly.
If several items were requested, do them all, then summarise.

## Rules

- **Never** call a consumption/stock tool — this skill only plans. If the user actually wants to
  log that they *ate* something (decrement stock), say that's a separate flow not covered here.
- **Never** invent product/recipe ids, amounts, or units. Ask when unclear.
- One item at a time end-to-end, but you may batch the final confirmation.
- Quote tool errors verbatim; don't paper over them.
- Nutrition context is informational only — always add what the user asked for unless they
  explicitly say to cancel after seeing the warning.
