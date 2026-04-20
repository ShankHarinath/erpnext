# Phoenix Triage Report — BOM Creator miscalculates sub-assembly raw-material cost

**Issue:** ShankHarinath/erpnext#1 — "BOM Creator Miscalculates the Raw Material Cost in sub-assembly and final product"
**Rev:** 1
**Repo(s) searched:** erpnext (single-repo scope — this is a self-contained Python bug in the ERPNext manufacturing module; no indexed sibling service touches BOM Creator internals)
**Classification confidence:** 0.95 (input) → **0.95 (after investigation)**

## Summary

The `BOMCreator.get_raw_material_cost()` method in `erpnext/manufacturing/doctype/bom_creator/bom_creator.py` computes the per-unit rate of a sub-assembly row by **multiplying** its total raw-material cost by `row.conversion_factor`, **without first dividing by the sub-assembly's quantity**. The result is that the sub-assembly's `row.rate` is the whole-batch cost (not per unit), and then `row.amount = rate * qty` multiplies by qty a second time — producing the `cost * qty * qty` figure the reporter observed (INR 350 × 1482 ≈ INR 518,934 in their example; the manual BOM path correctly divides and yields INR 0.4759/TB).

This is a known, fixed bug upstream. See **Phase D** — PR #54090 (merged 2026-04-07) plus backport #54091 contain the one-line corrective diff. The local clone is behind that fix.

## Phase A — Symptom mapped to code

### A.1 Search radius
- `list_repos`: erpnext indexed at 2026-04-20 (fresh).
- `group_list`: erpnext not in any group for this investigation.
- Pre-extracted signals ("BOM Creator", "sub-assembly cost", "Raw Material Cost") all point to ERPNext manufacturing domain — **scope = erpnext only**.

### A.2 Prime suspect
- **File:** `/Users/shashank/Canonix/phoenix-test/erpnext/erpnext/manufacturing/doctype/bom_creator/bom_creator.py`
- **Symbol:** `BOMCreator.get_raw_material_cost` (lines 182–211)
- **Buggy branch:** line 206, the `else` branch that handles expandable (sub-assembly) rows:

```python
else:
    row.rate = flt(self.get_raw_material_cost(row.item_code) * row.conversion_factor)

row.amount = flt(row.rate) * flt(row.qty)
amount += flt(row.amount)
```

`get_raw_material_cost(row.item_code)` recursively returns the *total* raw-material cost for the sub-assembly (sum of child amounts). This total is already the cost of producing `row.qty` units of the sub-assembly. To turn it into a per-unit rate, the code must divide by `row.qty` before multiplying by `row.conversion_factor`. It does not — so `row.rate` ends up equal to the total cost for the whole batch, and the next line multiplies by `row.qty` again, giving `total_cost * qty`.

### A.3 Contrast with the correct BOM path
`BOM.calculate_rm_cost` (`erpnext/manufacturing/doctype/bom/bom.py` lines 1006–1045) uses `stock_qty / bom.quantity` to derive per-unit consumption — which the issue reporter confirms works correctly when creating a BOM manually.

### A.4 Affected repos
Only `erpnext`. No other indexed service calls into BOM Creator.

### A.5 Impact (blast radius)

`mcp__gitnexus__impact(get_raw_material_cost, upstream)` — **LOW direct risk**:

| d | Symbol | Path |
|---|--------|------|
| 1 | `BOMCreator.set_rate_for_items` | bom_creator.py |
| 2 | `BOMCreator.before_save` | bom_creator.py (runs every save) |
| 2 | `delete_node` (whitelisted) | bom_creator.py |
| 2 | `edit_bom_creator` (whitelisted) | bom_creator.py |

All upstream callers live in the same module — a surgical single-line fix is safe.

## Phase B — Runtime evidence

Skipped. No stack trace in the issue, no pod/service target (BOM Creator is ERPNext site logic that runs in-process inside Frappe workers; the bug is deterministic and logic-only). Reporter's reproduction (INR 518,934 vs INR 350) plus the direct code read is sufficient evidence.

