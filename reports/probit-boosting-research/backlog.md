---
title: Research backlog
kind: backlog
status: living-document
created: 2026-08-25
created_by: docs/plans/archive/2026-08-24-repository-cleanup.md
---

# Research backlog

Open work, and the reason each item is still open. Assembled from the unresolved items in
[findings.md](findings.md) and from the deviations recorded in the archived plans.

> **A note on this file's provenance.** The cleanup plan named `RESEARCH_MEMORY.md` as the
> source for this backlog. No such file was present when the backlog was assembled, so it was
> built from the open items in the findings log instead, and that deviation was recorded here.
>
> It has since turned up: the file was added to the repository root by a later commit and moved
> to [memory.md](memory.md) on 2026-08-31. Reading it did not change this document. The two
> answer different questions -- this one lists work that is open and says why, while the memory
> holds hypotheses and external data-source leads that are not open work yet -- so it is indexed
> beside this file rather than merged into it.

## Blocking any profitability claim

**A prospective shadow test.** Every figure in this repository is retrospective: result-page
odds are final odds, not timestamped executable prices, and every test year through 2026 has
also been development data. Nothing here can be fixed by better analysis of the same data. The
step is a locked model run forward against archived pre-race price snapshots, and until it
exists the correct reading of every bankroll path is "diagnostic".

*Where it stands:* the pre-race snapshot capture exists in `hkjc.collection.live` but no locked
model has been run against it.

## Model research

**Recency-weighted training.** Truncating training history hurt badly overall while helping
2024-2026 -- last_3 beat full history by +7.9pp in 2024 and +21pp in 2025, and lost far more
across 2015-2021. The untested alternative is to weight rather than truncate:
`w = exp(-lambda * years_ago)`. This is the most direct open lead in the findings log.

**The two-factor probit variant.** Implemented and tested, never run canonically: at about 21
minutes per run there was no reason to spend it while the one-factor grid was flat. The
one-factor grid selected "no factor" in 7 of 11 years and the whole grid from scale 0.00 to
0.80 spans about 2e-5 of log loss, so the improvement is attributable to the normal
idiosyncratic distribution rather than to correlated pace shocks. Worth running only if a
reason appears to expect structure the one-factor search could not see.

**Three unmet probit promotion gates.** Gates 7 (an interaction diagnostic across predeclared
cohorts), 9 (reproduction from a clean identified commit) and 10 (prospective confirmation) are
not derivable from artifacts and are not met. Each published probit report says so in its
appendix. Production remains the five-seed Offset ensemble.

**Takeout-scale calibrated residuals.** The margin-memory work produced material but never
reached takeout-scale calibrated residuals, which is the bar it would have had to clear to
matter for staking.

## Staking

**Rebate-aware sizing solves before it rounds.** The rebate-aware rule solves a portfolio per
race against a reward function that includes the rebate, and the rebate is a cliff: 10% of the
race loss once that loss reaches HK$10,000. So the optimiser can add a runner whose own edge is
negative because it pushes the race's turnover over the threshold, which is correct. But stakes
are rounded to the nearest HK$10 *after* the solve, so a portfolio solved at the threshold can
round down below it. Both cases are in the smoke ledger: one race lands on HK$10,000 and collects
HK$1,000, another lands on HK$9,990 and collects nothing -- while both include a runner taken
only for the rebate.

Found in Phase 9 of the 2026-08-24 cleanup, when the settlement began covering Kelly fractions
above 0.01 and the cliff became reachable at all. Not fixed there: changing the solve is a
modelling change and the cleanup's rule was not to mix those in. The open question is whether to
solve against rounded stakes, add a rounding margin to the threshold, or drop the
rebate-motivated runner when the rounded turnover misses. A test pins the current behaviour so it
cannot change silently.

## Measurement and infrastructure

**The boosted probit costs 17x what the repository says it costs, and nobody knows why.** Measured on
2026-09-04 over the full sixty-cell sweep, from
`probit_offset_boosted_training_diagnostics.parquet`: per-cell fit time rises monotonically with
training history from **253s at test year 2015 to 943s at 2026**, mean 586s, totalling **35,165s --
9.8 CPU-hours**, or about 98 minutes at the six-wide fan-out. What `boosted_utilities` and the guide
both claimed was "cell cost plateaus around 137-184s" and a whole sweep of **2,023s**, i.e. ~34s per
cell.

The structural half of that measurement reproduced exactly -- `objective_share_of_fit` came in at
0.989-0.992 against a claimed 98.5-99.4%, so the integrator is still the bottleneck and the
fan-out reasoning still holds. Only the magnitudes moved, and the obvious candidates do not explain
it: `rounds` has been 400 and `factor_nodes` 32 since the architecture was integrated in `a934d63`,
the feature count went 28 to 33 which cannot cost 17x in an objective that does not touch columns,
and the learning-rate tuning in `1b511eb` changes the step size rather than the per-round work.

