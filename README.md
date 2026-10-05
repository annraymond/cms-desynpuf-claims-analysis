# CMS DE-SynPUF claims analysis

A self-directed project working with US Medicare claims data: pulling the CMS synthetic claims files, cleaning and validating them against CMS's own documentation, and running a small chronic condition and cost analysis. The pipeline was written by Claude Code from my instructions, and I audited it independently (see [How this was built](#how-this-was-built)).

**The data are synthetic.** The results demonstrate method and pipeline quality, not real Medicare patterns. CMS states that relationships between variables in the synthetic files were altered.

## Start here

| File | What it is |
|---|---|
| `Outputs/cms_synpuf_cleaning.html` | Pull, clean and validate all five file types |
| `Outputs/cms_synpuf_analysis.html` | Condition prevalence and correlations, cost by condition, reconciliation to the Beneficiary file |
| [`audit_log.md`](audit_log.md) | What I checked, what I found and changed, and limitations |

GitHub shows HTML files as source code, so download them and open them in a browser. The R Markdown sources are in `Markdown files/`.

## Data

CMS 2008-2010 Data Entrepreneurs' Synthetic Public Use File (DE-SynPUF), Sample 1 ([CMS download page](https://www.cms.gov/data-research/statistics-trends-and-reports/medicare-claims-synthetic-public-use-files/cms-2008-2010-data-entrepreneurs-synthetic-public-use-file-de-synpuf/de10-sample-1)). Diagnoses are ICD-9 coded. Raw row counts match the CMS Data User's Guide (Table 2) for every file.

| File type | Raw rows | Cleaned file (`Data/Clean/`) | Cleaned rows x columns |
|---|---:|---|---:|
| Beneficiary Summary 2008 / 2009 / 2010 | 116,352 / 114,538 / 112,754 | `beneficiary_2008_2010.csv` | 343,644 x 34 |
| Inpatient claims | 66,773 | `inpatient_claims_sample_1.csv` | 66,705 x 37 |
| Outpatient claims | 790,790 | `outpatient_claims_sample_1.csv` | 779,537 x 75 |
| Carrier claims (A + B) | 4,741,335 | `carrier_claims_sample_1.csv` | 4,741,335 x 21 |
| Prescription Drug Events | 5,552,421 | `prescription_drug_events_sample_1.csv` | 5,552,421 x 8 |

## What the notebooks do

**Cleaning** (`cms_synpuf_cleaning.Rmd`)
- Renames CMS's all-caps field codes to descriptive snake_case names, taken from the CMS Codebook.
- Recodes flags, race and state with the Codebook's category tables, and casts dates and amounts.
- Logs every row and column drop, collapse and bind (before and after), and validates every raw row count against CMS.
- Adds `total_claim_amt` per claim (Inpatient, Outpatient, Carrier) and `total_annual_amt` per beneficiary-year.
- Checks that every beneficiary in each claims file exists in the Beneficiary file (0 orphans in all four).

**Analysis** (`cms_synpuf_analysis.Rmd`)
1. Prevalence and pairwise correlations of the 11 CMS chronic condition flags.
2. Average and median claim amounts by condition for inpatient, outpatient and prescription claims.
3. Reconciliation of the Beneficiary file's annual reimbursement columns against totals rebuilt from the claims files. Inpatient and outpatient match to the dollar, for all six components and for every beneficiary-year.

## Key analytical decisions

| Decision | Reason |
|---|---|
| `SEGMENT = 2` rows are dropped from Inpatient (68) and Outpatient (11,253) | They have blank claim dates and mostly blank clinical fields. Dropping them is what makes claim totals reconcile to the Beneficiary file |
| Inpatient Medicare payment is `CLM_PMT_AMT + utilization days x per-diem pass-through` | This is the Codebook (Appendix 2) definition. Payment alone does not reconcile |
| A blank Part A deductible (2,173 inpatient claims) counts as $0 in `total_claim_amt`; the column itself stays `NA` | The deductible is charged once per benefit period. Dropping those claims instead changes the inpatient mean by 0.45% and leaves the median unchanged |
| Carrier is collapsed from 142 to 21 columns (service lines summarized per claim) and is not reconciled | Keeps the file manageable. Per-line detail would need a long-format reshape |
| A condition counts as present if it was flagged in any of 2008, 2009 or 2010, with one row per beneficiary (116,352 people) | Avoids counting a persistent condition three times |
| Cost by condition is per claim, not per person | A person with several conditions appears in several condition rows, so rows are not independent |
| 45 empty inpatient HCPCS columns and `BENE_COUNTY_CD` are dropped | Empty by design in inpatient claims; county dropped by choice |

`total_claim_amt` is Medicare-paid + primary-payer-paid + beneficiary responsibility. The beneficiary components are the Part A deductible, Part A coinsurance and blood deductible for inpatient, and the blood deductible, Part B deductible and Part B coinsurance for outpatient. Prescription cost uses `gross_drug_cost_amt`.

## Running it

Requires R with `tidyverse`, `lubridate`, `janitor` and `data.table` (installed automatically by the notebooks). Carrier and PDE are large (each Carrier segment is about 1.2 GB as CSV), and peak R memory in my run was roughly 8 GB.

1. Clone the repo and keep the folder structure. The notebooks use relative paths such as `../Data/Extracted`.
2. Download the CMS Sample 1 zips into `Data/Raw` and extract them into `Data/Extracted`. The download chunk in the cleaning notebook does this but is set to `eval=FALSE`, so run it once by hand. Raw data are not in the repo.
3. Knit `cms_synpuf_cleaning.Rmd` first (it writes `Data/Clean/`), then `cms_synpuf_analysis.Rmd`.

## Limitations

- Synthetic data, ICD-9 coding, 2008-2010 only. Not suitable for clinical or policy conclusions.
- Amounts are payments, not billed charges. Inpatient and outpatient totals exclude drug costs.
- Beneficiaries with HMO months have incomplete fee-for-service claims, and the analysis does not restrict to full-year fee-for-service enrollees.
- Carrier claims are not reconciled to the Beneficiary file.

The full list is in [`audit_log.md`](audit_log.md). Natural extensions I scoped out: reconciling Carrier, restricting to full-year fee-for-service enrollees, and a time-to-readmission analysis on the Inpatient file.

## How this was built

Claude Code wrote the pipeline code from my instructions. I reviewed the notebooks and independently re-checked the prevalence, cost and reconciliation figures using SQL (DuckDB) embedded in R, and spot-checked a sample of the correlations. [`audit_log.md`](audit_log.md) records the issues found and fixed (for example a mismatched download URL and a missing per-diem term in the inpatient total) and what I verified.

## Sources

CMS DE-SynPUF Data User's Guide, Codebook and FAQ (copies in `Docs/`), and the CMS download page linked above.
