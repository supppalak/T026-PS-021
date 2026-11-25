# Design Document

## Home Loan with Construction-Linked Disbursement Tranches and Pre-EMI

| Item | Detail |
|---|---|
| Project | PS-021 |
| Team | T026 |
| Section | CSDS-A |
| Implementation | Python 3.10+ |
| Interface | Command-file driven; no UI |
| Document version | 1.0 |
| Current status | Runnable prototype; final hardening and tests pending |

### Team

| Member | Roll number | Email |
|---|---|---|
| Nethraj T M | 1AT24CD052 | nethrajt@gmail.com |
| Niranjan | 1AT24CD053 | ushaniranjan03@gmail.com |
| Palak Raj | 1AT24CD054 | palaktina1@gmail.com |

### Document map

1. Executive summary and problem context
2. Goals, actors, and functional requirements
3. Architecture and domain model
4. Lifecycle and all business rules BR-01 to BR-26
5. Core algorithms and explicit edge cases
6. Inputs, outputs, validation, and reporting
7. Determinism, tests, status, gaps, and traceability
8. Viva notes, submission checklist, and glossary

---

## 1. Executive Summary

T026-PS-021 is a deterministic mortgage-servicing engine for loans on under-construction homes. The engine performs origination and sanction, controls construction-linked disbursements, bills pre-EMI interest on released principal only, converts the account to full EMI, processes rate resets and prepayments, and produces servicing reports.

The system is designed for command-file execution. It reads master data and an ordered event stream, processes that stream as a state machine, and produces audit-ready results. Financial arithmetic uses `Decimal` with `ROUND_HALF_UP`; date arithmetic uses calendar months with end-of-month clamping. Every material decision carries a business-rule identifier so that a result can be traced back to its cause.

The repository contains two execution paths:

1. `run_sample.py` is the currently runnable benchmark path for the official CSV sample.
2. `main.py` and `src/` form the modular JSON event engine. Their required JSON fixtures are not currently present in the repository.

The official sample currently runs successfully and reproduces the five checked-in CSV outputs. A consolidation to one execution path, a complete automated test suite, and several financial-state corrections remain before final submission.

---

## 2. Problem Context

A normal loan is usually represented by one disbursement, followed by EMI repayment. An under-construction home loan is different:

- the builder requests money in stages;
- a valuer certifies construction progress;
- the borrower pays part of the property cost;
- the bank releases principal in tranches;
- during construction the borrower pays interest without principal repayment;
- full EMI begins only after the final tranche or the moratorium cap;
- floating rates and prepayments change the remaining schedule.

The servicing engine must preserve the financial meaning of each event and calculate every bill and balance to the rupee. A command cannot be treated as an isolated update. It must be evaluated against the loan's current state, previously released funds, certification history, sanction cap, delinquency status, and outstanding schedule.

---

## 3. Goals and Non-Goals

### 3.1 Goals

- Register borrowers, co-applicants, properties, and stage plans.
- Sanction a loan using property value, income, tenure, credit score, and benchmark rate.
- enforce construction-linked disbursement controls and business-rule precedence.
- Accrue and bill pre-EMI interest per released tranche using actual days and a 365-day year.
- Convert to a full amortising EMI schedule on the required trigger.
- Process benchmark resets, EMIs, part prepayments, full prepayments, and delinquency.
- Produce sanction letters, a disbursement ledger, amortisation schedules, interest certificates, and audit logs.
- Make results deterministic and reproducible from identical inputs.
- Keep the implementation standard-library only and runnable on Python 3.10 or newer.

### 3.2 Non-Goals

- A user interface, web service, database, or authentication layer.
- Real-time integration with banks, valuers, builders, or payment networks.
- Credit underwriting beyond the supplied score and income inputs.
- Tax filing, e-signature, KYC, AML, or regulatory reporting integrations.
- Balance transfer and top-up, which are optional stretch goals.
- Ownership of legal dates such as bank holidays beyond the date and certification rules.

---

## 4. Actors and Inputs

| Actor or source | Responsibility |
|---|---|
| Borrower | Supplies income, co-applicant, age, and prepayment or repayment events. |
| Lender | Sanctions the loan, sets pricing, holds the sanction cap, and produces statements. |
| Builder | Raises stage demands. |
| Valuer | Certifies construction completion percentages. |
| Underwriting rules | Determine eligibility, LTV, tenure, rate, and fee. |
| Command file | Defines the ordered business event stream. |
| Reference data | Supplies benchmark history, property, borrower, and stage records. |
| Servicing job | Runs periodic billing, conversion, and delinquency checks. |

