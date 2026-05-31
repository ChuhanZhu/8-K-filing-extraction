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

For full keyword lists, matching rules, and performance metrics, see [`3.1 debt_classification_detail.md`](3.1debt_classification_detail.md).


### 3.2 Public Bond Classification (`debt_related = True` rows only)

Each debt-related filing is classified as `is_bond = True` or `False` using a two-step rule-based pipeline with no API calls.

First, the exhibit index (Item 9.01) is searched for bond-specific document names such as Indenture, Underwriting Agreement, Form of Note, and Legal Opinion. A match triggers is_bond = True, unless the exhibit section simultaneously contains loan-type document names (e.g. Credit Agreement, Term Loan, Promissory Note) and the filing body contains no bond-specific vocabulary. Second, if Step 1 does not return True, the narrative body text is searched for bond-specific terms including Rule 144A, senior notes, trustee, initial purchasers, and rate-and-maturity patterns such as 5.25% Senior Notes due 2031. Any single match triggers is_bond = True.

For full keyword lists, matching rules, and performance details, see [`3.2_3.3_classification_detail.md`](3.2_3.3_classification_detail.md).

---

### 3.3 Amendment Classification (`debt_related = True` rows only)

Each debt-related filing is classified as is_amendment = True or False in two steps.

First, the exhibit index (Item 9.01) is searched for amendment-specific document names: Amendment, Amended and Restated, Restatement, Supplemental Indenture, First–Seventh Supplemental, Waiver, Modification Agreement, Forbearance, and First–Fifth Amendment. A match immediately sets is_amendment = True with no further checks — an explicit amendment document in the exhibit index is treated as definitive.

Second, if no amendment exhibit is found, the narrative body text is searched for the same amendment terms. Here an anti-signal veto applies: if the body simultaneously contains new-issuance language (new notes, issuance of, new credit agreement, initial purchasers, underwriting), the match is overridden to is_amendment = False.

For full keyword lists and performance metrics, see [`3.2_3.3_classification_detail.md`](3.2_3.3_classification_detail.md).


### 3.4 Exhibit Type Extraction (all filings)

Exhibit labels are extracted from every filing regardless of `debt_related` status. The plain text is split at the first occurrence of `Item 9.01` or *Financial Statements and Exhibits*, and the exhibit section is parsed line by line using a regex that captures the exhibit number and label verbatim from each line. Labels are not normalised. All labels for a given accession are joined with ` | ` and stored in the `exhibit_types` column. Duplicate entries are dropped.

---

## Step 4: Extract Debt Terms

**Script**: `4.8k_debt_extractor_v4.py`

This script handles exhibit selection, new issuance flagging, public bond flagging, and extraction — all in one pipeline.

### 4.1 Exhibit Selection
- **Skip** filings classified as amendments or public bonds (from Step 3).
- **Exhibit priority**: If an agreement document exists, extract from the agreement only. If no agreement exists, fall back to the 8-K main file. Always skip opinion, consent, and press release documents.

### 4.2 New Issuance & Public Bond Flags
For each debt/lender identified within a document:
- Flag whether it is a **new issuance** (vs. a historical debt mentioned in passing).
- Flag whether it is a **public bond**.
- Only proceed to extraction if both flags are `False`.

### 4.3 Extraction at Debt-Lender Level
Extract terms for each debt-lender pair via separate API calls to ensure accuracy when a filing contains multiple debts or multiple lenders.

---

## Step 5: Clean Up

### Step 5.1: Coarse Deduplication
**Script**: `5.1 dedup_debt_lender.py`

- String matching on CSV fields.
- LLM-based comparison of string similarity for fuzzy duplicates.

### Step 5.2: Re-classify Public Bonds and Revolving/Term Loan
**Script**: `5.2 bond_classifier.py`

Split results into subgroups by instrument type:
- Only revolving/term loan rows.
- Only bonds/notes/other rows.
- Mixed rows.

Run a second-pass public bond classification on the subgroups.

**Manual checks on `public_bonds == False` rows:**
- Private placement notes should not be classified as public bonds.
- Watch for edge cases in the "other" category: warrants and stock purchase agreements (should be excluded); secured inventory-based revolving credit facilities (should be classified as revolving).

### Step 5.3: Check Fields
**Script**: `5.3 check_fields.py`

Data quality checks on revolving/term loan rows only.

#### 5.3.1 Check Instrument Type
**Manual checks:**
- `(cik 1018724, accession 119312517205287)` — flagged as merger, review classification.
- Bridge facilities: should not be classified as term loans.

#### 5.3.2 Check Rate, Maturity Date, Issuance Date
Verify extracted temporal and rate fields against source documents.

#### 5.3.3 Check Amount
Clean and standardize extracted amount fields.

### Step 5.4: Fine-Grained Deduplication
**Script**: `5.4 dedup_revolving.py`

Focus deduplication on filings where multiple documents were extracted under the same 8-K accession number — these are most likely to contain duplicates with different surface forms.

> **Future improvement**: More precise exhibit selection upstream (e.g., skipping the main file when an agreement is available) would reduce the deduplication burden here.

---

## Open Questions / TODOs

- [ ] `instrument_type` classification granularity: is a fine-grained split (revolving vs. term loan vs. bonds/notes) necessary, or is a coarser grouping sufficient?
- [ ] Edge cases in "other" category: Swingline Commitments, Subordination Agreements (loan agreement present → can skip?).
- [ ] Multi-debt, multi-lender extraction accuracy: validate that per-debt-lender API calls correctly isolate the right terms.
- [ ] Stock purchase / warrant rows should be excluded from the debt dataset entirely.
