# Triage Report — Issue #7 (rev 1)

**Repo:** ShankHarinath/erpnext
**Issue:** [#7 — Issue with Subcontracting Module - Excess Issue-Receipt Tolerance Not working as expected](https://github.com/ShankHarinath/erpnext/issues/7)
**Reporter:** ShankHarinath · **Created:** 2026-04-20
**Pre-fix SHA investigated:** `cb5926f59b9639f71def086f8b6edfbe9d174efd`

## Summary

Once a Subcontracting Order's `per_received` reaches 100 % (PO qty = 100, received = 100), ERPNext hides the "Create → Subcontracting Receipt" button and flips the SCO status to `Completed`, even if the configured *over-delivery / over-receipt* tolerance would otherwise allow a further receipt (e.g. 110 pcs). The reporter's scenario (PO 100, issued 110 under `over_transfer_allowance = 10 %`, received 102 so far, still needs 8 more) cannot proceed — the balance 8 pcs can only be booked by a convoluted return-and-reorder workaround.

The issue splits into three distinct code defects, all on the Subcontracting Order side, none on the receipt side (the receipt's actual `status_updater`/`over_delivery_receipt_allowance` validator *would* accept the overage if invoked).

## Root-cause hypothesis

Three linked defects all reduce to a single missing concept: the SCO never looks up `Stock Settings.over_delivery_receipt_allowance` (or per-Item override) when deciding whether more receipts can still be made. The code unconditionally compares against 100 %.

### Defect 1 — UI button gated strictly at `per_received < 100`
`erpnext/subcontracting/doctype/subcontracting_order/subcontracting_order.js:527-533`
```js
if (!["Closed", "Completed"].includes(doc.status)) {
    if (flt(doc.per_received) < 100) {
        this.frm.add_custom_button(__("Subcontracting Receipt"),
            this.make_subcontracting_receipt, __("Create"));
        ...
```
When `per_received == 100`, the button never gets added. Also, the outer guard on `doc.status` means once status transitions to `Completed` (see Defect 2), the whole `if` block is skipped.

### Defect 2 — Server flips status to `Completed` at exactly 100 %
`erpnext/subcontracting/doctype/subcontracting_order/subcontracting_order.py:325-347` (`update_status`)
```python
elif self.per_received >= 100:
    status = "Completed"
elif self.per_received > 0 and self.per_received < 100:
    status = "Partially Received"
```
No tolerance factored in. The moment `per_received ≥ 100`, status becomes `Completed`, which then makes the JS button block (Defect 1) also short-circuit on `!["Closed","Completed"].includes(doc.status)`.

A matching guard also exists in the PO pre-check used when creating a *new* SCO:
`erpnext/subcontracting/doctype/subcontracting_order/subcontracting_order.py:158`
```python
if po.per_received == 100:
    msg = f"Cannot create more Subcontracting Orders against the Purchase Order {po.name}."
    frappe.throw(_(msg))
```
(Not the primary cause of this bug but symptomatic of the same pattern.)

### Defect 3 — Mapper strips fully-received rows → the reporter's bullet #7
`erpnext/subcontracting/doctype/subcontracting_order/subcontracting_order.py:470-505` (`get_mapped_subcontracting_receipt`)
```python
def update_item(source, target, source_parent):
    ...
    target.qty = flt(source.qty) - flt(source.received_qty)   # can go negative / zero
    target.amount = (flt(source.qty) - flt(source.received_qty)) * flt(source.rate)
...
"condition": lambda doc: abs(doc.received_qty) < abs(doc.qty),
```
For a **multi-item** SCO where one line has already reached `received_qty ≥ qty`, `get_mapped_doc` skips that row entirely (condition `False`). The reporter's description matches exactly: *"this item is deleted by the system while creating a new receipt"*. Even if the button were visible (Defect 1 + 2 fixed), the excess qty of a fully-received item cannot be booked because the row never makes it into the target receipt.

Additionally, `target.qty = source.qty − source.received_qty` produces a non-positive default even when tolerance would still allow more. The postprocess should clamp qty to a sensible value (e.g. `max(0, tolerance_qty − received_qty)` or leave at 0 so the user can edit up to the tolerance ceiling).

### Why the underlying validator wouldn't block the receipt
`SubcontractingReceipt.status_updater` (`erpnext/subcontracting/doctype/subcontracting_receipt/subcontracting_receipt.py:100-113`) uses the generic `status_updater` framework, which respects `Item.over_delivery_receipt_allowance` and the `Stock Settings.over_delivery_receipt_allowance` global (see `erpnext/controllers/status_updater.py:745-754`). So submission of an over-tolerance receipt is permitted server-side — proving the bug is purely in the *approach path* to creating that receipt.

## Evidence

### Phase A — Code map (repo: erpnext)
| Signal | Location | Role |
|---|---|---|
| "Create > Subcontracting Receipt" button visibility | `erpnext/subcontracting/doctype/subcontracting_order/subcontracting_order.js:523-549` | Defect 1 |
| `SubcontractingOrder.update_status` (`Completed` at 100 %) | `erpnext/subcontracting/doctype/subcontracting_order/subcontracting_order.py:325-347` | Defect 2 |
| PO gate (`po.per_received == 100`) | `erpnext/subcontracting/doctype/subcontracting_order/subcontracting_order.py:158` | Same pattern |
| `make_subcontracting_receipt` / `get_mapped_subcontracting_receipt` | `erpnext/subcontracting/doctype/subcontracting_order/subcontracting_order.py:466-505` | Defect 3 |
| `SubcontractingReceipt.status_updater` (tolerance-aware) | `erpnext/subcontracting/doctype/subcontracting_receipt/subcontracting_receipt.py:100-113` | Counter-evidence |
| Item/global tolerance lookup | `erpnext/controllers/status_updater.py:745-754` | Counter-evidence |
| Buying-Settings `over_transfer_allowance` (issue side only) | `erpnext/subcontracting/doctype/subcontracting_order/subcontracting_order.py:110-111`; `subcontracting_order.js:572-576` (`has_unsupplied_items`) | Shows the issue-side tolerance already plumbed; receipt side has no analogue |

### Phase B — Runtime logs
Skipped per config: this is a local source repo (`kind-canonix` has no erpnext deployment). No runtime evidence attached.

### Phase C — Recent commits on the affected files
`git log -- erpnext/subcontracting/doctype/subcontracting_order/subcontracting_order.{py,js}` shows no touch to `update_status`, `get_mapped_subcontracting_receipt`, or the JS `refresh` in the last 24 months aside from unrelated work (feature `f2b948a` inward subcontracting, fix `57f9353d90` precision). The behaviour is long-standing, not a regression — correlation with any recent commit: **none**.

### Phase D — Similar past issues / PRs
`gh issue list --repo ShankHarinath/erpnext --search "subcontracting tolerance"` → only issue #7 itself. No prior PR on this fork addresses this region. Upstream `frappe/erpnext` lookups are disabled per counterfactual gate; no leak attempted.

### Phase E — Confidence
**0.88** — three concrete, independently verifiable defects fully account for every bullet in the reporter's description (including the subtle multi-item bullet #7), the receipt-side validator already honours tolerance so no deep architectural change is needed, and no recent commit noise complicates the fix.

## Recommended fix sketch (Architect will refine)

1. Introduce a helper `SubcontractingOrder.get_over_receipt_allowance()` that resolves per-item `over_delivery_receipt_allowance` → falling back to the global `Stock Settings.over_delivery_receipt_allowance` (see existing helper `erpnext/controllers/status_updater.py:745-754`). Expose via `onload` like `over_transfer_allowance` already is.
2. Replace `per_received >= 100` in `update_status` with `per_received >= 100 + allowance_pct` (row-level fallback: aggregate the per-item allowances). Mirror on PO pre-check at line 158.
3. `subcontracting_order.js:528` — change `flt(doc.per_received) < 100` to the allowance-aware bound, reading `__onload.over_receipt_allowance`.
4. `get_mapped_subcontracting_receipt`:
   - Relax the mapper `condition` to `received_qty < qty * (1 + allowance/100)` instead of `received_qty < qty`.
   - In `update_item`, clamp `target.qty = max(0, qty_ceiling − received_qty)` so fully-received items still appear in the mapped receipt at qty 0 (reporter's explicit request in bullet #7).
5. Add tests: extend `test_subcontracting_order.py` / `test_subcontracting_receipt.py` with a case that exercises 10 % over-receipt across multiple receipts and a multi-item SCO where one line is already at qty.

## Blast radius (per `gitnexus_impact`)
- `get_mapped_subcontracting_receipt` — upstream impact: LOW, one direct caller (`make_subcontracting_receipt` whitelisted wrapper). Safe to modify.
- `SubcontractingOrder.update_status` — widely used across SCO submit/cancel/receipt flows; changes MUST be evaluated with an impact run before editing (per CLAUDE.md). Architect: run `gitnexus_impact({target:"update_status", direction:"upstream"})` at plan time.

## Open questions / blockers
- **None blocking triage.** One product question for Architect: should the tolerance be read at *SCO creation* (snapshot) or at *each refresh* (live)? Existing `over_transfer_allowance` is snapshotted via `__onload`, so mirroring that pattern is the low-risk choice.
- `kubectl` Phase B skipped — no runtime deployment for this repo; no issue.

## Affected files (change scope)
- `erpnext/subcontracting/doctype/subcontracting_order/subcontracting_order.py` (update_status, make_subcontracting_receipt/get_mapped_subcontracting_receipt, onload, PO pre-check)
- `erpnext/subcontracting/doctype/subcontracting_order/subcontracting_order.js` (refresh → button visibility)
- `erpnext/subcontracting/doctype/subcontracting_order/test_subcontracting_order.py` (new test cases)
- `erpnext/subcontracting/doctype/subcontracting_receipt/test_subcontracting_receipt.py` (new test cases)

---

<!--phoenix:scout-summary
affected_repos:
  - ShankHarinath/erpnext
confidence: 0.88
recommended_next_step: proceed
primary_suspects:
  - erpnext/subcontracting/doctype/subcontracting_order/subcontracting_order.py:325-347   # update_status, 100% hard-coded
  - erpnext/subcontracting/doctype/subcontracting_order/subcontracting_order.py:466-505   # get_mapped_subcontracting_receipt, mapper condition + qty clamp
  - erpnext/subcontracting/doctype/subcontracting_order/subcontracting_order.js:523-549   # refresh(), button visibility on per_received<100
  - erpnext/subcontracting/doctype/subcontracting_order/subcontracting_order.py:158       # PO per_received==100 guard
phase_b_runtime_evidence: skipped_no_deployment
phase_d_prior_fix: none_on_fork
-->
