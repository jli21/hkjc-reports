# Boosted multinomial probit — research log

Branch `research/boosted-probit`. One entry per experiment, in the order it happened, with
hypothesis, configuration, runtime, result and interpretation. Failures are entries, not
omissions, and nothing is rewritten when a later result supersedes it — a correction is a new
entry that names the old one.

Kept here rather than as a top-level `RESEARCH_LOG.md` because `docs/research/` is where this
repository keeps `findings.md`, `backlog.md` and `memory.md`, and the plan asks for the
project's own conventions over its own suggested layout.

**The question.** Does training a nonlinear tree ensemble *directly* under the multinomial
probit race likelihood improve on (a) the existing link-only `probit`, (b) the existing
link-only `probit_offset`, and (c) the production market-anchored softmax booster?

**The gap this fills, in the repository's own words.** `docs/research/findings.md`, Phase 6:
"Both results are **link-only**. The mean learner is the existing softmax or Offset fit; no
native joint probit likelihood was trained, and metadata records
`mean_training=softmax_surrogate` for that reason."

---

## Phase 0 — reproduce the baselines before implementing anything

**Hypothesis.** The recorded probit results are reproducible from the committed artifacts. If
they are not, nothing downstream is interpretable.

**Method.** Read `results/runs/probit_offset/probit_offset_race_scores.parquet`, restrict to
`is_honest_scale` rows (which excludes 2015, the year with no earlier season to fit a scale
on), average per race then per seed then per year.

**Result — reproduced exactly.**

| Quantity | Reproduced | `findings.md` |
|---|---:|---:|
| market race log loss | 2.028805 | — |
| Offset (softmax link on its margins) | 2.025390 | — |
| `probit_offset` | 2.024447 | — |
| Offset − market | −0.003415 | — |
| `probit_offset` − market | −0.004358 | — |
| **`probit_offset` − Offset** | **−0.000943** | **−0.000943** |

Per-year, the Offset delta against the market runs from −0.000593 (2022) to −0.008154 (2024),
so 2024 is roughly twice as favourable a year as the average and a single-year result on it
would overstate.

**Interpretation.** The paired probit-offset effect is −0.000943 and the production model's
entire edge over the market is −0.003415 on this measurement (−0.004647 in `findings.md`,
which is the *calibrated* production figure rather than the raw-softmax-on-margins one). Those
two numbers are the scale everything below is judged against.

---

## E1 — analytic gradients of the anchored likelihood

**Hypothesis.** The existing quadrature integrator can be extended to supply exact gradients
without a new probability engine (§5, §47.1).

**What the plan asked for, and why it is not what was implemented.** §12 derives
`d(-log P_w)/dmu` for the *unanchored* probit: only the winner's probability appears, so one
row of the Jacobian suffices. This repository's probit is anchored, and its prediction is
`p = normalize(q · P(mu) / P(log q))`. Writing `s_j = log q_j + log P_j(mu) − log P_j(log q)`
gives `p = softmax(s)`, hence `dL/ds_j = p_j − y_j` — the production objective's own residual —
and

    dL/dmu_i = sum_j (p_j − y_j) · d log P_j / d mu_i

which needs the **full n×n Jacobian**. Training against §12's single-row gradient would
optimise a different objective from the one scored.

**The Jacobian reuses what the probability already forms.** With `a_jk = (mu_j − mu_k)/s + z`
and `lambda = phi/Phi`, `dP_j/dmu_j = +(1/s) E_z[prod · sum_k lambda]` and
`dP_j/dmu_i = −(1/s) E_z[prod · lambda(a_ji)]`. `lambda` is the same `(races, nodes, n, n)`
shape as the log-CDF array, so the gradient costs about one more probability evaluation rather
than `n` of them. Only the vector-Jacobian product is formed.

**Validation (§13, mandatory).**

| Check | Result |
|---|---|
| probabilities vs `quadrature_probit_probabilities` | **0.0** — bit-identical |
| anchor identity at zero correction | **1.1e-16** |
| gradient vs central difference, worst over 12 field-size × regime cells × 4 steps | **5.6e-9** relative |
| curvature nonnegative | yes, by construction |

Best step size 1e-4; 1e-6 degrades to ~1e-9 as expected from cancellation.

