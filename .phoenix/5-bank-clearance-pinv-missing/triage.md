# Triage Report — Issue #5

## Issue summary
**Title:** ERPnext 15 - Bank clearance not showing PINV - purchase invoices - paid by ticking Paid box
**Repo:** ShankHarinath/erpnext
**Reporter:** ShankHarinath
**Classification confidence (input):** 0.90

A Purchase Invoice that was paid directly via the "Is Paid" checkbox (with `cash_bank_account` = bank account 1210, and a `paid_amount`) posts a GL entry to the bank account but does **not** appear in the Bank Clearance tool's "Get Payment Entries" output — so the user cannot set its clearance date and reconcile it. Same behavior reported on v14 and v15; this is a long-standing issue, not a regression.

## Root-cause hypothesis

The Bank Clearance entry fetcher `get_payment_entries_for_bank_clearance` in `erpnext/accounts/doctype/bank_clearance/bank_clearance.py` (lines 180–300) returns four sources: Journal Entries, Payment Entries, POS Sales Invoices, and "POS" Purchase Invoices. The Purchase Invoice branch (lines 270–294) does query by `pi.cash_bank_account == account`, which is the right condition for a "Paid" purchase invoice — **but that entire branch is gated on the `include_pos_transactions` checkbox (line 238)**, which defaults to `0` (off) per `bank_clearance.json` line 72.

There are two concrete problems:

1. **Gating bug / UX mismatch (primary).** Purchase Invoice has no POS concept (no `is_pos` field in `purchase_invoice.json`). A user ticking "Is Paid" on a Purchase Invoice is performing a cash/bank payment, not a POS transaction, so they have no reason to expect "Include POS Transactions" to be relevant. When they leave it unchecked, their paid PINV is silently filtered out of Bank Clearance even though its GL entry hit the bank account. This is exactly the user's report — they see the entry on the GL of account 1210 but not in Bank Clearance.
2. **Missing `clearance_date` filter in the POS branches (secondary).** The `pos_sales_invoices` and `pos_purchase_invoices` subqueries do not apply the `condition`/`pe_condition` clearance-date filter that the JE and PE subqueries apply (lines 187–189). Consequence: toggling "Include Reconciled Entries" has no effect on already-cleared POS or paid-PINV rows, so they re-appear every time.

The `make_payment_gl_entries` method in `purchase_invoice.py` (lines 1540–1584) confirms that a paid PI with `cash_bank_account` credits that bank account — matching the user's GL observation. The sibling Bank Reconciliation Statement report (`erpnext/accounts/report/bank_reconciliation_statement/bank_reconciliation_statement.py`, function `get_pos_entries`, lines 174–192) has the **same omission** — it only selects from `Sales Invoice Payment`, never from paid `Purchase Invoice`. So this class of bug also affects that report, though it's not what the user reported.

**Confidence: 0.82.** High-confidence static analysis; I did not execute ERPNext against a live instance to reproduce, but the query text, the DocType schema, the GL entry construction, and the absence of a PI-focused test all corroborate the hypothesis.

## Evidence

### Code (Phase A)
- `/Users/shashank/Canonix/phoenix-test/erpnext/erpnext/accounts/doctype/bank_clearance/bank_clearance.py`
  - `get_payment_entries` (lines 40–88) — entry point, iterates hook implementations.
  - `get_payment_entries_for_bank_clearance` (lines 180–300) — the four subqueries.
  - Purchase Invoice subquery (lines 270–294) is inside `if include_pos_transactions:` (line 238).
  - No `condition`/`pe_condition` applied to PI/SI POS queries — only JE (line 203) and PE (line 225) respect "Include Reconciled Entries".
- `/Users/shashank/Canonix/phoenix-test/erpnext/erpnext/accounts/doctype/bank_clearance/bank_clearance.json`
  - `include_pos_transactions` field defaults to `0` (lines 71–76). Label: "Include POS Transactions" — does not mention paid Purchase Invoices.
- `/Users/shashank/Canonix/phoenix-test/erpnext/erpnext/accounts/doctype/purchase_invoice/purchase_invoice.json`
  - `is_paid` field (Check, line 276).
  - `cash_bank_account` field (Link Account, line 1027).
  - No `is_pos` field.
- `/Users/shashank/Canonix/phoenix-test/erpnext/erpnext/accounts/doctype/purchase_invoice/purchase_invoice.py`
  - `validate_cash` (lines 321–328) — enforces `cash_bank_account` when `is_paid`.
  - `make_payment_gl_entries` (lines 1540–1584) — emits GL entries debiting `credit_to` and crediting `cash_bank_account`, which is exactly what the reporter sees on bank account 1210.
