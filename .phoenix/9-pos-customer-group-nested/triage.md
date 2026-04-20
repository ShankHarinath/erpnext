# Triage Report — Issue #9: POS Profile: Filter Customer Group should be nested

## Summary

When a POS Profile has a non-leaf Customer Group configured in the `customer_groups` child table, the POS customer search (at the cashier terminal) only matches customers whose `customer_group` equals one of those exact names. Customers belonging to descendant (nested) groups are excluded. Historically (and still reflected in the POS Profile's item-side logic and the dead `test_pos_profile.get_customers_list` helper), the filter was expanded using nested-set descendants via `erpnext.accounts.doctype.pos_profile.pos_profile.get_child_nodes`. The new POS UI refactor (PR #20789, Jul 2020) built a parallel customer-side path that passes the raw flat list to a generic Frappe `Link` field query, skipping the tree expansion. No helper equivalent to `get_item_groups` was introduced for customers.

## Root-cause hypothesis

The client-side code in `pos_controller.prepare_app_defaults` (`erpnext/selling/page/point_of_sale/pos_controller.js:124`) flattens the POS Profile's `customer_groups` child table into a list of leaf names:

```js
this.settings.customer_groups = profile.customer_groups.map(group => group.customer_group);
```

`pos_item_cart.make_customer_selector` (`erpnext/selling/page/point_of_sale/pos_item_cart.js:295-306`) then sets this flat list as a hard filter on the customer search dialog:

```js
this.allowed_customer_groups = settings.customer_groups;  // ctor, line 7
...
if (allowed_customer_group.length) {
    query.filters = {
        customer_group: ['in', allowed_customer_group]  // line 304
    }
}
```

The `erpnext.controllers.queries.customer_query` (`erpnext/controllers/queries.py:82-115`) consumes the filter via `get_filters_cond(doctype, filters, conditions)` and produces SQL of the form `... and customer_group in ('Parent Group Name')`. Because the list only ever contains the configured parent names — never their nested-set descendants — customers in child customer groups never pass the `IN` predicate.

Contrast with the item-group path, which is correctly nested:
- `erpnext/accounts/doctype/pos_profile/pos_profile.py:107-116` → `get_item_groups` expands each configured item group using `get_child_nodes('Item Group', data.item_group)` (which walks `lft`/`rgt`).
- `erpnext/selling/page/point_of_sale/point_of_sale.py:163-169` → `get_item_group_condition` uses that expanded list.

There is NO analogous `get_customer_groups(pos_profile)` helper, and no server-side whitelisted method for the POS customer selector to route through that would perform the expansion. The legacy helper `get_customers_list` in `erpnext/accounts/doctype/pos_profile/test_pos_profile.py:34-47` DOES do the nested expansion correctly (using `get_child_nodes('Customer Group', …)`) — but it lives in the test module and is unused by the runtime POS UI path, which is itself a strong indicator that the logic was dropped during the refactor.

This matches the reporter's statement: "Earlier it was finding under the selected group and also under the nested groups." The legacy POS (pre-#20789) used helper functions like those still in `test_pos_profile.py`; the new POS skipped equivalent expansion on the customer side.

**Suggested fix shape** (for Architect, NOT applied by Scout):
- Add a `get_customer_groups(pos_profile)` helper to `erpnext/accounts/doctype/pos_profile/pos_profile.py` mirroring `get_item_groups` (iterate child table, expand each via `get_child_nodes('Customer Group', …)`, dedupe).
- Expose a whitelisted POS-side wrapper and switch `pos_item_cart.make_customer_selector` / `pos_controller.prepare_app_defaults` to use the expanded list (either inject the expanded list into `this.settings.customer_groups` after fetching the profile, or change the customer query to a POS-specific server-side query that expands on the fly).
- Keep the JS-side `in` filter semantics, just with the expanded list.

## Evidence

### Phase A — Code map (GitNexus + Read)

| Signal | Resolved to | File |
|---|---|---|
| POS Profile customer group filter (client) | `prepare_app_defaults` flattens `profile.customer_groups` | `erpnext/selling/page/point_of_sale/pos_controller.js:122-126` |
| POS Profile customer group filter (cart) | `POSItemCart.allowed_customer_groups` + `make_customer_selector` | `erpnext/selling/page/point_of_sale/pos_item_cart.js:7, 295-335` |
| Customer server-side query | `customer_query` (generic filter apply, no tree expansion) | `erpnext/controllers/queries.py:79-115` |
| Nested expansion for ITEM groups (reference pattern) | `get_item_groups` → `get_child_nodes` | `erpnext/accounts/doctype/pos_profile/pos_profile.py:107-121` |
| Nested expansion for CUSTOMER groups (present in tests, missing in runtime) | `get_customers_list` | `erpnext/accounts/doctype/pos_profile/test_pos_profile.py:34-47` |
| POS Profile schema confirms `customer_groups` child table | `pos_profile.json` fieldname `customer_groups` | `erpnext/accounts/doctype/pos_profile/pos_profile.json:33,145` |

GitNexus `query("POS Profile customer group filter")` surfaced process `proc_93_get_items` (item-side) but no customer-side process — consistent with the hypothesis that no equivalent runtime flow exists. `mcp__gitnexus__impact(get_item_groups, upstream)`: LOW risk, no callers outside POS UI; not the subject of the fix but used as a reference pattern.

### Phase B — Runtime (kubectl)

Skipped by design: the issue has no error signatures, no stack traces, no pre-extracted error regex; it is a silent filtering defect (missing rows in a search dropdown). `kubectl logs` would not surface anything actionable here — no server exception is raised, the SQL query returns an empty/short result set as designed. Recording explicitly rather than fabricating runtime signal.

### Phase C — Recent commits

- `a6f98d48bc` — "refactor: POS workflow (#20789)" (Jul 2020). Introduced the new POS (`selling/page/point_of_sale/*` with `pos_controller.js`, `pos_item_cart.js`). Also where `profile.customer_groups.map(group => group.customer_group)` was first introduced (`git log -S"profile.customer_groups.map"` points to this single commit). Strong root-cause correlation: this is when the customer-side nested expansion was lost, leaving only the flat list.
- `f591a220c9` — "Fetch items of item groups defined in the pos profile (#11763)" (Nov 2017). Introduced `get_item_groups` + `get_child_nodes` in `pos_profile.py`. Establishes the reference pattern and proves nested expansion was intentional.
- No commits in the last 30 days touch these files (`git log --since='30 days ago' --oneline -- erpnext/selling/page/point_of_sale/pos_item_cart.js erpnext/selling/page/point_of_sale/pos_controller.js erpnext/accounts/doctype/pos_profile/pos_profile.py`). Regression is long-standing, not recently regressed; matches reporter ("Earlier it was …" ≈ version before v13).

### Phase D — Similar past issues and PRs (fork only, per counterfactual constraint)

- `gh issue list --repo ShankHarinath/erpnext --search "customer group nested pos"` → only the present issue #9 returned. No other fork-side triage history.
- `gh pr list --repo ShankHarinath/erpnext --search "customer group pos nested"` → empty.
- Upstream `frappe/*` search is blocked by the counterfactual shim; no upstream PR was consulted.
- **Leak check:** no upstream fix SHA, PR number, or commit body was accessed during this investigation. None appeared in local `git log` either. Clean.

### Phase E — Confidence

High confidence that the missing nested expansion on the customer-side client path is the defect. The evidence is symmetric and strongly analogous to the item-group branch that still works: same project, same POS Profile doctype, parallel child tables (`item_groups` / `customer_groups`), parallel `get_child_nodes` helper already written for the Customer Group tree in the test module, and a single refactor commit that explains when the divergence was introduced.

## Affected files (where a fix will land)

- `erpnext/accounts/doctype/pos_profile/pos_profile.py` — add `get_customer_groups(pos_profile)` helper next to `get_item_groups`.
- `erpnext/selling/page/point_of_sale/pos_controller.js` — populate `this.settings.customer_groups` with the expanded list (via whitelisted server call) instead of the flat `map`.
- `erpnext/selling/page/point_of_sale/pos_item_cart.js` — no behavioural change needed if `allowed_customer_groups` already contains descendants; otherwise switch to a `get_query` that server-expands.
- (Optional) `erpnext/accounts/doctype/pos_profile/test_pos_profile.py` — keep/refresh the test that counts customers using the same expansion to prevent regression.

## Open questions for Architect

1. Should the nested expansion happen on the **server** (new whitelisted helper called from JS) or on the **client** by hydrating `this.settings.customer_groups` via a single server round-trip in `prepare_app_defaults`? Server-side tends to be safer — permission-aware and consistent with `get_item_groups`.
2. Should the fix also refactor the customer search to go through a POS-specific whitelisted query (parallel to `item_group_query`) so the expansion logic has a single owner, instead of leaning on Frappe's generic `customer_query`?
3. Is there any caller of the POS Profile's `customer_groups` child table besides the POS UI (e.g. reports, other pages) that should also be audited for the same flat-list bug? (A quick `Grep` for `customer_groups` across the repo showed only validation code in `pos_profile.py` and the JS/test references; no other consumer found — but worth confirming.)
4. No runtime log inspection was performed (no error signal). If Architect wants runtime confirmation before planning, a manual repro is needed: create a POS Profile with a parent customer group, assign a customer to a child group, open POS, search for that customer — expect: not found (current), Should-be: found (post-fix).

## Recommended next step

Architect should plan a small 1-file-add-plus-2-file-edit change: add `get_customer_groups` in `pos_profile.py`, hydrate `this.settings.customer_groups` with its output in `pos_controller.js`, and verify `pos_item_cart.js` needs no changes. Add a runtime test along the lines of the existing `test_pos_profile` customer-count assertion to prevent regression.

---

```yaml
# machine-readable
affected_repos:
  - erpnext
confidence: 0.88
recommended_next_step: plan
risk: LOW
leak_signals: none
runtime_evidence: skipped_no_error_signature
```
