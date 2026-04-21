# Triage Report — Wrong EAN naming (issue #23)

**Revision:** 1
**Home repo:** ShankHarinath/erpnext
**Classification confidence (input):** 0.95
**Scout confidence (root cause):** 0.95

## Summary

The `Item Barcode` DocType exposes a `barcode_type` Select field whose options list contains `EAN-12`. The EAN (International Article Number) standard does not define a 12-digit variant — the retail standard is **EAN-13** (13 digits), with EAN-8 as the short variant. The 12-digit code is UPC-A (already a separate option in the list). `EAN-12` is therefore a misnomer that can mislead users into tagging either UPC-A codes or EAN-13 codes with the wrong type. The fix is a naming correction: replace `EAN-12` with `EAN-13` in both the DocType JSON `options` string and the auto-generated Python `Literal` type.

## Phase A — Code mapping

**Searched repo set:** `erpnext` (only; the home repo's GitNexus-indexed counterpart). No group membership for this repo in GitNexus. Other indexed repos (canonix-*, langchain, mattermost, twenty) are unrelated domains — none handle stock/barcode logic. No cross-service contracts involved.

**Prime suspect files (both confirmed present at reported line numbers):**

- `/Users/shashank/Canonix/phoenix-test/erpnext/erpnext/stock/doctype/item_barcode/item_barcode.py` — line 23 inside the auto-generated `Literal[...]` annotation:
  ```
  23:			"EAN-12",
  ```
- `/Users/shashank/Canonix/phoenix-test/erpnext/erpnext/stock/doctype/item_barcode/item_barcode.json` — line 28, the Select field `options`:
  ```
  28:   "options": "\nEAN\nUPC-A\nCODE-39\nEAN-12\nEAN-8\nGS1\nGTIN\nISBN\nISBN-10\nISBN-13\nISSN\nJAN\nPZN\nUPC"
  ```

**Full search for the literal across non-locale files (complete enumeration):**

- `erpnext/stock/doctype/item_barcode/item_barcode.py:23` — the Literal entry.
- `erpnext/stock/doctype/item_barcode/item_barcode.json:28` — the Select options.
- No other code, JS, HOOK, patch, fixture, or test references `EAN-12` anywhere. `EAN-13` appears nowhere (the label is simply missing).
- `erpnext/locale/main.pot:16579` and 33 localized `.po` files contain `msgid "EAN-12"` — these are translation byproducts of the options string and will regenerate when `bench update-po-files` (or equivalent) is next run. They do not need manual editing for the fix to function.

**Downstream consumers of the literal value (Phase A + `mcp__gitnexus__context`):**

- `erpnext/stock/doctype/item/item.py`:
  - Line 469: `options = frappe.get_meta("Item Barcode").get_options("barcode_type").split("\n")` — options are read dynamically from the DocType meta, so renaming `EAN-12` → `EAN-13` in the JSON auto-propagates.
  - Lines 482–493: `Item.validate_barcode` calls `convert_erpnext_to_barcodenumber(item_barcode.barcode_type.upper(), item_barcode.barcode)` and then validates with the `barcodenumber` module.
  - Lines 1068–1091: `convert_erpnext_to_barcodenumber` explicitly handles `"EAN"` (dispatch by length → `EAN8`/`EAN13`), `"UPC-A"`, `"CODE-39"`, `"ISBN-10"`, `"ISBN-13"`. It has **no** branch for `"EAN-12"`, meaning any stored row with that type falls through to `barcodenumber.barcodes()` (not a valid key), so validation is silently skipped today. Renaming to `"EAN-13"` also has no converter branch (the converter's `"EAN"` branch handles length-13 via `EAN13`), so behavior is unchanged at the converter level.
- `erpnext/stock/doctype/item/test_item.py` lines 608–659: tests enumerate several `barcode_type` values (EAN, UPC-A, CODE-39, GS1, GTIN, ISBN-*, ISSN, JAN, PZN, UPC). **None** of the test barcodes use type `EAN-12` or `EAN-13`, so tests are unaffected.
- No patch, fixture, or Frappe `custom_field` reference uses `EAN-12`.

**`gitnexus_impact` on `ItemBarcode`:** LOW risk. Single d=1 importer: `erpnext/stock/doctype/item/item.py`, and that file reads options dynamically — no hard-coded `"EAN-12"` string anywhere in it.

**Affected repos (code must change):** `ShankHarinath/erpnext` only.

## Phase B — Runtime evidence

Attempted `kubectl --context kind-canonix get pods` — the cluster is unreachable (`connection refused` on 127.0.0.1:59249). This is acceptable for this issue: it is a data-correctness / UX-label bug with no runtime exception reported (the issue body states "Relevant log output … No response"). No runtime signal is required to corroborate a label misnaming; the evidence is structural (the literal string is visibly wrong in two files at the cited line numbers).

## Phase C — Recent commits

`git log` on `erpnext/stock/doctype/item_barcode/item_barcode.*`:

- `2a90fad710` — *feat(stock): Support more barcodes in an item by validate the barcode with the barcodenumber module (#33863)* (Feb 19 2023). **This is the introducing commit.** The diff expanded the options string from `"\nEAN\nUPC-A"` to `"\nEAN\nUPC-A\nCODE-39\nEAN-12\nEAN-8\nGS1\nGTIN\nISBN\nISBN-10\nISBN-13\nISSN\nJAN\nPZN\nUPC"` — adding `EAN-12` alongside `EAN-8` (EAN-13 was apparently intended but mis-typed / confused with UPC-A's 12-digit length).
- Subsequent touches (`63313eef6f` auto-annotations, `dfde490c02` sort order) only regenerated/shuffled the file; none corrected the misnomer.

No recent commits on either file in the past 6 months.

## Phase D — Similar past issues / PRs

`gh issue list --repo ShankHarinath/erpnext --search "EAN barcode"` returns only this issue (#23). The fork has no prior EAN issues. Upstream searches are blocked by the hardened `gh` shim per the dispatch note, so no upstream PR numbers are referenced in this report.

## Phase E — Root-cause hypothesis

The option label `EAN-12` was introduced in commit `2a90fad710` (PR #33863) when the barcode-type list was expanded from 2 to 14 entries. EAN-12 does not exist in the EAN standard — 12-digit codes are UPC-A, which is already a separate option in the same list; the 13-digit code is EAN-13 (missing). The label should be `EAN-13`. Because (a) the options string is loaded dynamically from DocType meta, (b) `convert_erpnext_to_barcodenumber` does not special-case either `EAN-12` or `EAN-13` by literal, and (c) no test uses the literal, the fix is a pure naming change with negligible behavioral impact. Existing stored records (if any) with `barcode_type = "EAN-12"` would become invalid against the new Select options — this is the only migration consideration.

**Confidence:** 0.95 — the bug is a visible string-literal error at the exact cited lines, structurally clean, with a narrow blast radius and an obvious correct value.

## Recommended fix

1. `erpnext/stock/doctype/item_barcode/item_barcode.json` line 28: replace `EAN-12` with `EAN-13` inside the `options` string.
2. `erpnext/stock/doctype/item_barcode/item_barcode.py` line 23: replace `"EAN-12"` with `"EAN-13"` inside the auto-generated `barcode_type` `Literal[...]`.
3. Optional data migration: add a Frappe patch under `erpnext/patches/v15_0/` (or current patch dir) that runs `UPDATE \`tabItem Barcode\` SET barcode_type = 'EAN-13' WHERE barcode_type = 'EAN-12'` so any existing records adopt the corrected label and continue to render/select correctly. Low risk — the rename does not collide with any other valid option.
4. Regenerate `.po` files in a follow-up localization pass (or let the next translation refresh do it); no manual editing of 33 locale files is required for functional correctness.

## Open questions / caveats

- kind-canonix cluster was offline during triage (connection refused). Not blocking for a string-literal bug, but runtime corroboration is absent.
- GitNexus index maps to repo name `erpnext` rather than `ShankHarinath/erpnext`; queries succeeded under that alias.
- Whether upstream has already corrected this is out of scope for this triage run (per dispatch counterfactual note).

---

<!--phoenix:scout-summary
affected_repos:
  - ShankHarinath/erpnext
confidence: 0.95
risk: LOW
recommended_next_step: proceed
introducing_commit: 2a90fad7106646229fcecfe3844082581bd56d1e
fix_files:
  - erpnext/stock/doctype/item_barcode/item_barcode.json
  - erpnext/stock/doctype/item_barcode/item_barcode.py
fix_optional_files:
  - erpnext/patches/v15_0/rename_ean12_to_ean13.py
blast_radius: LOW
direct_importers: 1
processes_affected: 0
runtime_evidence: unavailable (kind-canonix cluster offline; not required for label-correctness bug)
leak_signals: none
-->
