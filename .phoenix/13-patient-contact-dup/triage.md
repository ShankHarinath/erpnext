# Triage Report — Issue #13

**Repo:** ShankHarinath/erpnext
**Issue:** #13 — Duplicated Contact Name on patient creation using already existing customer
**URL:** https://github.com/ShankHarinath/erpnext/issues/13
**Reporter:** ShankHarinath
**Rev:** 1
**Triaged at:** 2026-04-21

## Summary

When a user creates a `Patient` and links it to a pre-existing `Customer` that already has a primary `Contact` auto-created from Customer creation, saving the Patient raises `DuplicateEntryError` on the `Contact` DocType ("Duplicated Contact Name"). The Healthcare Patient flow unconditionally inserts a brand-new `Contact` whose autoname (`first_name` + " " + `last_name`) collides with the Contact that already exists for the linked Customer. The reporter's own workaround (change the Contact name) confirms the autoname collision.

## Root cause

**File:** `/Users/shashank/Canonix/phoenix-test/erpnext/erpnext/healthcare/doctype/patient/patient.py`
**Function:** `Patient.set_contact` (lines 161–182)

The branch logic only asks *"Is there already a Dynamic Link from a Contact to this Patient?"* — if not, it inserts a new Contact. It never inspects the Contact(s) already linked to `self.customer`. When the linked Customer was previously created with a `mobile_no`/`email_id`, `Customer.create_primary_contact` (`erpnext/selling/doctype/customer/customer.py:152`) already created a Contact named after the same person; Patient's brand-new `Contact` insert then hits a name collision.

Relevant excerpt (patient.py:161–182):

```python
def set_contact(self):
    if frappe.db.exists('Dynamic Link', {'parenttype':'Contact', 'link_doctype':'Patient', 'link_name':self.name}):
        ...
    else:
        self.reload()
        if self.email or self.mobile or self.phone:
            contact = frappe.get_doc({
                'doctype': 'Contact',
                'first_name': self.first_name,
                ...
            })
            contact.append('links', dict(link_doctype='Patient', link_name=self.name))
            if self.customer:
                contact.append('links', dict(link_doctype='Customer', link_name=self.customer))
            contact.insert(ignore_permissions=True)   # <-- DuplicateEntryError here
            self.update_contact(contact)
```

The missing guard is: *"if `self.customer` is set and a Contact already exists linked to that Customer, append a Patient link to that existing Contact instead of inserting a new one."*

## Evidence

### A. Code mapping (GitNexus + static read)

- `Patient.set_contact` is called only from `Patient.on_update` (impact analysis: direct callers = 1, risk LOW, module "Patient"). Blast radius for a fix is contained to the Patient DocType.
- `Customer.create_primary_contact` (customer.py:152) auto-creates a Contact when the Customer has `mobile_no` or `email_id`; `make_contact` is invoked, which creates a Contact under a name derived from the Customer's name fields. Since the reporter fills the Patient form with the same person's name as the existing Customer, the two Contacts collide.
- Frappe's `Contact` DocType uses `first_name` + " " + `last_name` as its default autoname and the framework enforces uniqueness — hence the "Duplicated Contact Name" message.

### B. Runtime logs (kubectl)

BLOCKER: `kubectl --context kind-canonix get pods` failed — the kind cluster API server is unreachable (`connection refused on 127.0.0.1:59249`). Runtime corroboration against live logs was not possible. The code path is simple and deterministic; the static reasoning is sufficient to pinpoint the fault.

### C. Recent commits on `patient.py`

- `ee9b6d158a` — *feat(Healthcare): Capacity for Service Unit … Patient enhancements (#24860)* (Aug 19 2021) — **introduced `set_contact` verbatim** in its current buggy form. This is the commit that introduced the bug.
- `28cdff10cf` — *fix: update linked Customer on Patient update only if Link Customer to Patient is enabled (#25926)* (Jun 12 2021) — restructured `on_update` but did not touch the duplicate-Contact flaw.
- `4b2be2999f` — chore: Cleanup imports (#27320) — unrelated.

The bug has existed unchanged in the repo since `ee9b6d158a`, which matches the 13.10.0 version the reporter is running.

### D. Similar past issues / PRs

BLOCKER: `gh` search against `frappe/erpnext` upstream is denied by the counterfactual-triage allowlist (`gh-shim: DENIED — arg 'frappe/erpnext' references owner 'frappe'`), so I could not confirm whether an upstream fix PR exists. Within the `ShankHarinath/erpnext` fork, no similar issue/PR surfaced in the bug's natural keyword search.

### E. Synthesis

The three independent lines of evidence agree:
- **Static code** shows `set_contact` builds a new Contact without consulting the linked Customer's existing Contacts.
- **Git history** places the flawed block in commit `ee9b6d158a`, which is present at the reporter's version 13.10.0.
- **Reporter's own workaround** ("I changed the name of the Contact and saved and it worked") isolates the failure to Contact-name uniqueness, exactly what the missing guard would have prevented.

## Recommended fix direction

Inside `Patient.set_contact`, before constructing the `new Contact`, look up Contacts already linked to `self.customer` (via `Dynamic Link` with `parenttype='Contact'`, `link_doctype='Customer'`, `link_name=self.customer`). If one exists:
- append `('Patient', self.name)` to its `links` table and save it, then call `update_contact` on that existing Contact;
- skip the `frappe.get_doc({...}).insert(...)` entirely.

Only if no Customer-linked Contact exists should the function fall through to creating a new Contact. This aligns with how `Customer.create_lead_address_contact` already re-uses an existing Lead-linked Contact rather than duplicating it (customer.py:197–207) — same pattern, Customer→Patient instead of Lead→Customer.

## Confidence

**0.88** — static evidence + git archaeology converge unambiguously on a single call site; impact analysis confirms a low-blast-radius fix; the reporter's workaround is exactly the signature of an autoname collision. Points withheld for (a) runtime verification via cluster logs was blocked, and (b) upstream fix-pattern lookup against `frappe/erpnext` was blocked.

## Open questions / blockers

- **kubectl cluster unreachable** (`127.0.0.1:59249` refused). No runtime log corroboration; the skill may want to attempt log capture in a follow-up once the cluster is back up. For this bug the code path is deterministic, so this does not change the recommended action.
- **gh upstream search denied** for `frappe/erpnext`. If an upstream fix already exists, porting it would be preferable to a clean-room implementation. Fix author should cross-check `frappe/erpnext` PR history manually for an existing patch on `patient.py::set_contact`.
- Should the Patient flow also handle the symmetric case (`link_customer_to_patient` is ON, no customer yet, `create_customer` is called in `on_update`) to avoid creating a Contact for the new Customer and then again for the Patient? Worth confirming whether `Customer.create_primary_contact` fires in that sub-flow given how `create_customer` inserts with `ignore_mandatory=True` and without mobile/email — likely no collision there, but author should verify.

## Recommended next step

`architect` — the fault, its location, and the corrective pattern are concrete; Architect can draft the guard (look up Customer-linked Contact, reuse or create-new branch) and an accompanying regression test in `test_patient.py` without additional reconnaissance.

---

```yaml
# --- triage-machine-readable ---
affected_repos:
  - ShankHarinath/erpnext
confidence: 0.88
recommended_next_step: architect
primary_files:
  - erpnext/healthcare/doctype/patient/patient.py
primary_symbols:
  - Patient.set_contact
introducing_commit: ee9b6d158aa441388925219b4143aa1bc79d0e71
blockers:
  - kubectl cluster kind-canonix unreachable (api server refused)
  - gh search against frappe/erpnext upstream denied by allowlist
# --- end ---
```