**One bug, recorded because it looked numerical and was not.** The first version failed its own
check at `0.800000000006817`. That is 4/5, not noise: the analytic gradient differentiates the
race-loss **sum** (the convention `softmax_obj` uses, since XGBoost sums rows) while the
finite-difference helper differenced the **mean**, and the fixture had five races. Both losses
are now returned under separate names.

---

## E2 — §24: how many quadrature nodes does the gradient need?

**Method.** Real 2024 training slice, 130,629 rows / 10,496 races, against a 128-node
reference.

| nodes | max abs prob error | max rel gradient error | race log-loss error | seconds |
|---:|---:|---:|---:|---:|
| 8 | 3.2e-03 | 1.4e-01 | 2.4e-05 | 0.16 |
| 16 | 1.8e-04 | 1.2e-02 | 3.0e-06 | 0.27 |
| 24 | 1.6e-05 | 1.3e-03 | 2.4e-07 | 0.39 |
| **32** | **2.2e-06** | **1.5e-04** | **2.5e-08** | **0.51** |
| 48 | 3.2e-08 | 3.3e-06 | 6.7e-10 | 0.75 |
| 64 | 6.5e-10 | 8.5e-08 | 1.4e-12 | 0.97 |

**Decision: 32 for training, 64 for scoring.** At 32 the induced race-log-loss error is 2.5e-8
against an effect of 9.4e-4 — **38,000× smaller**. The stage already draws the same distinction
for its scale fit (`fit_nodes=32`).

---

## E3 — §23: profile before optimising

**Two bottlenecks found, both fixed.**

*Three integration passes per round became two.* The objective computed probabilities, then the
gradient VJP, then the Fisher curvature — three `log_ndtr` arrays. The gradient and curvature
cotangents now share one pass (the VJP accepts a stack), and the probabilities-only path calls
the integrator's own `_race_stack_probabilities` rather than building a `lambda` array it
discards. **62% of a round → 44%.**

*Threading.* The integrator releases the GIL in `log_ndtr` and already had a thread knob; the
gradient inherits it.

| threads | nodes | prob | grad | round | 400 rounds |
|---:|---:|---:|---:|---:|---:|
| 1 | 32 | 0.86s | 1.38s | 2.24s | ~15 min |
| 1 | 64 | 1.63s | 2.66s | 4.29s | ~29 min |
| **8** | **32** | **0.14s** | **0.22s** | **0.36s** | **~2 min** |
| 8 | 64 | 0.27s | 0.43s | 0.70s | ~5 min |

**Interpretation.** Direct probit training is computationally feasible (§47.3): ~10–16× from
threads alone, and a 400-round fit over a decade of races is two minutes. This is why cells are
fitted serially — each already saturates the machine, so fanning cells across processes would
divide the same cores twice.

---

## E4 — the plan's Option C for the utility scale, and why it fails

**Hypothesis (§18 Option C).** Alternate: boost, refit `sigma` on the training window,
continue.

**Result — unstable, and its instability is systematic rather than random.** Test year 2024,
seed 7, refitting every 64 rounds:

| rounds | fitted sigma | boosted probit − market |
|---:|---:|---:|
| 100 | 0.66 | **−0.016128** |
| 200 | 0.41 | **+0.004470** |

At 100 rounds it appeared to beat the market by 3.5× the production edge. At 200 rounds it lost
to the market. `sigma` was running to its lower bound.

**Why.** Refitting on the *training* loss selects a small scale, because the training correction
is overfit and dividing by a small scale sharpens it. Sweeping `sigma` on the softmax booster's
own margins shows the same thing from the other side:

| sigma | on 100-round margins | on 400-round margins |
|---:|---:|---:|
| 0.66 | −0.007820 | **+0.008432** |
| 1.00 | −0.007288 | −0.009001 |
| 1.50 | −0.005326 | **−0.010168** |
| 2.00 | −0.004144 | −0.008923 |

The test-optimal scale *rises* as the mean function grows (≈0.8 at 100 rounds, ≈1.5 at 400)
while the training-optimal scale *falls*. An alternation that cannot see held-out data chases
the wrong one, and its fixed point is a bound.

