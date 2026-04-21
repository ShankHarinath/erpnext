# Triage Report — POS Invoice 100% Discount Fails Due to Decimal Rounding Error (0.01)

**Issue:** ShankHarinath/erpnext#19
**Reporter:** ShankHarinath
**Revision:** 1
**Classification confidence (from triage skill):** 0.95

## Summary

When applying a 100% `additional_discount_percentage` to a POS Invoice whose single item carries tax **included in the print rate** (e.g. selling price ₹1300 with 18% GST inclusive), the client-side and server-side `calculate_taxes_and_totals` pipeline distributes the discount over a `total_for_discount_amount` value that does **not** match `discount_amount`. The numerator (`discount_amount`) is the rounded `grand_total` (₹1300), but the denominator (`grand_total_for_distributing_discount`) is captured as `doc.taxes[-1].total` **before** `adjust_grand_total_for_inclusive_tax` applies the 0.01 rounding correction (so ₹1299.99). The resulting per-item `distributed_amount` is not exactly `item.net_amount`, so the final `net_total` ends at ±₹0.01 while `grand_total` rounds to ₹0.00. The ≥ 0 requirement on `base_grand_total` (Currency field with `reqd: 1`) still validates, but the residual net_total breaks internal consistency checks and POS submission fails with `"Incorrect value: Grand Total (Company Currency) must be >= 0.0"`.

Root cause lives in `calculate_taxes_and_totals.calculate_taxes` and `get_total_for_discount_amount` (both Python and JS mirrors). It was introduced by commit `5741458c94` ("fix: get total without rounding off tax amounts for distributing discount", PR upstream of frappe/erpnext v15, backported onto the current `develop` HEAD `dd910bb4cc`).

## A. Symbol → code mapping

| Signal | Symbol | File | Lines |
|---|---|---|---|
| 100% discount on tax-inclusive POS Invoice | `calculate_taxes_and_totals.apply_discount_amount` | `erpnext/controllers/taxes_and_totals.py` | 800–855 |
| `discount_amount` computed from rounded `grand_total` | `calculate_taxes_and_totals.set_discount_amount` | `erpnext/controllers/taxes_and_totals.py` | 756–798 |
| `grand_total_for_distributing_discount` capture (pre-rounding) | `calculate_taxes_and_totals.calculate_taxes` | `erpnext/controllers/taxes_and_totals.py` | 455–475 |
| `total_for_discount_amount` uses pre-rounding total | `calculate_taxes_and_totals.get_total_for_discount_amount` | `erpnext/controllers/taxes_and_totals.py` | 857–898 |
| Rounding correction happens AFTER `grand_total_for_distributing_discount` is captured | `calculate_taxes_and_totals.adjust_grand_total_for_inclusive_tax` | `erpnext/controllers/taxes_and_totals.py` | 648–672 |
| JS mirror of `calculate_taxes` (POS form, live calc) | `TaxesAndTotals.calculate_taxes` | `erpnext/public/js/controllers/taxes_and_totals.js` | 452–478 |
| JS mirror of `apply_discount_amount` | `TaxesAndTotals.apply_discount_amount` | `erpnext/public/js/controllers/taxes_and_totals.js` | 776–831 |
| JS mirror of `get_total_for_discount_amount` | `TaxesAndTotals.get_total_for_discount_amount` | `erpnext/public/js/controllers/taxes_and_totals.js` | 833–870 |
| Currency field marked `reqd:1`, triggering Frappe framework `>= 0` validation | `base_grand_total` / `grand_total` | `erpnext/accounts/doctype/pos_invoice/pos_invoice.json` | 920–977 |
| POS discount control setting `additional_discount_percentage` | `PointOfSale.ItemCart.show_discount_control` | `erpnext/selling/page/point_of_sale/pos_item_cart.js` | 397–434 |

**Search radius:** GitNexus has 11 repos indexed. The home repo `ShankHarinath/erpnext` is not in any group and the bug is contained within the erpnext monorepo (Python controller + its JS mirror). No cross-repo investigation needed. GitNexus index for erpnext is fresh (indexed 2026-04-21T09:31:38Z, matching HEAD `dd910bb4cc`).

## B. Runtime evidence (kubectl)

**Blocker:** The `kind-canonix` cluster is not reachable from this environment (`kubectl get pods` → connection refused on `127.0.0.1:59249`). Runtime corroboration from logs is not possible.

Falling back to static reproduction trace using the reporter's steps:

Scenario: single line item, `rate=1300`, GST 18% with `included_in_print_rate=1`, `apply_discount_on="Grand Total"` (POS default, see `pos_invoice.json:880`), `additional_discount_percentage=100`.