### 4.1 Accepted command types

| Command | Meaning |
|---|---|
| `SANCTION_REQUEST` or `SANCTION` | Run eligibility and create a loan account. |
| `OWN_CONTRIBUTION` | Record the borrower's own contribution. |
| `STAGE_CERTIFIED` | Record valuer certification of cumulative construction completion. |
| `DEMAND` | Record a builder request against a stage. |
| `RELEASE` | Apply disbursement checks and release bank funds. |
| `RUN_TO` or `MONTHLY_JOB` | Advance periodic servicing through a date. |
| `MISSED` | Mark a due bill as unpaid. |
| `PAYMENT` or `RECORD_EMI` | Record repayment against dues. |
| `PREPAY` or `PREPAYMENT_PART` | Apply a part prepayment. |
| `PREPAYMENT_FULL` | Close the loan by full prepayment. |
| `BENCHMARK_CHANGE` | Schedule a floating-rate reset. |
| `PRINT_SCHEDULE` | Emit amortisation data as of a date. |
| `PRINT_INTEREST_CERT` | Emit an FY interest certificate. |

The official CSV command format uses pipe-delimited records. The modular engine uses JSON events. Both feed the same domain concept: an ordered event stream.

---

## 5. Functional Requirements

| ID | Requirement |
|---|---|
| FR-01 | Load borrowers, properties, stage plans, benchmark history, holidays, and commands. |
| FR-02 | Validate identifiers, required fields, dates, numeric ranges, stage percentages, and event order. |
| FR-03 | Register a borrower with gender, DOB, income, credit score, and optional co-applicant. |
| FR-04 | Register a property with agreement value, builder, and stage plan. |
| FR-05 | Calculate the LTV limit from agreement value. |
| FR-06 | Calculate combined eligible income and the maximum eligible EMI. |
| FR-07 | Derive the age-capped maximum tenure. |
| FR-08 | Calculate the interest rate from benchmark, credit spread, and women concession. |
| FR-09 | Calculate the sanction as the minimum of requested amount, LTV limit, and eligibility limit. |
| FR-10 | Calculate the processing fee, GST, and two-part payment schedule. |
| FR-11 | Enforce six-month sanction validity. |
| FR-12 | Process own contributions before the first bank tranche. |
| FR-13 | Validate stage certification coverage and freshness. |
| FR-14 | Release the demand subject to own-contribution absorption and the remaining sanction cap. |
| FR-15 | Enforce the defined tranche-rule precedence and record the failing rule. |
| FR-16 | Accrue pre-EMI interest daily per tranche and bill on the fifth for the previous month. |
| FR-17 | Stop pre-EMI on conversion and start full EMI from the required month. |
| FR-18 | Build an exact amortisation schedule with interest and principal splits. |
| FR-19 | Apply benchmark resets from the first of the following month. |
| FR-20 | Record overdue EMIs and penal interest after the grace period. |
| FR-21 | Freeze further disbursement after three consecutive missed EMIs. |
| FR-22 | Process part prepayments with tenure or EMI reduction. |
| FR-23 | Process full prepayment and close the loan. |
| FR-24 | Lapse undisbursed sanction at month 36. |
| FR-25 | Produce the required ledgers, schedules, certificates, and audit records. |
| FR-26 | Produce deterministic outputs for identical inputs. |

---

## 6. Architecture

### 6.1 Target architecture

```text
Command files + master data
            |
            v
       Input parsers
            |
            v
    Validation and registry
            |
            v
  LoanEngine state machine  <------ benchmark / holiday data
     |      |       |
     |      |       +--> eligibility, fees, sanction
     |      +----------> disbursement and chronology rules
     +-----------------> pre-EMI, EMI, reset, prepayment, delinquency
            |
            v
   Domain records and event log
            |
            v
   Report writers and output files
```

### 6.2 Module responsibilities

