# Glydways Sutton corridor — stated-preference survey package

Imperial College Business School MBA capstone with Glydways.
The Glydways service in this survey is **hypothetical**. It is not approved, funded or planned. Keep that wording in every public material.

Trip studied: Sutton town centre → South Wimbledon station, weekday ~08:00, staying ~2 hours.
Alternatives (fixed order and codes): 1 `personal_prt`, 2 `shared_prt`, 3 `public_transport`, 4 `taxi_tnc`, 5 `car`.
Game B (station boarding): 1 `board_now`, 2 `wait_next`, 3 `upgrade`.

## Status

| Stage | Status |
|---|---|
| 1 Journey assumptions | Defaults applied. Several values are **PROVISIONAL** (see `config/study_config.json`, field `status`) and must be confirmed before the pilot |
| 2 Experimental design | Done. Bayesian D-efficient, 40 tasks in 4 blocks of 10 + 1 holdout per block; Game B 16 tasks, 4 per block |
| 3 Pilot survey | Built and tested (random data, synthetic data, mobile UI). **Real pilot (40–60 people) still to run** |
| 4 Final survey | Generated, but regenerate and freeze only after the pilot review |
| 5 Analysis | Scripts proven on synthetic data (parameter recovery + 50-run Monte Carlo) |

## Folder map

```
config/study_config.json      baselines, sources, levels, priors, Game B settings, design settings
design/generate_design.py     design generator  -> design_gameA.csv, design_gameB.csv, design.json, design_validation.md
survey/template.html          survey source (single page app)
survey/build_survey.py        builds survey/pilot/, survey/final/ and collector/Code_deploy.gs
survey/img/                   card and explainer images
collector/Code.gs             Google Apps Script collector (source)
collector/Code_deploy.gs      same, with the design embedded  <- paste THIS into Apps Script
collector/run_local.js        runs the collector in Node against a JSONL file (for testing)
testing/                      random, synthetic, Monte Carlo and mobile UI tests
analysis/estimate_models.py   Biogeme estimation, WTP, pricing, holdout validation, report
docs/google_sheets_setup.md   step-by-step Google Sheets and Apps Script setup
data/test_random, data/test_synthetic   test outputs (NOT study data)
```

## Requirements

Python 3.11+, Node 18+ (testing only). `pip install -r requirements.txt`.
Pin `biogeme_optimization==0.0.12`: version 0.0.13 breaks Biogeme 3.3.3.

## 1. Changing journey assumptions

1. Edit `config/study_config.json` (baselines, levels, fares). Update each item's `status` and `source`.
2. Regenerate the design with the same seed:
   `python design/generate_design.py`
   Check `design/design_validation.md`: D-error must beat all random designs; check dominance counts, correlations and the Personal–Shared fare gap.
3. Rebuild the survey (step 3 below).
4. Rerun the tests (section 5).

If the TfL Journey Planner route turns out to be bus + bus (Hopper, £1.75), public transport becomes the cheapest option. Update `pt_*` baselines and levels and regenerate the design; the dominance checks will change.

## 2. Collector (Google Sheets + Apps Script)

Follow `docs/google_sheets_setup.md`. In short: one spreadsheet for the pilot and a separate one for the final survey, each with `Code_deploy.gs`, `setup()` run once, script property `ACCEPT_MODE` set to `pilot` or `final`, and deployed as a web app (Execute as: Me; Who has access: Anyone). Copy the two `/exec` URLs.

## 3. Build the survey

```
cd survey
python build_survey.py --pilot-endpoint https://script.google.com/macros/s/PILOT_ID/exec \
                       --final-endpoint https://script.google.com/macros/s/FINAL_ID/exec \
                       --contact team-email@imperial.ac.uk
```
Add `--no-gameb` to drop Game B. The final build has all test code stripped; the build fails if any is left.
Every rebuild re-writes `collector/Code_deploy.gs`. If the design changed, paste the new file into both Apps Script projects and create a **new deployment version** (Deploy → Manage deployments → Edit → New version) so the URL stays the same.

## 4. Host the survey

The survey is static: `index.html` plus `img/`.

