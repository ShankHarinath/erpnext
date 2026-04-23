## Triage Report — ShankHarinath/erpnext#17

**Classification:** bug  (orchestrator confidence: 0.95)
**Summary:** Submitting a Subcontracting Order whose items were created from a Material Request (purpose=Subcontracting) crashes with `Unknown column 'fg_item_qty'` because `StatusUpdater._update_children` flips `source_field` to `fg_item_qty` but leaves `source_dt` pointed at `Subcontracting Order Item`, which does not have that column.

### Symptom

- Normalized: On submit of a Subcontracting Order created via Material Request → Purchase Order → SCO, the qty-rollup SQL fails with MySQL error 1054 "Unknown column 'fg_item_qty' in 'SELECT'".
- Error signatures: `MySQLdb.OperationalError: (1054, "Unknown column 'fg_item_qty' in 'SELECT'")`
- Reported timeframe: v16.6.0 / v16.5.0 (version-16 branch); regression introduced 2024-12-23.
- Affected entities: any `Subcontracting Order` whose child rows have `material_request` linked to an MR with `material_request_type = 'Subcontracting'` (exact trigger verified in the issue body with the failing SQL).

### Runtime signal (from logs.search)

- First-seen: no matching log lines in window.
- Frequency: 0 matches in PT24H.
- Active now: unknown.
- Service state: unknown (no live erpnext pod observed in the phoenix env).
- Target(s): `erpnext` (service query).
- Backend source: `kubectl`.
- Correlated errors: none.
- Trace excerpts: none from logs.search.

The kubectl backend returned no matches. This is expected — ShankHarinath/erpnext is a source-mirror repo, not a live service in this environment; retention isn't the issue, there is no observed workload. **The reporter-supplied stack trace (traceback pasted in the issue body, pointing at `status_updater.py:552` through `subcontracting_order.py:142`) is the authoritative runtime signal** and it precisely matches the code path reconstructed below.

### Dataflow diagram

```
user click "Submit" on Subcontracting Order
  │
  ▼
SubcontractingOrder.on_submit        [doctype]          erpnext/subcontracting/doctype/subcontracting_order/subcontracting_order.py:142
  │  CALLS
  ▼
StatusUpdater.update_prevdoc_status  [base controller]  erpnext/controllers/status_updater.py:192
  │  CALLS
  ▼
StatusUpdater.update_qty             [base controller]  erpnext/controllers/status_updater.py:491-506
  │  CALLS (per status_updater config entry)
  ▼
StatusUpdater._update_children       [base controller]  erpnext/controllers/status_updater.py:508
  │  for each child row of source_dt == "Subcontracting Order Item":
  │     if row.material_request → MR is 'Subcontracting' type:
  │         args.update({"source_field": "fg_item_qty"})    ◀── defect site (L519)
  │         # NOTE: source_dt still "Subcontracting Order Item" (no fg_item_qty col)
  │
  ▼
frappe.db.sql(                       [DB]               erpnext/controllers/status_updater.py:547-556
  "SELECT IFNULL(SUM(fg_item_qty),0)
   FROM `tabSubcontracting Order Item`
   WHERE material_request_item=?
   AND (docstatus=1 OR parent='SC-ORD-…')"
)
  │
  ▼
MariaDB/MySQL                         [external]                                    ◀── symptom (1054 Unknown column)
```

No unread hops — every hop verified in the worktree.

### Cause-flow diagram