| Module | Responsibility |
|---|---|
| `src/models/` | Domain data classes: borrower, co-applicant, property, stage, loan, tranche, pre-EMI bill, EMI row, and event log. |
| `src/rules/` | Business-rule constants and shared formulas: LTV, income ratio, rates, PMT, fees, and reset maths. |
| `src/calculations/` | Pure calculation and decision functions for sanction, disbursement, pre-EMI, EMI, and fees. |
| `src/engine/loan_engine.py` | Stateful lifecycle processor for one loan. |
| `src/parsers/` | JSON parsers and input coercion. |
| `src/output/` | Sanction letters, ledgers, schedules, certificates, summaries, and event logs. |
| `main.py` | JSON CLI orchestration and multi-loan engine registry. |
| `run_sample.py` | Official CSV benchmark implementation and aggregate CSV writer. |

### 6.3 Architecture decision

The modular engine separates calculation from state transition so formulas remain testable without constructing complete loan histories. The official path demonstrates end-to-end behavior against the supplied benchmark, while the modular path provides domain boundaries for final development.

The two paths currently duplicate concepts and formulas. The final implementation should converge on one executable path while retaining the modular domain boundaries. Until that consolidation is complete, changes to shared business rules must be checked in both paths.

---

## 7. Domain Model

### 7.1 Core entities

| Entity | Key attributes |
|---|---|
| `Borrower` | ID, name, DOB, gender, monthly net income, credit score. |
| `CoApplicant` | Name, monthly income, relationship (`SPOUSE` or other). |
| `Property` | ID, agreement value, builder, stage plan. |
| `StageItem` | Stage name and cumulative or incremental percentage. |
| `SanctionResult` | Principal, tenure, rate, LTV, eligibility, fee, validity, rules. |
| `LoanAccount` | Sanction fields and live servicing records. |
| `Tranche` | Demand, certification, own contribution, release, decision, and rule IDs. |
| `PreEmiBill` | Billing period, total interest, and per-tranche breakdown. |
| `EmiRow` | Due date, rate, amount, interest, principal, balance, payment state. |
| `EventLog` | Timestamped event outcome, reason, rule IDs, and details. |

### 7.2 Invariants

- A loan belongs to exactly one borrower and one property in the current model.
- The agreement value excludes stamp duty and registration.
- Stage percentages are cumulative for the supplied stage-plan format and must end at 100%.
- Initial available own contribution equals agreement value less the requested or sanctioned bank loan, subject to the final interpretation of partial contribution events.
- Sum of released bank principal never exceeds the sanctioned bank loan.
- A tranche release cannot occur after the later-date-plus-seven-day SLA unless explicitly flagged as a breach; the core release decision is governed by precedently validated events.
- Pre-EMI accrual is non-negative and applies only from each tranche's release date.
- Full EMI principal is total net disbursed principal, not sanctioned principal.
- Closed loans accept no new disbursement or prepayment.
- Monetary values are represented as `Decimal`, not binary floating point.
- No output depends on hash iteration where order affects financial results.

---

## 8. Lifecycle State Machine

```text
SANCTIONED
   |
   | own contribution + stage certification + release
   v
PARTIALLY_DISBURSED / PRE_EMI
   |
   | final tranche or month-36 lapse
   v
ACTIVE_EMI
   |                     |
   | full prepayment     | all EMIs paid
   v                     v
CLOSED                 CLOSED / MATURED

SANCTIONED -- no disbursement at month 36 --> LAPSED
```

### 8.1 Event order

Events are processed in file order. An event that must not be backdated is rejected when its date predates an already applied servicing event. Monthly jobs fill missing monthly periods up to the job date. Report-print commands observe state as of the reported date.

---

## 9. Business Rules

The following rules are registered independently so that calculations, outputs, and audit logs can refer to stable identifiers.

### BR-01 - Loan-to-value limit

Let `P` be agreement value.

| Agreement value | LTV |
|---|---:|
| `P <= Rs 30,00,000` | 90% |
| `Rs 30,00,000 < P <= Rs 75,00,000` | 80% |
| `P > Rs 75,00,000` | 75% |

`LTV_limit = P x ratio`.

Boundary values exactly Rs 30 lakh and Rs 75 lakh use the lower band. Stamp duty and registration are excluded from `P`.

### BR-02 - Maximum tenure

The loan term ends before the primary borrower turns 70:

`max_tenure = min(360 months, months until primary borrower's 70th birthday)`

The candidate tenure must be capped by this value. A sanction at or after the 70th birthday is ineligible unless a documented project convention permits a minimum one-month term.

