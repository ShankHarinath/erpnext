# Triage Report — Bug: Update status not working when closing a work order with reserve stock enabled

**Issue:** ShankHarinath/erpnext#3
**Reporter:** ShankHarinath (2026-04-20)
**Rev:** 1
**Module:** `manufacturing` / Work Order
**Classifier confidence (input):** 0.95
**Scout confidence (root cause):** 0.92

## Summary

When a user closes a Work Order that was submitted with `reserve_stock` enabled, the document's `status` column silently reverts from `Closed` back to a computed production status (typically `Stock Reserved`, `Stock Partially Reserved`, `Not Started`, `In Process`, or `Completed`). The Close action executes the status update first, then `on_close_or_cancel()` runs and — only when `reserve_stock` is true — calls `update_stock_reservation()`, whose last line re-runs `get_status()` and writes the result to the DB, clobbering the just-persisted "Closed" status.

The fix is localized and straightforward: `get_status()` must preserve `"Closed"` the same way it already preserves `"Stopped"` (or, equivalently, `update_stock_reservation()` must not clobber `"Closed"`/`"Stopped"`).

## Root-cause hypothesis

Primary defect: `WorkOrder.update_stock_reservation()` at `erpnext/manufacturing/doctype/work_order/work_order.py:826-829` unconditionally recomputes and writes the WO status after `make_stock_reservation_entries(self)` — including in the close path, where the preceding `update_status("Closed")` call (invoked from `close_work_order` at lines 2522-2547) has already written `status="Closed"`.

The recomputation function `WorkOrder.get_status()` at lines 582-624 has explicit preservation logic for `"Stopped"` (line 590: `if status != "Stopped": status = "Not Started"`) but no equivalent guard for `"Closed"`. As a result, when `get_status()` is invoked on a just-closed, submitted (`docstatus == 1`) WO, it forcibly overwrites `"Closed"` with `"Not Started"` (then further refines to `"Stock Reserved"` / `"Stock Partially Reserved"` at lines 613-622 when `reserve_stock` is true, or `"In Process"` / `"Completed"` based on transferred/produced qty).

The defect only manifests when `reserve_stock` is true because the `on_close_or_cancel()` path only calls `update_stock_reservation()` under `if self.reserve_stock:` (line 821). Without reserve stock, nothing rewrites status after `update_status("Closed")`, so the bug is invisible in the default code path — which is exactly what the reporter described.

## Evidence chain

### A. Symptom → Code (GitNexus)

- Execution-flow query `work order close status update reserve stock` surfaced `WorkOrder.set_reserve_stock` and `StockReservationEntry` — confirming the reserve-stock feature lives under `erpnext/manufacturing/doctype/work_order/` and `erpnext/stock/doctype/stock_reservation_entry/`.
- Query `update_status work_order manufacturing` resolved `WorkOrder.update_status` at `erpnext/manufacturing/doctype/work_order/work_order.py:569-580`.
- `context(update_status)` shows five callers: `update_production_plan_status`, `update_qty_in_stock_reservation`, `update_consumed_qty_in_stock_reservation`, `stop_unstop`, and — crucially — `close_work_order`.
- `context(close_work_order)` (at `work_order.py:2522-2547`) confirms the call chain: `close_work_order(..., status="Closed")` → `work_order.update_status("Closed")` → `work_order.on_close_or_cancel()`.
- The second step, `on_close_or_cancel()` at `work_order.py:808-824`, calls `self.update_stock_reservation()` **only when `self.reserve_stock` is truthy** (lines 821-822).
- `update_stock_reservation()` at `work_order.py:826-829` ends with `self.db_set("status", self.get_status())`, overwriting the "Closed" status that `update_status` just wrote.

### Trace (concrete)

Reproduced manually from the code:

1. `close_work_order("WO-xxx", "Closed")` is invoked (whitelisted endpoint hit by the Close button at `erpnext/manufacturing/doctype/work_order/work_order.py:2522`).
2. `work_order.update_status("Closed")` runs (line 2543).
   - `self.status != "Closed"` → True.
   - `status not in ["Stopped", "Closed"]` → `"Closed" in [...]` → False → `get_status` is **skipped** (correct).
   - `self.db_set("status", "Closed")` — **DB now says Closed** (line 576).
3. `work_order.on_close_or_cancel()` runs (line 2544).
   - `self.reserve_stock` is True → enters `update_stock_reservation()` (line 822).
4. `update_stock_reservation()` runs:
   - `set_qty_change()`, `make_stock_reservation_entries(self)` — side effects on SRE rows.
   - `self.db_set("status", self.get_status())` — calls `get_status()` with no `status` arg → `status = self.status` (which is still `"Closed"` on the in-memory doc, because `db_set` updates DB but not always re-fetches).
5. Inside `get_status()`:
   - `self.docstatus == 1` → True → enters `elif` branch (line 589).
   - `status != "Stopped"` → True (status is `"Closed"`, not `"Stopped"`) → `status = "Not Started"` (line 591). **"Closed" is lost here.**
   - Material transfer / produced-qty logic may bump to `"In Process"` or `"Completed"`.
   - At lines 613-622, if `status == "Not Started"` and `reserve_stock` is true, status becomes `"Stock Reserved"` or `"Stock Partially Reserved"` based on `required_items.stock_reserved_qty`.
6. `db_set("status", <that value>)` — **DB now says "Stock Reserved" / "Not Started" / etc.**, which is exactly the "silent failure" the user reports.

### B. Runtime evidence (kubectl)

- The `kind-canonix` cluster does not run ERPNext. `kubectl get pods` lists only Canonix platform services (`app`, `cortex`, `mcp-gateway`, `postgres`, `redis`, etc.). No ERPNext container to pull logs from.
- Runtime corroboration is therefore not available for this issue. The root cause is static-evidence only, but the code path is fully deterministic and re-walkable from the sources cited above, so confidence remains high.

### C. Recent commits on affected files

- `git log -L 826,830:.../work_order.py` traces the offending line back to commit `4d050441b3 feat: stock reservation for Work Order` (introduction of stock reservation for Work Order). Commit `f2b948a483 feat: subcontracting inward (#47728)` later added `self.set_qty_change()` above it but did not touch the clobbering `db_set("status", self.get_status())`.
- Recent (last 120 days) commits to `work_order.py` include `4d40c84a31 fix(manufacturing): update status for work order before calculating planned qty` — confirming status-ordering bugs in this file have been a recurring theme and suggesting a similar reviewer lens would apply here.
- No commit in the last 30 days plausibly introduced this specific regression; the defect has existed since the reserve-stock feature landed (~2023/2024 era by file history).

### D. Similar past issues / PRs

- `gh issue list --repo ShankHarinath/erpnext --search "work order close status reserve"` returns only issue #3 itself (fork has no other matches).
- Upstream `frappe/erpnext` search is blocked by the counterfactual shim (exit 77). Noted in Leakage section.
- Adjacent pattern found in the same file: `get_status` already preserves `"Stopped"` (line 590) — the same guard is simply missing for `"Closed"`. This is the analog pattern the fix should extend.
- Existing test `TestWorkOrder.test_close_work_order` (`test_work_order.py:1086`) exercises `close_work_order` but with a WO that does NOT have `reserve_stock` enabled, which is why this regression slipped past CI. `TestWorkOrder.test_reserved_qty_for_production_closed` (line 232) closes a reserve-stock WO but only asserts bin `reserved_qty_for_production`, never `wo.status`.

### E. Synthesis

All four pre-extracted signals map to code in the single file `erpnext/manufacturing/doctype/work_order/work_order.py`. The causal chain is a deterministic two-step DB write where the second step overwrites the first, gated exactly by the `reserve_stock` flag the reporter mentioned. The fix surface is 1–5 lines. Confidence: **0.92** (drops from ≥ 0.85 fast-path territory only because runtime logs were not obtainable in `kind-canonix`).