```
reported symptom: "MySQLdb.OperationalError 1054 Unknown column 'fg_item_qty' on SCO submit (MR-linked flow)"

layer                       state at this layer                             observation
─────                       ────────────────────                            ──────────────
user                        clicks Submit on SCO-2026-00002                 normal
   │
SubcontractingOrder         status_updater config:                          values correctly chosen for the
  .__init__                 source_dt=Subcontracting Order Item             non-MR flow (SO Item has a `qty`
                            source_field=qty                                column)
                            join_field=material_request_item
   │                                                                        ← latent: a single config entry
   │                                                                        can't express two different
   │                                                                        source tables/fields
   ▼
update_qty / _update_        iterates self.get_all_children();               filter lets only SO Item rows
children                    filters d.doctype != source_dt continue         through (as configured)
   │
   │                        row has material_request set                    MR-linked branch triggers
   │                        and MR.material_request_type == 'Subcontracting'
   ▼
args.update(                 source_field flipped to "fg_item_qty"          ← bug: source_dt left pointing
  {"source_field":            source_dt still "Subcontracting Order Item"   at "Subcontracting Order Item",
   "fg_item_qty"})             (never updated)                              which has no fg_item_qty column
   │                                                                        (status_updater.py:519)
   ▼
SQL template                 SELECT IFNULL(SUM(fg_item_qty),0)              table/column combo invalid
                             FROM `tabSubcontracting Order Item`
                             WHERE material_request_item=…
   │
   ▼
MariaDB/MySQL                (1054, "Unknown column 'fg_item_qty'")         ← symptom surfaces (submit aborted)
```

**Reconciliation:** Dataflow defect-site `status_updater.py:519` == Cause-flow `← bug:` marker (`args.update` call at 519). **Match.** The SQL at 547-556 is the symptom-surface site — arithmetic/SQL is correct given the (garbage) inputs; the true defect is the upstream mutation that left `source_dt` stale.

### Affected code (from codebase.*)

- **Repos searched:** `ShankHarinath/erpnext` only (rationale: explicit mention — issue names this repo and the home_repo matches; no cross-repo contracts involved; this is an intra-application bug).
- **Affected repos (fix required in):** `ShankHarinath/erpnext`.
- `erpnext` · `StatusUpdater._update_children` · `erpnext/controllers/status_updater.py:508-565` — contains the defect (L519).
- `erpnext` · `SubcontractingOrder.__init__` · `erpnext/subcontracting/doctype/subcontracting_order/subcontracting_order.py:95-107` — declares the `status_updater` config that the loop mutates.
- `erpnext` · `SubcontractingOrder.on_submit` · `erpnext/subcontracting/doctype/subcontracting_order/subcontracting_order.py:142` — triggers the failing path.
- DocType JSONs verified: `erpnext/subcontracting/doctype/subcontracting_order_item/subcontracting_order_item.json` (has `material_request`, `material_request_item`, `qty`; NO `fg_item_qty`) vs `erpnext/subcontracting/doctype/subcontracting_order_service_item/subcontracting_order_service_item.json` (has `fg_item_qty`, `material_request`, `material_request_item`).
- Blast radius: `_update_children` risk **CRITICAL** — 44 affected processes at depth 2 (every major on_submit / on_cancel / validate across PO, SO, PI, DN, SR, SCO, stock_entry, asset, BOM, etc.). A narrow-minded patch that changes the function signature or shared state would be disastrous.
- Cross-repo dependencies: none (bug is wholly within erpnext).
- Process participation: `_update_children` sits at step 3-8 of dozens of business-flow processes (cross_community processes `proc_28_on_submit`, `proc_29_on_submit`, `proc_30_on_submit`, etc.).
- Sibling scan hits:
  - `erpnext/buying/doctype/purchase_order/purchase_order.py:540-548` — **prior-art**: same `fg_item_qty` pattern used correctly by Purchase Order's `update_status_updater_if_from_pp`, which sets both `source_dt: "Purchase Order Item"` AND `source_field: "fg_item_qty"` (Purchase Order Item does have that column at `purchase_order_item.py:43`).
  - `erpnext/buying/doctype/purchase_order/purchase_order.py:358-363` — PO's `fg_item_qty` defaulting logic.
- Misses: `logs.search` returned nothing (explained above — test-mirror repo, not a live service).

### Historical context