**GitHub Pages:** create a repository, upload `survey/pilot/` as `/pilot` and `survey/final/` as `/survey` (any folder names), then Settings → Pages → Deploy from branch → `main` / root. Links become `https://<user>.github.io/<repo>/pilot/`.
**Netlify:** drag the `survey` folder onto app.netlify.com/drop.

Before sharing any link, open it on a phone and check: the route map and images load (upload the whole `img` folder), and a test submission reaches the Sheet.

Test mode: add `?test=1` to the **pilot** link to show Autofill and Reset buttons. Submissions made this way are marked `is_test = 1`. The final build ignores `?test=1`.

## 5. Tests

```
python testing/generate_random_responses.py && node collector/run_local.js testing/out/random_payloads.jsonl data/test_random pilot && python testing/check_random_pipeline.py
python testing/generate_synthetic_mnl.py && node collector/run_local.js testing/out/synthetic_payloads.jsonl data/test_synthetic pilot
python analysis/estimate_models.py --data data/test_synthetic --out analysis/output_synthetic --truth testing/out/synthetic_truth.json
python testing/monte_carlo_recovery.py
python testing/ui_test.py          # needs: playwright install chromium (or set CHROMIUM_PATH)
python survey/make_route_map.py    # redraws the route map picture (only if the route or labels change)
node testing/gas_mock_test.js      # collector against stand-in Google services
```
Latest results (1.2.1): random pipeline 21/21; mobile UI 90/90 (pilot and final, 390×844); Apps Script stand-in test passes (attached and standalone); synthetic recovery 10/11 inside 95% CI (B_STAND just outside in that one sample; the 50-sample Monte Carlo shows no bias); Game B 5/5.
To push random test data into a live Sheet: `python testing/generate_random_responses.py --post <pilot /exec URL>`, then run **Glydways → Delete test rows** in the Sheet.

## 6. Pilot → final

1. Run **Delete test rows** in the pilot Sheet. Share the pilot link with 40–60 real people.
2. Review: `qc_flags` (especially comprehension and fast), the pilot feedback questions, choice shares (any alternative near 0% or 100%?), completion time, and the holdout predictions.
3. Estimate a pilot MNL: `python analysis/estimate_models.py --data <pilot exports> --out analysis/output_pilot`.
4. Optionally update the priors in `study_config.json` with the pilot estimates, regenerate the design, rebuild, and redeploy the final collector.
5. Freeze: record the design version hash (printed by `build_survey.py`) and do not change the final survey during fieldwork.

## 7. Exporting and analysing data

- In the Sheet: **Glydways → Export CSVs to Drive** writes `pilot_*.csv` or `final_*.csv` to the Drive folder `glydways_exports`. Download them into one folder.
- Or by link: `<exec URL>?action=export&tab=responses_wide&key=<EXPORT_KEY>&real=1` (`real=1` drops test rows).
- Analyse: `python analysis/estimate_models.py --data <folder> --out analysis/output_final [--drop-flags fast,comprehension]`.
  Report the model with and without flagged respondents. Flags never delete data.

Outputs: `report.md`, `parameters_*.csv`, `wtp.csv`, `pricing_scenarios.csv`, `holdout_validation.csv`.
Headline model is MNL. Nested Logit (nest PRT = Personal + Shared) becomes headline only if the nest parameter is sensible, stable and improves fit; the script applies and reports this rule. Game B is a separate MNL.

## Caveats to state in the report

- Stated-preference shares usually overstate demand for a new mode. Treat pricing outputs as indicative; capacity and operating costs are not modelled.
- Standard errors are robust but not clustered by respondent.
- The sample is a convenience sample. Compare age, income and car access with Sutton/Merton census figures and describe the gaps.
- Taxi is rarely chosen, so its constant is weakly identified.

## Change log

