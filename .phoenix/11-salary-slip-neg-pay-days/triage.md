## Triage Report — ShankHarinath/erpnext#11

**Classification:** bug  (orchestrator confidence: 0.95)
**Summary:** For Employees with status "Left", Salary Slip shows negative `payment_days` and an empty Earning & Deduction table because `get_payment_days` can subtract holiday count without a floor and because `check_sal_struct` / `validate_dates` allow the slip to proceed even when the employee's `relieving_date` is before the slip period.

### Symptom
- Normalized: Creating a Salary Slip for an employee whose status is "Left" produces a negative `payment_days` value and does not populate the earnings/deductions child tables.
- Error signatures: none (silent arithmetic + empty child table, no traceback shared by reporter)
- Reported timeframe: ERPNext v13.11.1 (reporter); reproducible on current HEAD (`385830bb51`) — the code paths responsible are unchanged since 2020.

### Runtime signal (from kubectl)
- First-seen: n/a — cluster unreachable
- Frequency: n/a
- Active now: unknown
- Pod/container state: unknown — `kubectl --context kind-canonix get pods` returns `connection refused` on 127.0.0.1:59249 (kind cluster not running)
- Target(s): none — Phase B blocked
- Affected entities: none extractable
- Correlated errors: none — Phase B blocked

### Affected code (from GitNexus)
- **Repos searched:** `erpnext` (rationale: only this repo is referenced by the issue; ShankHarinath/erpnext maps to indexed repo `erpnext`; no group membership; single-service bug).
- **Affected repos (fix required in):** `erpnext`
- `erpnext` · `SalarySlip.get_payment_days` · `erpnext/payroll/doctype/salary_slip/salary_slip.py:333-359` — prime suspect. Returns `date_diff(end_date, start_date) + 1 − len(holidays)`; no clamp to ≥ 0. When `relieving_date` falls inside the period and the shortened `[start_date, relieving_date]` window contains more holidays than working days (common for employees relieved early in the month over a weekend), `payment_days` goes negative.
- `erpnext` · `SalarySlip.get_working_days_details` · `erpnext/payroll/doctype/salary_slip/salary_slip.py:266-321` — propagates the negative. Line 304 `if flt(payment_days) > flt(lwp):` uses a negative `payment_days` against `lwp` computed over the **full** period (line 290 runs `calculate_lwp_or_ppl_based_on_leave_application(holidays, working_days)` with the full-period `working_days`), amplifying the mismatch; the else branch (line 321) zeroes it, but the signed value from the `if` branch is assigned when `payment_days > lwp` is satisfied.
- `erpnext` · `SalarySlip.validate_dates` · `erpnext/payroll/doctype/salary_slip/salary_slip.py:143-158` — only blocks slip creation when `relieving_date < start_date`. A Left employee with relieving_date inside the slip period passes through and hits the bug below. `validate_active_employee` (`erpnext/hr/utils.py:510-513`) blocks only `Inactive`, never `Left`, so there is no upstream guard.
- `erpnext` · `SalarySlip.check_sal_struct` · `erpnext/payroll/doctype/salary_slip/salary_slip.py:227-250` — secondary suspect for the **empty Earning & Deduction** symptom. The SQL at lines 233-241 filters Salary Structure Assignments only by `from_date`, with no upper bound against `relieving_date`; however, if the Left employee's Salary Structure Assignment was **ended/terminated** (from_date cleared or SA cancelled), `st_name` is empty, `self.salary_structure = None`, and `pull_sal_struct` is never invoked (`get_emp_and_working_day_details` line 207 gates on `if struct`). That leaves the earnings/deductions tables empty but the slip proceeds to save.
- `erpnext` · `SalarySlip.get_amount_based_on_payment_days` · `erpnext/payroll/doctype/salary_slip/salary_slip.py:951-976` — prorates with `self.payment_days / self.total_working_days`. If `self.payment_days` is negative, earnings/deductions that did get pulled become negative, compounding the report.
- Blast radius (upstream from `get_payment_days`, per `impact`): 1 direct caller (`get_working_days_details`); 4 transitive (`validate`, `get_emp_and_working_day_details`, `process_salary_structure`, `process_salary_based_on_working_days`); 1 module (`Salary_slip`); risk LOW; 0 tracked processes affected.
- Cross-repo dependencies: none.
- Process participation: `get_payment_days` itself does not appear in a tracked GitNexus process; `execute → Format` (`proc_224_execute`) in `salary_register.py` reads the values indirectly when rendering the Salary Register report — a negative `payment_days` in DB will corrupt that report too.
- Misses: "Earning & Deduction" as a textual signal matched nothing literal (that is the UI label for the `earnings` / `deductions` child tables on the Salary Slip doctype — interpretation: data path, not a missing module).

### Historical context
**Similar past issues / PRs:**
- None reachable. `gh pr list --repo ShankHarinath/erpnext --search "salary slip payment days"` returns empty; upstream `frappe/erpnext` is blocked by the `gh` shim (per constraints); local git log reveals no prior fix specifically addressing negative `payment_days` for Left employees.