### BR-03 - EMI-to-income limit

| Combined monthly net income | Maximum EMI ratio |
|---|---:|
| `<= Rs 1,00,000` | 55% |
| `> Rs 1,00,000` | 60% |

`max_EMI = combined_income x ratio`.

### BR-04 - Co-applicant weighting

`combined_income = borrower_income + coapplicant_income x weight`

| Relationship | Weight |
|---|---:|
| Spouse | 100% |
| Other | 50% |

### BR-05 - Credit spread

| Credit score | Spread over benchmark |
|---|---:|
| `>= 775` | 0.50% |
| `725-774` | 0.90% |
| `< 725` | 1.40% |

### BR-06 - Women borrower concession

If the primary applicant is female, subtract 0.05 percentage points from the annual interest rate.

### BR-07 - Sanction amount

`sanction = min(requested_amount, LTV_limit, eligibility_limit)`

The sanction cannot be negative.

### BR-08 - Processing fee

`gross_fee = clamp(0.5% x sanction, Rs 10,000, Rs 25,000)`

`GST = 18% x gross_fee`

`total_fee = gross_fee + GST`

The fee is split equally between sanction and first bank disbursement.

### BR-09 - Sanction validity

The sanction remains valid for six calendar months from sanction date. A release after validity is rejected unless the sanction was already availed under the agreed convention. A lapsed sanction forfeits the first half of the fee.

### BR-10 - Income eligibility via PMT

Monthly rate:

`r = annual_rate_pct / 1200`

Maximum principal supported by the maximum EMI:

`eligibility_limit = max_EMI x ((1 + r)^n - 1) / (r x (1 + r)^n)`

where `n` is the capped tenure in months. The result is rounded to the nearest rupee according to the project convention.

### BR-11 - Own contribution before first tranche

Own contribution must be fully paid before the first bank tranche. A demand may be partly or fully absorbed by the borrower's own-contribution balance according to the final contribution model. The first bank release must not occur while the required contribution remains unpaid.

### BR-12 - Stage certification coverage

Let `C` be cumulative certified stage percentage and `D` cumulative demanded stage percentage after the current demand. Release requires:

`C >= D`

Equality is accepted. A later demand requires new certification when it raises cumulative demanded percentage above the latest certified percentage.

### BR-13 - Sanction cap

`remaining_sanction = sanctioned_amount - cumulative_gross_bank_releases`

`bank_release = min(bank_portion_of_demand, max(0, remaining_sanction))`

Cumulative gross releases are used for the cap; prepayments reduce outstanding principal but do not restore sanctioned availability.

### BR-14 - Delinquency freeze

Three consecutive missed EMIs freeze further tranche releases until the arrears are cleared under the servicing convention. The event log must identify BR-14 as the rejection reason.

### BR-15 - Release timing

The target time is no later than seven calendar days after the later of:

`max(demand_date, certification_date)`

A late release is represented as an SLA breach rather than silently treated as on time. Whether a release after the SLA is rejected or merely flagged is a documented decision; the supplied expected output should decide it.

### BR-16 - Certification freshness

Certification is valid for 90 calendar days. Exactly 90 days old on the release date is accepted. More than 90 days requires fresh certification.

### BR-17 - Pre-EMI interest

For each tranche `i`:

`interest_i = principal_i x annual_rate_pct / 100 / 365 x active_days_i`

`active_days_i` is the overlap between the billing period and the tranche's post-release life. The monthly bill sums all active tranches. Interest is reported separately from principal repayment.

### BR-18 - Moratorium

Pre-EMI remains active until the earlier of final disbursement and the 36-month moratorium cap. Prepayments during this period reduce disbursed principal and future pre-EMI accrual.

### BR-19 - Full EMI start

Full EMI starts from the first of the month after final disbursement or the 37th month, whichever applies. The schedule must not collect a full EMI before conversion.

### BR-20 - EMI principal base

The full EMI is calculated on total disbursed principal, less applicable pre-EMI prepayments, for the remaining tenure. Sanctioned but undisbursed principal is excluded.

### BR-21 - Benchmark and rate reset

A benchmark change applies from the first day of the following month. First attempt to hold EMI constant and solve for new tenure:

`n_new = ceil(-ln(1 - r_new x P_outstanding / EMI) / ln(1 + r_new))`

