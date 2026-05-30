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

**Script**: `3.8k_classifier.py`

For each 8-K, classify along three dimensions: debt relevance, amendment status, and public bond status.

### 3.1 Debt-Related Classification
Read the 8-K main file and determine `debt_related`:

> Set `debt_related = True` if this 8-K primarily or significantly concerns any debt instrument (bonds, notes, debentures, credit facilities, term loans, revolving credit, indentures, supplemental indentures, debt amendments, debt repayments, or any other borrowing arrangement). Set to `False` otherwise.

Also extract: topic details, exhibit type.

### 3.2 Public Bond Classification (`debt_related == True` rows only)
Check exhibits for the presence of underwriting agreements or similar documents as a token-efficient proxy for identifying public bond offerings.

### 3.3 Amendment Classification (`debt_related == True` rows only)
Read the 8-K main file and determine `amendment`:

> Read this document and determine whether it primarily describes an AMENDMENT to an already-existing debt obligation (i.e., modifying rate, maturity, amount, covenants, or other terms of a prior agreement).

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

> **Open question**: How to guarantee that extracted rates/amounts are correctly matched to the right debt-lender pair when multiple are present in a single document?

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
