# HKJC model reports

Published evidence for four winner-probability models on Hong Kong racing. Generated from
[jli21/hkjc-mod](https://github.com/jli21/hkjc-mod) by `python -m hkjc build-report`; nothing here
is written by hand.

**Open [`index.html`](index.html)**, or go straight to a report:

| Model | Comparator | Report |
|---|---|---|
| Bagged Offset (production) | the market | [HTML](reports/bagged_offset/report.html) · [PDF](reports/bagged_offset/report.pdf) |
| Standalone Probit | standalone Softmax | [HTML](reports/probit/report.html) · [PDF](reports/probit/report.pdf) |
| Probit Offset (market-anchored) | the incumbent Offset | [HTML](reports/probit_offset/report.html) · [PDF](reports/probit_offset/report.pdf) |
| Probit Offset with a pace factor | the incumbent Offset | [HTML](reports/probit_offset_pace/report.html) · [PDF](reports/probit_offset_pace/report.pdf) |

Each HTML file is self-contained: no network, no server, no external stylesheet. GitHub will not
render them in the file view, so download the file or use a raw-HTML viewer; the PDFs open in
place.

## Nothing here establishes profitability

Every stake, return and bankroll figure is computed against historical result-page odds and
official dividends. Those are the prices after the pool closed: a bet placed at them could not
have been placed. Every test year through 2026 has also been development data, so the forward
figures are probability progress, not a locked out-of-sample result. The confirmation step none
of this substitutes for is a prospective shadow test against timestamped pre-race price
snapshots. Each report restates this beside its betting numbers and again at the end.

## What is in a report

The same fourteen sections for every model, so two reports can be read against each other:
identity and provenance, executive summary, comparator and evaluation population, overall and
year-by-year probability performance, calibration and sharpness, top-choice movement, ensemble
stability, feature and correction effects, representative races, betting and bankroll analysis,
simulation and uncertainty, a technical appendix, and limitations.

A section that cannot apply to a model says so and why -- a standalone probit has no market
offset, so there is no correction to attribute -- rather than being omitted. Each report states
the comparator it is scored against and why that comparator is the right one; the deltas are
**not** comparable across families, because each model is measured against its own.

The last section is an interactive Kelly panel over the four fractions the run settled (1%, 2%,
5%, 10%), showing three staking rules in two bankroll modes. Every path was computed in Python
and embedded; the control switches between pre-computed series and does no arithmetic, because a
browser recomputing a stake would be a second implementation of the settlement rules.

## Beside each report

| File | What it is |
|---|---|
| `summary.json` | the identity, the section capabilities and every settled Kelly path |
| `manifest.json` | what the report wrote, by digest, the PDF quality-gate verdict, and the recorded visual review |
| `run_lock.json` | what the report *read*, by content digest, and the code and environment that read it |

`run_lock.json` is what ties a figure back to the run that produced it: the run inputs are not
published here, but their digests are, so a claim about which bytes a number came from is
checkable.

## How these were checked

* Every PDF passed an automated gate: A4 dimensions, page count, all fourteen headings, no blank
  pages, every chart's axis label present, headline values matching the HTML, and no suppressed
  failure text.
* Every page of every PDF was rasterised and visually inspected; the review is recorded in each
  `manifest.json`.
* The artifacts behind them reproduce 694 pinned per-column digests across 19 files, verified by a
  full rebuild from a clean source tree.
