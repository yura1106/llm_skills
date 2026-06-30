---
name: shopping-list
description: Shows the user's Grocy shopping list and adds items to it via the grocy-nutrients MCP server. Manual add + show only — it does NOT suggest what to buy and does NOT change stock or meal plans. Use when the user says "додай молоко в список покупок", "що мені купити", "покажи список покупок", "add milk to my shopping list", "show my shopping list", "what's on my list", or wants to put a specific item onto their shopping list.
---

# Shopping List

Show the user's Grocy shopping list and add items they name. This is **manual** — it does not
recommend what to buy and does not touch stock or the meal plan. It operates on the **default
Grocy shopping list**.

## Prerequisites

Depends on the **`grocy-nutrients` MCP server**. Tools used:

- `mcp__grocy-nutrients__get_shopping_list` — current items on the default shopping list
- `mcp__grocy-nutrients__search_product` — fuzzy-find a product, returns its local `id`
- `mcp__grocy-nutrients__add_to_shopping_list` — add a product to the list (local id + amount)

If these tools are not available, stop and tell the user to configure the MCP server. Do **not**
invent ids or write anything blindly.

## Workflow

### Show the list

When the user asks to see the list ("що в списку", "show my shopping list"), call
`get_shopping_list` and present the items: product name, amount, and note. If the list is empty,
say so plainly. Never fabricate items.

### Add an item

1. **Parse** the request into: product name, amount, unit (if any), note (if any). "додай 2 л
   молока" → name "молоко", amount 2, unit "л". "додай яйця" → name "яйця", amount 1, no unit.
2. **Resolve the id** — call `search_product`. If there are multiple plausible matches or the top
   match looks wrong, **show the options and ask** which one; do not guess.
3. **Write** — call `add_to_shopping_list(product_id, amount, unit?, note?)`.
   - If the user gave a unit, **pass it** — omitting it defaults to the product's stock unit.
4. **Handle unit clarification** (same as `/plan-meal`):
   - `{"status": "needs_unit", "available_units": [...]}` → show the unit names, ask which to use,
     then retry with that `unit`.
   - `{"status": "needs_units"}` → units couldn't be loaded from Grocy; tell the user to check the
     household's Grocy key/URL, then retry.
5. **Confirm** — on `{"status": "added"}` report what was added (item, amount, resolved unit).

## Rules

- **Manual only.** Do not suggest items, do not cross-reference stock or consumption — if the user
  wants suggestions, that's a different request.
- **Never** consume stock or write to the meal plan — this skill only touches the shopping list.
- **Never** invent product ids, amounts, or units. Ask when unclear.
- Reply in the language the user wrote in. Quote tool errors verbatim.