The most likely explanation, and it is a guess: the old figure was taken over a restricted
`--years`/`--seeds` subset and written up as a full sweep. `2,023s / 60 = 34s` per cell, which matches
nothing measured -- the *standalone* mean-learner cells are ~5s each (observed 4.1-5.5s on
2026-09-04, recorded as `~3.4s` in `probit_widths`) and the boosted cells are 253-943s -- so it is
unlikely to be a mislabelled measurement of the other stage. Check it before concluding a performance
regression: one cell at the recorded configuration, timed, decides it.

Until it is settled, treat the documented figures as the measured ones and budget an hour and a half
for a cold boosted sweep. The cost of getting this wrong is a plan, not a number: it is what made a
2h sweep get scheduled as a 30-minute one on 2026-09-04.

**The prediction cell cache is not keyed on the feature set.** `_cell_stem` in
`hkjc/workflows/stages/predictions.py` names a checkpoint `y{test_year}__seed{seed}` and
`cell_complete` resumes any cell whose `__done.json` marker exists. Neither reads the feature-set
digest. So re-running a root after `production.json` changes **resumes the previous feature set's
fits and writes them out under the new set's identity** -- fast, silent, and wrong. Found on
2026-09-04 during the gear promotion, where a 34-column run reused 28-column cells; the caches were
cleared by hand and the generation re-fitted from cold.

`refuse_mismatched_reuse` does not cover this. It compares the stage's *declared inputs* against the
recorded lock and never looks inside the cell directory, so the guard that exists for exactly this
class of mistake does not reach the place the mistake happens. The reason it went unnoticed is that
until a feature set changed, `(test_year, seed)` was a complete key.

**The boosted probit has a second, independent cache with the same defect, and it is worse.**
`boosted_cells` under `results/runs/probit_offset_boosted/` is written by `boosted_utilities` in
`hkjc/workflows/stages/probit.py`, which reuses a cell on `cell_{test_year}_{seed}.parquet` merely
*existing*. The companion `.json` records `fit`, `history` and `diagnostics` and **no feature list at
all**, so unlike the prediction cells there is not even a stale digest to compare -- a cached cell is
not self-describing and its provenance cannot be recovered from the file. It is worse than the
prediction cache in a second way: this architecture fits its own mean function, so a reused cell is a
whole booster trained on the wrong columns rather than a cached scoring pass.

This is the defect that actually bit. On 2026-09-04 the gear promotion cleared the two softmax roots'
caches but not this one, and the interrupted refit left 48 of 60 cells behind. Refitting one of them
(`2026`, seed 7) with the current 33-column set reproduced `market_prob` exactly and `raw_margin` **not
at all** -- a maximum absolute difference of 0.503 on a margin -- so the cached cells were provably not
the current feature set's. They were moved aside and the generation re-fitted from cold. Note what
made this *findable*: the fit is otherwise deterministic, so a one-cell refit-and-compare is a cheap
audit for any cache whose contents are in doubt, and it is the only check available here.

**There is a third, and the pattern is the finding.** `betting_cells` in
`hkjc/workflows/stages/betting.py` resumes on `_cell_stem(scenario, view, seed)` plus a
`__done.json` marker, with no reference to the predictions it settled. It did no damage on
2026-09-04 only because the canonical softmax roots happened to hold no leftover cell directory --
the smoke root did, and it was cleared by hand. So three independent caches across three stages
each key on the loop variables and none on the inputs, which means this is a *convention* the
repository has drifted into rather than three separate oversights, and it should be fixed as one.

*The fix:* put the feature-set digest (and arguably the model-spec digest) in the cell stem or in the
`__done.json` marker, and have `cell_complete` refuse a marker whose digest does not match the run's.
Do the same for `boosted_cells`, where the companion JSON is the obvious place to record the ordered
columns it was fitted on, and for `betting_cells`, which should record the digest of the
`predictions.parquet` it settled -- a cache entry that cannot say what it is cannot be validated, only
distrusted. That makes a feature-set change invalidate the cache automatically instead of relying on
whoever runs it to remember. Worth doing before the next promotion, because the failure mode is a
published number that is wrong rather than a run that stops.

**Seed-set luck.** The production bag (`7, 17, 42, 123, 256`) was the luckiest of five
independent 5-seed meta-sets: across the five, mean cumulative ROI at Kelly 0.02 was +107% with
a standard deviation of 130% and a minimum of +24%. The published reports are single-bag
numbers. A distribution over meta-sets would be the honest headline, and it costs five canonical
runs to produce.

**The betting activity floor.** The model places 339 bets in twelve years at Kelly 0.01, and
activity falls by roughly two thirds from the early years to the late ones as the training
window grows. Whether that is the market getting harder, the model getting more conservative, or
the `min_overround` filter binding more often has not been separated.

**Edge-bucket support.** The expected-versus-realised check in each report's betting section
rests on buckets holding as few as nine bets, where one winner moves the bucket. The check is
the right one and the sample is too small to conclude from; more years or a pooled-across-models
version would help.

