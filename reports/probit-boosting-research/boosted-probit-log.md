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

CANONICAL_PLACEHOLDER