- `/Users/shashank/Canonix/phoenix-test/erpnext/erpnext/accounts/doctype/bank_clearance/test_bank_clearance.py`
  - `TestBankClearance` has `test_bank_clearance` (Payment Entry path) and `test_update_clearance_date_on_si` (POS Sales Invoice path). **No test for a paid Purchase Invoice** — explains how the gap persisted.
- `/Users/shashank/Canonix/phoenix-test/erpnext/erpnext/accounts/report/bank_reconciliation_statement/bank_reconciliation_statement.py`
  - `get_pos_entries` (lines 174–192) — same omission of paid Purchase Invoices.
- Blast radius from GitNexus `impact(get_payment_entries_for_bank_clearance, upstream)`: zero static callers (invoked via `frappe.get_hooks` dynamic dispatch). **Risk: LOW.** The function's contract is a list-of-dicts with a known shape.

### Runtime (Phase B)
Skipped. This is a deterministic filter/selection bug in Python/SQL — there is no runtime crash, no log line, and no `kind-canonix` cluster hosting a reproduction of this ERPNext install. Static evidence is sufficient.

### Git history (Phase C)
Recent commits on `bank_clearance.py` (none touch the Purchase Invoice POS branch):
- `e8510287e3` chore: remove unused imports
- `24c8cfe128` fix: add comment and validation for clearance date updation
- `62acc4aeb5` fix: update button was getting freezed after validation
- `302ff49b7f` Merge PR #49470 — bank-clearance-tax-calculation
- `d9a72c1e61` feat: reference for POS SI payments (#39523, Feb 2024) — the last commit that touched the PI/SI POS subqueries, by Gursheen Kaur Anand. Rewrote the SI/PI queries using `frappe.qb` but did not introduce a PI-paid path; the gating on `include_pos_transactions` predates it.

No recent commit plausibly introduced the bug — consistent with the reporter's observation that V14 has it too. This is a structural gap from the original feature.

### Related issues / PRs (Phase D)
Only issue #5 exists on `ShankHarinath/erpnext` for this keyword set. The `gh` shim blocks queries against `frappe/erpnext`, so I cannot check upstream for prior reports. Recommend Architect check `frappe/erpnext` manually for history if relevant.

## Proposed fix direction (non-binding — Architect decides)

Two reasonable directions, listed in rough order of preference:

1. **Unconditionally include paid Purchase Invoices & POS Sales Invoices in the base query** (drop the `include_pos_transactions` gate for the PI branch, or at least rename the checkbox and have it default-on). Concretely:
   - In `bank_clearance.py`, move the `pos_purchase_invoices` query out of the `if include_pos_transactions:` block (or run it whenever `is_paid`-style PIs reference `account`).
   - Apply the `condition` / `pe_condition` clearance-date filter to both `pos_sales_invoices` and `pos_purchase_invoices` so "Include Reconciled Entries" works consistently.
   - Optionally rename `include_pos_transactions` to a clearer label (or split into two flags) — though that is a DocType/JSON migration and may be out of scope.
2. **Minimal patch**: apply the `condition`/`pe_condition` filter to the POS subqueries AND document the "Include POS Transactions" label to include paid Purchase Invoices. Less invasive but leaves the UX trap in place.

Whichever direction is chosen, add a regression test in `test_bank_clearance.py`: create a Purchase Invoice with `is_paid=1`, `cash_bank_account=<bank>`, submit, then assert `get_payment_entries` returns it (without toggling `include_pos_transactions`).

## Open questions

- Does Architect want to also fix the sibling `bank_reconciliation_statement.py` report in the same PR, or leave it for a separate issue? The fix is small and the omission is identical. (Only ShankHarinath/erpnext is in scope for the PR, so this is a same-repo decision.)
- Upstream `frappe/erpnext` could not be checked via `gh` due to the owner allowlist — worth a manual pass to confirm no pending PR already addresses this.

## Affected repos

Only `ShankHarinath/erpnext` requires a code change. The symptom, the filter, the DocType, the GL entry construction, and the test gap are all inside this one Python/JSON module set. No cross-repo contracts apply (group `backend` exists but Bank Clearance is an in-tree accounts DocType, not an API contract).

---

```yaml
affected_repos:
  - ShankHarinath/erpnext
confidence: 0.82
recommended_next_step: plan
```