**Also tested — §10's recalibration control.** Amplifying the softmax correction by a factor
fitted on training data and scoring through the softmax link: the factor ran to the grid ceiling
(3.0) and bought only −0.000466 at 100 rounds. So the 100-round result was *not* explained by
rescaling — it was explained by being a lucky point on an unstable trajectory, which the
200-round row exposes.

**Action.** `scale_mode="fixed"` is now the default: trees train at a declared scale (2.0, inside
the 1.99–2.23 range the repository's own forward fits produce) and the *scoring* scale is fitted
forward per year by the stage — the identical procedure `probit_offset` gets. That is what keeps
the comparison a comparison of likelihoods rather than of scale-selection procedures.
`scale_mode="alternating"` is retained so the failure reproduces.

**A second instance of the same error, one level out.** A development sweep then fitted the
scoring scale on *the previous year's margins from the same model* — but that model was trained
on years including the previous one, so those margins are in-sample and overfit, and the scale
fit again ran to 0.3–0.45 and the model lost to the market by +0.006 to +0.018. The existing
stage avoids this by construction: its margins are out-of-sample per test year. The fix is to
produce a full out-of-sample utilities frame first and let the stage fit the scale, which is
what `boosted_utilities` does.

**The transferable lesson.** The utility scale is the single most dangerous parameter in this
architecture, exactly as `findings.md` warned ("the utility scale is the whole game, and theory
gets it wrong"), and it is dangerous in a specific way: *any* procedure that fits it on data the
mean function has already seen will drive it to a bound and produce a model that looks
spectacular at one round count and broken at the next.

---

## E5 — development run: all twelve years, one seed, honest forward scale

**Configuration.** `rounds=400`, `n_nodes=32` train / 64 score, `scale_mode="fixed"` at 2.0,
seed 7, hyperparameters identical to production Offset. Utilities produced out-of-sample per
test year, then scored through `stages/probit.run` — the same forward per-year scale fit,
nested chronological calibration and paired day-block interval the published `probit_offset`
report uses.

Runtime: 48s for the 2015 cell rising to ~155s for 2026 as the training window accumulates.

**Result — the honest reading needs care, because the stage's comparator changes meaning.**

| quantity | value |
|---|---:|
| `honest_delta` (year-weighted, 11 honest years) | **+0.0000113** |
| `honest_calibrated_delta` | −0.0000875 |
| 80% day-block interval | [−0.000252, +0.000347], **excludes zero: no** |
| honest years better | 5 of 11 |
| fitted scoring scale | 1.726 – 1.802 (no bound hit) |

The fitted scale is the first good news: 1.73–1.80, interior to the bounds and close to the
1.99–2.23 the link-only fits produce. The `scale_mode="fixed"` decision worked.

**But `mean_comparator_loss` here is not the incumbent.** The stage computes its comparator as
the *softmax link applied to the same batch*, and for this architecture the batch carries the
**boosted probit's own margins**. So `honest_delta = +0.0000113` answers "does the probit link
beat the softmax link on probit-trained margins" — not "does this beat the production model".
Reading it as the latter would have been the single most likely misreading of this whole
experiment, and it is the reason the ladder below is computed explicitly.

## E6 — the comparison ladder, paired per race on identical races

Seed 7, 11 honest years, 7,933 races, every model scored on the same race keys.

| | model | loss | Δ vs market |
|---|---|---:|---:|
| A | market | 2.028662 | — |
| B | production Offset (softmax booster) | 2.024705 | −0.003957 |
| C/D | link-only `probit_offset` | **2.023854** | **−0.004808** |
| E | softmax link on boosted-probit margins | 2.025836 | −0.002826 |
| F | boosted probit (directly trained) | 2.025885 | −0.002777 |

Paired deltas, negative meaning the candidate is better:

| comparison | Δ | se | t | races better |
|---|---:|---:|---:|---:|
| boosted probit vs production Offset | **+0.001181** | 0.001195 | +0.99 | 0.519 |
| boosted probit vs link-only `probit_offset` | **+0.002032** | 0.001149 | +1.77 | 0.550 |
| link-only `probit_offset` vs production Offset | −0.000851 | 0.000334 | −2.55 | 0.527 |
| probit link vs softmax link, *on boosted margins* | +0.000050 | 0.000242 | +0.21 | 0.543 |

**Two findings, and at this point I believed them.**

*The margins themselves are worse.* Row E against row B compares the **same link** on the two
sets of margins: 2.025836 against 2.024705. So the probit-trained mean function is worse by
+0.001131 regardless of which link scores it.

*And it destroys the link's advantage.* On softmax-trained margins the probit link is worth
−0.000851; on probit-trained margins it is worth +0.000050, i.e. nothing. That is
mechanistically coherent and it reframes the recorded −0.000943: the link-only gain is not
"normal shocks beat Gumbel ones", it is "the probit link partially repairs a mismatch between
softmax-trained utilities and the choice model it is scored under". Train under probit and
there is no mismatch left to repair.

**This was very nearly the final answer. It is wrong.**

## E7 — is the deficit the likelihood, or the optimiser?

**Hypothesis.** Before concluding that the probit objective learns a worse mean function, check
whether it is *optimising* comparably. Training loss under each objective on the identical
split, test year 2024:

| rounds | softmax-trained margins (softmax link / probit link) | probit-trained margins (probit link / softmax link) |
|---:|---|---|
| 100 | 2.01833 / 2.02082 | **2.01815** / **2.01663** |
| 200 | 2.00508 / 2.00944 | 2.00720 / **2.00492** |

The probit-trained ensemble reaches *lower training loss under both links*. So the gradient is
correct and the objective is being minimised — the model is **overfitting**, not underfitting.

**Why.** The gradient carries a factor 1/σ and the Gauss-Newton Hessian carries 1/σ², so the
leaf weight −G/(H+λ) scales like **σ**. At σ = 1.8–2.0 the effective step is about twice what
the same nominal learning rate gives the softmax objective. Measured directly: at 400 rounds the
softmax correction has sd 0.145 and the probit correction sd 0.194.

**So "identical hyperparameters" is not identical capacity.** §22 warns about exactly this and
I had reasoned it the wrong way round — I had matched the *nominal* parameters and assumed that
matched the capacity, when the objective's own parameterisation rescales the step. This is §19's
identifiability point with teeth: σ, the tree scale and the learning rate are not independent.

## E8 — §22's third requirement: best validated configuration for each objective

**Method.** Honest validation *inside* the training window: fit on years ≤ 2022, validate on
2023, for the 2024 cell. σ held fixed at 1.8 so only the step size varies. No test-year
information selects anything. The softmax booster is already at its own tuned production
configuration, so it needs no sweep.

| learning rate | validation Δ vs market | correction sd |
|---:|---:|---:|
| 0.000625 | −0.002661 | 0.043 |
| 0.00125 | −0.003628 | 0.068 |
| 0.002 | −0.004139 | 0.090 |
| 0.0025 | −0.004269 | 0.102 |
| **0.003** | **−0.004529** | 0.112 |
| 0.005 | −0.003703 | 0.143 |
| 0.0075 | −0.001193 | 0.173 |
| **0.01 (the nominal, production value)** | **+0.000437** | 0.194 |
| *softmax booster, softmax link* | *−0.003646* | *0.145* |
| *softmax booster, probit link* | *−0.004086* | *0.145* |

**The whole of E6 was a step-size artifact.** At the nominal learning rate the boosted probit
*loses to the market* on this validation year. At 0.002–0.003 it beats both the production
softmax booster (−0.003646) and the link-only probit (−0.004086) — and does so with a
**smaller** correction than the softmax booster's, which is the opposite of a capacity
advantage.

**Which learning rate goes to canonical, and why not the argmax.** 0.003 is the argmax on this
single validation year. I took **0.0025** instead. Selecting a hyperparameter at the argmax of
one year's curve is precisely the error this programme has documented repeatedly, the plateau
from 0.002 to 0.005 all beats the baselines, and 0.0025 sits inside it rather than on the peak.
The canonical claim is therefore made at a validated but deliberately non-argmax setting, which
can only understate it.

## E9 — canonical result, and the third near-miss

Canonical: 11 honest-scale test years x 5 seeds x 400 rounds at lr 0.0025, every arm scored on
the same race keys, deltas paired per race.

| Arm | race log loss | vs market |
|---|---|---|
| boosted probit | 2.023757 | -0.004905 |
| link-only `probit_offset` | 2.023818 | -0.004844 |
| softmax link on boosted margins | 2.024052 | -0.004610 |
| production Offset | 2.024662 | -0.004000 |
| market | 2.028662 | --- |

The boosted architecture is top of the ladder. **It does not follow that it works**, and the
reason took a third correction to see.

**The near-miss.** The first version of the report generator computed significance across the
five seeds and got the headline `boosted vs production = -0.000905, t = -16.03, 5/5 seeds`. I
would have published that. It is close to meaningless: five seeds score the *same races*, so five
deltas are five measurements of one sample, and their agreement measures how deterministic the
training is. Recomputed across **test years** -- the independent replicates:

| Comparison | mean delta | se (years) | t (years) | years favourable | t (seeds) |
|---|---|---|---|---|---|
| boosted vs production Offset | -0.001352 | 0.000968 | **-1.40** | 6/11 | -16.03 |
| boosted vs link-only probit | -0.000410 | 0.000900 | **-0.46** | 6/11 | -1.24 |

The same model, the same races, two notions of an independent observation, and a factor of
eleven in the *t*. Only the second answers "will this hold next season".

Compounding it, `recommendation_section` tested only the **sign** of the link-only comparison,
not its significance. Both faults together would have filed this run as *"promising, requires
another canonical confirmation"*. Both are fixed, `year_level` is now primary in `findings.json`,
and the seed-level figures are labelled as a stability measure rather than a significance one.

**The conclusion, which reversed.** Direct probit training does not improve on either baseline at
any defensible level. Almost the entire gain over the production booster belongs to the probit
**link**: link-only vs production is -0.000844 of the -0.001352 total, and it costs no fitting at
all -- it is a rescoring of utilities another stage already produced. Direct training adds
-0.000410 on top at t = -0.46.

The per-year spread is the tell: 2026 -0.0080, 2022 -0.0056, 2024 -0.0037 against 2016 +0.0023,
2025 +0.0027, 2018 +0.0022. A mean carried by three good years out of eleven.

**Recommendation: keep the softmax booster.** Filed under "Negative", not under "no meaningful
difference from post-hoc probit" -- though it is nearly that too. The mean function genuinely
does differ from the incumbent's: scored through the *identical* softmax link the boosted margins
beat the Offset margins by -0.000610 at t = -11.15 across seeds. The probit objective learns
something real. It just does not learn something that pays.

This is the nineteenth candidate this programme has carried to canonical scale across two
repositories, and the nineteenth not to confirm at t < -2.

## E10 — integration, and three guards that fired

Publishing through the repository's own machinery rather than beside it surfaced four defects in
my integration, each caught by an existing contract:

1. **The run lock refused to certify.** The boosted branch of `run_from_config` never wrote
   `probit_offset_boosted_utilities.parquet`, which `write_run_lock` declares as an input.
   `RunLockError: cannot lock a run whose inputs are absent` -- correct, and the fix is to
   persist it as the standalone architectures do. The two training-diagnostics frames now
   persist beside it; nothing else was recording them.
2. **A closed vocabulary rejected the model view.** `simulation_policies` validates `model_view`
   against `MODEL_VIEWS`, which did not admit `bagged_probit_offset_boosted`. The guard working:
   an unlisted view is what a typo looks like.
3. **Two hygiene contracts rejected my test module.** a separate `test_probit_gradients` module
   imported `pytest` (this suite is unittest-only) and pushed the module count to 21 against a
   15-20 target. Both say the same thing -- the tests belong in `tests/unit/test_probit.py` --
   so the 26 gradient checks are now `AnchoredProbitGradientTests` there, as subTests.
4. **Two pins asserted exactly four published models.** Resolved by publishing the fifth
   properly rather than by hiding it: `standard` status is correct for it (the vocabulary notes
   `experimental` cannot appear in this file at all, since every entry has a canonical report),
   and it joins the production sweep, because the invariant those pins encode is that every
   published report is rebuildable -- one the sweep skips goes stale silently. It dominates the
   sweep's runtime, being the only entry that fits a mean rather than laying a link over
   utilities; the per-cell cache makes a warm rebuild cheap.

Full suite: 1638 passed, 12 skipped, 8760 subtests.

**Artifacts.** `results/reports/probit_offset_boosted/report.html` through the same bundle and
sections as the other four; `docs/research/boosted-probit/report.pdf` for the derivation.



---

## E11 — consolidation onto `main`: three things this experiment left behind

Not an experiment. The branch was merged into `main` on 2026-09-01 and reviewed as a whole, and
three of its own loose ends are recorded here because two of them were published claims.

**1. A diagnostics module that never ran.** `hkjc.workflows.stages.boosted_probit_diagnostics`
was added at `59543f1` as a reader module for §§20, 32-35 and 43: depth ladder, correction
comparison, per-feature gain and split shares, root-to-leaf pair counts, longshot buckets,
calibration-with-sharpness, paired delta. It was never called. It had no `main()` despite a
docstring advertising itself as a runnable module with an `--output-dir` flag (the invocation is
not reproduced here, because a documentation test resolves every module a live command names),
it imported `argparse`, `json` and `sys` and used none of them, and the two report tools written
after it (`f17f2cb`, `3e4c309`) computed their own ladder and never imported it. **Not one number
in the PDF came from it.** Deleted; retrievable at `59543f1`. The same judgement, for the same
reason, as the two compiled kernels in `findings.md` that were correct and never wired in.

**2. Two published paragraphs credited it anyway.** `interactions.tex` opened by saying split
frequency, gain and pair counts "are computed by
`boosted_probit_diagnostics.interaction_diagnostics`" and then showed none of the three;
`longshots.tex` said the §33 buckets "are computed by … `longshot_table`" and showed no buckets.
Both were statements about code that existed rather than about evidence that was produced. Both
now say what was and was not run. §20's answer here rests on the depth ablation, which is real and
sits in `depth_ladder.csv`, and on §11's identical-link comparison.

**3. Regenerating the sections silently deleted a number.** `tools/boosted_probit_sections.py`
took `--summary` as optional, and without it `ladder_section` dropped the fitted-scale sentence
the committed `ladder.tex` carries. It now defaults to the run's own `_state.json`, so a bare
invocation reproduces every committed section byte for byte. Its docstring also advertised
`--canonical`, `--diagnostics`, `--curve`, `--depth` and `--interactions`, none of which `main`
has ever accepted, so every command it documented failed on an unrecognised argument.

**And one thing the integration had got wrong about this architecture.** The probit reporting path
was written for two link-only architectures and this one inherited its reason strings, so the
published boosted report stated that it "has no model-specific learned features to attribute" and
that there is "no ensemble to be stable … not a bag of independently fitted members". Both are
false for an architecture that fits its own booster per seed. It was also absent from
`probit_diagnostics.INPUT_COLUMN` — the architecture list written twice, the same defect `47c3cd6`
fixed once in `probit.py` — so `build()` raised for it and its report rendered without the
reliability and link-movement blocks the other two probit reports carry. All three now have them;
the reasons are architecture-aware and pinned against the stage's own list; and the attribution
gap is recorded in `backlog.md` with the producer change it needs.

**A fifth thing, and the one place the fix stops short.** The run identity hard-coded
`features_learned=()` for every probit architecture, on the comment "a choice link learns nothing".
True of two; false of this one, which learns a correction on twenty-seven columns with the market's
implied probability as a base margin, exactly as `OffsetModel` does. `sweep --check` printed that
as `learned 0 consumed 28`. `probit._feature_split` now derives the split from the architecture and
from the model class's own declared market column.

The *published* boosted identity still says `learned 0`, and that is deliberate. Regenerating it
means `--force-overwrite`, because the merge of canonical block C4 added four columns to
`race_features.parquet` and every published run root's lock therefore names a feature table that no
longer exists — the guard that refused the re-run is working, not broken. Regenerating one root of
a five-root generation would leave `sweep --check` reading reports standing on two different
feature-table digests, which is worse than a uniformly old one: today all five name one commit, one
feature-set digest and one race population, and the gate passes. So the correction lands in the
code and in a test that reads the code, and the next full generation writes it into the artifact.

---

## E12 — tuning the architecture for its own objective, and what the validation years did instead

**Why there was anything to tune.** Every booster setting this architecture uses was **copied from
`OffsetModel.DEFAULT_PARAMS` and never tuned**. That was deliberate and correct at the time: §22
required the capacity comparison to hold everything but the likelihood fixed, so the probit objective
was fitted at the *softmax* objective's tuned settings. The consequence is that the architecture had
never been optimised for its own objective, and E9's negative result was a negative result about a
borrowed configuration.

**The hypothesis, and the misreading it rested on.** Two earlier measurements looked like evidence
that the probit objective wants more capacity and less shrinkage than the settings it borrowed:
§35's depth ablation is monotone from +0.002665 at depth 1 to **−0.000814 at depth 6** and stops
there, while production depth is 5; and E8 found the objective wanted lr 0.0025 against the softmax
booster's 0.01 while producing a *smaller* correction (sd 0.102 against 0.145).

The depth reading was wrong, and the ablation's own table says so. It reports probit **minus**
softmax at each depth, and the softmax arm degrades with depth on that validation year — best at
depth 1 (−0.004279), −0.003646 at depth 5. So a monotone *gap* was partly softmax getting worse, not
probit getting better. Measured directly on a fat fit, depth is flat and then harmful:

| max_depth | 5 | 6 | 7 | 8 |
|---|---:|---:|---:|---:|
| Δ vs market, validating 2023 | −0.004200 | −0.004191 | −0.004210 | −0.003889 |

A relative quantity read as an absolute one. Worth recording as its own lesson: a monotone
difference between two arms is not a monotone claim about either.

### The validation year is the whole problem

The canonical profile scores **2015–2026** and trains from 2010, so only **2011–2014 are never
scored**. That gives two options and no third:

* **Validate on 2014.** Leak-free — the canonical measurement never scores it — but the fit gets
  **36,160 training rows** where a canonical cell gets 121,000.
* **Validate on 2023, as E8 did.** 120,986 training rows, and the canonical run *scores 2023*, so a
  setting chosen there leaks into 2023's canonical cell. E8's framing ("no *2024* information
  selected anything") is true and insufficient.

Both were run, on one declared five-configuration grid over the regularisation axis, plus depth and
step-size grids. `utility_scale` fixed throughout at the declared 2.0, because a scale fitted on the
training loss runs to its lower bound (E4) and letting it move would make the sweep a comparison of
scales. Scoring goes through the model's own `predict_proba`, so the loss cannot differ from the one
the architecture reports.

**The two fat years anti-rank the grid. Spearman −0.4; 2014 against 2023 is −0.9.**

| configuration | 2014 (36k rows) | 2022 (112k) | 2023 (121k) | fat mean | 2022↔2023 swing |
|---|---:|---:|---:|---:|---:|
| baseline (mcw 20, λ 10) | **−0.000314** | −0.004371 | −0.004200 | −0.004285 | 0.000171 |
| `reg_lambda=5.0` | −0.000316 | −0.004514 | −0.003802 | −0.004158 | 0.000712 |
| `reg_lambda=2.0` | **−0.000451** ① | **−0.005092** ① | **−0.003558** ⑤ | −0.004325 | **0.001534** |
| `gamma=0.0` | −0.000288 | −0.004383 | −0.004186 | −0.004284 | 0.000197 |
| `min_child_weight=10` | **+0.000653** ⑤ | −0.004628 | **−0.004836** ① | **−0.004732** | 0.000208 |

Circled digits are the rank within that year. **Every candidate is both first and last depending on
which year you ask.** `reg_lambda=2.0` is the argmax on 2014 *and* 2022 and the worst of five on
2023; `min_child_weight=10` is the worst on 2014 and the best on 2023.

The mechanism is training-set size, and it points the way it should: a looser per-leaf minimum only
pays when there is data to fill the leaves, so `min_child_weight=10` needs the fat fit and hurts the
thin one. Which means **the leak-free year is not merely weak, it is actively misleading**: its whole
signal is −0.0003 against the fat years' −0.0043, a factor of fourteen, and its ranking is inverted.
Selecting on 2014 would have chosen the configuration that comes last on the fits that matter.

### The selection rule, stated before the canonical run

One year's argmax is exactly the error E8 named and then half-committed. Two years allow a rule that
an argmax does not:

> Take the configuration that (a) improves on the baseline in **both** fat validation years and
> (b) swings no more between them than the baseline itself does.

Clause (b) is what disqualifies the largest average gain. `reg_lambda=2.0` has the second-best fat
mean, and it buys it with a 2022↔2023 swing of 0.001534 — **nine times** the baseline's 0.000171. A
setting whose value depends that strongly on which season you validate on has not been shown to
generalise; it has been shown to be sensitive. `min_child_weight=10` gains **+0.000447** on the fat
mean and swings 0.000208, statistically the control's own swing.

**Selected: `min_child_weight` 20 → 10, and nothing else.** Depth stays at 5 (flat then harmful),
`reg_lambda` at 10.0, `gamma` at 0.2, the learning rate at E8's 0.0025, and rounds at 400 — more
rounds was worse on both fat years (2023: 600 → −0.004011, 800 → −0.003595).

**2022 and 2023 selected it, so both are excluded from the canonical comparison it is judged by.**
Nine scored years remain of eleven. That is the price of using fat validation years, and paying it
explicitly is cheaper than the alternative, which is a tuned number whose validation year is inside
its own test set.

`tools/tune_boosted_probit.py` is the harness; `tuning-2014.csv`, `tuning-2022.csv` and
`tuning-2023.csv` beside it are what it wrote.

### The canonical result: the tuning is a null, and it surfaced the link's own number

60 cells, 5 seeds, 400 rounds, `min_child_weight=10`, every arm scored on the same race keys and
paired per race. **Control first**: borrowed settings against the production Offset reproduces §9's
published figure exactly — year-mean −0.001352, se 0.000968, **t = −1.40**, 6/11 years — so the
comparison machinery below is the same one that produced §9.

On the nine years neither validation year touched:

| arm | race log loss | vs market |
|---|---:|---:|
| tuned boosted probit (mcw 10) | 2.021544 | −0.004918 |
| boosted probit (borrowed settings) | 2.021473 | −0.004988 |
| link-only `probit_offset` | 2.021003 | **−0.005459** |
| softmax link on the tuned boosted margins | 2.021897 | −0.004565 |
| production Offset | 2.021764 | −0.004697 |
| market | 2.026461 | — |

| comparison, paired at year level | year-mean | se | t | years |
|---|---:|---:|---:|---:|
| tuned vs borrowed settings | **+0.000062** | 0.000099 | **+0.63** | 5/9 |
| tuned vs link-only probit | +0.000054 | 0.001046 | +0.05 | 3/9 |
| tuned vs production Offset | −0.000836 | 0.001097 | −0.76 | 4/9 |
| **probit link, the tuned mean held fixed** | **−0.000351** | 0.000108 | **−3.24** | **8/9** |

**The tuning did nothing.** A 15% improvement in the validation year's edge (−0.004836 against
−0.004200) became **+0.000062 at t = +0.63** across nine independent years, and the sign alternates
year to year: −0.00011, +0.00056, −0.00031, −0.00009, +0.00038, +0.00003, −0.00023, +0.00033,
−0.00001. That is the twentieth candidate this programme has carried to canonical scale without
confirming, and the first where the thing carried was a hyperparameter rather than a feature.

Worth stating plainly, because it is the transferable part: **one validation year moved 15% and nine
test years moved nothing.** The grid was declared in advance, the selection rule was stated before
the run, the rule deliberately rejected the largest average gain for a stability criterion, and the
selection years were excluded from the judgement. Every one of those precautions was taken and the
gain still did not survive. A single-year validation delta at this effect size is not evidence about
a hyperparameter, and this is the measurement that says so rather than an argument that it might be.

**What the run did establish, and it is the strongest number in the programme's record.** Scoring
the *identical* boosted margins through the probit link and through the softmax link — the mean
function held fixed, only the shock distribution changing — gives **−0.000351 at t = −3.24 with 8 of
9 years favourable** (−0.000285, t = −2.91, 9/11 on all eleven; the exclusion is not needed for this
comparison because no selection touched it). §9 reported this contrast at *seed* level, where five
deltas over the same races measure determinism rather than generalisation. At year level it is the
first t below −2 in twenty candidates.

It is also not a promotion case, and the reason is the same one §9 gave. The comparison needs the
boosted margins, which cost two and a half hours of fitting and are themselves worth nothing:
`tuned vs link-only probit` is +0.000054 at t = +0.05, so the free link over the incumbent's own
utilities is as good as the expensive mean function underneath it. The link is what pays; the
architecture that made it measurable is not what should ship. **Recommendation unchanged: keep the
softmax booster, and keep `probit_offset` as the published link.**