**Recent commits on affected files (local history, newest first):**
- `d011a3f82c` — `fix(Salary Slip): TypeError while clearing any amount field in components (#29931)` — unrelated component-amount fix.
- `5f03292aba` — `fix: Future recurring period calculation for addl salary (#29578)` — recurring additional salary; unrelated to payment_days sign.
- `bab644a249` — `fix(Payroll): incorrect component amount calculation if dependent on another payment days based component (#27349)` — touches `get_amount_based_on_payment_days`; adjacent surface, but does not clamp `payment_days`.
- `74818c7b62` / `5657fddb7a` — `fix: improve filter for from_date; validation for joining and relieving date` (2021-05-29, Sagar Vora). Introduced the current `validate_dates` that throws when `relieving_date < start_date`. Flag: this was the last time joining/relieving logic was hardened; it closed the "relieving before period" hole but did **not** clamp the holiday-subtracted `payment_days` for the "relieving within period" case.
- `ba70e7e8bce` (2020-04-26, Nabin Hait) — introduced the current `get_payment_days` signature and the internal fallback to `frappe.get_cached_value`; the holiday subtraction without floor has existed since this commit.
- `a236f4e5867` (2017-04-13) — original `payment_days = date_diff(...) + 1` line; the `payment_days -= len(holidays)` subtraction has shipped unclamped since then.

### Root-cause hypothesis
For a Left employee whose `relieving_date` falls inside the slip's `[start_date, end_date]` window, `SalarySlip.get_payment_days` (`salary_slip.py:333-359`) shortens `end_date` to `relieving_date`, computes `payment_days = date_diff(end_date, start_date) + 1`, and then unconditionally subtracts `len(holidays)` for the shortened window (line 357). There is no floor at zero, so when the shortened window is dominated by holidays (e.g. relieved on a Monday after a Saturday/Sunday start, with an additional public holiday), the return value becomes negative. `get_working_days_details` (lines 301-321) then compares that negative value with `lwp` and, when the `if` branch is taken, writes the signed value straight to `self.payment_days`. Separately, the empty Earning & Deduction table is driven by `check_sal_struct` returning `None` for Left employees whose Salary Structure Assignment no longer satisfies the `from_date <= start_date` / `<= end_date` / `<= joining_date` filter (`salary_slip.py:227-250`); because `get_emp_and_working_day_details` only calls `pull_sal_struct` when `check_sal_struct` returns a struct (line 207), the child tables stay empty while the slip is still allowed to persist. Together the two gaps produce the reported symptom. The fix should (a) clamp `payment_days` to `max(0, …)` in `get_payment_days` before returning, and (b) either refuse to create / short-circuit to zero totals when `check_sal_struct` finds no active structure for a Left employee, or broaden the filter to permit the last-active SA to be picked up when `relieving_date` is within the slip period.

**Confidence:** 0.72

### Evidence chain
1. Issue says "negative payment days" + "Earning & Deduction table is not fetched" for Left employees → two distinct failures in the same flow, both in `SalarySlip.validate`.
2. Code read of `salary_slip.py:353-357` shows `payment_days` is set by `date_diff + 1 − len(holidays)` with no `max(0, …)` clamp → arithmetic path to a negative number exists.
3. Code read of `salary_slip.py:207` shows `pull_sal_struct` is gated on `check_sal_struct` returning a struct → if the Left employee's SA filter fails, earnings/deductions remain empty; slip still proceeds because `check_sal_struct` only `msgprint`s, it does not `throw`.
4. `validate_active_employee` (`hr/utils.py:510-513`) only blocks `Inactive`, never `Left` → Left employees are an un-guarded code path into the buggy math.
5. `validate_dates` (lines 143-158) blocks only when `relieving_date < start_date`; it does not guard the in-period relieving case that causes the bug.
6. Git blame: the unclamped subtraction has shipped since `a236f4e5867` (2017) and the `get_payment_days` shape since `ba70e7e8bce` (2020); the last touch to Left-employee validation is `74818c7b62` (2021) which fixed an adjacent hole but not this one → regression is long-standing, reporter's v13.11.1 matches.
7. `gitnexus impact` on `get_payment_days`: only 1 direct caller and 4 transitive; risk LOW → fix is localized, safe to apply with clamp + filter widening.
8. Phase B (kubectl) unreachable — cannot corroborate with runtime logs; hypothesis stands on static analysis and reported symptoms alone, hence confidence is held at 0.72 rather than ≥ 0.85.

### Recommended next step
proceed to fix

### Open questions
- Blocker: `kubectl --context kind-canonix` cluster is not running (`connection refused on 127.0.0.1:59249`); no runtime reproduction in logs — Architect / Implementer should spin up the cluster or add a unit test exercising `get_payment_days` with a relieving_date inside the period and a holiday-heavy shortened window.
- Design choice for Architect: should a Salary Slip for a Left employee with no active Salary Structure Assignment (a) hard-throw in `check_sal_struct`, (b) soft-zero all amounts, or (c) broaden the SA filter to allow the last-active assignment up to `relieving_date`? The reporter's expected behavior is unclear.
- `gh` shim blocks upstream `frappe/erpnext` issue/PR search, so we could not confirm whether this has been reported/fixed upstream on newer versions.
- Upstream issue breadcrumb was intentionally skipped (config disabled `disable_issue_comments`; no `## Issue comment` block in dispatch payload).

<!-- MACHINE-READABLE FOOTER — DO NOT REMOVE; downstream skills parse this block -->
<!--phoenix:scout-summary
affected_repos: [erpnext]
confidence: 0.72
recommended_next_step: proceed
-->