If the new tenure would extend past the primary applicant's 70th birthday, cap the tenure and recompute EMI:

`EMI_new = P x r_new x (1 + r_new)^n / ((1 + r_new)^n - 1)`

### BR-22 - Part prepayment

- Minimum amount: Rs 50,000.
- No fee.
- Default is tenure reduction with EMI unchanged.
- On request, tenure is held and EMI is reduced.
- During pre-EMI, reduce net disbursed principal and future accrual.
- After conversion, rebuild the remaining EMI schedule from the next unpaid due date.
- Preserve paid rows and audit history.

### BR-23 - Full prepayment

A full prepayment closes the loan without foreclosure charge. The payment must equal or exceed the valid outstanding principal plus any remains due under the agreed settlement convention. A short payment cannot close the loan. Closing must not fabricate future paid EMIs.

### BR-24 - Penal interest

After 15 calendar days of grace, overdue EMI principal attracts 2% per month, prorated according to the project's documented day or month convention. The charge must be shown separately for the overdue EMI.

### BR-25 - Month-36 lapse

At month 36, undisbursed sanction lapses. If principal has already been disbursed, convert to full EMI on the remaining disbursed principal. If no principal has been disbursed, mark the sanction lapsed with no EMI schedule.

### BR-26 - Tranche precedence

Checks execute in this order and stop at the first failure:

1. BR-09 - sanction validity
2. BR-11 - own contribution paid
3. BR-12 and BR-16 - certification coverage and freshness
4. BR-13 - sanction cap
5. BR-14 - delinquency freeze

The decision record contains one primary failing outcome and all relevant supporting rule identifiers.

---

## 10. Algorithms

### 10.1 Monetary and date arithmetic

- Monetary calculations use `Decimal` with 28 digits of precision.
- Output monetary values are rounded to two decimal places with `ROUND_HALF_UP`, except rupee-denominated eligibility caps where the established output convention applies.
- Calendar additions clamp day-of-month to the last valid day.
- Pre-EMI uses actual days and a fixed 365-day denominator.
- EMI monthly rate is annual percentage divided by 1200.
- All persisted values retain their meaning as principal, interest, fee, penalty, or GST.

### 10.2 Tranche release pseudocode

```text
on RELEASE demand:
    locate loan and demand
    apply any monthly jobs due before release
    if loan is closed or converted: reject
    if release date after sanction validity and no money released: reject BR-09
    if required own contribution remains unpaid before first bank tranche: reject BR-11
    if cumulative certified percentage < cumulative demanded percentage: reject BR-12
    if certification age > 90 days: reject BR-16
    if remaining sanction <= 0: reject BR-13
    if consecutive overdue EMIs >= 3: reject BR-14
    consume applicable own contribution
    bank_portion = demand - own_contribution_used
    bank_release = min(bank_portion, remaining sanction)
    record partial when bank_release < bank_portion
    update gross released principal and outstanding principal
    mark final stage or final cap when applicable
    persist rule IDs, reason, decision and SLA flag
```

### 10.3 Pre-EMI billing pseudocode

```text
on MONTHLY_JOB date:
    period = previous calendar month
    bill_date = fifth of current month
    for each released active tranche:
        from = max(period.start, tranche.release_date)
        days = period.end - from
        if days > 0:
            add principal x annual_rate / 100 / 365 x days
    round and persist the monthly bill
```

### 10.4 Full EMI generation

For each due month:

`interest = outstanding x r`, rounded to 2 decimal places

`principal = EMI - interest`

The final instalment is the outstanding principal plus its interest. The schedule stops when outstanding reaches zero.

### 10.5 Rate reset

The engine applies a benchmark effective from the next month's first day. During pre-EMI, only the accrual rate changes. During full EMI, the engine computes a new schedule using the outstanding principal and unpaid tenure. A constant-EMI solve is used first, subject to the age cap; otherwise EMI is recalculated at the capped remaining tenure.

---

## 11. Explicit Edge Cases

