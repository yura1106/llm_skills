---
name: nutrition-today
description: Shows today's nutrition progress vs daily targets — calories, protein, fat, carbs, and other tracked nutrients. Reads the current day's totals and goals from the grocy-nutrients MCP server. Use when the user asks "скільки я сьогодні з'їв", "який прогрес по харчуванню", "скільки ккал залишилось", "покажи харчування на сьогодні", "nutrition today", "how am I doing today", "what's my calorie balance", or wants a quick nutrition snapshot for the current day.
---

# Nutrition Today

Show the user a clear snapshot of today's nutrition: what they've eaten so far, what's in the
plan for later, how it compares to their daily targets, and what's left to fill.

## Prerequisites

Depends on the **`grocy-nutrients` MCP server**. Tools used:

- `mcp__grocy-nutrients__get_day` — nutrition totals, breakdown by meal, and planned (uncosumed)
  entries for a given date
- `mcp__grocy-nutrients__get_nutrition_targets` — daily targets for calories and macros

If these tools are not available, stop and tell the user to configure the MCP server.

## Workflow

### 1. Fetch data (parallel)

Call both tools simultaneously:
- `get_day("today")` — returns consumed totals, planned totals, per-meal breakdown, and any
  `omitted_lines` (entries whose nutrients couldn't be computed)
- `get_nutrition_targets("today")` — returns daily targets per nutrient

### 2. Compute remaining

For each nutrient where a target exists:
- **consumed** = already eaten (from `get_day`)
- **planned** = in the plan but not yet consumed (from `get_day`)
- **remaining** = target − consumed − planned (can be negative = over target)

### 3. Present

Render a compact table with a progress indicator for each tracked nutrient:

```
📊 Харчування · Сьогодні, 25 червня

Нутрієнт    З'їдено   В плані   Залишок   Ціль
──────────────────────────────────────────────
Калорії     1 240     580       280       2 100
Білки         68 г     32 г      30 г     130 г
Жири          52 г     18 г       0 г      70 г  ⚠️ майже вичерпано
Вуглеводи   140 г     60 г      40 г     240 г
```

Use ✅ when remaining ≥ 20% of target, ⚠️ when remaining is 0–20%, 🔴 when already over target.

After the table, add a one-sentence plain-language summary:

> "До кінця дня залишилось ~280 ккал і 30 г білка — можна додати невеликий білковий перекус."

Or if targets are exceeded:

> "По жирах вже на межі — решта дня краще без жирних страв."

### 4. Offer next actions

End with a brief offer:
- "Додати щось у план на сьогодні?" → triggers plan-meal skill
- "Що приготувати з того що є?" → triggers recipe-from-stock skill

## Output rules

- Never show nutrients for which both consumed and target are zero/null — skip them.
- If `omitted_lines` is non-empty, mention how many entries couldn't be computed and why
  (e.g. "2 продукти без даних нутрієнтів не враховані").
- If no targets are set, show consumed totals only without a remaining column, and suggest
  setting targets in the app.
- Reply in the language the user wrote in.