**Replacing `gear_change` rather than adding beside it.** Production's `gear_change` compares raw gear
strings, and measured on the shipped table it fires on 0.2654 of starts while the normalised equipment
set changes on 0.1467 -- so **44.73% of its fires are HKJC's first-time marker decaying** (`B1`
becoming `B`) with nothing on the horse changing. It misses 0.0000 of real changes, so it is a noisy
superset rather than a wrong signal. `gear_true_change` is the same event without the false half.

*Where it stands:* the addition was measured and rejected (2026-09-02, `-0.0001370` equal-cell over 55
cells, above the `-0.0005` bar), and that is *not* the same question as the replacement. `feature-iterate`
appends a block to the incumbent and cannot substitute a column, so the cleanest form of the fix is not
expressible as a candidate. What it would cost: a variant feature-set config (`production_gear_fixed.json`,
28 columns with `gear_change` redefined) and a way for the harness to name an incumbent other than
`production.json`. That is a change to what "the incumbent" means and needs its own decision. The
addition result bears on it: a corrected bit that adds almost nothing on top of the defective one
suggests the replacement is a **cleanup** -- worth doing for the definition's honesty, not for the log
loss -- and it should be argued on those terms rather than as a modelling improvement. The semantics are
now documented where the column is defined, which was the other half of the finding.

**A `race-stewards-reports` dataset for the prose era, 2003-2023.** `race-incidents` now begins in
2024, because that is when the per-runner incident *table* begins: verified against the live site on
2026-09-02, `?Date=2024/01/01` serves ten incident tables and 124 runners while 2023/09/10,
2023/01/01, 2022, 2021, 2020 and 2019 serve real pages with **zero** tables. The 24 year files that
used to cover 2000-2023 were the endpoint's fail-open default meeting relabelled, and they are gone.

What the earlier pages *do* carry is the same channel in a different shape: prose stewards' reports
keyed by race, holding exactly the official-intervention statements this programme went looking for
-- *"Race 10: SUPER FORM on 31.12.18 on veterinary advice"*, jockey censures, per-race veterinary
findings. That is a meeting-level or mention-level schema, not a runner-level one, so it cannot be
backfilled into `race-incidents` and needs its own parser, contract and key.

*Where it stands:* recommended and out of scope for the 2026-09-02 repair, which deliberately did
not parse prose into a runner-level schema. It is the only place the official-intervention channel
exists before 2024, and the production model trains from 2010 -- so a runner-level intervention
feature is confined to the last three seasons until this exists. Note the shape of the gap before
building it: `gear_first_n`'s own measured value was confined to 2023-2026 for a related reason
(declaration coverage rising from 0.74 to 0.87), and a recent-only channel is a different finding
from a stationary one.

**Feature attribution for the two probit architectures that could have it.** Neither the standalone
`probit` nor `probit_offset_boosted` publishes any per-feature evidence, and the two gaps are one
producer change. The standalone link fits its own utilities and `standalone_utilities` persists
their raw margins *without* SHAP; the boosted architecture fits its own booster under the probit
likelihood and `BoostedProbitModel.predict_contrib` returns SHAP on exactly the correction it
learns, and the fitting stage does not persist it. In both cases the only attribution available
today would be a separately fitted Softmax run's, which is another run's work under this model's
heading and is refused. What it costs: the probit fitting stage persisting `correction_shap` and
`feature_values` for its own sixty cells per architecture, after which the existing transforms
(`feature_effects.build_feature_effects_yearly` and `build_feature_stability`) produce the yearly
response curves and the whole stability table -- including the direction-consistency fields
(`sign_agreement`, `shape_correlation`, `monotone_share`, `seed_variation`) -- with no new
mathematics, because those transforms are architecture-agnostic reducers over a SHAP frame.

*Where it stands:* refused deliberately rather than overlooked. `probit_offset` renders the Offset
run's attribution instead, gated by `probit_diagnostics.correction_identity`, which proves the
inherited frame describes the exact correction the link consumed -- 535,295 rows matched, margins
agreeing to 0.0. That gate is the reason the other two get nothing rather than the nearest
available thing. Leave-one-feature-out through the link is separately ruled out on cost: it needs a
quadrature pass per feature per runner, which is 28 features over 97,636 runners.

## Closed during the 2026-08-24 cleanup

Kept here briefly because each was an open question that now has an answer, and the answer is
easy to lose:

* **Does a rebate-aware Kelly objective find more bets?** No -- at the production bankroll the
  stakes never reach the HK$10,000 cliff, and at larger banks the rebate-aware rule loses more.
* **Do the refactors change any number?** No. A full rebuild from the cleaned tree reproduced
  all 694 pinned per-column digests across 19 artifacts, including a 12.2M-row prediction table.
* **Is the probit utility scale identified by theory?** No. The fitted values run 1.99 to 2.23
  against a theoretical 0.78, and at the theoretical value the market-relative candidate loses
  instead of winning.
