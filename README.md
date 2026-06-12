# How this project fits together

This is the full analysis behind the country selection and power analysis
for a Phase 2 trial in Relapsing Multiple Sclerosis (RMS, N=125, 2:2:1 high
dose / low dose / placebo).

There are three parts:

- **`01_country_selection/`** (`master_analysis.py`) - pulls live data from
  ClinicalTrials.gov to check country-level signals: how many MS trials are
  already running in each country, how many use drugs that would make a
  patient ineligible (washout), and how much active RMS research is already
  happening there. Also runs its own smaller power and safety simulation as a
  cross-check on the numbers from `02_power_analysis`.
- **`02_power_analysis/`** (`run_final.py`) - the main analysis: statistical
  power, the country scorecard, retention/dropout, and subgroup checks.
  Everything goes into `MS_Strategy_Analysis.xlsx` (10 sheets).
- **`03_charts/`** (`build_charts.py`) - reads `MS_Strategy_Analysis.xlsx`
  and produces 4 charts summarizing the power analysis results.

For how to run each script (install steps, runtime, etc.), see the README
inside each folder.

The two analyses don't read each other's output files. They look at related,
but distinct, questions about the same trial design.

## Where the CSVs come from

### `01_country_selection/data/`

| File | What it is | Where it's used |
|---|---|---|
| `01_dmt_trial_level_extract.csv` | Every MS trial found for the 8 drug classes that would make a patient ineligible (washout rules), one row per trial | Backs up `02_washout_proxy_index_by_country.csv` |
| `02_washout_proxy_index_by_country.csv` | Count of those trials by country - a rough stand-in for "how many patients here have probably already been on one of these drugs" | Supporting evidence that Poland and Czech Republic have lower prior DMT exposure than other countries |
| `02_live_trial_level_extract.csv` | Phase 2/3 MS trials by recruiting status, with countries and enrollment numbers | Background context |
| `05_active_rms_research_concentration_by_country.csv` | Trial counts by country, split into relapsing/active SPMS vs. progressive only vs. unclear | Supports the "low competition from other trials" point for Argentina |
| `01_simulated_patient_visit_data_sample.csv` | One example simulated dataset: 125 patients, 6 visits | Example output only, nothing else reads it |
| `02_power_simulation_results.csv` | A second power check on the same two efficacy scenarios as `run_final.py`, using the Wk 0-12 window | Cross-check against `MS_Strategy_Analysis.xlsx`, the numbers line up |
| `03_safety_simulation_results.csv` | How often the side effect check fires, under "no extra risk" vs. "somewhat more risk" | Source of the safety detection rate numbers for those two scenarios |

### `02_power_analysis/data/`

| File | What it is | Read by `run_final.py`? |
|---|---|---|
| `country_selection_scorecard.csv` | Hand-built table (prevalence, trial competition, washout risk, cost, etc per country), based on public sources | Yes, main input for the country scorecard |
| `endpoint_anchoring_comparison.csv` | Results from 4 published oral drug trials on the same lesion count endpoint this trial uses | Yes, feeds the efficacy scenarios and the `Literature_Anchor` sheet |
| `live_trial_counts_by_country.csv`, `live_trial_level_extract.csv` | `run_final.py`'s own live ClinicalTrials.gov query (separate from the one in `01_country_selection`) | Written, then read again by `run_final.py` each run |
| `ms_prevalence_by_country.csv` | GBD 2016 + Atlas of MS 2020 prevalence numbers | No, kept so you can see where the scorecard's prevalence numbers came from, but the script doesn't load it |
| `power_analysis_reference_trials.csv` | The trials behind the efficacy assumptions (frexalimab, ocrelizumab, evobrutinib) | No, kept for traceability only |

## How the two country checks connect

`country_selection_scorecard.csv` (in `02_power_analysis`) is the table that
drives the country recommendation. The washout and active RMS files in
`01_country_selection` are an independent check on two of that table's
calls: how much prior DMT exposure a country probably has, and how much
relapsing MS trial activity is already happening there. They use live trial
counts from ClinicalTrials.gov instead of the scorecard, which is based on
published literature.

Each of those two files has a column naming the scorecard's tier for that
country, and the washout file adds a one-line note like "scorecard called LOW
prior DMT exposure / lower washout risk." Spain's trial count (45) comes out
close to Germany's (49), even though the scorecard rates Spain's prior DMT
exposure as "Moderate" and Germany's as "Higher".

## The two power simulations

