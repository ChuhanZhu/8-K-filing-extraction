# 8-K Debt Instrument Dataset Pipeline

## Overview

This pipeline processes SEC 8-K filings to extract structured data on debt instruments issued by non-financial firms.

---

## Step 1: Download 8-K Dataset

**Script**: `1.sec_8k_batch_downloader.py`

Download raw 8-K filings from SEC EDGAR in batches.

---

## Step 2: Classify Financial vs. Non-Financial Firms

### Step 2.1: Download CIK-SIC Mapping
**Script**: `2.1.download_cik_sic_edgar.py`

Download the CIK-to-SIC mapping from EDGAR to identify firm industry codes.

### Step 2.2: Classify Firms
**Script**: `2.2.split_fin_nonfin.py`

Use SIC codes to separate financial firms from non-financial firms. Exclude financial firms from downstream processing.

---

## Step 3: Classify Debt-Related 8-Ks

**Script**: `3.scan_debt_8k.py`

For each 8-K, classify along three dimensions: debt relevance, amendment status, and public bond status.

### 3.1 Debt-Related Classification

Each 8-K main file (`MAIN_*.htm`) is classified as `debt_related = True` or `False` using a three-layer rule-based string matching pipeline — no API calls are made. The pipeline processes ~200 files per second, reducing a 30,000-filing dataset to a debt-relevant subset in under 5 minutes.

The three layers are applied in sequence; the first match terminates the check:

1. **SEC Item number** — searches for `Item 2.03`, `Item 2.04`, or `Item 1.03` in the full text. These SEC-mandated headings unambiguously identify debt creation, debt acceleration/default events, and bankruptcy filings respectively.

2. **Exhibit index keywords** — searches the `Item 9.01` exhibit listing for standardised document names (e.g. *Indenture*, *Credit Agreement*, *Term Loan*). Ambiguous terms such as *Purchase Agreement* and *Underwriting Agreement* require corroborating debt vocabulary in the filing body before triggering a positive classification.

3. **Body text keywords** — searches the narrative text for debt-specific terminology. High-specificity terms (e.g. *SOFR*, *senior notes*, *maturity date*, *administrative agent*) trigger classification alone; lower-specificity terms (e.g. *debt*, *collateral*, *covenants*) require two or more co-occurrences and the absence of counter-signals such as *earnings*, *press release*, or *employment agreement*.

For full keyword lists, matching rules, and performance metrics, see [`Step3.1 debt_classification_detail.md`](Step3.1debt_classification_detail.md).


### 3.4 Exhibit Type Extraction (all filings) (need to be modified)

Exhibit labels are extracted from every filing regardless of `debt_related` status. The plain text is split at the first occurrence of `Item 9.01` or *Financial Statements and Exhibits*, and the exhibit section is parsed line by line using a regex that captures the exhibit number and label verbatim from each line. Labels are not normalised. All labels for a given accession are joined with ` | ` and stored in the `exhibit_types` column. Duplicate entries are dropped.

---

## Step 4: Extract Debt Terms (LLM)
**Script**: `4.debt_identify_classify.py`

This script screens, classifies, and extract new issued revolving/term loan terms from 8-K filings.

### Phase 1 — Main File Screening and extract new issued debts
Each 8-K main file is passed to the LLM in a single call that extracts all new issued debts and classifies these debts (revolving/term loan, public bonds, notes, others):

1. The LLM reads the full Main file and identifies all debt instruments, flagging each as a new issuance or a reference to an existing one.

2. **What type of new debt instrument does it describe?** Classified by the type of debt *newly issued* in this filing — a draw-down or reference to an existing facility does not qualify. Priority order: `revolving_term_loan` → `public_bonds` → `notes` → `others`. Only filings classified as `revolving_term_loan` proceed to *phase 2*.



### Phase 2 — Exhibit Classification
For each 8-K (non-amendment revolving/term loan related 8-K) that passed Phase 1, all non-main files in the same directory are read and classified by the LLM. The label (`file_type_label`) is extracted directly from the document's own title or opening lines. Each file is also flagged for `is_loan_agreement`: whether it is a primary agreement governing a revolving credit facility or term loan (Credit Agreement, Loan and Security Agreement, and variants).

The main file is added as a record without an LLM call, identified by its `MAIN_` prefix.

Once all files in a filing are classified, a `skipped` flag is set for every file:
- If any exhibit is a loan agreement → only the loan agreement file(s) are marked `skipped = False`; all other files including the main file are marked `skipped = True`.
- If no loan agreement exists → only the main file is marked `skipped = False`; all exhibits are marked `skipped = True`.

### Phase 3 — Debt Extraction
Files where `skipped == False` are extracted — directly reflecting the Phase 2 logic above.

**Instrument and lender identification (Step 2):** The LLM reads the full file and identifies all debt instruments, flagging each as a new issuance or a reference to an existing one. For each instrument it also identifies lenders (excluding arrangers, bookrunners, trustees, and guarantors), and whether their commitment amount is stated individually or shared across the syndicate.

**Field extraction (Step 3):** Only new issuances proceed to extraction. Non-new-issuance instruments are still recorded but with extraction fields left blank. Fields extracted: `instrument_type1` (name as it appears in the document), `instrument_type2` (classify as Revolving or Term Loan), `lender_name`, `lending_amount`, `rate`, `maturity_date`, `issuing_date`. Each field has a corresponding verbatim evidence column.

See [`Step4_detail.md`](Step4_detail.md) for full field-level documentation of all output files.

---

## Step 5: Clean Up

### Step 5.1: Cross-File Deduplication （LLM)
**Script**: `5.1.dedup_debt_lender.py`

For 8-K filings where lender rows were extracted from multiple source files under the same accession, the LLM compares rows across files and identifies the less complete file — signals include fewer lenders covered, aggregated-only amounts, and fewer fields populated. All overlapping rows from that file are marked `is_repeat=True`, retaining the more detailed source agreement exhibit over summary documents (main filing, press release). Single-file groups pass through unchanged.

> **Future improvement**: Selecting only the most detailed exhibit upstream would reduce the cross-file overlaps that need to be resolved here.

---

## Open Questions / TODOs

- [ ] `instrument_type` classification granularity: is a fine-grained split (revolving vs. term loan vs. bonds/notes) necessary, or is a coarser grouping sufficient?
- [ ] Edge cases in "other" category: Swingline Commitments, Subordination Agreements (loan agreement present → can skip?).
- [ ] Multi-debt, multi-lender extraction accuracy: validate that per-debt-lender API calls correctly isolate the right terms.
- [ ] Stock purchase / warrant rows should be excluded from the debt dataset entirely.
