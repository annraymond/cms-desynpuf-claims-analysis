# Audit log: CMS DE-SynPUF claims pipeline

Author: Ann Raymond | Last updated: October 2026

## Scope and method

This log covers two notebooks: `cms_synpuf_cleaning.Rmd` (pull, clean, validate) and `cms_synpuf_analysis.Rmd` (condition prevalence and correlations, cost by condition, reconciliation).

Claude Code wrote the pipeline code from my instructions. I reviewed the notebooks and independently re-checked the reported numbers using SQL (DuckDB) embedded in R, run against the raw and cleaned CSVs. I also used Claude (chat) for syntax help and to explain unfamiliar CMS fields. Where blank values needed a decision, I followed standard missing-data reporting practice: quantify the blanks, state the assumption, and test an alternative (Faria et al. 2014, PharmacoEconomics; Leurent et al. 2018, Health Economics). Raw CMS data is not in the repo (`.gitignore`); download steps are in the cleaning notebook.

## Issues found and fixed

| # | Issue | How found | Resolution |
|---|---|---|---|
| 1 | The 2010 beneficiary download URL pointed to the Sample 20 file but was saved as Sample 1 | Code review | URL corrected to Sample 1. The 2010 row count (112,754) matches CMS |
| 2 | 68 inpatient rows with `SEGMENT = 2` were incomplete or inconsistent | SQL checks on dates, nulls and dollar amounts; one claim inspected by hand | Dropped before any total is calculated (details below) |
| 3 | Inpatient `total_claim_amt` left out the per-diem pass-through term, which the Codebook (Appendix 2) includes in Medicare's payment | Compared the cleaning formula with the reconciliation formula | Term added |
| 4 | Zero-filling the blank Part A deductible column broke the exact reconciliation | Reconciliation re-run after the change | Column left as `NA`. The blank is treated as $0 inside `total_claim_amt` only (see below) |
| 5 | Cost by condition used Medicare payment only for inpatient and outpatient (inpatient without per-diem) but total drug cost for prescriptions | Reviewed how each column was built | Inpatient and outpatient now use all-payer `total_claim_amt`; prescriptions use gross drug cost |
| 6 | 116,352 was labeled "ever-diagnosed" but counts all unique beneficiaries | Questioned the denominator | Relabeled; prevalence table added |
| 7 | Several labels and statements were inaccurate: a payment-only column named as Medicare-paid, "nine" annual columns when six were tested, "continuation records", "billed components" | Final review of both notebooks | Corrected. Added race and state `NA` counts and a beneficiary-year mismatch table |

## What I checked

- **Row counts.** The notebook validates every raw file against CMS Data User's Guide Table 2 (all match). I re-counted in DuckDB.
- **Beneficiary ID uniqueness.** DuckDB `SUMMARIZE` showed an approximate distinct count (90,973) below the row count, which looked like duplicates. An exact `COUNT(DISTINCT)` showed IDs are unique. `approx_unique` is an estimate and should not be used to test keys.
- **Reconciliation to the Beneficiary file.** For inpatient and outpatient, Medicare-paid, beneficiary responsibility and primary-payer amounts match the annual totals to the dollar (inpatient: 648,456,860 / 73,822,082 / 26,565,700; outpatient: 209,292,350 / 64,492,100 / 8,006,730). At beneficiary-year level, none of the six components differs by more than half a cent, and none is `NA` in the Beneficiary file. An earlier notebook version reported an exact match before the segment filter existed. I could not confirm that for the non-Medicare components, because `SEGMENT = 2` rows carry real amounts (coinsurance $9,000, primary payer $70,000, deductible $66,864 in total). I rely on the post-filter result.
- **`SEGMENT` in inpatient.** Of 66,705 `SEGMENT = 1` claims, claim end date equals discharge date in all, and claim start equals admission date in 66,652 (53 differ, 0.08%). The 68 `SEGMENT = 2` rows have blank claim dates and mostly blank clinical fields, but non-zero payments that differ from segment 1. For the claim I inspected, the admission date was earlier than segment 1's. This does not look like a simple interim-billing continuation, so I treat these rows as unreliable and exclude them.
- **Blank Part A deductible.** 2,173 inpatient claims (3.3%) have a blank deductible. CMS documentation says the deductible is charged once per benefit period and that the field is filled only when a deductible value code is present, so I read a blank as "no deductible applied" and treat it as $0 inside `total_claim_amt`. This is an interpretation of the field definition. The column itself stays `NA`, because CMS's annual figures only reconcile when each component is summed separately. Dropping these claims instead changes the inpatient mean claim amount by 0.45% (11,184 vs 11,234) and leaves the median at 8,068. The dropped claims are somewhat more expensive (implied average about $12,700 vs $11,200), so I did not treat them as missing at random.
- **Total vs components.** `total_claim_amt` sums to 749,345,102. The three reconciliation components sum to 748,844,642. The difference of $500,460 is the coinsurance and blood-deductible amounts on the blank-deductible claims, which the reconciliation's beneficiary-responsibility row leaves out (as CMS's figure does) and `total_claim_amt` keeps.
- **Outpatient `SEGMENT = 2`.** 11,253 rows (1.4% of outpatient claims; 790,790 raw minus 779,537 cleaned) are dropped under the same rule as inpatient. All of them have blank claim start and end dates and 5,597 (50%) have no primary diagnosis, which matches the inpatient pattern. The outpatient reconciliation still holds after the drop.
- **Outpatient blanks.** None of the five outpatient amount columns has a blank (0 of 779,537 claims), so outpatient `total_claim_amt` has no `NA`.
- **Recodes.** Race and state have 0 `NA` values after recoding.
- **Reported figures.** I recomputed condition prevalence, cost by condition (inpatient, outpatient and prescriptions, after the changes above), the reconciliation totals and the beneficiary-year mismatch counts in SQL. For the correlation matrix I spot-checked 5-6 pairs, including diabetes-CHF.

## Limitations to keep in mind

- The data are synthetic. CMS states that relationships between variables were altered, so condition correlations and cost patterns demonstrate method, not clinical fact. Claims use ICD-9 and cover 2008-2010 only.
- Amounts are payments, not billed charges. Inpatient and outpatient totals exclude drug costs (prescriptions are separate).
- Beneficiaries with HMO coverage months will have incomplete fee-for-service claims. The analysis does not restrict to full-year fee-for-service enrollees. Part A and Part B months can legitimately differ.
- A condition counts as present if it was flagged in any of 2008, 2009 or 2010. Cost by condition is per claim (not per person), reflects all of a person's claims (associated, not attributable), and counts a person once per condition.
- Blank handling differs by file. Inpatient: a blank Part A deductible is treated as $0 in the total (an interpretation, with the sensitivity result above). Outpatient: no blanks in the amount columns. Carrier: blank line-level amounts are treated as zero (unused service-line slots).
- Carrier was collapsed to one row per claim and is not reconciled (CMS's carrier formulas depend on a processing indicator dropped in cleaning).
- Tool note: DuckDB read the cleaned CSVs' literal `NA` as text. Use `nullstr = 'NA'` when creating views, or `TRY_CAST`.