| Edge case | Required behavior |
|---|---|
| Agreement value exactly Rs 30 lakh | Use the 90% band. |
| Agreement value exactly Rs 75 lakh | Use the 80% band. |
| 20% certified, demand 20%, then demand 10% without new certification | First demand is eligible. Second demand fails BR-12. |
| Benchmark change on the first day of a month | The new rate is effective from the first of the next month. |
| Final tranche on the last day of month 36 | Treat the final tranche as conversion-triggering and prevent an extra pre-EMI bill. |
| Prepayment on the same day as a tranche release | Define ordering. Expected convention: release first, then prepayment; accrue interest only on the post-prepayment balance from the release date. |
| Rate reset pushes tenure beyond age 70 | Cap tenure and recalculate EMI. |
| Sanction expires on a holiday | Validity is date based. Holidays may mark processing or SLA, but do not extend the financial expiry unless the stated convention says so. |
| Certification exactly 90 days old on release date | Accept. Only ages greater than 90 are stale. |

---

## 12. Input Contracts

### 12.1 Official CSV sample

| File | Purpose |
|---|---|
| `borrowers.csv` | Borrower demographics, income, score, co-applicant. |
| `properties.csv` | Agreement value and stamp/registration reference. |
| `stage_plans.csv` | Construction stages and cumulative completion percentages. |
| `benchmark_history.csv` | Dated benchmark rates. |
| `holidays.csv` | Holiday reference dates. |
| `commands.txt` | Pipe-delimited ordered lifecycle events. |

### 12.2 Modular JSON sample

```json
{
  "borrowers": [
    {
      "borrower_id": "B001",
      "name": "Example Borrower",
      "dob": "1990-06-20",
      "gender": "F",
      "net_income_monthly": "150000.00",
      "credit_score": 780,
      "coapplicant": {
        "name": "Example Co-applicant",
        "income_monthly": "120000.00",
        "relationship": "SPOUSE"
      }
    }
  ]
}
```

The repository must supply the missing `data/sample` JSON files before the modular CLI can be demonstrated from a clean clone.

---

## 13. Output Contracts

| Output | Required content |
|---|---|
| Sanction letter | Borrower, property, LTV, eligibility, sanction, rate, tenure, fee, validity, rule IDs. |
| Disbursement ledger | Demand, certification, decision, own contribution, release amount, cumulative releases, rule IDs, reason, SLA. |
| Pre-EMI bill table | Bill date, period, per-tranche accrual and monthly total. |
| Amortisation schedule | Version, due date, rate, EMI, interest, principal, balance, status. |
| Interest certificate | FY principal repaid, EMI interest, pre-EMI interest, and total. |
| Event log | Ordered outcomes with rule IDs and machine-readable details. |
| Run summary | Input counts, sanction results, success/error counts, and generated artifacts. |

The official sample emits aggregate CSV files. The modular path emits per-loan text and JSON reports.

---

## 14. Validation Strategy

Validation belongs at input and state-transition trust boundaries.

### 14.1 Input validation

- Required files and fields exist.
- IDs are non-empty and unique where required.
- Dates are valid ISO dates and ordered where chronology requires it.
- Borrowers have supported gender values and non-negative income.
- Credit score is within the intended numeric domain.
- Property value is positive; stamp duty is excluded from LTV.
- Stage names exist in the property plan.
- Stage cumulative percentages are monotonic and end at 100%.
- Benchmark history is sorted by effective date.
- Amounts are valid `Decimal` values and non-negative unless debit semantics are supported.
- Unknown commands and unknown loan or demand IDs fail with an explicit result.

### 14.2 State validation

- No release exceeds the sanctioned principal.
- No bank release before required own contribution.
- No certification used beyond 90 days.
- No duplicate full prepayment or post-closure events.
- No payment of an unknown EMI.
- No full closure on a short payment.
- No full EMI before conversion.
- No future-dated charge included in an earlier certificate.
- No duplicate monthly bill for an already billed period.

### 14.3 Failure policy

Invalid master data is a fatal input error. Invalid sequence events produce a rejected audit record and do not corrupt existing financial state. Partial outputs must be written atomically or clearly marked incomplete.

---

## 15. Reporting and Financial-Year Rules

The Indian financial year runs from 1 April through 31 March.

- Pre-EMI interest is reported separately from principal repayment.
- EMI interest and principal are split from the applied schedule row.
- Penal interest is separately identifiable.
- Full prepayment does not become principal repayment of all future instalments.
- Certificates report cash and charges applicable in the selected FY.
- Bill date and bill period are retained so reporting can use the project's chosen accounting convention.

---

## 16. Determinism and Auditability

