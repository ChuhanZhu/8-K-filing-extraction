### 3.1 Debt-Related Classification — Technical Reference

Debt classification is performed via a three-layer rule-based string matching pipeline on the 8-K main file (`MAIN_*.htm`), with no API calls. Each layer is applied in sequence; a match at any layer immediately classifies the filing as `debt_related = True`.

---

#### Pre-processing

The raw HTML is stripped of all tags via regex to produce plain text. The text is then split at the `Item 9.01 Financial Statements and Exhibits` section heading, yielding two zones used separately by later layers:
- **`body`** — everything before Item 9.01 (the narrative text describing the transaction)
- **`exhibit_section`** — everything from Item 9.01 onward (the exhibit index listing attached documents)

---

#### Layer 1 — SEC Item Number

Searches the full text for SEC-mandated item headings. These headings are required by SEC rules and appear verbatim in every 8-K, so a match is unambiguous.

| Item | Full Title | Rationale |
|------|-----------|-----------|
| **Item 2.03** | Creation of a Direct Financial Obligation or an Obligation under an Off-Balance Sheet Arrangement of a Registrant | By definition, the company has incurred new debt — a new loan, bond issuance, or credit facility. 100% debt-related. |
| **Item 2.04** | Triggering Events That Accelerate or Increase a Direct Financial Obligation or an Obligation under an Off-Balance Sheet Arrangement of a Registrant | A covenant breach, rating downgrade, or other event has accelerated or increased an existing debt obligation. Effectively 100% debt-related in practice. |
| **Item 1.03** | Bankruptcy or Receivership | The company has filed for bankruptcy or entered receivership, which necessarily involves debt restructuring. |

**Known gap:** Many genuine debt filings are reported only under **Item 1.01** (*Entry into a Material Definitive Agreement*), which covers any material contract — including credit agreements and indentures — but is not debt-specific. These are not caught by Layer 1 and require Layers 2–3. In practice, Layer 1 captures approximately 29% of debt-related filings.

---

#### Layer 2 — Exhibit Index Keywords

Searches the `exhibit_section` zone only. Exhibit names in the Item 9.01 index are highly standardised (e.g. "Indenture", "Credit Agreement"), making them reliable debt signals when matched precisely.

**Layer 2a — High-precision exhibit terms** (match → `True` immediately, unless equity guard fires):

These terms appear in the exhibit index almost exclusively in debt filings:

```
Indenture / Supplemental Indenture / First–Seventh Supplemental
Credit Agreement / Amended and Restated Credit Agreement / Amendment to Credit Agreement
Loan Agreement / Loan and Security Agreement
Term Loan / Revolving Credit / Credit Facility
Promissory Note / Note Purchase Agreement / Bond Purchase Agreement
Bridge Loan / Bridge Facility / Debenture
Receivables Purchase Agreement / Securitization
DIP Facility / DIP Term Sheet / DIP Credit
Forbearance Agreement / Debt Conversion Agreement
Intercreditor Agreement / Pledge Agreement / Collateral Agreement
Guarantee Agreement / Guaranty Agreement
Senior Secured / Senior Unsecured / Junior Subordinated
First–Seventh Supplemental (Supplemental Indenture series)
```

**Layer 2b — Ambiguous exhibit terms** (match → `True` only if `body` also contains a debt core word):

These terms are too broad to trust alone, as they appear in both debt and equity filings:

| Term | Why ambiguous |
|------|--------------|
| `Underwriting Agreement` | Used for both bond underwriting and equity offerings |
| `Purchase Agreement` | Covers Note Purchase Agreement (debt), Securities Purchase Agreement (equity), and Asset Purchase Agreement (M&A) |
| `Security Agreement` | Sometimes relates to equity securities rather than collateral |
| `Conversion Agreement` | May refer to equity conversion rather than debt conversion |

A match here only triggers `True` if the `body` simultaneously contains at least one debt core word: `notes`, `bonds`, `debentures`, `indenture`, `credit agreement`, `term loan`, `revolving`, `SOFR`, `LIBOR`, `maturity date`, `principal amount`, or `administrative agent`.

**Equity guard:** If the exhibit index contains equity-specific terms (`pre-funded warrant`, `at-the-market`, `placement agent`, `subscription agreement`, `certificate of designation`, etc.), Layer 2a is suppressed entirely and only Layer 2b logic applies (with mandatory body verification). This prevents equity offering filings from being misclassified due to incidental matches on terms like `Underwriting Agreement`.

---

#### Layer 3 — Body Text Keywords

Searches the `body` zone. This layer captures filings where the exhibit index contains only generic entries (e.g. "Press Release", "Amendment") but the narrative text describes a debt transaction.

**Layer 3a — Strong keywords** (any single match → `True`):

Highly specific debt terms that rarely appear in non-debt filings:

```
senior notes / senior secured notes / senior unsecured notes
subordinated notes / convertible notes / floating rate notes / fixed rate notes
secured notes / unsecured notes / debentures / junior subordinated debentures
indenture / supplemental indenture / trust indenture act
credit agreement / amended and restated credit / credit facility
term loan / revolving credit / incremental term / term A loan / term B loan
bridge loan / bridge facility / promissory note / note purchase agreement
SOFR / LIBOR / administrative agent / collateral agent
maturity date / borrower / lenders / guarantors
DIP facility / forbearance agreement / chapter 11 / bankruptcy
first lien / second lien / mezzanine / PIK notes / payment-in-kind
asset-backed / securitization / tranche / covenant-lite
receivables purchase agreement / intercreditor agreement
[pattern] notes due {year}  (e.g. "Notes due 2031")
[pattern] {rate}% senior notes  (e.g. "5.25% Senior Notes")
```

**Layer 3b — Weak keywords** (≥ 2 matches required, and fewer than 3 exclude keywords present):

General financial terms that are individually insufficient but collectively indicative:

```
debt / financing / interest rate / principal amount / principal balance
borrowing / repayment / repaid / redemption / collateral
guarantee / covenants / default / amortization / prepayment
acceleration / trustee / investment grade
Rule 144A / Regulation S / registration rights
offering memorandum / offering circular / prospectus supplement
```

**Exclude keywords** (suppress Layer 3b if ≥ 3 are present):

These indicate the filing is primarily an earnings release, governance event, or personnel change rather than a debt transaction:

```
earnings / quarterly results / annual results / financial results
press release / conference call / dividend
share repurchase / stock repurchase / proxy / annual meeting
election of directors / employment agreement / severance
merger agreement / acquisition agreement
```

If none of the three layers match, the filing is classified `debt_related = False`.

---

#### Performance

Evaluated against LLM-labelled ground truth (n = 27,585 accessions, year 2017):

| Metric | Value |
|--------|-------|
| Precision | 0.96 |
| Recall | 0.71 |
| Processing speed | ~200 files/sec (single core) |

The ~29% recall gap primarily reflects LLM over-labelling: filings where a debt-issuing company discloses earnings or an officer change, and the LLM flagged as debt-related due to incidental mentions of existing obligations. True debt event recall is estimated above 0.95.