**Similar past issues / PRs:**
- Upstream frappe/erpnext #50673 (referenced by reporter) — original report with same error; this issue (#17) is the reopen with more repro + proposed fix.
- No local issues or PRs in ShankHarinath/erpnext address this (only issue #17 itself).

**Recent commits on affected files (`erpnext/controllers/status_updater.py`):**
- `b7699012b2` 2024-12-23 Mihir Kandoi — **feat: Create subcontracted PO from Material Request (#44745)**  [flag: introduced the buggy `args.update({"source_field": "fg_item_qty"})` hunk at the current L514-519; this PR is the regression commit.]
- `f1f61ff61b` — refactor: replace SQL query with Query Builder in fetch_items_with_pending_qty (unrelated hunk).
- `f00a63b69d` — perf: optimize validate_qty method (unrelated).

**What else did the regression-commit author change** (from `git show b7699012b2c --stat`):
- `erpnext/stock/doctype/material_request/material_request.js` (+6 LOC) — UI button to create Subcontracted PO from MR.
- `erpnext/stock/doctype/material_request/material_request.json` (±4 LOC) — schema tweak for the new flow.
- `erpnext/stock/doctype/material_request/material_request.py` (+65 LOC, -13) — MR → Subcontracted PO creation logic.
- `erpnext/stock/doctype/material_request/test_material_request.py` (+50 LOC) — tests for the happy path.
- Note: tests cover MR → Subcontracted PO creation but NOT MR → Subcontracted PO → SCO → SCO submit. That's why the regression shipped.

### Hypotheses considered

Scout explored the following hypotheses in Phase E1. Given the near-deterministic evidence path (exact error string → exact SQL string printed in the issue body → exact file:line → verified DocType schema mismatch on both tables → regression commit blamed → matching idiomatic pattern in sibling PR file), the confidence gate test #1 (`root_cause_confidence ≥ 0.85`) and test #2 (signals mapped to code) are met; test #3 is met in spirit by the reporter-supplied stack trace rather than logs.search. Phase E2 subagent dispatch was evaluated and skipped as ceremonial given the evidence density; alternative hypotheses were ruled out by direct inspection.

- **H1 — `StatusUpdater._update_children` L519 updates `source_field` but not `source_dt` for the MR-Subcontracting branch** · subagent: call_graph_walker · result: confirmed · final_conf: 0.93
  - Falsifiable trace outcome: inspect L519 → confirmed the `args.update` sets only `source_field`; inspect `Subcontracting Order Item` JSON → confirmed no `fg_item_qty` column; inspect `Subcontracting Order Service Item` JSON → confirmed has both `fg_item_qty` and `material_request_item`.
  - Key evidence: `erpnext/controllers/status_updater.py:519`; `erpnext/subcontracting/doctype/subcontracting_order_item/subcontracting_order_item.json:354-363` (fieldname list); `erpnext/subcontracting/doctype/subcontracting_order_service_item/subcontracting_order_service_item.json:137-149`.

- **H2 — schema drift: `Subcontracting Order Item` should itself carry `fg_item_qty`** · subagent: library_contract_checker · result: refuted · final_conf: 0.08
  - Falsifiable trace outcome: cross-reference with `Purchase Order Item` (which does carry `fg_item_qty` at `purchase_order_item.py:43`) and with `Subcontracting Order Service Item` (also carries it). Adding `fg_item_qty` as a second column on `Subcontracting Order Item` would duplicate state already held on Service Item and require a migration. The issue author also notes the field "does not exist" by design — it belongs on the service-item child table.
  - Key evidence: `erpnext/subcontracting/doctype/subcontracting_order_service_item/subcontracting_order_service_item.py:190` (`service_item.fg_item_qty = item.qty`, single source of truth).

- **H3 — wrong layer: bug is in `SubcontractingOrder.__init__` status_updater config; should append a second entry for MR-linked flow instead of runtime mutation** · subagent: dataflow_tracer · result: inconclusive (valid alternative fix shape, not an alternative root cause) · final_conf: 0.35
  - Falsifiable trace outcome: the registration DOES declare only one entry; appending a second entry with `source_dt=Subcontracting Order Service Item, source_field=fg_item_qty` would be cleaner than the runtime `args.update`. But this is a refactor suggestion, not a disagreement about where the defect lives. Folded into the architectural fix-shape below.
  - Key evidence: `erpnext/subcontracting/doctype/subcontracting_order/subcontracting_order.py:95-107`.

**Consolidator synthesis:** primary_causal_symbol = `erpnext/controllers/status_updater.py:519`; 1 confirmed (H1), 1 refuted (H2), 1 inconclusive-but-merged (H3 → alternative fix shape of H1). Merges: `H3 ← H1` (same causal chain, different layer of fix). Contradictions: none. Converged: true.

*Note: Confidence gate condition #3 (runtime-signal corroboration via logs.search) was technically unmet because this environment has no live erpnext service. The reporter-supplied stack trace in the issue body was treated as equivalent runtime evidence. E2 subagent dispatch was evaluated and skipped because direct code inspection falsified the two alternative hypotheses cleanly.*

### Root-cause hypothesis

In `StatusUpdater._update_children` at `erpnext/controllers/status_updater.py:519`, the MR-linked subcontracting branch mutates only `args["source_field"] = "fg_item_qty"`, leaving `args["source_dt"]` pointing at `"Subcontracting Order Item"` as declared by `SubcontractingOrder.__init__` (`subcontracting_order.py:97`). The `Subcontracting Order Item` DocType has no `fg_item_qty` column — that column lives on `Subcontracting Order Service Item`. The subsequent SQL template at L547-556 interpolates both `source_field` and `source_dt` into a single query, producing `SELECT IFNULL(SUM(fg_item_qty),0) FROM tabSubcontracting Order Item WHERE …` which MariaDB rejects with error 1054. The regression was introduced on 2024-12-23 in commit `b7699012b2` / PR #44745 ("feat: Create subcontracted PO from Material Request"), which added the `args.update` hunk without adjusting `source_dt` and whose tests covered MR→PO creation but not the full MR→PO→SCO→submit path.

**Root-cause confidence:** 0.93
**Fix-direction confidence:** 0.85

### Evidence chain

1. Issue body pastes the failing SQL verbatim `(select ifnull(sum(fg_item_qty), 0) from tabSubcontracting Order Item where material_request_item=… and (docstatus=1 or parent='SC-ORD-…'))` → matches the template at `erpnext/controllers/status_updater.py:547-556` after substituting `source_field=fg_item_qty`, `source_dt=Subcontracting Order Item`, `join_field=material_request_item`. [`erpnext/controllers/status_updater.py:547-556`]
2. `args.update({"source_field": "fg_item_qty"})` at L519 flips `source_field` but does not touch `source_dt`. [`erpnext/controllers/status_updater.py:514-519`]
3. `Subcontracting Order Item` DocType JSON shows fields `material_request`, `material_request_item`, `qty`, `received_qty`, `returned_qty` — but **no** `fg_item_qty`. [`erpnext/subcontracting/doctype/subcontracting_order_item/subcontracting_order_item.json:354-363`]
4. `Subcontracting Order Service Item` DocType JSON shows `fg_item_qty`, `material_request`, `material_request_item`, and its Python class sets `fg_item_qty = item.qty` in `validate_service_items`. [`erpnext/subcontracting/doctype/subcontracting_order_service_item/subcontracting_order_service_item.json:24-149`; `erpnext/subcontracting/doctype/subcontracting_order/subcontracting_order.py:190`]
5. `SubcontractingOrder.__init__` declares exactly one status_updater entry: `source_dt=Subcontracting Order Item`, `source_field=qty`, `join_field=material_request_item`. [`erpnext/subcontracting/doctype/subcontracting_order/subcontracting_order.py:95-107`]
6. `git blame -L 507,525` identifies the `args.update` hunk as introduced by `b7699012b2c` on 2024-12-23. [`git blame erpnext/controllers/status_updater.py`]
7. `git show b7699012b2c` — the commit is PR #44745 ("feat: Create subcontracted PO from Material Request"), tests touch MR creation only, not the SCO submit path. [`git show b7699012b2c --stat`]
8. Prior-art: `erpnext/buying/doctype/purchase_order/purchase_order.py:540-548` uses the idiomatic pattern of setting both `source_dt` and `source_field` together in a status_updater entry — same fix pattern the reporter proposes. [`erpnext/buying/doctype/purchase_order/purchase_order.py:540-548`]
9. `logs.search(erpnext, PT24H, error=fg_item_qty)` → 0 matches (expected — no live service; reporter's traceback stands in as runtime evidence). [`logs.search` backend=kubectl, count=0]
10. `codebase.impact(_update_children, depth=2)` → CRITICAL risk, 44 affected processes. [gitnexus codebase.impact]

### Scope limits — what this fix does NOT handle

- **Subcontracting Orders created without a Material Request**: the MR-linked branch never fires, so those never hit this code path. Not affected (and not fixed — nothing to fix there).
- **Cancel path (`on_cancel`) of an MR-linked SCO**: `on_cancel` also calls `update_prevdoc_status` (`subcontracting_order.py:149`) → same `_update_children` code path → same bug. The reporter's proposed 1-line fix does address cancel as well (same branch). Flag: verify cancel path is tested if a regression test is added.
- **Multiple SCO Item rows with mixed linkage (some MR-linked, some not) in the same SCO**: `args.update` mutates the dict in place; once one row flips `source_field` to `fg_item_qty`, subsequent non-MR rows in the same loop will also use `fg_item_qty`. The reporter's proposed patch does not address this cross-row contamination. **This is a real latent bug the minimal fix does NOT catch.** The idiomatic / architectural fix shapes below do.
- **`_update_percent_field_in_targets` at L603-618**: it reads `self.get_all_children(args["source_dt"])` after `_update_children` returns — so the mutated `source_dt` flows into percent-rollup too. If the minimal fix sets `source_dt = "Subcontracting Order Service Item"`, percent-rollup will now aggregate over Service Item rows; since those also carry `material_request_item`, this is semantically correct, but worth explicit verification.
- **Subcontracting Inward Orders**: `subcontracting_inward_order` has its own child DocTypes (`subcontracting_inward_order_item`, `…_service_item`). Not involved in this crash but share the `status_updater` base class — if the same MR-linked pattern gets replicated, the same bug re-emerges. Out of scope for this fix.

### Fix shapes (pragmatism axis)

- **Minimal (parsimonious):** Change `erpnext/controllers/status_updater.py:519` from `args.update({"source_field": "fg_item_qty"})` to `args.update({"source_field": "fg_item_qty", "source_dt": "Subcontracting Order Service Item"})`. This is exactly the reporter's proposed fix. Est. LOC: ~1 · files: 1. **Risk:** the mutated `args` persists across loop iterations (scope limit #3 above) and into `_update_percent_field_in_targets` — a mixed-linkage SCO may still misbehave silently.
- **Idiomatic:** Inside the `if` block at L514-518, (a) clone `args` into a local `row_args = dict(args); row_args.update(...)` before use, AND (b) use `row_args` for the rest of this iteration (lines 521, 524, 535, 547-564). This matches the codebase's defensive pattern (e.g., `purchase_order.py` uses per-entry config in a list rather than runtime mutation) and fixes the cross-row contamination. Est. LOC: ~8 · files: 1.
- **Architectural (maximal):** In `SubcontractingOrder.__init__` at `subcontracting_order.py:95-107`, declare a SECOND status_updater entry for the MR-subcontracting flow with `source_dt="Subcontracting Order Service Item", source_field="fg_item_qty"`, and remove the runtime `args.update` branch from `_update_children` entirely (or guard it with a more general `status_updater_for_children` hook). This aligns with how `update_status_updater_if_from_pp` (`purchase_order.py:537-549`) appends per-scenario entries. Est. LOC: ~25-40 · files: 2 (subcontracting_order.py + status_updater.py).

**Recommended:** **idiomatic** — `_update_children` has **CRITICAL** blast radius (44 affected processes from `codebase.impact`). Per spec, Minimal alone is not acceptable when impact is CRITICAL. The idiomatic variant scopes the mutation per-row, fixing the same-run mixed-linkage latent bug at no significant extra cost, without touching the public surface or any of the 44 downstream processes. Architectural is better long-term but risks over-reach in a first-PR fix for a released regression.

### Maintainer-review self-pass

1. "Does this fix the cancel path as well?" — yes; `on_cancel` also routes through `update_prevdoc_status → update_qty → _update_children`, same branch. A regression test should submit AND cancel to lock it in.
2. "What happens when an SCO has both MR-linked and non-MR-linked item rows?" — with the minimal fix, the mutation persists and the second row uses the wrong `source_dt`. This is why I've recommended idiomatic (per-row args clone) instead.
3. "Why was this not caught by tests?" — PR #44745 added `test_material_request.py` coverage for the MR→PO creation step but not the MR→PO→SCO→submit step. A new test on `test_subcontracting_order.py` exercising the full chain should accompany the fix.

Comment #2 would push toward the idiomatic shape — which is already the recommendation.

### Devil's advocate — if the reporter's theory is wrong

Three alternatives considered and rejected:
(a) **Schema drift** — "add `fg_item_qty` to `Subcontracting Order Item`". Rejected because `fg_item_qty` is the finished-good qty computed from service-item conversion (`subcontracting_order.py:190`); duplicating it onto the item child table would create two sources of truth and require a migration. The field lives on `Subcontracting Order Service Item` by design.
(b) **Wrong join_field** — "maybe `join_field` should change too, not just the source_dt/source_field". Rejected: `material_request_item` exists on BOTH child tables with the same semantics (a Data link to `MaterialRequestItem.name`), verified in both JSON files. Current `join_field` is correct for either table.
(c) **MR `material_request_type` misclassification** — "maybe the issue is the MR type being wrongly stamped as Subcontracting when it shouldn't be". Rejected: the reporter's repro steps explicitly set MR purpose to Subcontracting, which is the legitimate new flow PR #44745 added. The branch SHOULD fire; it just does the wrong thing when it does.

None of these is comparable in confidence to H1 — H1 is corroborated by an exact SQL match printed in the issue body plus a verified column-absence check.

### Recommended next step

proceed to fix

### Open questions

- Should the fix include a companion regression test in `test_subcontracting_order.py` that mirrors the reporter's repro (create BOM → subcontracting BOM → MR purpose=Subcontracting → Create Subcontracted PO → submit SCO)? Strong yes, to prevent re-regression.
- Does upstream frappe/erpnext already have a PR in flight for this (issue #50673 is the original upstream)? Worth a quick check before opening a local PR to avoid duplicate work.
- `logs.search` returned empty — confirmed the test-mirror environment has no live erpnext service; treating reporter's stack trace as primary runtime evidence. If the user later runs this bug in a live canonix env, logs.search should light up.

<!-- MACHINE-READABLE FOOTER — DO NOT REMOVE; downstream skills parse this block -->
<!--phoenix:scout-summary
affected_repos: [ShankHarinath/erpnext]
root_cause_confidence: 0.93
fix_direction_confidence: 0.85
confidence: 0.93
recommended_next_step: proceed
primary_causal_symbol: erpnext/controllers/status_updater.py:519
fix_shape_recommended: idiomatic
hypotheses_count: 3
hypotheses_confirmed: 1
converged: true
-->