- Inputs and commands are processed in ascending source order.
- Rates, amounts, dates, and identifiers are preserved with explicit string or text formatting.
- Every state transition writes an event with a sequence number, date, type, outcome, reason, rule IDs, and details.
- Report files are overwritten only for the same loan and report type.
- Randomness is not used in financial calculations.
- Floating-point routine use is limited to logarithmic tenure solving where exact closed-form integer algebra is impractical; schedule generation and post-solve amounts return to `Decimal`.

---

## 17. Test Strategy

### 17.1 Required suites

| Suite | Coverage |
|---|---|
| Sanction unit tests | LTV boundaries, income weighting, credit spread, female concession, fee clamps, GST, age tenure. |
| Disbursement tests | Own contribution, coverage, freshness, cap, partial release, delinquency, precedence. |
| Pre-EMI tests | Per-tranche start dates, month ends, leap years, rate changes, prepayment effect. |
| EMI tests | Schedule exactness, final row, conversion, resets, age cap, prepayment modes. |
| Payment tests | Grace period, penalty, partial payment, three missed EMIs, payment clearing. |
| Reporting tests | FY boundaries, pre-EMI separation, principal/interest split, certificate totals. |
| Integration tests | Official sample, exact expected outputs, multi-loan command isolation. |
| Property-based tests | Monetary conservation and no-negative-balance invariants. |

### 17.2 Acceptance gate

A release candidate must satisfy all of the following:

- all supplied expected outputs match exactly to the rupee;
- all visible and hidden sample command sequences complete without unexplained rejections;
- no output contains an unhandled exception or missing rule ID;
- all explicit edge cases have automated tests;
- two independent runs from the same inputs are byte-identical;
- the team can modify and explain the code live in viva.

The current checkout does not yet contain the advertised automated test suite. This is a final-submission blocker.

---

## 18. Current Implementation Status

### 18.1 Verified

- `run_sample.py` executes against the official sample data.
- It generates 7 sanctions, 28 tranche decisions, 110 pre-EMI bills, 3,297 EMI rows, and 21 FY certificate rows.
- Its five generated CSV files match the checked-in reference outputs byte for byte.
- Python compilation succeeds for `main.py`, `run_sample.py`, and `src/`.

### 18.2 Implementation gaps requiring resolution

1. The clean clone lacks the documented `data/sample` JSON fixtures needed by `main.py`.
2. The clean clone lacks the documented test suite.
3. The two engine paths duplicate state, formulas, and outputs.
4. Rule IDs in `run_sample.py` diverge from the BR-01 to BR-26 registry for own contribution, stale certification, delinquency, and sanction cap.
5. Partial tranches are omitted from the cumulative cap in the modular path.
6. Payment commands do not fully validate and allocate the received amount.
7. Full prepayment can close a loan without validating the amount and can fabricate future paid EMIs.
8. Pre-EMI prepayment does not consistently reduce accrual principal in the modular path.
9. Rate-reset schedule generation can fail to honor the constant-EMI tenure solve.
10. Stage-plan validation, unknown-stage checks, date ordering, duplicate events, and command-level error reporting need hardening.

These gaps must be closed before the design can be treated as implemented.

---

## 19. Ambiguities and Decisions Needed

| Topic | Decision needed |
|---|---|
| Own contribution events | Are partial payments accepted before the first tranche, and does every demand consume the remaining contribution before bank funds? |
| Sanction expiry | Does expiry stop only an unavailed sanction, or every release after six months? |
| Holiday effect | Do holidays change validity, SLA, or release date, or are they reference-only? |
| Same-day order | Does tranche release precede prepayment on the same day for interest purposes? |
| Pre-EMI rate change | Which rate applies to each day within a rate-change month? |
| Penal interest | Is the 2% monthly rate applied to days/30 or complete calendar months? |
| EMI prepayment mode | Which cash-flow is preserved exactly when reducing EMI versus tenure? |
| Payment allocation | What order applies among penalty, interest, principal, and future dues? |
| Final prepayment | Is closure based on principal only or principal plus accrued interest and charges? |
| Lapsed sanction | Does a sanction with prior releases lapse only undisbursed amount, or invalidate later releases as in existing BR-26 precedence? |

---

## 20. Traceability Matrix

