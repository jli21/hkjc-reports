# HKJC model reports

Published evidence for four winner-probability models on Hong Kong racing, generated from
[jli21/hkjc-mod](https://github.com/jli21/hkjc-mod) by `python -m hkjc build-report`, plus four
exploratory research studies that inform them. Nothing in the model reports is written by hand.

**Open [`index.html`](index.html)**, or go straight to a report:

| Model | Comparator | Report |
|---|---|---|
| Softmax (unanchored baseline) | the market | [HTML](reports/softmax/report.html) |
| Softmax Offset (production) | the market | [HTML](reports/softmax_offset/report.html) |
| Standalone Probit | standalone Softmax | [HTML](reports/probit/report.html) |
| Probit Offset (market-anchored) | the incumbent Offset | [HTML](reports/probit_offset/report.html) |

Each HTML file is self-contained: no network, no server, no external stylesheet. GitHub will not
render them in the file view, so download the file or use a raw-HTML viewer.

These four are the models `hkjc-mod` currently publishes. `reports/bagged_offset/` and
`reports/probit_offset_pace/` are kept as archived evidence from the 2026-08-25 generation under
the earlier naming; they are not regenerated and their figures describe a different race
population from the four above.

## Research studies

Four exploratory studies of the distribution of horse performance in Class 1-5 racing, 2008-2026,
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

Study IV closes the six sections of Study II's brief that Study II did not execute, and
**supersedes `dynamic-state.pdf` as the answer to that brief**; Study II is kept as its own
artifact rather than as the current answer. Read them in that order if reading all four.

Only Study I's LaTeX source is published here, beside its PDF, because it was already. The sources
for the other three, every stage that produced their numbers, and the research log recording several
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
