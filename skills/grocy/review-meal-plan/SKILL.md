---
name: review-meal-plan
description: Shows the user's planned meals for a day or range and edits them — remove a line, change recipe servings, or clear a day — via the grocy-nutrients MCP server. This reviews and EDITS existing meal-plan entries; to ADD new items use /plan-meal. Use when the user says "покажи меню на тиждень", "що в мене заплановано", "прибери борщ з плану", "зроби 2 порції", "очисти вівторок", "show my meal plan", "what's planned this week", "remove X from the plan", "clear that day", or wants to view/tidy what's already on the plan.
---

# Review Meal Plan

Show what the user has planned and make small edits: remove a line, change a recipe's servings,
edit a note, or clear a whole day. This is the **read/edit** counterpart to `/plan-meal` (which
**adds** new entries). It writes to both the local DB and Grocy.

## Prerequisites

Depends on the **`grocy-nutrients` MCP server**. Tools used:

- `mcp__grocy-nutrients__get_meal_plan` — planned lines for a date range, each with a `removable` flag
- `mcp__grocy-nutrients__remove_from_meal_plan` — delete one line (local + Grocy)
- `mcp__grocy-nutrients__edit_meal_plan_line` — change a recipe's servings or a note's text

If these tools are not available, stop and tell the user to configure the MCP server.

## Workflow

### Step 1 — Resolve the range

Map the request to `start_date`/`end_date` (ISO or `today`/`tomorrow`/`yesterday`). "на тиждень"
→ today through today+6. "на вівторок" → that Tuesday (both start and end). A single day → start =
end.

### Step 2 — Show the plan

Call `get_meal_plan(start_date, end_date)`. Present it grouped by day (and section if useful):
each line's name, amount/servings, and unit. Note which lines are **not removable**
(`removable: false` — already consumed/done or still syncing) so the user knows what can't be
touched.

### Step 3 — Act on the request

- **Remove one line:** find its `line_id` from the shown plan, call `remove_from_meal_plan(line_id)`.
  - `{"status": "removed"}` → confirm it's gone.
  - `{"status": "cannot_remove"}` → tell the user why (already eaten/done, or still syncing) — do
    **not** retry.
- **Change recipe servings:** call `edit_meal_plan_line(line_id, servings=N)`.
  - `{"status": "updated"}` → confirm. `{"status": "cannot_edit"}` → explain (done/syncing).
  - `{"status": "invalid"}` → servings only apply to recipe lines; for a **product** amount change,
    tell the user to remove it and re-add via `/plan-meal` (this tool can't convert product units).
- **Edit a note:** call `edit_meal_plan_line(line_id, note="...")`.
- **Clear a day:** call `get_meal_plan` for that day, then call `remove_from_meal_plan` for **each**
  line where `removable: true`. Report how many were removed and how many were skipped (and why).

### Step 4 — Confirm

Summarise what changed. If you removed several lines, list them.

## Rules

- **Only edit what the user asked for.** Don't clear a day unless explicitly told to.
- **Respect `removable`/`cannot_*` results** — never force a removal of a consumed line; surface
  the reason instead.
- Product-amount edits are **out of scope** here — route them to `/plan-meal` (remove + re-add).
- **Never** consume stock — this skill only edits the plan. Reply in the user's language; quote
  tool errors verbatim.
