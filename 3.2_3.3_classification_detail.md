### 3.2 Public Bond Classification — Technical Reference

Applied to `debt_related = True` rows only. Classification uses the same plain-text pipeline as Section 3.1, in two steps.

---

#### Step 1 — Exhibit index signals

The exhibit section (everything after `Item 9.01 Financial Statements and Exhibits`) is searched for document names that appear almost exclusively in public or Rule 144A/Reg S bond offerings:

```
Indenture / Supplemental Indenture / First–Seventh Supplemental
Underwriting Agreement
Form of Note / Form of Global Note / Form of Senior Note(s)
Legal Opinion / Opinion of Counsel / Consent of Counsel / Opinion and Consent
Initial Purchaser
```

A match triggers `is_bond = True`, subject to one veto: if the exhibit section also contains loan-type document names (`Credit Agreement`, `Term Loan`, `Promissory Note`, `Forward Sale Agreement`) **and** the filing body contains no bond-specific vocabulary, the match is overridden to `is_bond = False`. This prevents credit facility filings from being misclassified when they incidentally attach a legal opinion.

---

#### Step 2 — Body text fallback

If no bond exhibit is found — common when the filing attaches only a Press Release — the narrative body text is searched for bond-specific terms using whole-word regex matching:

```
Rule 144A / 144A / Regulation S
indenture / trustee
notes due {year}              (e.g. "Notes due 2031")
{rate}% senior notes          (e.g. "5.25% Senior Notes")
senior notes / junior subordinated / debentures
initial purchasers / offering memorandum / prospectus supplement
euro notes
```

Any single match → `is_bond = True`.

---

#### Performance

Evaluated against LLM labels (debt=True subset, n=5,127):

| Metric | Value |
|--------|-------|
| Precision | ~0.87 |
| Recall | ~0.62 |

The ~38% recall gap is structural: many bond 8-Ks attach only a Press Release with no bond-specific exhibit names, and the body text may not contain the specific terms above. These cases require LLM review.

---

### 3.3 Amendment Classification — Technical Reference

Applied to `debt_related = True` rows only. Classification uses two steps with an anti-signal veto.

---

#### Step 1 — Exhibit index signals

The exhibit section is searched for amendment-specific document names:

```
Amendment / Amended and Restated / Restatement
Supplemental Indenture / First–Seventh Supplemental
Waiver / Modification Agreement / Forbearance
Amendment No.X / First–Fifth Amendment to [Agreement]
```

Note: Supplemental Indentures are counted as amendments because they modify the terms of an existing indenture rather than establishing new debt.

---

#### Step 2 — Body text fallback

If no amendment exhibit is found, the narrative body text is searched using whole-word regex matching:

```
amendment / amended and restated
supplemental indenture / waiver
modification agreement / forbearance
amendment no.X / first–fifth amendment
```

---

#### Anti-signal veto

If the body text contains any of the following new-issuance signals, `is_amendment` is set to `False` even when amendment keywords are present — **unless** the exhibit section explicitly names an amendment document (in which case the exhibit signal takes precedence):

```
new notes / issuance of / new credit agreement
initial purchasers / underwriting
entered into a new
```

This prevents filings that describe a new bond offering alongside an amendment from being misclassified as amendments.

---

#### Performance

Evaluated against LLM labels (debt=True subset, n=5,127):

| Metric | Value |
|--------|-------|
| Precision | ~0.77 |
| Recall | ~0.81 |

The main source of false positives is Supplemental Indentures filed as part of a new bond offering (correctly a new issuance, but the supplemental indenture exhibit triggers the amendment rule). The anti-signal veto reduces but does not eliminate this error.

---

### Exhibit Type Extraction

For all filings (regardless of `debt_related`), exhibit labels are extracted directly from the `Item 9.01` section of the plain text using a regex that matches the standard SEC exhibit index format:

```
Exhibit 4.1    Indenture
Exhibit 10.1   Credit Agreement
99.1           Press Release
```

Pattern: `(?:exhibit\s+)?(\d+\.\d+\w*)\s{2,}(.+?)(?=\s{2,}|\n|$)`

- **Group 1** captures the exhibit number (e.g. `4.1`, `99.1`)
- **Group 2** captures the label verbatim (e.g. `Indenture`, `Press Release`)

Labels are **not normalised** — they are taken exactly as written in the filing. Multiple exhibits are joined with ` | ` and stored in the `exhibit_types` column. Duplicate entries (same number and label) are dropped.

This approach is fast and requires no API calls. The trade-off versus LLM extraction is that labels may vary in phrasing across filings (e.g. `"Trust Indenture"` vs `"Indenture"`), but substring matching in downstream classification handles this variation without issue.
