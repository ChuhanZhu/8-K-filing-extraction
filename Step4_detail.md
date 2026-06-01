# Step 4: Detailed Documentation

## Input

`3_debt_scan_debt_only.csv`, filtered to `is_bond == False` and `is_amendment == False` (~2,426 rows). Each row is one 8-K; the `path` column points to the main file (`MAIN_*.htm`).

---

## Phase 1 — Main File Screening (`step1.1`)

**Checkpoint key**: `(cik, accession)`

Each 8-K main file is read up to 12,000 characters. One LLM call returns answers to three questions simultaneously.

### Step 0-pre: `is_debt_related`
Is this filing primarily about a debt instrument?  
- `False`: equity issuances, press releases, employment agreements, non-debt M&A, etc.  
- When `False`: `is_amendment`, `instrument_class`, and related fields are all set to `null` (Questions 2 and 3 are skipped).  
- `8-K_topic`: always populated — a brief plain-language description of what the filing is actually about. Especially useful when `is_debt_related == False`.

### Step 0a: `is_amendment`
Is this filing an amendment or modification to an existing debt instrument?  
- `True`: "Amendment No. 2", "First Amendment to Credit Agreement", waiver and amendment language, etc.  
- Only answered when `is_debt_related == True`.

### Step 0b: `instrument_class`
Classify by the type of **newly issued** debt in this filing. A draw-down or reference to an already-existing facility does not count as a new issuance. Priority order:

| Class | Condition |
|-------|-----------|
| `revolving_term_loan` | New revolving credit facility or term loan (closing/effective date present, "entered into", "established", etc.). Wins over all other types. |
| `public_bonds` | New publicly placed bonds (Rule 144A / Reg S / CUSIP / underwriters). Only if no new revolving/term loan. |
| `notes` | New privately placed notes (Note Purchase Agreement, no CUSIP, no underwriters). Only if no revolving/term loan or public bonds. |
| `others` | No new debt — press release only, equity, employment, existing facility draw-down, etc. |

Only answered when `is_debt_related == True`.

### Pass condition
`is_debt_related == True` AND `is_amendment == False` AND `instrument_class == revolving_term_loan`.  
All others are recorded with a `skip_reason` and excluded from subsequent phases.

### Output: `4.step1.1_results.csv`
One row per 8-K.

`cik` / `accession` / `filing_date` / `filename` / `path` / `exhibit_types` / `is_debt_related` / `8-K_topic` / `is_amendment` / `amend_evidence` / `instrument_class` / `instruments_found` / `class_evidence` / `phase1_pass` / `skip_reason` / `step0_error`

---

## Phase 2 — Exhibit Classification (`step1.2`)

**Requires**: `4.step1.1_results.csv` (reads `phase1_pass == True` rows).  
**Checkpoint key**: `(cik, accession)` — completion is determined by whether the main file row has been written.

For each passing 8-K, four steps are executed:

### Step 1: Classify exhibits (LLM)
All non-`MAIN_` files in the directory are read up to 3,000 characters each. One LLM call per file returns:
- `file_type_label`: extracted directly from the document's title or opening lines, free-form lowercase (e.g. "credit agreement", "subordination agreement", "consent of independent auditors"). Not forced into a fixed category list.
- `is_loan_agreement`: `True` if the document primarily establishes or governs a revolving credit facility or term loan (Credit Agreement, Loan Agreement, Loan and Security Agreement, and variants). `False` for everything else.
- `skipped`: temporarily set to `True`; updated in Step 3.

### Step 2: Add main file row (no LLM)
The `MAIN_*.htm` file is added directly as a row with `file_type_label = "main file"` and `is_loan_agreement = False`. No LLM call is made.

### Step 3: Compute `skipped` for the whole 8-K
Once all exhibits are classified, `skipped` is determined for every file in the 8-K:

| Condition | Result |
|-----------|--------|
| At least one exhibit has `is_loan_agreement == True` | Loan agreement file(s) → `skipped = False`; all others (including main file) → `skipped = True` |
| No exhibit has `is_loan_agreement == True` | Main file → `skipped = False`; all exhibits → `skipped = True` |

### Step 4: Write to CSV
Existing exhibit rows in the CSV are updated with the final `skipped` values; the main file row is appended.