## Proposed fix direction (informational — Architect decides)

Minimal, least-surprise options, in order of preference:

1. **Preserve `"Closed"` in `get_status()` symmetrically with `"Stopped"`.** Change line 590 from `if status != "Stopped":` to `if status not in ("Stopped", "Closed"):`. This matches the invariant that both terminal states are user-set and must not be recomputed from qty/reservation state. Smallest blast radius, consistent with existing intent. Also fixes `update_qty_in_stock_reservation` → `update_status(...)` → `get_status(...)` paths if they ever reach a Closed doc.
2. **Guard `update_stock_reservation()` against terminal states.** Replace line 829 with `if self.status not in ("Closed", "Stopped"): self.db_set("status", self.get_status())`. Narrower scope but leaves the general weakness in `get_status` for any future caller.
3. **Reorder `close_work_order` to run `update_status` after `on_close_or_cancel`.** Risky — inverts the invariant that "the thing that closed a WO wrote Closed last." Not recommended.

Architect should also add a regression test: create a WO with `reserve_stock=1`, submit it, `close_work_order(wo.name, "Closed")`, reload, and assert `frappe.db.get_value("Work Order", wo.name, "status") == "Closed"`. This extends `test_reserved_qty_for_production_closed` or adds a sibling.

## Affected files (touched by fix)

- `erpnext/manufacturing/doctype/work_order/work_order.py` — 1-line change in `get_status` (lines 582-624) or in `update_stock_reservation` (lines 826-829).
- `erpnext/manufacturing/doctype/work_order/test_work_order.py` — add regression test asserting terminal status is preserved for reserve-stock WOs.

## Impact / blast radius

`mcp__gitnexus__impact(update_stock_reservation, upstream)` → **LOW risk**, 4 affected symbols, 0 processes:
- d=1: `WorkOrder.on_submit`, `WorkOrder.on_close_or_cancel`
- d=2: `WorkOrder.on_cancel`, `close_work_order`

Option 1 (modify `get_status`) expands the radius slightly (more callers) but only changes behavior for docs whose in-memory `status` is already `"Closed"` — a narrowly scoped class that today is only produced by `close_work_order` and would otherwise be overwritten anyway. Still LOW.

## Open questions / blockers

- Unable to corroborate in runtime logs because `kind-canonix` does not host ERPNext. Confidence rests on static reasoning.
- Unable to search upstream `frappe/erpnext` for prior art / related PRs (counterfactual shim blocks non-fork owners).

## Leakage / process notes

- `gh issue list --repo ShankHarinath/erpnext ...` and local `git log` / `git show` worked normally.
- No denied shim invocations occurred during this triage (I did not attempt upstream/global `gh` calls). `WebFetch` and `WebSearch` were not used.
- No issue-comment breadcrumb was posted — `disable_issue_comments=true` per dispatch instructions.

## Recommended next step

**PATCH** — this is a small, well-scoped single-file bug fix with an obvious analogous pattern already in the same function (`"Stopped"` preservation). Architect should plan a 1–5 line edit in `work_order.py` plus a regression test in `test_work_order.py`.

---

```yaml
# machine-readable footer
affected_repos:
  - ShankHarinath/erpnext
confidence: 0.92
recommended_next_step: PATCH
primary_symbols:
  - erpnext/manufacturing/doctype/work_order/work_order.py:WorkOrder.get_status
  - erpnext/manufacturing/doctype/work_order/work_order.py:WorkOrder.update_stock_reservation
primary_files:
  - erpnext/manufacturing/doctype/work_order/work_order.py
  - erpnext/manufacturing/doctype/work_order/test_work_order.py
runtime_evidence: none_available
kubectl_context: kind-canonix
blockers:
  - no_erpnext_pod_in_cluster
  - upstream_search_blocked_by_counterfactual_shim
```