First JS `_calculate_taxes_and_totals` pass (pre-discount):
1. `determine_exclusive_rate` → `item.net_amount = 1101.69` (1300 / 1.18, rounded to 2dp).
2. `calculate_net_total` → `doc.net_total = 1101.69`.
3. `calculate_taxes`: `tax.tax_amount = 1101.69 × 0.18 = 198.30` (rounded); `last_tax.total = 1299.99`.
4. Because `additional_discount_percentage = 100` is already truthy, the discount-on-grand-total branch (line 452 JS / line 455 Python) executes on the **first** pass and captures `this.grand_total_for_distributing_discount = 1299.99` (from `doc.taxes[-1].total`, **before** `adjust_grand_total_for_inclusive_tax`).
5. `adjust_grand_total_for_inclusive_tax` → `grand_total_diff = 0.01`.
6. `calculate_totals` → `doc.grand_total = 1299.99 + 0.01 = 1300.00`. Display is correct at this point.

`calculate_discount_amount`:
7. `set_discount_amount` → `discount_amount = 1300 × 100/100 = 1300`.
8. `apply_discount_amount` → `total_for_discount_amount = get_total_for_discount_amount()` returns `(this.grand_total_for_distributing_discount(1299.99) || doc.grand_total) - 0 = 1299.99`.
9. For the single item: `distributed_amount = 1300 × 1101.69 / 1299.99 ≈ 1101.698463`. `adjusted_net_amount = 1101.69 − 1101.698463 = −0.008463`. `flt(…, 2) = −0.01`. `item.net_amount = −0.01`. The rounding-difference re-adjustment loop (lines 813–820) computes `rounding_difference = flt(expected_net_total − net_total, 2) = flt(0.001537, 2) = 0`, so no correction is applied.
10. Recursive `_calculate_taxes_and_totals` runs with `discount_amount_applied = true`: `calculate_item_values` and `determine_exclusive_rate` short-circuit on that flag; `calculate_net_total` re-sums `item.net_amount` → `doc.net_total = −0.01` (reporter sees the sign as `+0.01` depending on which rounding direction the reporter's precision hits — the user-visible residual magnitude is `0.01`). Taxes recompute to 0, `last_tax.total = 0`. `adjust_grand_total_for_inclusive_tax` with `discount_amount_applied` and `discount_amount = 1300` → `diff = doc.total(1300) + 0 − 0 − 1300 = 0`, so `grand_total_diff = 0`. `grand_total = 0`.

End state: **`net_total = −0.01`, `grand_total = 0.00`** — exactly the mismatch the reporter describes. The "must be >= 0.0" error is Frappe's framework-level Currency validation on `base_grand_total` (a `reqd:1` Currency field — `pos_invoice.json:920–928`); the reqd/non-negative check rejects the row because of the inconsistency surfaced after `calculate_total_advance` / consolidation, not an `erpnext/`-owned `frappe.throw`. The only erpnext validation that mentions grand total semantics (`set_discount_amount:785`) permits this case (discount ≤ grand_total).

## C. Suspicious commits on affected files

`git log --since="18 months ago"` on `erpnext/controllers/taxes_and_totals.py` and `erpnext/public/js/controllers/taxes_and_totals.js`:

| SHA | Date | Summary | Relevance |
|---|---|---|---|
| `5741458c94` | 2025-04-19 | **fix: get total without rounding off tax amounts for distributing discount** | **Introduced `grand_total_for_distributing_discount` capture before `adjust_grand_total_for_inclusive_tax`.** This is the root-cause commit. Touches both Python and JS. |
| `f25bf6dbd2` | 2025-03-31 | fix: improved rounding adjustment when applying discount (#46720) | Introduced the rounding-difference re-adjustment loop in `apply_discount_amount`. That loop's `flt(…, precision("net_total"))` guard fails at 2dp when the residual is ≈ 0.001, so the residual passes through silently. |
| `b3fdef8d19` | 2025 | fix: ensure fresh `grand_total_diff` is used for each calculation | Related. Resets `grand_total_diff` per calc. |
| `fd252da6b1` | 2025 | fix: use `grand_total_diff` instead of `rounding_adjustment` in `taxes_and_totals` | Background for the naming. |

The earlier chain (`3ffd50c772` "decimal break for discount amount", `0e8f8677b8` "decimal break with dirty") are EOL-normalisation-only commits (CRLF → LF), not logic changes.

## D. Similar past issues and PRs

- Local/test-fixture coverage: `TestPurchaseInvoice.test_apply_discount_on_grand_total` (`erpnext/accounts/doctype/purchase_invoice/test_purchase_invoice.py:2825`) and `test_apply_discount_on_grand_total_with_previous_row_total_tax` (:2859) exercise discount-on-grand-total, but **neither tests the 100%-discount-plus-inclusive-tax combination** that the reporter hit.
- `TestPOSInvoiceMergeLog.test_consolidation_round_off_error_1` (`erpnext/accounts/doctype/pos_invoice_merge_log/test_pos_invoice_merge_log.py:196`) tests a sibling round-off problem at consolidation time, reinforcing that the `grand_total` vs `net_total` residual is a known failure mode in the POS pipeline.
- `gh issue list --search "100% discount rounding"` against the current repo returns only issue #19 itself; `gh pr list --search "100% discount rounding"` returns nothing merged here. `gh issue list --repo frappe/erpnext` failed (exit 77 — auth/permissions against upstream). Cross-ref with upstream cannot be completed from this environment.

## E. Root-cause hypothesis

In `calculate_taxes` (both Python `erpnext/controllers/taxes_and_totals.py:455–475` and JS `erpnext/public/js/controllers/taxes_and_totals.js:452–478`), the value stored in `grand_total_for_distributing_discount` is `doc.taxes[-1].total`, which for **tax-inclusive** pricing is ₹0.01 less than the final `grand_total` (the gap is later filled by `adjust_grand_total_for_inclusive_tax` via `grand_total_diff`). `get_total_for_discount_amount` returns that pre-adjustment value as the denominator for the discount distribution. Meanwhile `set_discount_amount` multiplies the already-adjusted `grand_total` by `additional_discount_percentage / 100`, so the numerator is the post-adjustment ₹1300. For a 100% discount, the ratio `discount_amount / total_for_discount_amount = 1300/1299.99 ≠ 1`, so `distributed_amount` does not equal `item.net_amount` and a ±₹0.01 residual remains in `item.net_amount` (and therefore `net_total`). The rounding-difference re-adjustment loop inside `apply_discount_amount` cannot detect a sub-cent residual because its precision is `net_total` (2dp), so the residual propagates.

The fix direction (at whichever level the team prefers) is one of:

1. **In `calculate_taxes`** (preferred — fixes both Python & JS symmetrically): after `adjust_grand_total_for_inclusive_tax` has computed `grand_total_diff` and `calculate_totals` has produced the final `grand_total`, re-set `grand_total_for_distributing_discount = doc.grand_total` (i.e., include the inclusive-tax rounding adjustment in the distribution denominator). Current ordering in `_calculate()` captures it too early.
2. **In `get_total_for_discount_amount`**: when inclusive taxes are present, add back the `grand_total_diff` to the returned total, so numerator and denominator agree.
3. **In `apply_discount_amount`**: when `additional_discount_percentage == 100`, short-circuit the per-item distribution and set every `item.net_amount = 0` and `distributed_discount_amount = item.net_amount` directly (mirrors the existing `item.discount_percentage == 100 → item.rate = 0.0` fast-path in `calculate_item_values:169`).

Option 3 is the smallest, most localised, and symptom-targeted change; options 1/2 address the broader class of inclusive-tax + discount rounding drift.

**Confidence: 0.82** — static trace is unambiguous and consistent with the reporter's exact symptom (net_total ₹0.01, grand_total ₹0.00, submission blocked by >= 0 validation), the root-cause commit is identifiable (`5741458c94`) and the affected symbols have low upstream blast radius (LOW per `gitnexus_impact`). Confidence is not 0.9+ because I could not corroborate with a live reproduction (cluster unreachable) or confirm the user's sign convention for the 0.01 residual.

## Open questions / blockers

- **Cluster unreachable.** `kubectl --context kind-canonix get pods` → connection refused. No runtime logs could be pulled; Phase B is static-only. If Phoenix can route me to a working `kind-canonix` (or a site with a reproducible POS Invoice submission), a single-shot repro + log confirms the exact sign of the residual and the exact validator that rejects the save.
- **Upstream issue/PR search failed.** `gh issue list --repo frappe/erpnext` exited 77 (auth). Worth confirming from a session with upstream `gh` auth whether this has already been reported/fixed upstream.
- **Exact Frappe version and Currency precision.** The trace assumes 2-decimal Currency precision (INR default). If the reporter's Currency precision is configured differently (e.g., System Settings → float precision 3), the residual magnitude and sign may shift. Precision setting is not captured in the issue.
- **`apply_discount_on` actually used.** POS's JSON default is "Grand Total" (`pos_invoice.json:880`); the reporter didn't explicitly state which was in effect, though the symptom (net_total 0.01 with grand_total 0.00) is most consistent with "Grand Total" on an inclusive-tax line.

## Recommended next step

**Fix** in `ShankHarinath/erpnext`. The bug is fully isolated to `calculate_taxes_and_totals` (Python) and its JS mirror; both must be patched in lock-step to avoid the JS showing a different total than the server computes on save. Architect should prefer Option 3 (100% short-circuit) for a minimal, well-tested fix, or Option 1 (re-order `grand_total_for_distributing_discount` capture) for the broader inclusive-tax rounding-drift class. New tests to add: `test_pos_invoice_100_percent_discount_with_inclusive_tax` in `erpnext/accounts/doctype/pos_invoice/test_pos_invoice.py`, and a matching `test_apply_discount_on_grand_total_with_inclusive_tax_100_percent` in `test_purchase_invoice.py`/`test_sales_invoice.py`.

---
```yaml
affected_repos:
  - ShankHarinath/erpnext
confidence: 0.82
recommended_next_step: fix
```
