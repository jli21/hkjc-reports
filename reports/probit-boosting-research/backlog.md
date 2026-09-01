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
