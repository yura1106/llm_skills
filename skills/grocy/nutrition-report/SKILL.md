---
name: nutrition-report
description: Analyzes nutrition trends over a week or month — where the user consistently falls short, where they overshoot, and what patterns emerge by day of week or meal. Uses get_nutrition_history from the grocy-nutrients MCP server. Use when the user asks "проаналізуй моє харчування за тиждень", "де я не добираю", "покажи тренди харчування", "nutrition report", "weekly nutrition summary", "what's my average calorie intake", "where am I missing nutrients", or wants to understand patterns in their eating over time.
---

# Nutrition Report

Analyze the user's nutrition history over a time period and surface patterns: consistent
shortfalls, systematic overshoots, best and worst days, and day-of-week trends. The goal is
actionable insight, not just averages.

## Prerequisites

Depends on the **`grocy-nutrients` MCP server**. Tools used:

- `mcp__grocy-nutrients__get_nutrition_history` — daily totals per nutrient over a date range
- `mcp__grocy-nutrients__get_nutrition_targets` — daily targets to compare against

If these tools are not available, stop and tell the user to configure the MCP server.

## Workflow

### 1. Determine the period

If the user specified a range ("за тиждень", "last month", "за червень"), use it.
Default to the **last 7 days** if no period is mentioned.

Map to ISO dates: `start = today − N days`, `end = today − 1` (exclude today — it's incomplete).

### 2. Fetch data (parallel)

- `get_nutrition_history(start, end)` — one row per day, each row has totals per nutrient
- `get_nutrition_targets(today)` — daily targets (assumed stable; use as baseline)

### 3. Compute statistics

For each nutrient that has a target:
- **average** consumed over the period
- **days on target** (within ±10% of target)
- **days under** / **days over**
- **worst day** (furthest from target in the bad direction)
- **best day** (closest to target or best surplus on protein/fiber)

Also compute:
- **day-of-week pattern** — is Monday consistently low on protein? Are weekends high in calories?
- **trend** — is the last 3 days better or worse than the first 3?

### 4. Present

Start with a one-paragraph executive summary in plain language:

> "За останні 7 днів ти в середньому їв 1 780 ккал при цілі 2 100 — стабільно ~15% нижче.
> Білок найслабший: 68 г при цілі 130 г — норму дотримано лише 1 день із 7. Жири і вуглеводи
> загалом у нормі. Найкращий день — вівторок, найгірший — неділя."

Then a table per nutrient:

```
Нутрієнт    Ціль    Середнє   На нормі   Низько   Високо
──────────────────────────────────────────────────────────
Калорії    2 100    1 780 🔴    2 / 7       5        0
Білки       130 г    68 г 🔴    1 / 7       6        0
Жири         70 г    65 г ✅    5 / 7       2        0
Вуглеводи  240 г   210 г ✅    4 / 7       3        0
```

Then highlight the top 2–3 actionable patterns:

> **📌 Що робити:**
> 1. Білок критично низький — особливо в неділю та понеділок. Додай білковий сніданок або перекус
>    у план на ці дні.
> 2. Калорії стабільно нижче цілі — схоже, пропускаєш вечерю або перекус. Спробуй планувати
>    другий сніданок.

### 5. Offer next actions

- "Додати білковий сніданок у план на завтра?" → triggers plan-meal
- "Знайти рецепти з високим вмістом білка?" → triggers recipe-from-stock with protein focus

## Output rules

- Skip nutrients with no target set and less than 3 days of data.
- If fewer than 3 days have data in the range, say so and note results may not be representative.
- Day-of-week analysis requires at least 2 weeks of data — skip it for shorter ranges.
- Never fabricate data for days with no entries — omit those days from averages.
- Reply in the language the user wrote in.
