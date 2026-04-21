# Triage Report: Issue #15 — Standalone Sales Return uses selling rate / Return Rate as fallback for stock valuation

- **Repo:** ShankHarinath/erpnext
- **Issue:** [#15](https://github.com/ShankHarinath/erpnext/issues/15) — Standalone Sales Return uses selling rate / Return Rate as fallback for stock valuation
- **Reporter:** ShankHarinath
- **Classification confidence (input):** 0.92 (bug)
- **Triage revision:** 1

## A. Symptom → code mapping

**Repo radius selected:** `erpnext` only. The `list_repos` showed `frappe/erpnext` and `ShankHarinath/erpnext` as the only ERPNext forks; the home repo (`ShankHarinath/erpnext`) is the only one indexed as `erpnext`, has no group peers, and the symbols referenced (`get_valuation_rate`, `get_incoming_rate`, `Item.standard_rate`) all live in the ERPNext Python codebase. No cross-repo search radius required.

**Pre-extracted signals mapped:**

| Signal | Match | File | Line |
|---|---|---|---|
| `get_valuation_rate` (fallback chain) | `get_valuation_rate(item_code, warehouse, voucher_type, voucher_no, …)` | `/Users/shashank/Canonix/phoenix-test/erpnext/erpnext/stock/stock_ledger.py` | 1903–2017 |
| `Item.standard_rate` fallback in valuation | `valuation_rate = frappe.db.get_value("Item", item_code, "standard_rate")` | `/Users/shashank/Canonix/phoenix-test/erpnext/erpnext/stock/stock_ledger.py` | 1977 |
| `Item.valuation_rate` fallback (higher priority) | `valuation_rate = frappe.db.get_value("Item", item_code, "valuation_rate")` | `/Users/shashank/Canonix/phoenix-test/erpnext/erpnext/stock/stock_ledger.py` | 1973 |
| Buying `Item Price` fallback (last resort) | `"Item Price", dict(item_code=item_code, buying=1, currency=currency), "price_list_rate"` | `/Users/shashank/Canonix/phoenix-test/erpnext/erpnext/stock/stock_ledger.py` | 1981–1983 |
| `get_incoming_rate` entry point | `@frappe.whitelist() def get_incoming_rate(args, …)` → calls `get_valuation_rate` | `/Users/shashank/Canonix/phoenix-test/erpnext/erpnext/stock/utils.py` | 240, 320 |
| "aligning with return document's rate" — selling-rate-as-fallback | `if self.get("is_return") and not d.incoming_rate and not self.get("return_against") … : d.incoming_rate = d.rate` | `/Users/shashank/Canonix/phoenix-test/erpnext/erpnext/controllers/selling_controller.py` | 603–610 |
| `return_against` branch (standalone path) | `elif not self.get("return_against") or (get_valuation_method(...) == "Moving Average" and self.get("is_return") …)` | `/Users/shashank/Canonix/phoenix-test/erpnext/erpnext/controllers/selling_controller.py` | 534–591 |
| `get_rate_for_return` (used only when `return_against` is present) | `get_rate_for_return(voucher_type, voucher_no, item_code, return_against, …)` | `/Users/shashank/Canonix/phoenix-test/erpnext/erpnext/controllers/sales_and_purchase_return.py` | 732–813 |

**Call chain for a standalone Sales Return into Warehouse B (no prior SLE, no `return_against`):**

1. `SellingController.set_incoming_rate` (selling_controller.py:500) is invoked during `validate` for the Delivery Note / Sales Invoice return.
2. Because `return_against` is falsy, the `elif not self.get("return_against")` branch at line 534 runs.
3. It calls `erpnext.stock.utils.get_incoming_rate(...)` (utils.py:240) with `{item_code, warehouse: Warehouse B, qty: +qty, voucher_type, …}`.
4. With no serial/batch bundle, control reaches the `else` at utils.py:304–316 → `get_previous_sle(args)` returns empty because Warehouse B has never held this item → `in_rate` stays `None`.
5. `get_valuation_rate(item_code, Warehouse B, …)` is called (utils.py:320, stock_ledger.py:1903). In order:
   - Warehouse/batch SLE query returns empty (stock_ledger.py:1957–1968).
   - `Item.valuation_rate` → may be zero for a item that has never been purchased into any warehouse, or present but stale.
   - **stock_ledger.py:1977 — falls to `Item.standard_rate`, which is the selling price seeded on the Item master** (confirmed by `erpnext/stock/doctype/item/item.py:290` where `standard_rate` is written into a selling `Item Price`).
   - If that too is zero, it tries a buying `Item Price` entry (stock_ledger.py:1981–1983).
6. If `get_valuation_rate` still returns 0, control returns to `selling_controller.py:603–610`, which hits the last-resort fallback `d.incoming_rate = d.rate` — i.e. the selling/return-document rate is written as the incoming/valuation rate.

Both fallback paths (the Item master `standard_rate` in step 5, and the selling `d.rate` assignment in step 6) confirm the reporter's exact claim: *"the system may end up valuing the returned stock using `standard_rate` or aligning with the return document's rate."*

**Prime suspect files / lines to change:**

- `/Users/shashank/Canonix/phoenix-test/erpnext/erpnext/stock/stock_ledger.py:1971–1984` — remove the `standard_rate` branch from the valuation-rate fallback chain. Keep `Item.valuation_rate` and buying `Item Price` as the only fallbacks. When none yield a rate, and perpetual inventory is on, raise the existing "Valuation Rate Missing" error (already at 1985–2015).
- `/Users/shashank/Canonix/phoenix-test/erpnext/erpnext/controllers/selling_controller.py:603–610` — for a standalone Sales Return (`is_return` and no `return_against`), do NOT silently set `d.incoming_rate = d.rate`. Either leave zero and let `allow_zero_valuation_rate` gate it, or `frappe.throw` with guidance to create an opening Stock Entry / Stock Reconciliation to establish valuation in the destination warehouse.

**Impact analysis (`gitnexus_impact` on `get_valuation_rate`, upstream, minConfidence=0.8):** **CRITICAL risk**. 45 upstream symbols; 4 direct callers (d=1):

- `erpnext.stock.utils.get_incoming_rate`
- `update_entries_after.get_fallback_rate` (stock_ledger.py:1643) — used on negative stock only
- `SubcontractingReceipt.get_scrap_items`
- `StockEntry.set_basic_rate`

Affected processes: `validate` (purchase_invoice), `validate`/`get_items`/`get_stock_and_rate` (stock_entry), plus a v13 patch script. 10 modules touched. A change to remove `standard_rate` from the fallback list changes behaviour for Purchase Invoice/Receipt, Stock Entry, Stock Reconciliation, Subcontracting, and Asset Capitalization as well — any flow that relies on `Item.standard_rate` being the final-fallback valuation will now either land on a buying `Item Price` or throw "Valuation Rate Missing". The author of the fix MUST add regression tests across these flows and confirm `allow_zero_valuation_rate` behaviour is preserved.

## B. Runtime evidence (kubectl)

**Not applicable / not verifiable in this environment.** This is a correctness/semantic bug with no exception signature, so there is nothing to match in `kubectl logs`. Additionally, the `kind-canonix` context is unreachable from this host:

```
$ kubectl --context kind-canonix get pods
The connection to the server 127.0.0.1:59249 was refused - did you specify the right host or port?
```

No runtime corroboration was attempted beyond that check. Reproduction would require an ERPNext instance with (a) an Item that has `standard_rate` set (typical), (b) no historical SLE for that item in a target warehouse, and (c) a Sales Return filed without `return_against` into that warehouse — then inspecting the resulting Stock Ledger Entry's `valuation_rate` / `incoming_rate`.

## C. Recent commits on the affected files

Relevant commits within the last 180 days on `erpnext/stock/stock_ledger.py`, `erpnext/controllers/selling_controller.py`, `erpnext/controllers/sales_and_purchase_return.py`:

| SHA | Title | Relevance |
|---|---|---|
| `eb2119e292` | fix: zero valuation rate if returning from different warehouse | **Most relevant.** Cherry-pick from upstream that rewrote `get_filters` in `sales_and_purchase_return.py` to only trust SLE matches whose warehouse participated in the original voucher. It addresses a neighbouring case (return to a warehouse that was not on the source document) but does NOT remove `standard_rate` from the stock-ledger fallback chain, and does NOT touch the standalone-return path in `selling_controller.py:603–610`. It establishes the project's recent direction: prefer zero valuation over incorrect valuation when cost can't be determined. |
| `2d6b43fd54` | fix: reset incoming rate in selling controller if there are changes in item | Touches the same `set_incoming_rate` block; shows active maintenance of this surface. |
| `e78c750b4e` | fix(credit-note): set incoming rate as zero for expired batch | Another "prefer zero over wrong" precedent for standalone credit notes (selling_controller.py:523–532). |
| `d420ec0b22` | fix: purchase return issue | Adjacent to the return-flow area. |
| `316b6d6867` | feat: company wise valuation method | Touches `stock_ledger.py` but unrelated to the fallback chain. |

`git log --grep="standard_rate" -- erpnext/stock/stock_ledger.py` returned no hits — the `standard_rate` fallback has been in place untouched for a long time. The reporter's claim that it predates the current behaviour is consistent with blame.

## D. Similar past issues and PRs

- `gh issue list ShankHarinath/erpnext --search "valuation standalone return"` — returns only issue **#15** itself. No prior duplicates on the fork.
- `gh pr list ShankHarinath/erpnext --search "valuation return standard_rate"` — empty.
- `gh issue list frappe/erpnext …` — blocked by local `gh-shim` deny rule (`owner 'frappe'` not in allow-list). Upstream duplicates on `frappe/erpnext` could not be confirmed from this session. **Open question — see below.**

**Fix pattern inherited from `eb2119e292` and `e78c750b4e`:** the repo's established direction for "cannot determine a cost-side rate" is to prefer **zero + explicit validation / user guidance** over silently using a rate from the selling side. The proposed change in the issue aligns with that pattern.

## E. Root-cause hypothesis

`get_valuation_rate` (stock_ledger.py:1903) walks a fallback ladder: last-SLE → `Item.valuation_rate` → `Item.standard_rate` → buying `Item Price`. `Item.standard_rate` is a **selling-side** field (see `erpnext/stock/doctype/item/item.py:290`, which writes it into a selling `Item Price`), so including it in a **cost-side** fallback chain is semantically wrong. Separately, `SellingController.set_incoming_rate` (selling_controller.py:603–610) has a terminal `d.incoming_rate = d.rate` assignment that activates exactly in the standalone-return-no-prior-SLE scenario — and `d.rate` is the return document's selling rate. Either path inflates Stock Ledger `valuation_rate`/`incoming_rate`, overstates inventory value, and distorts COGS.

The fix must: (1) remove `standard_rate` from stock_ledger.py:1971–1984's fallback ladder; (2) remove or harden the `d.incoming_rate = d.rate` fallback in selling_controller.py:603–610 for standalone returns — mirroring the "zero + validation" direction set by `eb2119e292` and `e78c750b4e`. Both edits are in the same repo (`ShankHarinath/erpnext`).

**Confidence: 0.86.** The source code for both fallback paths is plainly present and matches the reporter's narrative line-for-line; the fix direction is corroborated by very recent commits in the same files adopting the exact "zero valuation over wrong valuation" principle. Confidence is not higher only because (a) no upstream `frappe/erpnext` duplicate could be confirmed (gh deny rule), and (b) no live reproduction / SLE inspection was performed. The blast radius on `get_valuation_rate` is CRITICAL (see Phase A), so the PR MUST ship regression tests covering Purchase Invoice, Purchase Receipt, Stock Entry, Stock Reconciliation, Subcontracting Receipt, Asset Capitalization, and standalone Delivery Note/Sales Invoice returns to ensure the change does not break flows that today silently lean on `Item.standard_rate`.

## Open Questions

- Upstream duplicate / discussion on `frappe/erpnext` could not be searched (`gh` shim denies owner `frappe`). Architect should manually check whether this has already been raised or fixed on `version-15` / `develop` upstream before opening a PR on the fork.
- `kind-canonix` is not running in this session; no log-side corroboration. A reproduction Stock Ledger Entry captured from a test instance would lift confidence toward 0.9.
- Removing the `standard_rate` fallback from `get_valuation_rate` will cause *existing* flows that today quietly land on `standard_rate` (notably Stock Reconciliation and Stock Entry for items with no SLE, no `Item.valuation_rate`, and no buying `Item Price`) to instead throw "Valuation Rate Missing". That is arguably desirable — but Architect/owner should decide whether to hard-remove the branch, gate it behind a Stock Settings toggle (default off), or keep it only when `voucher_type` is not a sales return.

## Recommended next step

Proceed to planning/implementation. Two coordinated edits in `ShankHarinath/erpnext`:

1. `erpnext/stock/stock_ledger.py:1971–1984` — drop the `Item.standard_rate` branch from `get_valuation_rate`'s fallback ladder; rely on `Item.valuation_rate` then buying `Item Price`; on no rate + perpetual inventory, let the existing `frappe.throw("Valuation Rate Missing")` fire.
2. `erpnext/controllers/selling_controller.py:603–610` — for standalone returns (`is_return` and no `return_against`), stop assigning `d.incoming_rate = d.rate`. Either leave zero (gated by `allow_zero_valuation_rate`) or `frappe.throw` prompting the user to first establish valuation via Stock Reconciliation / an opening Stock Entry.

Add regression tests for: (a) standalone Sales Return into a warehouse with no SLE — assert `incoming_rate == 0` or clear error, never `d.rate` nor `standard_rate`; (b) Purchase Receipt with no prior SLE and `standard_rate` set but no `valuation_rate`/buying price — assert the "Valuation Rate Missing" error path stays intact; (c) Stock Entry / Stock Reconciliation paths that today rely on the existing behaviour.

---

<!--phoenix:scout-summary
affected_repos:
  - ShankHarinath/erpnext
confidence: 0.86
recommended_next_step: proceed
-->
