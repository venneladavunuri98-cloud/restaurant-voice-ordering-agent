# `voiceai` schema (PostgreSQL / Supabase)

| Table | Purpose | Key design points |
|---|---|---|
| `branches` | Outlet master data | timezone, currency, open/close time, `is_active` respected by every query |
| `branch_phone_numbers` | Dialled number → branch | PK on phone number; the backbone of multi-branch routing |
| `branch_hours` | Per-day opening hours | composite PK (branch, day_of_week 0–6), `is_closed` |
| `menu_items` | Menu | price ≥ 0, tax rate, spice level, veg/vegan/Jain/gluten-free/halal flags, allergens array, bestseller/seasonal/available |
| `customers` | Loyalty | unique phone with `CHECK (phone ~ '^\+?[0-9]{10,15}$')`, loyalty points/tier |
| `promotions` | Active offers | unique promo code, percentage/fixed discount, date range |
| `calls` | One row per call | text PK from the voice platform; branch and customer resolved progressively; summary, ended reason |
| `orders` | One draft order per call | `UNIQUE(call_id)`; status `draft → handed_off` |
| `order_items` | Cart lines | name and unit-price **snapshots**, quantity > 0, modifiers JSONB, generated `line_total` |
| `call_special_instructions` | Verbatim caller notes | de-duplicated per call with an `is_active` flag |
| `call_tool_invocations` | Audit log | `UNIQUE(tool_call_id)` for idempotent inserts; args and result as JSONB |
| `customers_phone_quarantine` | Staging table | Not used by the workflows |

Plus a function `get_branch_open_status(branch_id)` that returns `is_open, today_open_time, today_close_time, is_closed_today, next_open_at`.

## Idempotency patterns used
```sql
-- first tool call creates the call row; later calls never overwrite a resolved branch
INSERT INTO voiceai.calls (call_id, branch_id, started_at) VALUES ($1, $2, now())
ON CONFLICT (call_id) DO UPDATE SET branch_id = COALESCE(voiceai.calls.branch_id, EXCLUDED.branch_id);

-- audit rows can be retried safely
INSERT INTO voiceai.call_tool_invocations (call_id, tool_call_id, tool_name, arguments, result, success)
VALUES ($1, $2, $3, $4::jsonb, $5::jsonb, $6)
ON CONFLICT (tool_call_id) DO NOTHING;
```
`manageCart` handles add, remove, update quantity and clear in a **single CTE statement**, and only adds items that exist, are available and belong to the caller's branch.
