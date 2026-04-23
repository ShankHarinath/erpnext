> **Note to orchestrator:** `issues_comment` returned HTTP 403 (`Resource not accessible by personal access token`). The breadcrumb was NOT posted to ShankHarinath/erpnext#9. Retry with a token that has `issues:write` on the repo, or skip.

# Triage Report — POS Profile: Filter Customer Group should be nested

- **Issue:** ShankHarinath/erpnext#9
- **Reporter:** ShankHarinath (2026-04-20)
- **Home repo:** ShankHarinath/erpnext (GitNexus `erpnext`)
- **Affected repo(s):** erpnext (single-repo)
- **Rev:** 1
- **Classification confidence (input):** 0.95
- **Runtime source:** N/A — UX / deterministic bug, no logs provided, Phase B skipped

---

## 1. Summary

The POS v13 UI (`erpnext/selling/page/point_of_sale/`) filters the customer selector by `customer_group IN (...)` using **only the customer groups explicitly listed in the POS Profile**, without expanding to their descendant groups in the `Customer Group` nested-set tree. Pre-v13 behavior (and the behavior still used for `item_groups` on the same POS Profile, and by `test_pos_profile.get_customers_list`) walked the tree via `get_child_nodes` / the NSM `lft`/`rgt` range. The customer path lost that expansion in the v13 POS rewrite (`a6f98d48bcb`, PR #20789) and the subsequent filter-wiring commit (`7acb42d6df3`, PR #24102, 2020-12-10).

Result: a POS Profile set to parent group `Commercial` no longer surfaces customers in `Commercial > Retail` or `Commercial > Wholesale`; only customers whose `customer_group` equals `Commercial` exactly appear.

---

## 2. Dataflow diagram (A.9)

```
[POS Profile doc]
  customer_groups[] = [{customer_group: "Commercial"}, ...]
        │
        ▼
 pos_controller.js:122–126  prepare_app_defaults()
   ├─ frappe.db.get_doc("POS Profile", …)
   └─ this.settings.customer_groups =
        profile.customer_groups.map(g => g.customer_group)   ← bug: flat list; no NSM expansion
        │
        ▼
 pos_item_cart.js:7           constructor
   this.allowed_customer_groups = settings.customer_groups   [plain array]
        │
        ▼
 pos_item_cart.js:295–306     make_customer_selector()       ← symptom site
   query.filters = { customer_group: ['in', allowed_customer_group] }
        │
        ▼ (Frappe RPC: frappe.client.get_list → customer_query)
 erpnext/controllers/queries.py:82–115  customer_query()
   get_filters_cond(doctype, filters, conditions)
   → SQL: AND `customer_group` IN ('Commercial')             ← exact-match; descendants never matched
        │
        ▼
 [tabCustomer rows where customer_group = 'Commercial']      ← symptom observed by reporter
```

Sibling (contract cousin) — same POS Profile, item path, works correctly:

```
 pos_profile.py:107–116 get_item_groups()
   for data in pos_profile.get('item_groups'):
     item_groups.extend([d.name for d in get_child_nodes('Item Group', data.item_group)])
                                              ▲
                              pos_profile.py:118–121  get_child_nodes()
                                SELECT name FROM `tabItem Group`
                                WHERE lft >= {lft} AND rgt <= {rgt}      ← NSM walk; the pattern that's missing on customer side
```

---

## 3. Cause-flow diagram (A.10)

| Layer | Observed state | Expected state | Annotation |
|---|---|---|---|
| POS Profile (DB) | `customer_groups = [Commercial]` | same | baseline |
| pos_controller.js:124 build | `settings.customer_groups = ['Commercial']` (flat, literal) | `['Commercial', 'Commercial - Retail', 'Commercial - Wholesale', ...]` (expanded) | ← bug: flattening without NSM expansion |
| pos_item_cart.js:304 filter payload | `{customer_group: ['in', ['Commercial']]}` | `{customer_group: ['in', <descendants>]}` OR `{customer_group: ['descendants of', 'Commercial']}` | ← latent: filter operator choice |
| controllers/queries.py customer_query → `get_filters_cond` | emits `WHERE customer_group IN ('Commercial')` | `IN (<descendants>)` or equivalent tree predicate | mechanical consequence of upstream |
| Customer search UI | hides `Commercial - Retail` customers | shows them | ← symptom |

The A.9 defect-site (`pos_controller.js:124`) matches the A.10 `← bug` marker. Reconciled.

---

## 4. Root-cause hypothesis

**H1 — POS v13 customer_group filter skips the Customer Group NSM expansion that existed pre-v13 and still exists for item_groups on the same DocType.**

- Primary causal symbol: `prepare_app_defaults` in `erpnext/selling/page/point_of_sale/pos_controller.js` (build site), cooperating with `make_customer_selector` in `erpnext/selling/page/point_of_sale/pos_item_cart.js` (filter site).
- Falsifiable trace: set up a POS Profile with a single row `customer_group = <any parent group that has children>`, create test customers under a child group, open POS → customers under the child group do NOT appear in the selector. Replace the `map(group => group.customer_group)` line with an expansion through `get_descendants_of("Customer Group", …)` (or change the filter operator to `descendants of`) → child-group customers reappear.
- Root-cause confidence: **0.92**
- Fix-direction confidence: **0.88**

---

## 5. Hypotheses considered

| ID | Rank | Causal | Status | Notes |
|---|---|---|---|---|
| H1 | 1 | pos_controller.js:124 + pos_item_cart.js:304 — flat `IN` filter, no NSM expansion | **confirmed (static)** | Defect visible in source; sibling path (`item_groups`) uses `get_child_nodes`; test helper `get_customers_list` also uses it — confirms intended behavior. |
| H2 | 2 | `customer_query` (`erpnext/controllers/queries.py:82`) should transparently treat `customer_group` filter as tree | refuted | `customer_query` has ~60 consumers (`codebase_impact` = CRITICAL) — changing its semantics is a behavior change for Payment Entry, Project, Lead, Item, Sales Invoice, etc. Fix belongs at POS callsite. |
| H3 | 3 | POS Profile child-table schema changed (`customer_groups` field in `pos_profile.json`) silently dropped a join | refuted | `pos_profile.json:145` still defines `customer_groups` as a Table linking to `POS Customer Group`. Schema is intact; the regression is in how the consumer reads it. |
| H4 | 4 | Frappe core `get_filters_cond` stopped treating `customer_group` as an NSM link | refuted | Other ERPNext paths still use explicit `get_descendants_of` themselves (warehouse, account, company cases); `get_filters_cond` has always required explicit tree operator. |

Confidence gate triggered: H1 confidence ≥ 0.85, all pre-extracted signals mapped, runtime corroboration unavailable by bug class. Parallel investigator subagents skipped.

---

## 6. Evidence refs

- `erpnext/selling/page/point_of_sale/pos_controller.js:122–126` — build site, flattening map
- `erpnext/selling/page/point_of_sale/pos_item_cart.js:7` — settings → `allowed_customer_groups`
- `erpnext/selling/page/point_of_sale/pos_item_cart.js:295–306` — filter wiring into `customer_query`
- `erpnext/controllers/queries.py:82–115` — `customer_query`, uses `get_filters_cond`
- `erpnext/accounts/doctype/pos_profile/pos_profile.py:107–116` — `get_item_groups` (correct sibling path)
- `erpnext/accounts/doctype/pos_profile/pos_profile.py:118–121` — `get_child_nodes` helper
- `erpnext/accounts/doctype/pos_profile/test_pos_profile.py:34–47` — `get_customers_list` (correct reference implementation)
- `erpnext/setup/doctype/customer_group/customer_group.py:25` — `get_parent_customer_groups` (NSM utility exists for Customer Group)
- `erpnext/selling/report/item_wise_sales_history/item_wise_sales_history.py:8,148` — `from frappe.utils.nestedset import get_descendants_of` (idiomatic Frappe pattern)
- `erpnext/accounts/doctype/pos_profile/pos_profile.json:33, 145` — `customer_groups` child-table schema intact
- Commit `a6f98d48bcb` (2020-07-23, "refactor: POS workflow (#20789)") — introduced v13 POS file; regression inception
- Commit `7acb42d6df3` (2020-12-10, "feat(POS): hide images & auto add item checkbox (#24102)") — finalized the flat-map filter wiring at the current lines
- `codebase_impact(customer_query)` → CRITICAL, 64 affected items / 60 processes — caller-site fix mandated

---

## 7. Affected code

| File | Lines | Change kind | Why |
|---|---|---|---|
| `erpnext/selling/page/point_of_sale/pos_controller.js` | 124 | edit | Replace flat `map` with an async expansion call (either to a new whitelisted Python endpoint or via a Frappe RPC that returns descendants). Alternatively, leave the flat list here and push expansion to the filter site. |
| `erpnext/selling/page/point_of_sale/pos_item_cart.js` | 300–306 | edit | Either consume an already-expanded list, OR change operator to `['descendants of', <root>]` — but that operator accepts a single value, so if the POS profile has N parent groups the client must pass `['in', expanded_union]`. |
| `erpnext/accounts/doctype/pos_profile/pos_profile.py` | new function, ~117 | add | Whitelisted helper `get_child_customer_groups(pos_profile)` mirroring `get_item_groups`, returning the union of descendants for all configured customer_groups. Expose via `frappe.call`. |
| `erpnext/accounts/doctype/pos_profile/test_pos_profile.py` | 21, 27–30 | edit | Existing test uses a leaf group (`{'is_group': 0}` at line 91 / single leaf `_Test Customer Group` at line 21) so it cannot detect this regression. Add a parent-with-children test case. |

No schema / contract changes. `codebase_api_impact` for the modified JS functions: local to POS page only. No other repos in the GitNexus set reference these files.

---

## 8. Devil's-advocate pass

- **Alt A — Server-side filter in `customer_query` via an additional `customer_group_descendants_of` param** (instead of expanding client-side). Would be cleaner (one roundtrip) but adds a second channel of semantics to `customer_query`; low confidence alternative.
- **Alt B — Change the `Customer Group` link filter to use the Frappe `descendants of` operator with an OR across each POS-Profile-listed group.** Frappe supports `['descendants of', single_value]` per filter, not a list; would require OR-combining filters, which is awkward in `get_filters_cond`. Lower confidence than the expansion-at-build-site approach.
- **Alt C — The POS Profile `customer_groups` child table itself should be made to accept only parent groups and expand at save-time (persisted expansion).** Rejected: violates the DocType contract (the Table link is to `Customer Group` generally) and makes the POS Profile brittle to Customer Group tree edits after save.

None of these alternatives surface a comparable-confidence competing root cause. H1 stands.

---

## 9. Scope limits

The recommended fix does NOT handle:

1. **Live tree edits after a POS session opens** — if an admin creates a new child customer group while POS is running, the cached `allowed_customer_groups` list (built in `prepare_app_defaults`) will be stale until POS is reloaded. Same staleness existed pre-regression; fix does not regress it but also does not fix it.
2. **Disabled / group-header Customer Groups in the expansion** — `get_descendants_of` returns all nodes including `is_group=1` parents. Customers cannot actually be assigned to group-header rows, so including them is harmless but technically widens the `IN` list. If the team wants a tighter filter, exclude `is_group=1` — deliberate decision.
3. **Permission-scoped groups** — if the user has restricted access to the Customer Group tree (Frappe User Permissions), the expansion should respect that. The existing `get_descendants_of(..., ignore_permissions=False)` covers this; need to ensure we don't pass `ignore_permissions=True` (which would widen).
4. **Empty POS Profile customer_groups table** — must preserve "no filter at all" semantics (current behavior): if `customer_groups` is empty, don't add any `customer_group` filter. Easy to honor; call out in tests.

---

## 10. Maintainer-review self-pass

- Review comment 1: "Why are we duplicating the NSM walk in JS? Put the expansion on the Python side and call it once from `prepare_app_defaults`." — agreed; leads to the Idiomatic fix shape below.
- Review comment 2: "`get_child_nodes` in `pos_profile.py:118–121` uses f-string interpolation for `lft`/`rgt`. If we're adding a new helper for customer groups, use `frappe.utils.nestedset.get_descendants_of` instead of duplicating this pattern — and please don't introduce new f-string SQL." — agreed.
- Review comment 3: "The existing test `test_pos_profile.test_pos_profile` picks a leaf group (`is_group=0`), so it cannot regress on this. Add a fixture that uses a parent group to exercise the descendants walk." — agreed; mandatory to prevent re-regression.

---

## 11. Pragmatism axis — fix shapes

| Shape | LOC | Files | Approach |
|---|---|---|---|
| **Minimal** | ~8 | 1 (`pos_controller.js`) | Replace line 124 with a `frappe.call` to a new whitelisted helper that returns the flat descendants list, then assign. Keep pos_item_cart.js unchanged. **Not recommended alone** — no regression test, easy to re-break. |
| **Idiomatic (recommended)** | ~35 | 3 (`pos_profile.py` new helper; `pos_controller.js` call it; `test_pos_profile.py` add parent-group case) | Add `get_child_customer_groups(pos_profile)` in `pos_profile.py` using `frappe.utils.nestedset.get_descendants_of("Customer Group", root)` per profile row, return the dedupped flat list. `pos_controller.js:124` fetches it via `frappe.call`. Mirrors the existing `get_item_groups` shape. Update test to use a parent group with known children. |
| **Architectural** | ~80+ | 5+ | Refactor both `item_groups` and `customer_groups` expansion into a shared `pos_profile_group_expander(group_type, rows)` helper; migrate `get_item_groups`, `test_pos_profile.get_items_list`/`get_customers_list` to it; parameterize filter operators. Only justified if follow-up work on POS Profile group handling is already on the roadmap. |

**Recommendation:** Idiomatic. `codebase_impact` on `customer_query` is CRITICAL (60 processes, 64 affected), which mandates keeping the fix at the POS callsite rather than inside `customer_query`. The Idiomatic variant is a direct port of the already-correct `item_groups` pattern on the same DocType, so risk is low and the behavioral ceiling (parity with pre-v13 / test_pos_profile) is exactly what the reporter is asking for.

---

## 12. Open questions

- Are there user permissions considerations for Customer Group that should bound the expansion? (Passing `ignore_permissions` in either direction changes observable behavior.) Recommend: default to permission-respecting (`ignore_permissions=False`).
- Does the team want to filter out `is_group=1` rows from the expansion? Cosmetic — suggested: leave them in (matches `get_child_nodes`).
- `issues_comment` breadcrumb POST failed with 403 (personal-access-token scope). Maintainer should either broaden the token or post the breadcrumb manually. Triage was NOT blocked.

---

## 13. Recommended next step

Dispatch a Builder on the Idiomatic fix shape. Files to open:
- `erpnext/accounts/doctype/pos_profile/pos_profile.py` (add `get_child_customer_groups`)
- `erpnext/selling/page/point_of_sale/pos_controller.js:122–126` (consume expanded list)
- `erpnext/accounts/doctype/pos_profile/test_pos_profile.py` (parent-group regression test)

No worktree fan-out required — single repo.

---

```yaml
# machine-readable footer
affected_repos:
  - erpnext
confidence: 0.90
recommended_next_step: dispatch_builder
root_cause_confidence: 0.92
fix_direction_confidence: 0.88
primary_causal_symbol: "erpnext/selling/page/point_of_sale/pos_controller.js:prepare_app_defaults"
fix_shape_recommended: idiomatic
hypotheses_count: 4
hypotheses_confirmed: 1
converged: true
breadcrumb_posted: false
breadcrumb_error: "403 Resource not accessible by personal access token"
```