## Phase C — Recent commits on the affected file

`git log` on `bom_creator.py`:

| SHA | Date | Notes |
|-----|------|-------|
| **c9eda3c279** | **2025-12-11** | **`fix: bom creator page width and rate calculation`** — **regression introduced here.** Before this commit, the code did `row.rate = flt(row.amount) / (flt(row.qty) * flt(row.conversion_factor))`, which was correct (dividing total amount by qty). The refactor replaced it with `row.rate = flt(self.get_raw_material_cost(row.item_code) * row.conversion_factor)`, dropping the divide by `row.qty`. |
| 9c0c39381f | later | adds unrelated validation |
| f1ac0376fb | later | co-product feature |

Correlation with the regression: **very high**.

## Phase D — Similar issues / prior fixes

Identical bug has been triaged and fixed upstream:

- **Issue frappe/erpnext#52149** — same title as this issue, closed 2026-04-07 when PR #54090 merged.
- **PR #54090** — "fix: divide sub-assembly cost by qty to get per-unit rate in BOM Creator" (merged 2026-04-07, credits @ravindu2012).
- **PR #54091** — `version-16-hotfix` backport (merged 2026-04-07).
- Earlier PR #53389 (ravindu2012) and #52573 (belee-asp) proposed the same fix; both closed in favor of #54090.

The upstream patch (3 lines, single file):

```python
else:
    row.rate = flt(
        self.get_raw_material_cost(row.item_code) / flt(row.qty or 1) * row.conversion_factor
    )

row.amount = flt(row.rate) * flt(row.qty)
```

The `row.qty or 1` guard protects against a zero-qty sub-assembly row (division-by-zero).

## Phase E — Synthesis

**Root cause (confidence 0.95):** In `BOMCreator.get_raw_material_cost` at `erpnext/manufacturing/doctype/bom_creator/bom_creator.py:206`, the sub-assembly branch computes `row.rate = total_rm_cost * conversion_factor` instead of `row.rate = (total_rm_cost / row.qty) * conversion_factor`. Because `row.amount` is subsequently set to `row.rate * row.qty`, the error compounds to `total_rm_cost * row.qty`, giving the reporter's observed ~1482× inflation (INR 350 → INR 518,934). The regression was introduced on 2025-12-11 by commit `c9eda3c279` ("fix: bom creator page width and rate calculation"), which removed the explicit `flt(row.amount) / (flt(row.qty) * flt(row.conversion_factor))` divide without replacing it.

**Recommended fix (one-line):** Apply the exact upstream patch from PR #54090 — divide the recursive cost by `flt(row.qty or 1)` before applying `conversion_factor`. File: `erpnext/manufacturing/doctype/bom_creator/bom_creator.py`, lines 205–206.

**Test coverage:** `erpnext/manufacturing/doctype/bom_creator/test_bom_creator.py::TestBOMCreator::test_bom_sub_assembly` asserts `doc.items[0].amount == sum of raw amounts` at qty=1 — which is exactly the case that silently hides this bug (divide-by-1 is a no-op). A new assertion with `qty > 1` on the sub-assembly (e.g., the issue's 300-TB or 1482-CFC scenario) will fail on the buggy code and pass on the fix.

## Open questions

1. Should the Architect extend `test_bom_sub_assembly` with a `qty=2` (or higher) sub-assembly case to regression-guard this path?
2. Is any manual-data migration needed for BOM Creators that were already saved between 2025-12-11 and the fix date with inflated `raw_material_cost`/row amounts? (Likely re-save / re-enqueue suffices, but worth confirming.)

---

<!--phoenix:scout-summary
affected_repos:
  - ShankHarinath/erpnext
confidence: 0.95
recommended_next_step: proceed
primary_file: erpnext/manufacturing/doctype/bom_creator/bom_creator.py
primary_symbol: BOMCreator.get_raw_material_cost
regression_commit: c9eda3c279
related_issue: frappe/erpnext#52149
-->
