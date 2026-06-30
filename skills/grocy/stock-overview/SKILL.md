---
name: stock-overview
description: Shows the user's current Grocy pantry, highlighting what is expiring, overdue, or expired, via the grocy-nutrients MCP server. Read-only — it just reports stock, it does not cook, plan, or change anything. Use when the user asks "що в мене є", "що скоро зіпсується", "покажи запаси", "що протерміновано", "what's in my pantry", "what's expiring", "show my stock", "what's going off soon", or wants a plain view of what they have on hand.
---

# Stock Overview

Give the user a clear picture of what they have in stock, with expiring/overdue/expired items
called out. Read-only — this skill reports; it does not suggest recipes (that's
`/recipe-from-stock`) or change anything.

## Prerequisites

Depends on the **`grocy-nutrients` MCP server**. Tools used:

- `mcp__grocy-nutrients__get_all_stock` — the whole pantry (one line per product), sorted by urgency
- `mcp__grocy-nutrients__get_expiring_stock` — only items expiring soon / overdue / expired

If these tools are not available, stop and tell the user to configure the MCP server. Do **not**
invent stock data.

## Workflow

### Step 1 — Decide scope

- "що скоро зіпсується" / "what's expiring" → call `get_expiring_stock` only.
- "що в мене є" / "show my pantry" → call `get_all_stock` (the full list).
- General / unclear → call `get_all_stock`; it is already sorted by urgency so expiring items
  surface at the top.

### Step 2 — Present

Each item has `product_name`, `amount` + `quantity_unit_name`, `best_before_date`,
`days_until_expiry`, and `expiry_status`. Render a compact view:

- Lead with anything **overdue/expired** (urgent), then **expiring soon**, then the rest.
- Use a simple urgency marker: 🔴 expired/overdue, ⚠️ expiring within a few days, ✅ fine.
- Mention `synced_at` once if the data looks stale (so the user knows the freshness).
- For a full-pantry view, group sensibly (e.g. by urgency band) rather than dumping a flat list.

### Step 3 — Offer next steps

After showing stock, optionally offer:
- "Підібрати рецепт із того, що є?" → hands off to `/recipe-from-stock`.
- "Що приготувати, щоб вкластись у норми?" → hands off to `/what-to-plan`.

## Rules

- **Read-only** — never plan, consume, or change anything from this skill.
- Never claim an item is in stock if it's not in the tool result. `days_until_expiry`/`expiry_status`
  are recomputed live by the tool — trust them over the raw `best_before_date`.
- Reply in the user's language. If a tool errors, report it verbatim — do not guess.
