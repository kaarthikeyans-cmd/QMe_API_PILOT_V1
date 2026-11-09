# Step 1 notes: where quote calculation runs today (checked 2026-11-09)

- The calculation engine is TypeScript: `packages/core/calcTotals.ts` and `gst.ts` in the QME repo (line values, discounts,
  CGST/SGST/IGST per line, cess, round-off). The React app imports it everywhere (editor, customer page, PDF, lists, invoices).
- A Deno copy exists at `supabase/functions/_shared/calcTotals.ts` and `gst.ts` (only the `.ts` import suffix differs). It is used
  only by reporting functions. Nothing that creates a quote uses it. The two copies are kept in step by hand.
- No database function calculates a quote's tax or total. When a quote is sent, the browser works the totals out and calls
  `attach_quote_version(p_totals)`. The database checks that the taxable amount equals the sum of the line values (within one
  paisa per line) and freezes `frozen_total_paise`. The tax split and rounding are trusted from the engine.
- No public API or API keys exist today. Anonymous entry points: the enquiry form (`capture-lead`) and the customer quote link.
- Sending must follow the existing guarded path: `next_quote_number`, the frozen snapshot, and the member-approval rules.

## Open decisions
1. Who calls the API (own website, an ERP, a partner)?
2. Authentication: a signed-in user's token, or a per-company API key?
3. How the engine is shared with this repo (import from QME's `packages/core`, or a tested copy).