### Output: `4.step1.2_file_classify.csv`
One row per file (exhibits + main file).

`cik` / `accession` / `filename` / `file_type_label` / `is_loan_agreement` / `evidence` / `skipped` / `path`

---

## Phase 3 — Debt Extraction (`step2+step3`)

**Requires**: `4.step1.1_results.csv`. `4.step1.2_file_classify.csv` is optional — if absent, only the main file is scanned.  
**Checkpoint key**: `(cik, accession)`

### File selection
Files where `skipped == False` are extracted. This directly reflects the Phase 2 logic:
- If a loan agreement was found → only the loan agreement file(s) are extracted.
- If no loan agreement was found → only the main file is extracted.

If no `file_classify` record exists for a given 8-K, only the main file is used.

### Step 2: Identify instruments and lenders
The LLM reads the full text of each selected file and returns for each debt instrument:

- `is_new_issuance` / `is_new_issuance_evidence`: `True` = established for the first time by this filing (closing/effective date present, "entered into" language). `False` = reference to or draw-down under an existing facility.
- Lender list — includes administrative agent and named lenders; excludes arrangers, bookrunners, trustees, and guarantors. Each lender has:
  - `name`
  - `lender_name_type`: `specific` (named institution, e.g. JPMorgan Chase Bank) or `generic` (collective term, e.g. "the Lenders", "each Lender")
  - `amount_type`: `individual` (separately stated commitment) or `shared` (one total facility amount)

Opinion and consent documents return empty results and produce no rows.

### Step 3: Extract fields
Only executed for `is_new_issuance == True` debt instruments. Instruments with `is_new_issuance == False` are still written to `4.debt_lender_full.csv` with Step 3 fields left blank.

LLM call count depends on amount structure:
- All lenders `shared` → one LLM call; result copied to each lender row.
- Any lender `individual` → one LLM call per lender.

Fields extracted per debt × lender:

| Field | Description |
|-------|-------------|
| `instrument_type1` | Instrument name as it appears in the document (not the agreement name) |
| `instrument_type2` | `Revolving` (draw-repay-redraw credit line) or `Term Loan` (fixed advance repaid on schedule). If a document contains both, two rows are created. |
| `lender_name` | Legal name of the lender |
| `lending_amount` | Lender's commitment (individual) or total facility amount prefixed "Total: " (shared) |
| `rate` | Interest rate including benchmark + spread (e.g. "SOFR + 2.50%") |
| `maturity_date` | Final maturity date |
| `issuing_date` | Closing or issuance date |

Each field has a corresponding `_evidence` column containing the verbatim quote from the document.

### Output: `4.tracking.csv`
One row per 8-K processed in Phase 3.

`cik` / `accession` / `filing_date` / `filename` / `files_used` / `has_loan_agreement` / `n_debts` / `n_new_debts` / `n_lenders` / `skipped` / `skip_reason`

### Output: `4.debt_lender_full.csv`
One row per debt × lender. Includes all rows regardless of `is_new_issuance`.

`cik` / `accession` / `filing_date` / `filename` / `file_type_label` / `debt_id` / `inst_name_identified` / `is_new_issuance` / `is_new_issuance_evidence` / `instrument_type1` / `instrument_type2` / `lender_name` / `lender_name_type` / `amount_scope` / `lending_amount` / `rate` / `maturity_date` / `issuing_date` / `instrument_type1_evidence` / `instrument_type2_evidence` / `lender_name_evidence` / `amount_scope_evidence` / `lending_amount_evidence` / `rate_evidence` / `maturity_date_evidence` / `issuing_date_evidence`

### Output: `4.debt_lender.csv`
Subset of `4.debt_lender_full.csv`: only `is_new_issuance == True` rows, all `_evidence` columns removed.

---

## Checkpoint Resumption Summary

| Phase | Checkpoint key | Completion signal |
|-------|---------------|-------------------|
| Phase 1 | `(cik, accession)` | Row exists in `4.step1.1_results.csv` |
| Phase 2 | `(cik, accession)` | Main file row written to `4.step1.2_file_classify.csv` |
| Phase 3 | `(cik, accession)` | Row exists in `4.tracking.csv` |
