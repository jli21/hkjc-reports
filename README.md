# HKJC model reports

Published evidence for five winner-probability models on Hong Kong racing, generated from
[jli21/hkjc-mod](https://github.com/jli21/hkjc-mod) by `python -m hkjc build-report`, plus five
exploratory research studies that inform them. Nothing in the model reports is written by hand.

**Open [`index.html`](index.html)**, or go straight to a report:

| Model | Comparator | Report |
|---|---|---|
| Softmax (unanchored baseline) | the market | [HTML](reports/softmax/report.html) |
| Softmax Offset (production) | the market | [HTML](reports/softmax_offset/report.html) |
| Standalone Probit | standalone Softmax | [HTML](reports/probit/report.html) |
| Probit Offset (market-anchored) | the incumbent Offset | [HTML](reports/probit_offset/report.html) |
| Boosted Probit Offset | the incumbent Offset | [HTML](reports/probit_offset_boosted/report.html) |

Each HTML file is self-contained: no network, no server, no external stylesheet. GitHub will not
render them in the file view, so download the file or use a raw-HTML viewer.

These five are the models `hkjc-mod` currently publishes. The fifth is the only one that trains its
own mean function under the probit likelihood rather than laying a choice link over another run's
utilities -- and its own conclusion is that the gain belongs to the link rather than to the direct
training, so `hkjc-mod` keeps the softmax booster.

`reports/bagged_offset/` and `reports/probit_offset_pace/` are kept as archived evidence from the
2026-08-25 generation under the earlier naming; they are not regenerated and their figures describe a
different race population from the five above.

`reports/probit-boosting-research/` holds `hkjc-mod`'s own research documents rather than a model
report: the findings log, the open backlog, and the boosted-probit experiment's log and PDF. It is
the written record behind several results that were *rejected*, which is most of what is in it.

## Research studies

Six exploratory studies of the distribution of horse performance in Class 1-5 racing, 2008-2026,
from [RamenBurger/hkjc-research](https://github.com/RamenBurger/hkjc-research). They are written
documents rather than generated reports, they treat the production model as read-only, and they
compute no return anywhere: no dividends are loaded, deliberately, because at these effect sizes
ROI ranks noise.

| Study | Report | Subject |
|---|---|---|
| I. Static latent performance | [PDF, 45pp](reports/research/report.pdf) | latent horse and jockey ability, environmental sensitivity, the residual distribution, volatility, race-level shocks, groups of horses |
| II. Dynamic latent state and regimes | [PDF, 18pp](reports/research/dynamic-state.pdf) | ability as a dynamic state, run informativeness, per-run observation noise, opponent adjustment, transition regimes, dynamic analog peers, 200 m sub-splits |
| III. Deep feature research | [PDF, 29pp](reports/research/feature-research.pdf) | handicap/class/rating state, sectional and 200 m physical state, field-level pace and race shape, run-specific information quality, and two adversarial checks on the programme's own instruments |
| IV. Regime explorer | [PDF, 22pp](reports/research/regime-explorer.pdf) | latent switching regimes, a multidimensional sectional state, jockey interactions, metric-learned peer pools, per-run predictive information, and the residual mixture as a measurement model |
| V. Jockey booking information | [PDF, 12pp](reports/research/jockey-booking-research.pdf) | whether today's jockey booking adds anything to the market, the horse's history and production's `jockey_residual`: dynamic jockey state, booking change and upgrade, trainer appointment behaviour, barrier-trial continuity, and a tactical-execution gate |
| VI. Within-race geometry | [PDF, 8pp](reports/research/within-race-geometry.pdf) | the information a per-row booster provably cannot construct: price micro-structure, field-centred form, campaign position, and a decomposition of the stewards' comment channel |

Study IV closes the six sections of Study II's brief that Study II did not execute, and
**supersedes `dynamic-state.pdf` as the answer to that brief**; Study II is kept as its own
artifact rather than as the current answer. Read I to IV in that order if reading all of them.

**Study V is a null, and is published because of that rather than in spite of it.** Eleven
candidate blocks were built across four predeclared waves, eight reached the production model's
own canonical profile (8 years x 5 seeds x 400 rounds, 40 paired cells), and **all eight were
rejected**: the best is worth `-0.000075` of race log loss against a declared materiality bar of
`-0.0005`, and six of the eight make the model worse. Three results in it are worth more than the
feature would have been --- production's cumulative `jockey_residual` is reproduced exactly to
6.7e-17 and its *cumulative* form is confirmed better than any decayed one; the apparent
booking-upgrade signal loses to a permutation that preserves trainer and market-rank structure;
and a jockey tactical effect that beat its own control by 189 times produced the campaign's **worst**
block. No feature was promoted and none was handed to production.

**Study VI is also a null, and it is the sharpest one.** It asked whether the remaining signal
sits in the one class of information the production model *provably cannot construct*: a per-row
booster sees one runner at a time, and a racewise softmax with the market as a fixed offset
responds only to within-race variation, so the geometry of a field is invisible to it. Two of the
four waves were withdrawn on numerical proof before any canonical budget was spent --- including
one proof that invalidated the campaign's own declared control --- and the six remaining blocks
were rejected. Its two permutation controls are the result and they disagree: price
micro-geometry beat its control, so that geometry does attach to the individual runner, while
ability geometry lost to its control by a factor of thirteen. The principle is confirmed as a
principle and refuted as a source of usable signal.

**Across Studies III to VI, 24 candidate blocks have been carried to the production model's own
canonical profile and none has confirmed.** That is the programme's central practical finding and
it is stronger than any of the individual nulls. Two mechanisms once looked like near-misses --- a
pace-composition cross at t = -1.25 and a campaign-position block worth 37% of the production
edge --- and both dissolved on examination, the second because a single partial season accounted
for more than the whole effect.

Only Study I's LaTeX source is published here, beside its PDF, because it was already. The sources
for the other five, every stage that produced their numbers, and the research log recording several
results that were corrected after their mechanisms were traced, are all in `hkjc-research`.

## Nothing here establishes profitability

Every stake, return and bankroll figure is computed against historical result-page odds and
official dividends. Those are the prices after the pool closed: a bet placed at them could not
have been placed. Every test year through 2026 has also been development data, so the forward
figures are probability progress, not a locked out-of-sample result. The confirmation step none
of this substitutes for is a prospective shadow test against timestamped pre-race price
snapshots. Each report restates this beside its betting numbers and again at the end.

## What is in a report

The same seven sections for every model, so two reports can be read against each other: overview
and relationship to the market, predictive performance, calibration, model diagnostics, historical
settlement and bankroll, model-assumed simulation, and limitations.

A block that cannot apply to a model says so and why -- a standalone probit fits its own utilities
and persists no attribution for them, so there is no inherited correction to attribute -- rather
than being omitted, and a block whose evidence is simply absent is distinguished from one that does
not apply. Each report states the comparator it is scored against and why that comparator is the
right one; the deltas are **not** comparable across families, because each model is measured
against its own.

The evaluation window is 2016-2026, **7,933 races**, with 2015 excluded as a warm-up year whose
utility scale is not fitted. The race keys are compared by digest before publication, so the four
reports are known to cover the same population rather than assumed to.

The last section is an interactive Kelly panel over the four fractions the run settled (1%, 2%,
5%, 10%), showing three staking rules in two bankroll modes -- continuous and annual reset. Every
path was computed in Python and embedded; the control switches between pre-computed series and does
no arithmetic, because a browser recomputing a stake would be a second implementation of the
settlement rules.

The two Probit reports headline the **published bagged probability scored per race** -- the
probability settlement stakes and the simulations resample -- and publish two other constructions
of the same comparison beside it, per evaluated year and over the mean fitted member. The three do
not agree, and the disagreement is a property of the model rather than noise, so none of them is
hidden.

## Beside each report

| File | What it is |
|---|---|
| `summary.json` | the identity, the section capabilities and every settled Kelly path |
| `manifest.json` | what the report wrote, by digest |
| `run_lock.json` | what the report *read*, by content digest, and the code and environment that read it |

`run_lock.json` is what ties a figure back to the run that produced it: the run inputs are not
published here, but their digests are, so a claim about which bytes a number came from is
checkable. It also records whether the source tree was clean when the report was written; for all
four model reports it was.

## How these were checked

* The generation behind them was produced twice -- once into a scratch directory and once into the
  canonical run roots -- and every parquet and CSV column of all four models is bit-identical
  between the two, compared by content rather than by a tolerance.
* The artifacts reproduce 771 pinned per-column digests across 23 files, verified with the declared
  keys checked for uniqueness.
* 2,075 tests pass, and each report was rendered under canonical strictness, which refuses to
  publish a report whose declared evidence is missing.

No PDFs are published for the four model reports. The earlier generation shipped them with an
automated
dimension-and-heading gate and a recorded visual review; the current pipeline does not produce
them, and shipping the 2026-08-25 files beside 2026-09-01 HTML would have paired each report with a
PDF describing a different race population. The research studies under `reports/research/` are PDFs
by nature and are unaffected.