**1.2.1** (survey page only; same design 8658c41be3, so no Apps Script change is needed for the survey to work)
- Route map: now a fixed picture, `survey/img/route_map.png`, drawn by `survey/make_route_map.py` from open data kept in `survey/mapdata/` (real National Rail, Northern line and tram lines and stations; Sutton and Merton boundaries). No map servers are used: OpenStreetMap blocked the survey's tile requests under its usage policy, and CARTO needs an API key. Tap the map to enlarge it. The drawn diagram appears only if the picture fails to load.
- Sending: a pop-up asks "Send your answers?" before anything is sent ("Yes, send my answers" / "Go back and check"). A retry after a failed send does not ask again.
- Thank-you page: "Start a new response" (after a confirmation) lets someone else on the same device take part, or lets you test again. The new response carries `previous_ref`; the collector notes it in `qc_flags` so repeat responses from one device are easy to review.
- Choice tables: the selection circle has its own column on the left, clear of the pictures; lines separate walk, wait and travel; header shortened to "Time (minutes)".
- Collector: only the `previous_ref` note is new. Paste the new `Code_deploy.gs` and deploy a new version when convenient.

**1.2.0** (new experimental design, hash 8658c41be3; paste the new `Code_deploy.gs` and deploy a new version)
- Design logic: Personal and Shared pods now always have the same walking time and the same travelling time (same station, same guideway, no stops). Shared waiting time is never shorter than Personal (levels 3/5/8 vs 1/3/5). Only waiting, price, sharing and boarding method differ between the two pods.
- Pod capacity: 4 seats + 2 standing. Shared pod levels are now 1 to 5 other passengers; with 4 or 5 others the table shows "4 seated + 1 standing" / "4 seated + 2 standing" and the model has a new standing term `B_STAND` (people standing = max(0, others + 1 − 4)). Capacity is PROVISIONAL: confirm with Forrest.
- Design: D-error 0.0976, beats all 200 random designs (19% below the median); all levels balanced; B_STAND expected t ≈ 2.9 at n = 300. Monte Carlo (50 × 300): no bias on any parameter; B_STAND mean −0.315 vs true −0.30.
- Map: CARTO tiles now need an API key (they showed "API KEY REQUIRED"). The live map now uses OpenStreetMap, then Esri street tiles, then the drawn route diagram. `build_survey.py --static-map picture.png` shows a fixed picture instead (for example a screenshot of the live map).
- Group question replaced: fixed group of 3, Personal pod £5 each, 1 other passenger joins the Shared pod, all seated; asks the highest price each at which the group would share (`group3_share`). The 1.1 question `party_share` stays in the sheet for answers already collected.
- Collector: new `standing` column in `responses_long`; a submission made from an older copy of the survey (different design version) keeps its raw answers and respondent row but writes no task rows and is flagged `flag_invalid`. The 1.1 design is kept in `design/archive/` so such answers can be decoded.
- Answers collected with survey 1.1 remain valid: every row stores the attributes the person saw. B_STAND is estimated only when the data include standing tasks.
- Python: Biogeme 3.3.3 needs Python 3.12 or newer.

**1.1.0** (survey only; the experimental design is unchanged, hash 414faa8889)
- Consent page: removed the reference-number line and the "imaginary" note. The explanation screen still says the service is imaginary, and the thank-you page still gives the reference number and contact email.
- Order: all questions about the respondent come first (eligibility, travel, household, about you), then the explanation, check question, trip, Game A and Game B.
- New questions: `car_own` (household cars or vans), `group_size` (usual group on local trips), `party_share` (would the group take a Shared pod, and for what saving; asked after the games so it does not steer the choices).
- Explanation: new artist's-impression image; pods take up to 6 passengers; Shared pod "1 to 5 other passengers"; both sharing concepts described as shared rides.
- Games: stacked cards replaced by a comparison table, one row per option. Columns: option, total time, price, sharing, then walk, wait and travel.
- Collector: the three new answers are added at the END of `respondents`, `responses_wide` and `responses_gameB`. Paste the new `Code_deploy.gs`, run `setup` once (it adds the new column headings to an existing sheet), and deploy a new version.
- Note: the explanation says a Shared pod can carry 1 to 5 others, but the design only tests 1 to 3. The models cannot say anything about 4 or 5 other passengers unless `shar_occ` levels are widened and the design regenerated.
