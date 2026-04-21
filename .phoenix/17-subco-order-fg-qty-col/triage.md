# Triage Report — Issue #17

## Summary

Submission of a Subcontracting Order that originates from a Material Request (Purpose = Subcontracting) fails with `MySQLdb.OperationalError: (1054, "Unknown column 'fg_item_qty' in 'SELECT'")`. The `StatusUpdater._update_children` method overrides `source_field` to `fg_item_qty` for the MR-Subcontracting branch but leaves `source_dt` pointing at `Subcontracting Order Item` — which has no `fg_item_qty` column. That column lives only on `Subcontracting Order Service Item`. Single-line fix at `erpnext/controllers/status_updater.py:519` — add `source_dt: "Subcontracting Order Service Item"` to the existing `args.update(...)` call. This is a reopen of frappe/erpnext#50673 that was never actually resolved in the v16 line.

## Root cause

**File:** `erpnext/controllers/status_updater.py`
**Lines:** 508–565 (`_update_children`), specifically 514–519.

```python
# status_updater.py:514–519 (current, buggy)
if (
    d.get("material_request")
    and frappe.db.get_value("Material Request", d.material_request, "material_request_type")
    == "Subcontracting"
):
    args.update({"source_field": "fg_item_qty"})        # <-- forgot source_dt
```

Context — `SubcontractingOrder.__init__` registers:

```python
# erpnext/subcontracting/doctype/subcontracting_order/subcontracting_order.py:95–107
self.status_updater = [{
    "source_dt": "Subcontracting Order Item",
    "target_dt": "Material Request Item",
    "join_field": "material_request_item",
    "target_field": "ordered_qty",
    "source_field": "qty",
    ...
}]
```

When `on_submit → update_prevdoc_status → update_qty → _update_children` runs, the per-row branch at line 514 flips `source_field` to `fg_item_qty` without flipping `source_dt`. The SQL at lines 547–556 then becomes:

```sql
select ifnull(sum(fg_item_qty), 0)
  from `tabSubcontracting Order Item`    -- wrong table
 where `material_request_item` = '...'
   and (docstatus=1 or parent='SC-ORD-2026-00002')
```

`tabSubcontracting Order Item` has no `fg_item_qty` column (verified against `erpnext/subcontracting/doctype/subcontracting_order_item/subcontracting_order_item.json` — only `qty`, `stock_qty`, `received_qty`, etc.). `fg_item_qty` lives on `tabSubcontracting Order Service Item` (verified against `erpnext/subcontracting/doctype/subcontracting_order_service_item/subcontracting_order_service_item.json:92`), which also has `material_request_item` (line 149) — so `join_field = material_request_item` is still valid after swapping `source_dt`.

The regression was introduced in a single commit:

- `b7699012b2` — *feat: Create subcontracted PO from Material Request (#44745)* — Mihir Kandoi, 2024-12-23.
  The commit added the `if material_request + Subcontracting` branch but specified only `source_field`, not `source_dt`. No test exercises the MR→PO→SCO path end-to-end (the added tests in `test_material_request.py` cover creation of the sub-PO, not downstream SCO submission).

## Evidence

### Code-level
- `erpnext/controllers/status_updater.py:519` — the buggy `args.update({"source_field": "fg_item_qty"})` (no `source_dt`).
- `erpnext/controllers/status_updater.py:547–556` — the SQL template that references `{source_field}` and `{source_dt}` independently; both must match the same doctype.
- `erpnext/subcontracting/doctype/subcontracting_order/subcontracting_order.py:95–107` — base `status_updater` config with `source_dt = Subcontracting Order Item`.
- `erpnext/subcontracting/doctype/subcontracting_order_item/subcontracting_order_item.json` — has `material_request`, `material_request_item` but NO `fg_item_qty`.
- `erpnext/subcontracting/doctype/subcontracting_order_service_item/subcontracting_order_service_item.json:92,149` — has both `fg_item_qty` and `material_request_item`.

### Commit correlation
- Introducing commit: `b7699012b2` (PR #44745). All seven added lines land in the `_update_children` method; no other call site exists.
- Confirmed via `git log -L 515,520:erpnext/controllers/status_updater.py`: exactly one commit touches this block.

### Historical hits
- Issue references frappe/erpnext#50673 (closed without fix per reporter). The upstream repo is not reachable from this environment (allowlist denies `frappe` owner), so I cannot fetch the prior thread directly, but the reporter has already summarized it.
- No prior PR in `ShankHarinath/erpnext` addresses this. Search results empty for `fg_item_qty` / `subcontracting material request`.

### Runtime (kubectl)
- `kind-canonix` cluster is offline (`connection refused` on API endpoint). No live-pod log corroboration collected. Not required — the issue payload contains a canonical stack trace rooted exactly at `status_updater.py:552`, which matches the SQL template location.

## Impact analysis (upstream of `_update_children`)

- **Risk: CRITICAL** — 27 upstream symbols, 15 processes, 6 modules.
- d=1: `StatusUpdater.update_qty` (will be exercised on every submit/cancel of Sales Order, PO, DN, PR, SI, PI, Pick List, Packing Slip, MR, Installation Note, Subcontracting Order, Subcontracting Receipt).
- The proposed fix is inside the inner `if d.get("material_request") and … == "Subcontracting"` branch, which in the current codebase is only reachable when processing a `Subcontracting Order Item` row (the only child table among all callers that has a `material_request` field pointing at a MR of type `Subcontracting`). So the effective blast radius of the fix is limited to the `SubcontractingOrder.on_submit` / `on_cancel` code path — low regression risk for the other 14 callers.

## Recommended fix

Single-line addition inside the existing guard at `erpnext/controllers/status_updater.py:519`:

```python
# before
args.update({"source_field": "fg_item_qty"})

# after
args.update({
    "source_field": "fg_item_qty",
    "source_dt": "Subcontracting Order Service Item",
})
```

Additionally, the `if d.doctype != args["source_dt"]: continue` skip at line 511 evaluates before the override. On a `Subcontracting Order` document, `get_all_children()` yields rows from both `Subcontracting Order Item` and `Subcontracting Order Service Item`. Today it matches `Subcontracting Order Item` rows (base config), flips `source_field`, and fails on the SQL. After the fix, the branch will be reached from a `Subcontracting Order Item` row but queries the Service Item table via `source_dt` — that is the desired behavior and is what the reporter proposed. However, a cleaner variant is to update `args` based on the current row's doctype rather than the source row's doctype. The minimal patch matches the reporter's suggestion; an optional follow-up is to refactor the loop so the `source_dt` swap is expressed as a separate entry in `self.status_updater` on the SCO side instead of an inline override — but that's scope creep and should wait.

### Regression test to add

`test_subcontracting_order.py`: end-to-end case that (1) creates a Material Request with `material_request_type = Subcontracting`, (2) creates a Subcontracted Purchase Order from it, (3) creates the auto-generated Subcontracting Order, (4) submits the SCO and asserts no SQL error + MR's `per_ordered` / `ordered_qty` updates. This exact flow is the one the PR #44745 feature added but never covered.

## Affected repos

Only `ShankHarinath/erpnext`. No cross-service contracts touched; both the buggy code and the fix location are in a single file within a single Python service.

## Open questions / blockers

- Unable to fetch frappe/erpnext#50673 (upstream allowlist). The reporter's summary is authoritative but the upstream attempt(s) at a fix are opaque. Architect may want to check whether an upstream PR is in flight before proposing ours.
- `kind-canonix` cluster unavailable — no live-pod corroboration. Stack trace in the issue payload is sufficient; flagging for completeness.
- Minor: should the fix also handle the mirrored case where `source_dt` is still correct for non-MR rows? Yes — because `args` is mutated inside the per-row loop, the override persists for subsequent iterations. This is a pre-existing bug in the commit that introduced the branch. Scope of this triage is the crash; the pollution-of-args side effect is not observable today (all MR-backed rows need the Service Item table anyway), but Architect should call it out and consider making a `row_args = {**args, ...}` copy rather than mutating the shared dict.

## Recommended next step

Proceed to `/phoenix:plan` in the affected repo. The fix is a 1-line surgical patch inside an existing guard; risk of collateral damage to the 14 unrelated `_update_children` callers is low because the outer guard isolates the subcontracting-from-MR path.

---

```yaml
affected_repos:
  - ShankHarinath/erpnext
confidence: 0.9
recommended_next_step: plan
```
