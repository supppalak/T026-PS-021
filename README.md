# 🏠 T026-PS-021 · Home Loan with Construction-Linked Disbursement Tranches and Pre-EMI

**👥 Team T026 · Section CSDS-A**

| Member | Roll No | Email |
|--------|---------|-------|
| Nethraj T M | 1AT24CD052 | nethrajt@gmail.com |
| Niranjan | 1AT24CD053 | ushaniranjan03@gmail.com |
| Palak Raj | 1AT24CD054 | palaktina1@gmail.com |

---

## 📋 Overview

A complete, deterministic mortgage-servicing engine for under-construction property loans. Designed as a command-file-driven backend without external dependencies or UI overhead.

**✨ Core Capabilities:**
- **📝 Origination & Sanction:** Borrower/property registration, credit-based risk pricing, LTV & FOIR eligibility checks, and processing fee computation.
- **🏗️ Tranche Disbursement:** Construction-linked release validation enforcing stage certification freshness and strict rule precedence.
- **💸 Pre-EMI Billing:** Daily actual/365 interest accrual tracked independently per disbursed tranche during the construction moratorium.
- **📅 Amortisation & Servicing:** Automatic conversion to full EMI, floating benchmark rate resets, part/full prepayments, and overdue penal interest.
- **📊 Reporting & Audit:** Automated generation of sanction letters, disbursement ledgers, amortisation schedules, FY interest certificates, and deterministic event logs.

---

## 🛠️ Technology Stack

| Component | Implementation |
|-----------|----------------|
| **🐍 Language** | Python 3.10+ |
| **🔢 Precision Arithmetic** | `decimal.Decimal` (`ROUND_HALF_UP`, `prec=28`) |
| **📆 Date & Calendar Logic** | `datetime.date`, `calendar` |
| **💾 Data & File I/O** | `json`, `csv`, `pathlib` |
| **💻 CLI Interface** | `argparse` |
| **🧪 Testing Framework** | `unittest` (standard library) |
| **📦 External Dependencies** | None (100% Python Standard Library) |

---

## 📂 Project Structure

```text
T026-PS-021/
│
├── main.py                         ← 🚀 CLI entry point (Phase 1 + Phase 2)
├── README.md
│
├── data/
│   └── sample/
│       ├── borrowers.json          ← 👤 Sample borrower profiles (3 records)
│       ├── properties.json         ← 🏢 Sample properties (₹30L / ₹55L / ₹90L)
│       ├── commands.json           ← 1️⃣ Phase 1 input: SANCTION commands only
│       └── commands_phase2.json    ← 2️⃣ Phase 2 input: Full lifecycle event stream
│
├── design/
│   ├── design_document.md          ← 📐 System architecture & specification
│   └── decision_log.md             ← ⚖️ Edge-case resolutions & design assumptions
│
├── src/
│   ├── models/
│   │   ├── borrower.py             ← Borrower, CoApplicant dataclasses
│   │   ├── property.py             ← Property, StageItem dataclasses
│   │   └── loan.py                 ← SanctionResult, LoanAccount, DisburseRecord,
│   │                                  PreEmiBill, EmiRecord, EventLog
│   ├── rules/
│   │   └── business_rules.py       ← 📏 Centralized rule definitions (BR-01 → BR-26)
│   ├── calculations/
│   │   ├── eligibility.py          ← LTV, age-capped tenure, interest rate, PMT eligibility
│   │   ├── fees.py                 ← Processing fee + GST split calculation
│   │   ├── sanction.py             ← process_sanction() orchestrator
│   │   ├── disbursement.py         ← Tranche release checks & precedence (BR-26)
│   │   ├── pre_emi.py              ← Daily actual/365 accrual & monthly billing (BR-17)
│   │   └── emi_schedule.py         ← Full EMI schedule, rate resets & prepayments
│   ├── engine/
│   │   └── loan_engine.py          ← ⚙️ Stateful event processor (LoanEngine)
│   ├── parsers/
│   │   └── input_parser.py         ← 🔍 JSON schema validation & parsing
│   └── output/
│       ├── sanction_letter.py      ← 📄 Formatted sanction letter generator
│       ├── report_writer.py        ← 📑 JSON & summary report writers
│       └── ledger_writer.py        ← 📒 Ledger, schedule, FY certificate & audit log writers
│
├── tests/
│   ├── test_phase1.py              ← ✅ 46 Phase 1 unit tests
│   └── test_phase2.py              ← ✅ 58 Phase 2 unit & integration tests
│
└── output_files/                   ← 📤 Generated output directory