Both `01_country_selection` and `02_power_analysis` run a Monte Carlo power
check for the same two efficacy scenarios (89%/79%, and the TEMSO-anchored
80%/57%). Both simulate the Wk 0-12 window with a balanced placebo baseline:
no arm differences before Week 12, since nobody has been dosed yet at that
point. With that, the two are broadly consistent. For the SC-injectable
anchor scenario (89%/79%), `01_country_selection` (150 draws) gets about
90.0% / 87.3% power and `MS_Strategy_Analysis.xlsx` (400 draws) gets
93.0% / 83.0%. For the TEMSO scenario, `01_country_selection` gets about
76.7% / 47.3% and the xlsx gets 75.2% / 48.5%. Both are computed on the Wk
0-12 placebo-controlled window, which is the only period the trial has a
placebo group to compare against.

One thing worth noting: the model (`count ~ arm + visit_week`) assumes a
constant arm effect across Week 0 and Week 12, but with a balanced baseline
the real effect is zero at Week 0 and only appears at Week 12. This means the
model likely understates the true effect (a cross-check on the 89%/79%
scenario recovered only about 56.5% of the assumed 89% effect). A model that
isolates the Week 12 comparison, for example by adjusting for each patient's
Week 0 count, would probably show higher power for the same N, meaning even
the SC-injectable scenario's numbers above could be understated.

The side effect check (`03_safety_simulation_results.csv`) uses all 6 visits
over 104 weeks, since there's no Wk 0-12-only safety analysis in
`run_final.py` to compare it to.

## A couple of naming things

- **"Czechia" vs "Czech Republic" vs "Czech Rep."**: same country, written
  differently across files. `01_country_selection` uses ClinicalTrials.gov's
  "Czechia", the scorecard uses "Czech Republic", and `run_final.py` shortens
  it to "Czech Rep." to match its own lookups.

- **"Tier 1/2/3"** is the country role used in the country scorecard: Tier 1
  is the primary site (Poland, Czech Republic), Tier 2 is the anchor site
  (Germany, Spain), Tier 3 is the diversity site (Argentina), and Reference is
  the United States (not part of the plan). This is unrelated to the
  retention analysis below.

## The retention analysis

`Retention_Curve_Anchors` and `Retention_Dropout_Sensitivity` anchor patient
retention to the same two literature trials already used for the efficacy
scenarios:
- Frexalimab (NEJM 2024, NCT04879628): 97% retained at Wk12, 87% at Wk48, 82%
  at Wk96, reported values. Wk24/Wk76/Wk104 are filled in with a straight
  line between and after the known points.
- TEMSO (NEJM 2011): 73% retained at Wk108 (treated as Wk104), the only
  reported point, so the rest of the curve assumes people drop out at the
  same steady rate the whole time.

Both anchors imply almost the same retention at Wk12 (97% vs about 96%). The
gap between them only opens up later, reaching 81% vs 73% by Wk104.
`Retention_Dropout_Sensitivity` re-runs the Wk0-12 power simulation for the
Frexalimab-anchor efficacy scenario (89%/79%) with about 97% Wk12 retention
(effective N around 121) instead of the 100% assumed in
`Primary_Window_Power`, and finds it doesn't change the conclusion: power
moves from 92.0%/81.5% to 89.8%/81.8%, both still comfortably above 80%.

## Sources

Links to the published papers and registries behind the prevalence numbers,
efficacy assumptions, and diversity context used across this analysis.

| Source | Link |
|---|---|
| GBD 2016 (Lancet Neurology 2019) - global MS prevalence estimates | https://doi.org/10.1016/S1474-4422(18)30443-5 |
| He et al. 2023, JAMA Network Open | https://doi.org/10.1001/jamanetworkopen.2023.34675 |
| ClinicalTrials.gov | https://clinicaltrials.gov |
| Marrie et al. 2023, Multiple Sclerosis Journal - patient involvement in trial design | https://doi.org/10.1177/13524585231189678 |
| Onuorah et al. 2022, Neurology - non-white participant enrollment in MS phase 3 trials | https://doi.org/10.1212/WNL.0000000000013230 |
| RelevarEM (Argentina MS registry) | https://pubmed.ncbi.nlm.nih.gov/31128521/ |
| Vermersch et al. 2024, NEJM - frexalimab Phase 2 (NCT04879628) | https://doi.org/10.1056/NEJMoa2309439 |
| Kappos et al. 2011, Lancet - ocrelizumab Phase 2 | https://doi.org/10.1016/S0140-6736(11)61649-8 |
| O'Connor et al. 2011, NEJM - TEMSO trial | https://doi.org/10.1056/NEJMoa1014656 |
