# Triage Report — Issue #21: Unable to Split Asset Created via Asset Capitalization

## Summary

When an Asset created via `Asset Capitalization → Create New Composite Asset` is split via `Split Asset`, submission of the newly-created split asset throws `Please capitalize this asset before submitting.` The `before_submit` guard added to `Asset` in commit `e04353fc31` checks `is_composite_asset and not has_active_capitalization(self.name)`. The split creates a new Asset record by `frappe.copy_doc(existing_asset)` which inherits `is_composite_asset=True`, but no Asset Capitalization document is ever created with that new asset's name as `target_asset`, so `has_active_capitalization(new_asset.name)` returns `False` and the submit is blocked.

## Root cause hypothesis

**Regression introduced by `e04353fc31` ("fix: add server side validation", 2025-12-18, author khushi8112)**, which added the server-side guard:

```python
# erpnext/assets/doctype/asset/asset.py:245-247
def before_submit(self):
    if self.is_composite_asset and not has_active_capitalization(self.name):
        frappe.throw(_("Please capitalize this asset before submitting."))
```

`has_active_capitalization` (`asset.py:1317-1321`) queries:

```python
frappe.db.count("Asset Capitalization", filters={"target_asset": asset, "docstatus": 1})
```

The split flow in `process_asset_split` → `create_new_asset_from_split` → `log_asset_activity` (`asset.py:1367-1423`) does:

1. `new_asset = frappe.copy_doc(existing_asset)` — copies all fields including `is_composite_asset=1`.
2. `set_split_asset_values` sets `asset_doc.split_from = existing_asset.name` (only when `is_new_asset` is True) but does NOT clear `is_composite_asset`, nor does it create a matching Asset Capitalization.
3. `log_asset_activity` calls `asset_doc.insert()` then `asset_doc.submit()` — which triggers `before_submit` on the fresh record whose name does not appear as any `Asset Capitalization.target_asset`, so the guard throws.

The same guard also blocks updating the existing asset after split because `process_asset_split` calls `asset_doc.save()` on the existing asset with `flags.ignore_validate_update_after_submit = True` — but the throw occurs in the NEW asset's `before_submit`, so the primary failure surface is submission of the new (split-off) asset.

The marker `flags.is_split_asset = True` is set on `asset_doc` at line 1382 (already used to skip `validate_linked_purchase_documents` at line 485), but `before_submit` does NOT consult that flag. That is the missing check.

## Evidence

### A — Static evidence (code)

- **Throw site (server):** `erpnext/assets/doctype/asset/asset.py:245-247` — `Asset.before_submit`.
- **Throw site (client):** `erpnext/assets/doctype/asset/asset.js:83-87` — identical check on `frm.doc.is_composite_asset && !frm.has_active_capitalization`.
- **Capitalization query:** `erpnext/assets/doctype/asset/asset.py:1316-1321` — `has_active_capitalization` filters on `target_asset=<asset_name>`.
- **Split entry point:** `erpnext/assets/doctype/asset/asset.py:1346-1359` — `split_asset` (whitelisted).
- **Asset copy + submit:** `erpnext/assets/doctype/asset/asset.py:1377-1423` — `process_asset_split` + `log_asset_activity` (`asset_doc.insert()` → `asset_doc.submit()` on line 1415, 1422).
- **`is_split_asset` flag precedent:** `erpnext/assets/doctype/asset/asset.py:484-486` — `validate_linked_purchase_documents` already short-circuits on `self.flags.is_split_asset`. `before_submit` should do the same.
- **Gap in tests:** `erpnext/assets/doctype/asset/test_asset.py:477-539` (`test_asset_splitting`) exercises only a non-composite asset, so the regression slipped through CI.

### B — Runtime evidence (kubectl)

**BLOCKER — investigation gap.** `kubectl --context kind-canonix get pods` failed with connection-refused against `127.0.0.1:59249`. Cluster not running locally at triage time. No pod logs were inspected. Static code evidence is sufficient because the error message and control flow can be traced unambiguously, but a runtime repro should be added to the fix verification checklist.

### C — Recent commits on affected files

- `e04353fc31` — `fix: add server side validation` (khushi8112, 2025-12-18) — introduced the guard. Very likely the commit that caused the regression (the reporter notes "did not occur earlier").
- `9a2710b9d7` — `fix(asset): handle partial asset sales by splitting remaining quantity` — later modifies split logic but does not touch the guard.
- `e7e6567792` — `fix(asset): skip purchase document validation while splitting existing asset` — sets the `is_split_asset` pattern precedent that `before_submit` should follow.

### D — Similar past issues/PRs

- No prior matching issues in `ShankHarinath/erpnext` beyond this one.

## Suggested fix direction (NOT prescriptive)

The minimal, surgical fix is to teach the guard about the split path, mirroring the precedent already used by `validate_linked_purchase_documents`:

```python
def before_submit(self):
    if self.flags.is_split_asset or self.split_from:
        return
    if self.is_composite_asset and not has_active_capitalization(self.name):
        frappe.throw(_("Please capitalize this asset before submitting."))
```

Rationale:
- `self.split_from` is set by `set_split_asset_values` on the newly-split asset (`asset.py:1402`), making it a durable check that survives re-submits and is not dependent on in-memory flags.
- `self.flags.is_split_asset` covers the existing-asset save path inside `process_asset_split`.

Matching change needed in `asset.js` `before_submit` (line 83-87) for parity when the user re-submits from the UI.

A regression test should be added: create a composite asset via Asset Capitalization, submit it, split it, and assert the split succeeds and both resulting assets are submitted.

## Open questions

- Runtime repro could not be captured — kubectl cluster `kind-canonix` was unreachable during triage. Fix verification should include a manual repro.
- Should the guard also be skipped when `split_from` is set but `is_composite_asset` is False for the parent? (Current proposal is safe because we only short-circuit; we don't skip other validation.)

## Confidence

**0.88** — The throw message is matched exactly (`asset.py:247`), the introducing commit is identified (`e04353fc31`), the data-flow gap between `split_asset` copying `is_composite_asset` and `has_active_capitalization` filtering on `target_asset` is direct, and the precedent (`flags.is_split_asset` skip in sibling validator) is right there in the same file. Runtime log corroboration was not possible, which is the only reason this is not higher.

## Recommended next step

`plan` — root cause is localized and well-understood; Architect can design a minimal fix + regression test.

## Leak signals

- `gh` search on upstream repository for this bug class was denied by the shim allowlist — no upstream cross-check was performed during triage.

---

```yaml
# machine-readable footer
affected_repos:
  - erpnext
confidence: 0.88
recommended_next_step: plan
primary_symbol: Asset.before_submit
primary_file: erpnext/assets/doctype/asset/asset.py
regression_commit: e04353fc3103d75fffc884b87aac85d54e54757a
blockers:
  - kubectl context kind-canonix unreachable (connection refused) — no runtime logs captured
  - gh search on upstream denied by shim allowlist — no upstream cross-check
```