| Requirement | Rules | Main modules | Required evidence |
|---|---|---|---|
| Origination | BR-01 to BR-10 | `eligibility.py`, `fees.py`, `sanction.py` | Sanction letter, summary, tests. |
| Disbursement | BR-09, BR-11 to BR-16, BR-26 | `disbursement.py`, `loan_engine.py` | Tranche ledger with reason and rule IDs. |
| Pre-EMI | BR-17, BR-18 | `pre_emi.py` | Monthly bill with per-tranche breakdown. |
| Conversion | BR-19, BR-20, BR-25 | `loan_engine.py`, `emi_schedule.py` | Conversion event and amortisation schedule. |
| Rate reset | BR-21 | `emi_schedule.py`, `loan_engine.py` | Versioned schedule and reset event. |
| Prepayment | BR-22, BR-23 | `loan_engine.py`, `emi_schedule.py` | Versioned schedule and closure event. |
| Delinquency | BR-14, BR-24 | `loan_engine.py` | Overdue and penal records, freeze event. |
| Reporting | Feature outputs | `ledger_writer.py`, `report_writer.py` | Ledger, schedule, certificate, event log. |

---

## 21. Security, Privacy, and Operational Limits

- The engine processes local command and master files only.
- It does not accept network requests or expose credentials.
- Input files contain personal data; generated outputs should not be committed to public repositories without review.
- Logs include borrower and loan identifiers, so production deployment would require retention and access-control policies.
- A single-process CLI is appropriate for deterministic batch processing, not concurrent bank servicing.
- Atomic output replacement and idempotent monthly jobs are required before production use.

---

## 22. Viva Preparation

### 22.1 Pre-EMI accrual

Explain that each tranche begins its own accrual clock on its release date. For a monthly bill, intersect each tranche's active lifetime with the previous calendar month and multiply its principal by annual rate divided by 365 and actual active days. Prepayments reduce the active principal before future-day accrual.

### 22.2 Constant-EMI rate reset

At reset, the outstanding principal is known and the existing EMI is held constant. Solve the annuity tenure formula for `n`. If that term would end after age 70, cap `n` and use the PMT formula to recompute EMI. This preserves the contractual option in the valid tenure range and respects the eligibility cap otherwise.

### 22.3 Precedence

Explain that rule precedence prevents a misleading rejection reason. The first applicable failed gate is reported. Supporting rule IDs are recorded without changing the primary outcome.

### 22.4 Principal distinctions

Maintain separate concepts:

- sanctioned principal;
- cumulative gross bank releases;
- net disbursed principal after pre-EMI prepayments;
- current outstanding principal after EMI repayments;
- EMI interest;
- pre-EMI interest;
- processing fee and GST;
- penal interest.

---

## 23. Final Submission Checklist

| Submission item | Status |
|---|---|
| Consolidate the official sample and modular paths | Pending |
| Commit JSON fixtures or remove unsupported commands from documentation | Pending |
| Add the complete automated test suite | Pending |
| Make all visible sample outputs match after every correction | Pending |
| Correct rule-ID mappings | Pending |
| Validate payment, closure, prepayment, and partial-release state changes | Pending |
| Verify same-day event ordering and every documented edge case | Pending |
| Add a decision log | Pending |
| Update README to describe only runnable commands and artifacts | Pending |
| Generate and review the final PDF design document | Complete |
| Publish the final five-minute demo video link | Pending |
| Prepare viva walkthrough of pre-EMI and constant-EMI reset | Pending |

---

## 24. Glossary

| Term | Meaning |
|---|---|
| Agreement value | Property price used for LTV, excluding stamp duty and registration. |
| Amortisation | Schedule that separates each payment into interest and principal. |
| Benchmark | Reference market rate before credit spread. |
| Own contribution | Borrower's share of property cost paid outside the bank loan. |
| Cumulative certified percentage | Percentage of construction certified by the valuer to date. |
| Cumulative demanded percentage | Percentage of cost demanded by the builder to date. |
| EMI | Equated monthly instalment of interest and principal. |
| LTV | Loan-to-value ratio. |
| Moratorium | Construction-phase period billed by pre-EMI. |
| Pre-EMI | Interest-only accrual on released principal during construction. |
| Sanction | Maximum approved bank principal and its terms. |
| Total disbursed | Bank principal released less qualifying pre-EMI prepayments. |

---

*Prepared for Team T026, PS-021. The assignment email is the authoritative source for requirements. This design records the planned implementation and explicitly identifies current deviations.*
