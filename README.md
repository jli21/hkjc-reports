# HKJC model reports

Published evidence for four winner-probability models on Hong Kong racing. Generated from
[jli21/hkjc-mod](https://github.com/jli21/hkjc-mod) by `python -m hkjc build-report`; nothing here
is written by hand.

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
population from the four above. `reports/research/` is a separate written report, not a generated
one.

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
checkable. It also records whether the source tree was clean when the report was written; for these
four it was.

## How these were checked

* The generation behind them was produced twice -- once into a scratch directory and once into the
  canonical run roots -- and every parquet and CSV column of all four models is bit-identical
  between the two, compared by content rather than by a tolerance.
* The artifacts reproduce 771 pinned per-column digests across 23 files, verified with the declared
  keys checked for uniqueness.
* 2,075 tests pass, and each report was rendered under canonical strictness, which refuses to
  publish a report whose declared evidence is missing.

No PDFs are published for these four. The earlier generation shipped them with an automated
dimension-and-heading gate and a recorded visual review; the current pipeline does not produce
them, and shipping the 2026-08-25 files beside 2026-09-01 HTML would have paired each report with a
PDF describing a different race population.
