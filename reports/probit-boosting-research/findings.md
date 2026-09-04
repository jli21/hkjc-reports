---
title: Research findings
kind: findings
status: living-document
moved: 2026-08-25
moved_from: FINDINGS.md
moved_by: docs/plans/archive/2026-08-24-repository-cleanup.md
---

# HKJC model -- research findings

A record of what was tried and what it showed, in the order it happened. Moved here from
`FINDINGS.md` at the repository root by Phase 10 of the 2026-08-24 cleanup.

## How to read this

This is a log, not a specification, and parts of it are older than the model. Where a section's
numbers have been superseded, a note at the head of the section says so and names what replaced
it -- the notes were added during the cleanup rather than the sections being rewritten, because
a research log whose history is edited to match the present is no longer evidence of anything.

The current published numbers are in the four model reports under `results/reports/`. Where this
document and a report disagree, the report is right: it is generated from declared artifacts and
its inputs are locked by digest.

Three sections below are explicitly stale and marked:

* **TL;DR -- current best config** describes a 25-feature, Kelly 0.02/0.03 configuration whose
  cumulative returns (+332% at Kelly 0.02) come from a settlement that predates the current
  filters. Production is 28 declared inputs of which 27 are learned, Kelly 0.01, and the
  published bagged_offset report settles 339 bets over twelve years for a return near zero.
* **Feature set (final)** lists 25 columns and names two that are no longer inputs. The
  authority is `configs/features/production.json`.
* **Reproducibility** names `python -m hkjc.suites.production`, which does not exist. The
  supported commands are in `docs/pipeline.md`.

What has *not* gone stale, and is still the most useful part of this document, is the negative
and methodological findings: what was tried and did not work, and why a measurement that looked
convincing was not. Those are in **What we tried -- chronologically** and **Methodology
lessons**, and they are the reason several later designs were not attempted twice.


## 2026-09-02 `race-incidents` was a fixture: one meeting, 27 year files, and a contract that passed

**309,276 rows across 27 year files held 142 distinct horses and 99 distinct incident texts.** Every
check in this repository passed it. The row count was right, the dates were right, the schema was
right, the `Incident` prose was real -- and it came from one meeting, relabelled 2,178 times.

**Three failures compounding, established against the live site rather than inferred.**

1. **The wrong query parameter.** The collector built
   `racereportfull?racedate=YYYY/MM/DD&Racecourse=ST&RaceNo=1`. The endpoint reads none of those
   three; its own date selector navigates with `?Date=`. Verified on 2026-09-02:
   `?racedate=2024/01/01` answers *"Race Meeting: 15/07/2026"* at Happy Valley, and `?Date=2024/01/01`
   answers *"01/01/2024"* at Sha Tin.
2. **The endpoint fails open, and its siblings do not.** Given a parameter it cannot honour, or a date
   with no meeting, `racereportfull` returns **HTTP 200 and a complete, plausible page for the current
   default meeting**. Its sibling `exceptionalfactors`, fetched by the same collector in the same run
   with the same lowercase `racedate`, fails *closed* with "No information." That is why
   `race-exceptional-factors` is real and this was not, from one collector in one run.
3. **Identity came from the request.** `parse_report_meeting` stamped every row with the date and
   racecourse it had *asked for*, numbered races by table position, and synthesised a per-race
   `source_url` that was never fetched. So a fallback page was relabelled as the meeting requested and
   the recorded provenance could not contradict it.

The mechanism is visible in the data alone: in `race-incidents-2003.csv`, ST race 1 across **49
distinct dates holds exactly one horse set**, ST and HV rows for a race number are identical, and the
file contains `horse_id`s from the 2020-2025 seasons. Only `RaceNo` was ever read: 11 race numbers x
~13 runners = 142 horses.

**What was rebuilt, by this repository's own repaired collector.** 192 meetings, zero fetch failures,
zero discards:

| | rebuilt 2024-2026 | the archive it replaced |
|---|---:|---:|
| rows | 22,914 | 309,276 |
| races | 1,845 | 23,958 |
| meetings | 192 | 2,178 |
| **distinct horses** | **2,122** | **142** |
| **distinct incident texts** | **13,272** | **99** |

Race numbers came from the page's own `Race:N` headers for **100% of rows** -- the positional fallback
never fired. Every accepted page proved its own meeting date, and each row records the URL actually
fetched (192 URLs for 192 meetings, not one per race). 98.81% of `(race_id, horse_id)` pairs are in the
results panel and on those 22,641 rows the scraped horse number agrees with the panel's **exactly:
zero disagreements**. Mean runner-set agreement is 0.9927 with 90.6% of races agreeing exactly.

**The 273 rows not in the panel are fully accounted for, and most of them are the point of the
dataset.** 170 are withdrawn or excluded runners (`WV`, `WX`) that never started, so a results-derived
panel cannot contain them -- and their text is the intervention itself: *"Withdrawn on 2.1.24 by order
of the Stewards acting on veterinary advice (lame right fore). Before being allowed to race again,
LADY BILLIE will be subjected to..."* 102 sit in eight races the **panel itself** lacks on three 2024
dates, so they are an `aspx-results` gap rather than an incident error. One is a standby declared
starter with no placing and no number, recording a trainer fined for entering it in a barrier trial.
Zero rows are unexplained. A downstream join must not inner-join on the panel or it discards exactly
these.

**Two bounds on what can ever be recovered.** The per-runner incident *table* begins with the 2023/24
season: `2024/01/01` serves ten tables and 124 runners while 2023/09/10, 2023/01/01, 2022, 2021, 2020
and 2019 serve real pages with **zero** tables. So the 24 pre-2024 files were not mis-scraped, they
were never available in this schema, and they are gone. What those earlier pages *do* carry is the
same channel as prose keyed by race -- *"Race 10: SUPER FORM on 31.12.18 on veterinary advice"*,
jockey censures, per-race veterinary findings -- which is a meeting-level schema and a separate
dataset. It is in `backlog.md`.

**The methodology lessons, which are the transferable part.**

* **A sibling endpoint's contract is not evidence of this endpoint's contract, and neither is HTTP
  200.** Two pages in the same family, fetched by the same code path on the same day, disagreed about
  whether a query parameter exists and about what to do when it cannot be honoured.
* **Identity must come from the response.** A collector that stamps a response with the request it
  answered cannot detect a server that ignored the request, and its provenance is then a record of
  what was asked rather than of what arrived.
* **A row count cannot see replication.** `min_rows: 300000` was satisfied by the fixture. The
  contract now declares entity-cardinality floors (`min_distinct`) enforced on read and reported in
  `SCRAPE_COVERAGE.md`, and the regression test is that the replicated fixture fails its own contract.
* **The cross-dataset check was the cheap one and it was not there.** Comparing the page's runners
  against `aspx-results` for the same races returns ~0.99 on the rebuilt data and near zero on the
  archive. It would have caught this on the first meeting.

The primary key moved from `(race_id, "Horse No.")` to `(race_id, horse_id)`, because a withdrawn
runner has no number and those are the rows worth having. `verify --only historical` now checks 249
files rather than 273, and the freeze grew a rebaseline verb that demands a reason and records what
moved, because "source data does not change" cannot mean "a fixture stays because it is frozen".

## 2026-09-03 Tuning the boosted probit: one validation year moved 15%, nine test years moved nothing

Every booster setting in `probit_offset_boosted` was **copied from `OffsetModel` and never tuned** --
§22 of its plan required the capacity comparison to hold everything but the likelihood fixed. So its
negative result was a negative result about a borrowed configuration, and the architecture had never
been optimised for its own objective.

**The validation year is the whole problem, and it has no clean answer.** The canonical profile scores
2015-2026 and trains from 2010, so only 2011-2014 are never scored. Validating on 2014 is leak-free
and fits 36,160 rows where a canonical cell gets 121,000; validating on 2023, as the original work
did, has the rows and is a year the canonical run *scores*. Both were run over one declared grid.

**The two fat validation years anti-rank the grid: Spearman −0.4, and 2014 against 2023 is −0.9.**
Every candidate configuration is both first and last depending on which year is asked.
`reg_lambda=2.0` is the argmax on 2014 *and* 2022 and the worst of five on 2023. The mechanism is
training-set size -- a looser per-leaf minimum only pays when there is data to fill the leaves --
which makes the leak-free year not merely weak but **actively misleading**: its whole signal is
−0.0003 against the fat years' −0.0043, and its ranking is inverted.

The selection rule was stated before the canonical run and deliberately rejected the largest average
gain: take the configuration that improves on the baseline in **both** fat years *and* swings no more
between them than the baseline does. That picked `min_child_weight` 20 → 10 (+0.000447 on the fat
mean, swing 0.000208) over `reg_lambda=2.0` (+0.000040, swing 0.001534 -- nine times the control's).
Both selection years were then excluded from the judgement.

**It did nothing.** A 15% improvement in the validation year's edge became **+0.000062 at t = +0.63
over nine independent years**, sign alternating. Twentieth candidate to canonical scale without
confirming, and the first where the thing carried was a hyperparameter rather than a feature. The
transferable lesson is the precautions: grid declared in advance, rule stated before the run, argmax
deliberately refused, selection years excluded -- all taken, and the gain still evaporated. A
single-year validation delta at this effect size is not evidence about a hyperparameter.

**What the run did establish is the strongest number in the programme's record.** Scoring the
*identical* boosted margins through the probit link and through the softmax link -- mean function
fixed, only the shock distribution changing -- gives **−0.000351 at t = −3.24, 8 of 9 years
favourable** (−0.000285, t = −2.91, 9/11 on all years). The original work reported this contrast at
*seed* level, where five deltas over the same races measure determinism rather than generalisation.
At year level it is the first t below −2 in twenty candidates.

It is still not a promotion case, for the same reason as before: the contrast needs the boosted
margins, which cost two and a half hours of fitting and are worth nothing themselves -- the tuned
architecture against the free link-only probit is +0.000054 at t = +0.05. The link pays; the
architecture that made it measurable is not what should ship. **Recommendation unchanged: keep the
softmax booster, keep `probit_offset` as the published link.** Full methodology in
`docs/research/boosted-probit-log.md` §E12 and the report's tuning section.

## 2026-09-02 The declared-gear intervention: three blocks, three nulls, and a season split that did not replicate

**What was tested.** HKJC declares each runner's equipment on the card before the race in its own
notation -- a bare code is standing equipment, a trailing `1` is first time in that item, `2` is the
second start, `-` is removed today. Production reduces the whole channel to `gear_change`, one bit
comparing raw strings. Six columns decompose it (`gear_first_n`, `gear_removed_n`, `gear_load`,
`gear_vision_first`, `gear_true_change`, `gear_intervention_size`), measured as three blocks against
the locked incumbent. No new dataset, cache or join: the `Gear` string was already loaded.

**D1 confirmed on this repository's own table, not taken on trust.** Over 152,046 starts from 2010:

| Quantity | Measured here | The research's panel |
|---|---:|---:|
| `gear_change` fires | 0.2654 | 0.2769 |
| the equipment set actually changed | 0.1467 | 0.1524 |
| **fires with no equipment change** | **0.4473** | 0.4499 |
| real changes the bit misses | 0.0000 | 0.0000 |

`B1` at one start becomes `B` at the next, so the string changes while nothing on the horse's head
does. The bit never misses a real change; nearly half of what it reports is HKJC's first-time marker
decaying. **`gear_change` keeps its semantics** -- it is one of the 28 production inputs, so
redefining it is a modelling change behind the same bar as a promotion, and the corrected bit had to
be measured as an addition first.

**The result: three rejects.** `canonical_confirm`, 11 scored years x 5 seeds, 55 cells each,
complete, race universe consistent, no-op exactly `0.0` on the rebuilt table beforehand:

| Block | equal-cell | race-weighted | years | seeds | paired 95% interval | research | retention |
|---|---:|---:|---:|---:|---|---:|---:|
| all six | **-0.0002790** | -0.0002537 | 7/11 | **5/5** | -0.000486 to -0.000024 | -0.0014606 | 19% |
| first-time only | **-0.0001978** | -0.0002181 | **9/11** | **5/5** | -0.000391 to -0.000052 | -0.0012124 | 16% |
| corrected bit | **-0.0001370** | -0.0001639 | 7/11 | **5/5** | -0.000336 to +0.000010 | -0.0007010 | 20% |

Every equal-cell mean is above the pre-registered `-0.0005`, so all three stop there. At 3.0-6.2% of
the production model's market-relative edge, none is worth a production column.

**This is a different kind of null from C4's, and the difference is worth keeping.** C4 was
inconsistent *and* tiny: 5/11 years, 1/5 seeds, an interval spanning zero. These are **consistent and
tiny**: 5/5 seeds favourable in all three blocks, 9/11 years for the narrow one, and two of the three
intervals *exclude* zero. The signs are the most stable this programme has produced. The magnitude is
a fifth of what was predicted. A real, reproducible, sign-stable effect can still be too small to
promote, and that is a cleaner finding than noise would have been.

**The research's central prediction did not replicate, and it was pre-registered so it can be said
plainly.** The source repository put the *entire* effect in the last three seasons, identically across
all three blocks, with 2016-2022 positive in every one (+0.000329, +0.000466, +0.001133). Split at
this plan's 2023 boundary:

| Block | 2016-2022 | years | 2023-2026 | years |
|---|---:|---:|---:|---:|
| all six | -0.0000762 | 3/7 | -0.0006338 | **4/4** |
| first-time only | **-0.0001544** | **7/7** | -0.0002739 | 2/4 |
| corrected bit | -0.0001663 | 5/7 | -0.0000858 | 2/4 |

The early half is favourable in all three blocks, not positive. For the first-time block it is
**7/7 years favourable at t = -2.22**, more consistent than its own recent half. So the effect is not
confined to the seasons where declaration coverage rose (`gear_first_n` 0.0943 in 2010 to 0.1436 in
2024), and the "recent-only, therefore a restricted-span argument" branch of the pre-registration is
not the one that fired. Only the six-column block is recent-concentrated (4/4 favourable at
t = -2.97 over 2023-2026), which is the opposite of the wide-block-dilutes-its-contents story.

**Redundancy, pinned.** Against `gear_change`: `gear_true_change` 0.690, `gear_intervention_size`
0.650, `gear_removed_n` 0.455, `gear_first_n` 0.323, `gear_vision_first` 0.295, `gear_load` 0.121.
Against `early_style_ewm` and `trip_excuse_ewm` every column is below 0.07 in absolute value -- this
block is genuinely novel where C4 was not, and it still measured almost nothing. That is the third
study in a row in which novelty against the incumbent and usefulness ran in opposite directions.

**What was kept.** The six columns stay in both feature tables and in `ALL_CANDIDATE_FEATURES` as
research inputs; nothing reads them at fit time unless a candidate names them. `production.json` is
untouched. The three manifests carry their measured neutral shares and the pinned correlations. The
replacement test -- the corrected bit *instead of* `gear_change` rather than beside it -- is still not
expressible in this harness and is in `backlog.md` with what it would cost.

> **Superseded on 2026-09-04: the six columns were promoted.** The paragraph above records the
> decision as it stood under the `-0.0005` bar. The bar was reconsidered on the grounds that it
> rejects effects this measurement shows to be real, and the six-column block was promoted into
> `production.json`. See *2026-09-04 The gear promotion* below for the decision, what the relaxation
> means, and the measured effect on the published generation. Nothing in the measurement above was
> re-read or revised; only the rule applied to it changed.

## 2026-09-04 The measured result of both production changes

Production went **28 → 33** columns in one generation: the six gear-intervention columns added,
`trip_excuse_ewm` removed. Measured on the production `softmax_offset` root, racewise log loss against
the market on the winner, bagged over five seeds, 8,675 scored races over 2015-2026. The two changes
were fitted separately so each is attributable:

| | equal-year mean | years improved |
|---|---:|---:|
| the six gear columns added | **-0.000299** | 8/12 |
| `trip_excuse_ewm` removed | **-0.000105** | 8/12 |
| **net, 28 → 33 columns** | **-0.000404** | **9/12** |

Pooled market-relative loss moved from `-0.004647` to `-0.005025`, an **8.1% improvement in the
model's edge over the market**.

*The basis, because it is not the reports'.* Every number in this entry is over **2015-2026**, the
full set of test years the run fits -- 8,675 races. The five published reports standardise on
**2016 onwards** instead (7,933 races), because 2015 has no fitted forward utility scale and takes
the predeclared theoretical value, which makes its stakes and claimed edges non-comparable with a
fitted year's; `configs/publication/reports.json` carries that reasoning. So the same run reports a
pooled market-relative loss of `-0.0048228` in `results/reports/softmax_offset/` and `-0.005025`
here, and the two do **not** disagree -- they are the same quantity over two populations. Verified
against `race_scores.parquet` on 2026-09-04. A research entry measuring a feature change wants every
year the change was fitted on; a publication comparing five architectures wants one population all
five can be scored on.

**The gear result reproduced its own pre-registration almost exactly.** The candidate harness estimated
`-0.0002790` for the block; the production refit delivered `-0.000299`. That is the closest agreement
between a `feature-iterate` estimate and a production outcome this programme has recorded, and it is
the strongest available evidence that the harness measures what it claims to.

**The removal is the more interesting number, and it contradicts the magnitude ranking.**
`trip_excuse_ewm` was 18th of 33 correction features by mean absolute correction (0.012416) -- ahead of
`closer_setup`, `distance_log` and `gear_change` -- and removing it made the model *better*. So
**correction magnitude is not evidence of usefulness**: it measures how far the booster moves on a
column, not whether the movement generalises. A column can absorb a large share of the correction
budget and spend it on noise, which is what a text-derived feature with non-uniform coverage across
sixteen seasons is well placed to do.

This is the second time the two have run in opposite directions, and it is worth stating as a rule
rather than as two anecdotes. The seven race-constant columns have a leave-one-out probability effect of
exactly 0.000000 and removing them made the model **worse** (+0.000219). `trip_excuse_ewm` has the 18th
largest correction in the model and removing it made it **better** (-0.000105). Neither the magnitude
ranking nor the leave-one-out effect predicted the sign of its own removal. The only thing that did was
fitting the model without the column, which means **the ablation is the measurement and the attribution
is a description** -- a distinction the reports should be read with.

### What the change cost the two probit links, which is not nothing

The mean model improved and the two choice links laid over its utilities did **not** move together.
Measured on the standardised report population (2016-2026, 7,933 races), bagged and pooled:

| | before, 28 columns | after, 33 columns | |
|---|---:|---:|---|
| `probit` vs its standalone comparator | -0.01212623 | **-0.01230172** | improved |
| `probit_offset` vs the incumbent offset | -0.00076217 | **-0.00064190** | **gave back 16%** |

**And `probit_offset` lost a property the repository had asserted -- on one of its three
constructions.** This has to name the construction, because the two disagree and an unqualified "it
was 11 of 11" is wrong for the number the report headlines. Restated after the 2026-09-04
republication, from the artifacts on both sides:

| construction | before, 28 columns | after, 33 columns |
|---|---|---|
| **member mean** (`honest_better_years`, the assertion) | 11 of 11, clean sweep | **10 of 11**, worst year +0.000232 |
| **bagged**, which the report headlines | **already 10 of 11**, worst year +0.0000493 | 10 of 11, worst year +0.00030877 |

So the clean sweep that was lost is the member-mean one -- the construction
`test_the_offset_direction_holds_in_all_but_one_honest_year` asserts. The **published** headline never
had a clean sweep to lose: the 28-column report already recorded one degraded year, and what the
promotion did there was make that year **6.3 times larger**, from +0.0000493 to +0.00030877. Both
readings are true of the same link and neither is the whole statement, which is exactly why
`bundle.py`'s `_reference_delta` pins all three constructions rather than a headline.

Either way the gates hold: +0.000232 against a declared `YEAR_DEGRADATION_TOLERANCE` of 0.001, so
promotion gate 3 passes, as do the seed-agreement, day-block-interval and nested-calibration gates,
and the aggregate honest delta is still favourable at -0.00083009.

This is the one known unfavourable consequence of the promotion, and it is worth stating in the same
entry that reports the gain rather than leaving it to a test diff. It also says something about where
the six columns act: a block that improves the mean utilities can still shift the *shape* of the
per-race utility spread, and the anchored link -- which prices against the market's own ordering --
is more exposed to that than the standalone one. The two links moving in opposite directions is the
evidence; the mechanism is not established here and should not be read as though it were.

### The other three architectures, measured on the republished generation

The entry above was written before the five reports were regenerated, so it could only report the two
links. All five are now published from the 33-column tree, and two of the remaining three moved in
ways worth recording. Same standardised population throughout, 2016-2026 and 7,933 races.

| report | market delta before | after | year consistency |
|---|---:|---:|---|
| `softmax` (unanchored baseline) | -0.00099165 | **-0.00106825** | **7 of 11 -> 5 of 11** |
| `softmax_offset` (production) | -0.00453270 | **-0.00482279** | 10 of 11, unchanged |
| `probit_offset_boosted` vs its comparator | -0.00030523 | **-0.00029061** | 11 of 11, unchanged |

**The unanchored baseline is the uncomfortable one.** Its pooled edge over the market improved by 7.7%
and its *year* record fell from 7 of 11 to **5 of 11** -- worse than a coin flip. Both are true because
its edge is roughly a fifth of the anchored model's and its per-year deltas straddle zero, so a pooled
improvement of 0.00008 can move several years across the line in either direction. Read the year count
as the honest reading of a model this weak rather than as a regression the promotion caused: at this
effect size the count is not a stable statistic, which is itself the finding and is the reason the
promotion decision was never taken on it.

**The boosted arm gave back 4.8% of its edge and kept every other property**: 11 of 11 years, seed
signs in agreement, and its market-relative loss improved from -0.00476757 to -0.00479720. Its own
conclusion is therefore unchanged by the promotion -- the gain still belongs to the choice link rather
than to training the mean under the probit likelihood, which is what `configs/publication/reports.json`
records as the reason production keeps the softmax booster.

### The group ablation, re-run on the seven current groups, corroborates the promotion independently

The 2026-08-25 ablation described a group set that no longer exists -- it held `race_text` and had no
`gear_intervention` arm -- so it was re-run cold on 2026-09-04: 420 fits over 84 `(group, year)` cells
in **37.7 minutes**. Every cell refitted; none was falsely reused, because the ablation checkpoint's
`CellIdentity` carries `incumbent_columns` and `candidate_columns` and therefore rejected all eighty-four
28-column cells on its own. That is the same stage the three defective caches in `backlog.md` sit
beside, and it is the one that gets this right.

Positive means removing the group made the model worse, so a larger number is a more valuable group:

| group | columns removed | mean delta vs full | 95% interval |
|---|---:|---:|---|
| `barrier_trial` | 3 | +0.000772 | +0.000053 to +0.001929 |
| `pace` | 3 | +0.000673 | +0.000047 to +0.001395 |
| `preparation` | 6 | +0.000536 | -0.000058 to +0.001059 |
| `race_card` | 10 | +0.000430 | -0.000039 to +0.001256 |
| `body_weight` | 1 | +0.000422 | +0.000181 to +0.000929 |
| `historical_market` | 2 | +0.000316 | -0.000183 to +0.000853 |
| **`gear_intervention`** | **6** | **+0.000276** | **-0.000062 to +0.000566** |

**The corroboration is the point, and it was not designed as one.** The promotion was decided on a
paired candidate measurement that put the block at `-0.0002790`, and the production refit delivered
`-0.000299`. This ablation asks the opposite question on a different construction -- refit the promoted
model *without* the group and see what it costs -- and answers **+0.000276**. Three estimates of the
same quantity by three routes, agreeing to within 8%. Nothing in the ablation was tuned to reproduce
the promotion figure; it is the same seven-group procedure that ran in August, pointed at the new
group set.

Two honest qualifications. The block is the **least valuable of the seven**, which is exactly what a
promotion at 56% of the bar predicts, and its interval **includes zero** -- as do three of the other
six, including `race_card`, so that is a property of this method's power at twelve years and five
seeds rather than a verdict on the gear block. And the ablation cannot separate the six columns from
each other: a group is the unit, which is the reason `gear_intervention` was kept as its own group
rather than folded into `preparation`.

## 2026-09-04 The gear promotion: the same numbers, a different bar

**The decision, and whose it was.** The six gear-intervention columns were added to
`production.json`. The measurement behind them is the one recorded above and was not re-run to
support the promotion: `-0.0002790` equal-cell over 55 cells, favourable in **5 of 5 seeds** and 7 of
11 years, 95% paired interval `-0.000486` to `-0.000024`. That interval excludes zero. What changed
is the rule, not the evidence -- the pre-registered `-0.0005` promotion bar was judged too stringent
for an effect that is small, reproducible and sign-stable, and the block was promoted at 56% of it.

**Why this block and not C4.** The distinction the earlier entry drew is exactly what the relaxation
turns on. C4 was inconsistent *and* tiny -- 5/11 years, 1/5 seeds, an interval spanning zero -- and a
more generous bar does not reach it, because there is nothing there to reach. This block is
consistent and tiny. A bar that rejects both treats "too small to matter" and "indistinguishable
from noise" as the same finding, and they are not.

**What the promotion costs, stated rather than discovered later.**

* **Two of the six are near-collinear by construction.** `gear_true_change` and
  `gear_intervention_size` are non-zero on exactly the same rows, and correlate 0.690 and 0.650 with
  the `gear_change` bit that stays in the model beside them. The block was measured as a block, so
  its result is a statement about all six together and not a licence for any one of them.
* **`gear_first_n` measured flat** -- probability effect 0.00000 on the pre-promotion run -- and
  `gear_vision_first` rests on a judgement about which item codes affect vision, pinned in
  `GEAR_VISION_ITEMS`. Both ride in on the block's result rather than their own.
* **Two columns are non-stationary.** `gear_first_n` rises from 0.0943 in 2010 to 0.1436 in 2024 and
  `gear_load` from 1.01 to 1.35, as declaration coverage rose. A pooled mean over the full span mixes
  two coverage regimes, and the model now trains on that.
* **The `gear_change` replacement is still open**, and the promotion makes it more attractive rather
  than less: production now carries the defective bit *and* its corrected form. `backlog.md` has the
  cost.

**What was retired by it, and why that was forced.** Promotion made the four gear candidates
unrunnable -- `feature-iterate` appends a candidate's columns to the incumbent, so a candidate whose
columns are already in the incumbent scores an arm identical to the incumbent and a delta of exactly
zero for the wrong reason. `validate-configs` refuses that outright, which is how it was caught
rather than run. The four candidates, the `production_plus_gear` variant set and its two experimental
run configs are in `evidence/archive/feature-research`. `production_plus_gear` was retired because the
arm it existed to score -- the incumbent *plus* the gear block -- is now the incumbent; note it is not
a copy of production but production's 33 columns **plus `trip_excuse_ewm`**, at 34, because it was
written against the pre-removal incumbent. `production_minus_race_constant` was resynchronised from 21
to 26 columns (33 - 7) so that its name kept meaning "production minus the race-constant seven"; the
2026-09-03 run that fitted its 21-column form is preserved beside a `STALE.md` and is *not* re-runnable
as the same test.

**`trip_excuse_ewm` was removed in the same change.** Production went 28 → 34 → **33**: the six gear
columns were added and `trip_excuse_ewm` was taken out, both on 2026-09-04. The two are independent
decisions that happen to share a generation, and they are recorded separately because they are not the
same kind of edit.

The removal is a **real subtraction, not a cleanup.** Measured on the 34-column run before it was
taken out, `trip_excuse_ewm` ranked **18th of 33** correction features by mean absolute correction at
0.012416 -- above `closer_setup`, `distance_log` and `gear_change`, and an order of magnitude above the
seven race-constant columns (ranks 23, 26-29, 31, 32) whose leave-one-out probability effect is exactly
0.000000. So this is not the race-constant case, where removal was free by construction and still made
the model worse; here there was something to lose. What it cost is measured in the generation this
entry heads, and the honest reading of the column's own caveat is that its value was always partly a
text-coverage artefact: the 0.0 default means "no excuses recorded", which is also what a missing text
source produces, and comment coverage is not uniform across the sixteen seasons.

Removing it emptied the `race_text` group, which held that column alone. The group was dropped from
`groups.json` rather than left declared and empty, because an empty group classifies nothing and would
still be offered as an ablation arm. The group count is therefore still seven -- through two offsetting
edits rather than by standing still, which is worth saying because a reader checking the count would
otherwise conclude nothing had changed.

The column's dictionary entry was **moved rather than deleted**, into a new *Retired columns* section
of the modelling guide at a heading level the coverage tests do not read as production. It still exists
in `race_features.parquet` and in `ALL_CANDIDATE_FEATURES`, so a candidate can still name it and the
definition is still worth having; what must not happen is a reader believing the production model reads
it, which is the drift direction `test_no_documented_feature_is_absent_from_the_manifest` exists to
catch.

**A latent hazard the regeneration exposed.** The per-cell prediction checkpoint is keyed on
`y{test_year}__seed{seed}` alone -- `_cell_stem` in `workflows/stages/predictions.py` -- and does
**not** include the feature-set digest. Re-running a root after a feature-set change therefore
resumes 28-column fits and relabels them as 34-column ones, silently and quickly. The caches were
cleared by hand before this generation. `refuse_mismatched_reuse` guards the stage's declared inputs
and does not reach inside the cell directory, so this is a real gap rather than a procedural note; it
is in `backlog.md`.

## 2026-09 Pace composition x own style (canonical block C4): a null, and where the research effect went

**The claim tested.** A feature-research campaign in a separate repository took six candidate
blocks to canonical scale (8 years x 5 seeds x 400 rounds, 40 paired cells) and exactly one
survived: `pace composition x own style`, at a paired racewise log-loss delta of **-0.0014903**,
t = -1.25, 25/40 cells and 6/8 years favourable, with a development-scale effect that carried into
canonical intact. That is 32.9% of the production model's entire edge over the market
(-0.004533), and it was the most novel of the finalists against the incumbent's 28 columns. The
plan for this work is `docs/plans/` 2026-09-01; the implementation is
`src/hkjc/features/pace_composition.py`.

**The mechanism, which is the reason the block is four columns rather than one.** A racewise
softmax is invariant to anything constant within a race, so the *level* of a projected pace
cancels out of the ordering however good the forecast is -- which is what every previous pace
attempt in this programme measured. What does not cancel is the projected shape multiplied by
this runner's own race-centred prior style. Three of the four columns are that product; the
fourth is the runner's own early-speed score less its race mean.

**Three corrections applied before anything was measured**, each a place where the research
number used information a pre-race feature cannot have:

| Correction | What it was | What it is here |
|---|---|---|
| the forecast's track-speed term | `race_track_speed`, the field's own mean sectional residual **in the race being predicted** | a strictly-prior per-course/going estimate from completed meetings only |
| `z_pace_hat`'s scale | standardised over the whole panel, future years included | training-fold moments, applied forward |
| missing values | a full-panel median | a declared neutral 0.0, measured and recorded |

The first one is the load-bearing one and its cost is measurable rather than speculative: the
pace forecast's out-of-sample correlation with the realised early pace falls from the research
build's **r = 0.680 to r = 0.337** (R^2 = 0.086 on 10,098 races). The forecast was reading a
partial answer to its own question. Study II of the same campaign had independently reached
r = 0.414 for a legitimate field-composition forecast, so 0.337 is the plausible band and 0.680
was not.

**The result, on this repository's own chronology and seeds** (`canonical_confirm`: 12 test years
from 2015 with 2015 a fitted warm-up, 5 seeds, 400 rounds, 55 scored cells, complete):

| Quantity | Research canonical | Measured here |
|---|---:|---:|
| equal-cell mean delta | -0.0014903 | **+0.0000659** |
| race-weighted mean delta | not reported | **-0.0000226** |
| years favourable | 6/8 | 5/11 |
| seeds favourable | not reported | 1/5 |
| worst year | 2021, +0.00913 | 2020, +0.00130 |
| paired race-level 95% interval | not reported | -0.000301 to +0.000232 |
| share of the production edge | 32.9% | 1.5% at most, and not distinguishable from zero |

Retention from the research canonical figure is **-4% on the equal-cell mean and +2% on the
race-weighted one**. The recommendation is `reject`, the pre-registered thresholds
(`MATERIAL_DELTA = -0.0005`, a 60% sign majority) are not met on any reading, and
`configs/features/production.json` was not edited at any point.

**The two means disagree in sign, and that is worth a sentence rather than a footnote.** The
equal-cell mean gives every (year, seed) the same weight; the race-weighted mean gives every race
the same weight. 2026 holds 201 races against 744-799 for every other year, and it is one of the
block's worst (+0.00129), so it carries 1/11 of the equal-cell mean and 1/40 of the
race-weighted one. Reporting either as though it were the other is how this block could be
written up as marginally positive or marginally negative at will. It is neither: the paired
interval spans zero and 1 of 5 seeds favours it.

**The redundancy, measured on the shipped table rather than assumed.** Production already carries
`early_style_ewm`, `pace_pressure` and `closer_setup`. Against `early_style_ewm`,
`pacecomp_lead_x_style` correlates 0.77, `pacecomp_entropy_x_style` 0.79 and
`pacecomp_own_speed_vs_field` 0.82. Only `pacecomp_pace_x_style` is substantially novel, at 0.18
-- and that is the column whose forecast lost half its correlation to the P0.1 correction. So the
one genuinely new thing in the block is the one the correction weakened, and the other three are
largely a restatement of a column the incumbent has had for years. The research campaign's own
novel-share measurement (0.682 for the pace cross, 0.317, 0.278 and 0.180 for the others) pointed
at the same conclusion from the other direction.

Correlation against `sectional_projected_pace` is exactly zero, and that number is arithmetic
rather than evidence: it is race-constant and all four new columns are race-centred. A test pins
it so nobody reads it as a redundancy finding.

**What this contributes, given that it is a null.**

* **A leak in a predictor can be the whole effect, and a plausible-looking forecast quality figure
  is where to look for one.** r = 0.680 for a pre-race pace forecast should have read as too good.
  The three corrections cost -0.001490 and the first is most of it.
* **Development-scale paired deltas remain uninformative at this effect size, in both
  directions.** The 15-cell `development` profile returned +0.000310 here against the research
  campaign's -0.0014519 for the same block -- a sign flip. The campaign had already recorded
  eight development-to-canonical retentions of 43%, 15%, -23%, 2%, -155%, -401%, -567% and +427%;
  this is the ninth, and the lesson has not changed. A development profile is for finding
  mistakes, not effects.
* **Redundancy with the incumbent, not novelty against nothing, decides whether a feature is
  worth anything.** Three of these four columns were a re-derivation of `early_style_ewm` by
  another route, and no amount of mechanism argument changes that.
* **One structural feature does replicate and is not noise.** 2020 and 2021 are the block's worst
  years in both studies (research 2021: +0.00913; here 2021: +0.00112, 2020: +0.00130). Whatever
  the crosses do, they do the opposite of helping through the disruption seasons. That is the only
  part of this block a future attempt should start from.

**What was kept.** The four columns remain in both feature tables and in
`ALL_CANDIDATE_FEATURES`, the cache builder is step four of the documented build sequence, and the
candidate manifest, its measured `missing_behaviour` and the pinned redundancy correlations are in
the tree. They are research inputs, not production inputs: nothing reads them at fit time unless a
candidate names them, and the cost of keeping them is one minute of build time. Re-running the
measurement takes `hkjc feature-iterate --candidate pace_composition_x_own_style --profile
canonical_confirm`, and the run's content digest is
`sha256:97fdf464dc3b18e009dd454f1e514aabf55793dff7a8571bb0a0b129685d3d28`.

## 2026-08 Phase 6: the normal choice link beats the Gumbel one, on both scoreboards

Replacing independent Gumbel choice shocks (softmax) with normal latent-performance
shocks (probit) improves held-out race log loss in both of the plan's paired
comparisons. Both are isolated *link* comparisons: the mean learner is held fixed and
the same raw utility rows feed both links, so the only thing that differs is the shock
distribution.

| Comparison | Delta | Honest years better | 80% day-block interval | After nested calibration |
|---|---:|---:|---|---:|
| `probit` vs `softmax` (standalone) | **-0.012557** | 11 of 11 | [-0.01403, -0.01007] | -0.007069 |
| `probit_offset` vs `offset` (market-relative) | **-0.000943** | 11 of 11 | [-0.00124, -0.00044] | -0.000493 |

Negative is better. Both intervals exclude zero favourably, every one of the five seeds
is negative in both comparisons, and no year degrades. For scale, the Offset model's
entire edge over the market is -0.004647.

**The shape of the answer.** The link buys an order of magnitude more where the model
must construct the whole ordering itself than where the market has already done most of
the work. That is coherent rather than surprising: a market price already encodes much
of the substitution structure between runners, so swapping the link on top of it adds
less. Standalone probit closes about 86% of the gap between standalone softmax (2.0434)
and the market (2.0288), reaching 2.0308 -- better than softmax, still short of the
market.

**The utility scale is the whole game, and theory gets it wrong.** It is the single free
parameter of the iid link. Matching a logit's difference variance (pi^2/3) to a probit's
(2) suggests `sqrt(2/(pi^2/3)) = 0.78`; the walk-forward fit on real margins says about
2.1 for the offset side and 1.43 for the standalone side. At 0.78 the offset candidate
*loses* by +0.016, at 1.0 by +0.005, at 2.0 it wins, and by 40 it has converged back
toward the market. A run that took the theoretical value on faith would have reported
the wrong sign. The fitted values are stable across years (offset 1.99-2.23, standalone
1.39-1.45) despite being refitted each year on that year's history alone, so they are
not tracking noise.

**The plan's specified integrator could not have found this.** QMC winner-counting over
`m` draws can only land on a multiple of `1/m`, so at the plan's canonical 2,048 draws
nothing below 4.9e-4 is resolvable -- and HKJC longshots sit at 1e-3 to 1e-4, which is
exactly the range race log loss is dominated by. Measured on the canonical 2024 seed-7
slice, the QMC loss was still drifting with budget (2.333 at 512 draws, 2.196 at 2,048,
2.151 at 32,768) and its 4-replicate standard error at 32,768 was 0.0032 -- 3.4x larger
than the -0.000943 effect. Promotion gate 6 demands the improvement exceed integration
error by 10x; QMC fails it by a factor of three at any affordable budget, because the
error falls only as `m^-0.5` for a discontinuous `argmax` integrand.

For iid normal shocks the winner probability collapses to a one-dimensional integral
over the winner's own shock:

```text
P(i wins) = E_z[ prod_{j != i} Phi(m_i - m_j + z) ]
```

so Gauss-Hermite quadrature gives it outright. It matches the analytic binary result to
1.1e-16, resolves a probability of 1.1e-9, and runs 30x faster than QMC at 32,768 draws.
Gate 6 then passes by seven orders of magnitude. With shared factors the runners are
conditionally independent given the factor draw, so the integral nests and the structured
variants are exact too. QMC is retained as an independent cross-check -- it shares no code
with the quadrature path beyond the batch, and the two agree to 1.8e-4 on real utilities,
consistent with QMC's own reported error.

**The market anchor is not optional.** Direct probit does not reproduce the market from
the market's own utilities: on real margins `C_probit(log q, Sigma)` differs from `q` by
up to 0.20, where `softmax(log q)` is `q` exactly. The unanchored diagnostic therefore
scores 2.0424 against the incumbent's 2.0254 -- losing by four times the model's whole
edge, entirely from a baseline shift that would have been attributed to the link. The
anchored form `normalize(q * p1 / p0)` returns `q` to 1.1e-16 at zero correction.

### Limitations

- Both results are **link-only**. The mean learner is the existing softmax or Offset
  fit; no native joint probit likelihood was trained, and metadata records
  `mean_training=softmax_surrogate` for that reason.
- The utility scale is fitted on earlier *test* years, which are themselves development
  data under the standing limitation below. This is not a locked prospective result.
- The first year has no history and falls back to the theoretical scale; it is labelled
  `insufficient_history` and excluded from the headline. Including it, the offset delta
  is +0.001017 -- which is how much the theoretical scale costs.
- **Structured covariance does not help.** A one-factor pace variant loading on
  standardised `early_style_ewm`, with its scale chosen from a predeclared grid on
  earlier years, comes out *worse* than iid: -0.000785 against -0.000943 raw and
  -0.000341 against -0.000493 calibrated. It degrades one year past the +0.001
  tolerance where iid degrades none, and its interval is wider. The grid selected "no
  factor" in 7 of 11 years, and the entire grid from scale 0.00 to 0.80 spans about
  2e-5 of log loss -- so the factor scale is essentially unidentified by this data.
  The improvement is attributable to the normal idiosyncratic distribution, not to
  correlated pace shocks. A two-factor variant is implemented and tested but not run
  canonically: at ~21 minutes per run there is no reason to spend it when the
  one-factor grid is this flat.
- Promotion gates 7 (an interaction diagnostic), 9 (clean-commit reproduction) and 10
  (prospective confirmation) are not yet satisfied. Production remains the five-seed
  Offset ensemble.

---

## 2026-08 Landing-layer duplicates in 2026, absorbed by an unasserted safety net

Found while validating primary keys in the Phase 4 scraping work. **The model
numbers are not affected** — but the reason they are not is worth knowing.

`data/historical-data/aspx-results/aspx-results-2026.csv` holds 4,184 rows for 213
races — 19.64 runners per race, where a Hong Kong race fields about 12–14. 1,513 of
those rows are **byte-identical duplicates**: all 21 columns match a twin. The cause
is a re-scrape that rewrote the year's file instead of merging into it.

They do not reach the feature table. `features/build.py:1777` does
`df[~df.index.duplicated(keep='first')]` immediately after
`set_index(['race_id', 'Horse No.'])`. Verified rather than assumed:
`race_features_extended.parquet` has **zero** duplicate `(race_id, Horse No.)` keys
in any year, 2026 shows 2,527 rows over 201 races = 12.57 runners/race, and each of
the 1,437 duplicated landing keys that survive filtering appears exactly once. So
`mean_delta_vs_market = -0.003685` is computed on clean data and **no rebaseline is
required.**

The residual issue is the safety net itself:

- It announces the problem only as a `print` — the Phase 4 T2 build log contains
  `Dropped 1501 duplicate (race_id, Horse No.) entries.` on line 46 of
  `prep-data.log`, inside a 19-minute build. The pipeline has been reporting this
  for as long as the defect has existed and nobody read it.
- It is `keep='first'`, so which copy survives is positional. Harmless while the
  copies are byte-identical; silent and arbitrary if they ever diverge.
- Nothing asserts it. Compare `formline_shared_races` at `build.py:483`, which
  raises an explicit `AssertionError` if the dedup it depends on ever moves — the
  codebase already knows this pattern, it just is not applied here.

Phase 5 step 5.0 (swallowed exceptions → recorded diagnostics) is the right home
for promoting that count into a real diagnostic. Phase 4 removed the *cause*:
`hkjc.collection.orchestrate` merges a re-scrape into the existing year by
`primary_key` with `keep="last"`, and `audit_keys` reports the existing duplicates
in `data/historical-data/SCRAPE_COVERAGE.md`.

## 2026-08 Rebate-Aware Kelly

The Kelly objective now includes the HKJC loss rebate, so bet sizing and
settlement optimise the same growth function. Previously the optimiser was
rebate-blind and the rebate was added afterwards as a bankroll credit, meaning
"with rebate" described a strategy nobody would run: rebate-blind stakes plus a
rebate cheque.

### Confirmed rebate terms

The rebate pays once the loss reaches HK$10,000: 10% on Win/Place, 12% on
Quinella/Quinella Place. This model bets WIN only, so 10% is the live rate.
Two corrections followed:

- The gate is a **cliff**, not an excess. At a $10,000 loss the rebate is 10% of
  the whole loss, a discontinuous $0 → $1,000 jump. `betting.py` was right and
  the `.tex` write-up (`0.1 × max(0, loss − 10k)`) was wrong.
- The base is the **race loss**, not losing turnover. The old
  `(stakes × (1 − won)).sum()` paid a rebate even on net-*winning* races.
  Corrected to `max(0, staked − returned)`; the legacy behaviour survives as
  `basis='losing_stake'` purely to reproduce published figures.

### Why this is a bankroll question, not a rate question

Model probabilities are normalised to sum to 1 while `b = Win Odds − 1` stays
un-devigged, and mean race overround is `1.2280`. A model that reproduces the
devigged market therefore scores `edge = 1/1.228 − 1 = −18.6%` on every runner,
and clearing `edge > 0` needs a lift ratio above `1.228` against a realised
Offset lift of only p99 `= 1.128`. That is why the model placed 15 bets in
107,038 runner-rows.

A thresholded rebate does **not** shift the marginal break-even: at `f → 0` the
loss is infinitesimal and never clears the gate, so `dG/df|₀` is unchanged. The
rebate is a *non-local* incentive to stake enough to clear the gate, which makes
the objective non-concave and requires a multi-start solve; a single solve from
`f = 0` finds nothing. Because the gate is a fixed dollar amount while Kelly
stakes scale with the bankroll, reachability is governed by `t = 10,000 / B`.

### Result (bagged Offset, 2015-2026, retrospective final odds)

| Pool | `t` | Kelly | Bets | Races | Turnover | Rebate | ROI on turnover |
|---|---|---|---|---|---|---|---|
| baseline (blind) | — | 0.01 | 13 | 13 | 4,470 | 0 | +0.93% |
| HK$10M | 1e-3 | 0.05 | 178 | 178 | 1.90M | 143,879 | +1.75% |
| HK$100M | 1e-4 | 0.01 | 405 | 404 | 5.29M | 411,108 | −1.10% |
| HK$1B | 1e-5 | 0.01 | 1,054 | 1,049 | 61.3M | 4,837,215 | −2.04% |
| HK$1B | 1e-5 | 0.10 | 1,225 | 1,219 | 617.4M | 48,714,886 | −2.01% |

Activity rose from 13-15 bets to ~1,200, an ~80x increase, and the share of
solves won by a gate-clearing seed tracks pool size exactly as the mechanism
predicts: 34% at $10M, 69% at $100M, 92% at $1B. `rebate_model_error` falls to
exactly 0 at $1B, confirming sizing and settlement agree once the gate is
comfortably cleared.

**The extra bets are not profitable.** At scale, ROI on turnover settles at
about −2%. The rebate recovers roughly 10 of the ~18.6 percentage points of
takeout, which converts a heavily losing book into a mildly losing one — it does
not manufacture an edge. The few positive cells ($10M at Kelly 0.02 and 0.05)
rest on 30 and 178 bets and should not be read as signal; note also that ROI is
not monotone in the edge threshold, consistent with the weak residual ordering
recorded below.

Sizing deliberately over-bets to clear the gate, which is +EV on the rebate
alone and −EV on the underlying wager. That is a faithful consequence of the
real terms, not a modelling artefact.

These use retrospective final odds for sizing and official WIN dividends for
settlement. They are diagnostics, not executable returns.

### Reproducing

```bash
# published behaviour, bit-identical
python -m hkjc.suites.return_suite --reuse-predictions --rebate-basis losing_stake

# rebate-aware sizing
python -m hkjc.suites.return_suite --reuse-predictions --rebate-aware-sizing \
    --rebate-basis net_loss --rebate-threshold-mode cliff --initial-bankroll 1000000000
```

`rebate_aware_sizing` defaults to off, so every prior figure reproduces. A
golden-value regression in `tests/test_rebate_policy.py` pins that exactly.

## 2026-08 Point-In-Time Feature Iteration

This section supersedes the old production recommendation below. The older
notes remain as research history but their ROI claims are not comparable after
race, settlement, and Kelly corrections.

### Dynamic horse-memory research

A new date-atomic state engine now replays all runners by complete race date.
It emits every pre-race snapshot before applying any outcome from that date and
maintains these separately ablatable states:

- Bayesian opponent-adjusted rank ability with broad surface/distance/venue
  aptitude, posterior covariance, mean reversion, and short performance
  innovation.
- A market-independent dynamic speed mean, short speed innovation, and
  uncertainty based on strictly prior context-standardized finish speed.
- Explicit historical market-mispricing memory, kept separate from the core
  ability state.
- Reliability-weighted early-sectional strength, including exact-field
  relative values.

The first direct-tree screens showed why engineered memory cannot simply be
added as a large feature family. Rank state improved winner residual AUC but
worsened log loss; compact dynamic speed improved only 15/27 year-seed runs;
and unrestricted combinations overfit 2025.

`src/hkjc/suites/memory_residual_suite.py` therefore fits small bounded utility tilts
against the incumbent model's chronological OOF probabilities. Across
2018-2026 and seeds `[7, 17, 42]`, the single reliability-weighted early
sectional tilt produced:

- Mean candidate-minus-incumbent race log loss: `-0.000211`.
- Better log loss in 23/27 year-seed runs.
- Improvements in eight of nine yearly seed averages; 2020 regressed by
  `+0.000997`, while 2025 was effectively neutral at `-0.000012`.
- Stable standardized coefficients from `0.016` to `0.025`, far inside the
  configured `[-0.35, 0.35]` bound.

Adding short dynamic speed improved the mean delta to `-0.000249` but only
18/27 runs, so the single sectional correction is the more stable research
candidate. Neither layer is promoted to production yet. All years are reused
development data, and the residual-ordering metrics did not improve alongside
log loss.

Artifacts:

- `results/contextual_feature_suite/memory_residual_validation/`
- `results/contextual_feature_suite/memory_residual_pair_validation/`
- `results/contextual_feature_suite/horse_memory_validation_2018_2022/`
- `results/contextual_feature_suite/horse_memory_validation_2023_2026/`

The independently cacheable workflow is:

```powershell
python -m hkjc.features.build_horse_state_cache
python -m hkjc.features.materialize_horse_state
python -m hkjc.suites.memory_tilt_diagnostic
python -m hkjc.suites.memory_residual_suite --years 2018,2019,2020,2021,2022,2023,2024,2025,2026 --seeds 7,17,42
```

### Independent base and margin-aware iteration

The independent model was expanded beyond one-hot winner training and generic
EWMs:

- A strict inference allowlist prevents current or historical market-derived
  columns from entering the independent feature matrix.
- Historical market distributions were tested as dense training targets. They
  did not improve hard winner log loss and were rejected.
- A 10% dense finishing-order target improved the selected independent model
  without using odds.
- Opponent-adjusted beaten-margin state now distinguishes close and dominant
  performances using 99.8%-covered parsed `LBW` histories.
- Official rating-plus-carried-weight residual states expose persistent
  performance beyond handicap expectations.
- A new 2008-2026 sectional observation cache reconstructs physical first-
  section lengths, quality-controls times, standardizes absolute race pace on
  complete prior-date contexts, and maintains separate absolute pace,
  field-relative speed, workload, and late-sustain states.

Development selection on 2018-2021 chose rank-plus-margin memory with 10%
placing supervision. Locked 2022-2026 evaluation over seeds `[7, 17, 42]`
produced:

- Old strict independent base mean log loss: `2.213981`.
- Rank-plus-margin independent mean log loss: `2.193902`.
- Improvement: `-0.020079` per race across 15 year-seed runs.
- Remaining delta versus market: `+0.157366`; the independent model is much
  stronger but still not competitive with current odds.

A nonlinear residual mapper could beat the raw market in some later regimes,
but it was unstable and did not improve the incumbent Offset. Probability-ratio
tails remained far below the approximately `1.227` takeout hurdle.

The final constrained Offset frontier separates probability and ordering:

- Sectional reliability tilt: `-0.000211` log loss versus incumbent, 23/27
  wins, but lower residual AUC.
- Margin-rank tilt: `+0.000027` log loss, but residual AUC improved from
  `0.724153` to `0.732025`.
- Sectional plus margin-rank tilt: `-0.000174` log loss, 21/27 wins, residual
  AUC `0.724131` (effectively incumbent-level).

No new layer is promoted automatically. The independent-base improvement is
material, but it has not yet produced takeout-scale calibrated residuals or a
stable improvement over the incumbent market model.

Additional artifacts:

- `results/distilled_independent_suite/development_feature_target_screen/`
- `results/distilled_independent_suite/rank_margin_evaluation_2022_2024/`
- `results/distilled_independent_suite/rank_margin_evaluation_2025_2026/`
- `results/contextual_feature_suite/sectional_margin_rank_validation/`
- `data/processed/cache/sectional_observations.parquet`
- `data/processed/cache/sectional_memory_state.parquet`

### Promoted result

- Model: racewise XGBoost market offset using `log(q_market)` plus learned
  corrections and softmax winner likelihood.
- Inputs: 28 total: 2 market, 10 race-card/context, and 16 promoted correction
  features.
- Calibration: chronological expanding-window predictions; no random-fold or
  in-sample isotonic calibration.
- Evaluation: expanding-year tests from 2015-2026, seeds `[7, 17, 42]`, 100
  rounds, probability metrics only.
- Result: 35/36 individual year-seed runs and all 12 bagged yearly runs beat
  the market on race multinomial log loss.
- Mean bagged candidate-minus-market log loss: `-0.004064` per race.
- Bagged year-level 95% interval: `[-0.005574, -0.002555]`.
- All 12 yearly seed averages beat the market.
- Mean calibration slope: `0.9926`.

Artifacts:

- `results/promoted_suite_no_speed_sectional_2015_2021/`
- `results/promoted_suite_no_speed_sectional/`
- `results/promoted_ablation.csv`
- `results/feature_audit/`

### Promoted correction blocks

- Preparation: layoff, class/rating/distance changes, gear change, and carried
  weight versus the horse's own history.
- Historical market residuals: strictly prior-date trainer-track and jockey
  performance relative to prior market expectations.
- Pace: structured early-position history, projected pressure, and closer setup.
- Trip context: prior comments-derived trouble state.
- Barrier trials: effort-adjusted finishing state, margin, and recency.
- Body condition: horse-relative declared-weight surprise without target encoding.

The new speed and sectional blocks remain available in the 44-column
fundamental research set, where they are strongly predictive without current
odds. They were removed from the promoted market-edge correction because
actual-model ablation found no stable incremental value after the market and
other feature blocks were present.

### Correctness changes

- Raw duplicates are removed before odds normalization.
- Malformed and incomplete race fields are rejected atomically.
- Trainer and all target aggregates update after complete dates, never by runner
  row order.
- Corrupt historical race incidents are quarantined.
- Sparse cohort-selected trackwork/vet/movement data is excluded from the
  long-history core model.
- The Kelly optimizer now includes the probability that an unbet runner wins.
- Dividends join by race and runner without expanding evaluation rows.
- Feature promotion is based on paired race log loss, not bankroll ROI.

### Remaining limitation

All years through 2026 are development data, and result-page odds are final
odds rather than timestamped executable prices. This is substantial probability
progress, not proof of a tradable edge. The next confirmation step is a locked
prospective shadow test against archived pre-race snapshots.

### Corrected retrospective returns

`src/hkjc/suites/return_suite.py` retrains the promoted model walk-forward for every
year, averages seeds `[7, 17, 42]`, sizes bets with the corrected multinomial
Kelly objective, and settles against official WIN dividends.

Bagged results over 2015-2026:

| Kelly fraction | No-rebate bankroll ROI | With-rebate ROI | Max DD no rebate |
|---:|---:|---:|---:|
| 0.01 | -0.029% | -0.029% | 0.040% |
| 0.02 | -0.057% | -0.057% | 0.080% |
| 0.05 | -0.143% | -0.143% | 0.200% |
| 0.10 | -0.285% | -0.271% | 0.400% |

At Kelly `0.02`, the bag placed only 10 bets across 10 races, turned over
`$9,040` from a `$10m` starting bankroll, and lost `$5,681.50`. Equal-stake ROI
on all positive-edge selections was `-13.5%`.

This final result follows two additional integrity corrections:

- Races with reciprocal WIN-odds sums below `1.10` are incomplete markets and
  are rejected before feature construction. Earlier apparent profits were
  dominated by synthetic arbitrage in these partial fields.
- DNF/PU/DISQ runners remain in the pre-race field as losing starters. Removing
  them using post-race status was field-definition look-ahead.

The corrected conclusion is that probability quality improves consistently,
but the improvement is not large enough to overcome HKJC takeout. The bagged
model still beat market race log loss in all 12 yearly tests, averaging
`-0.004064` per race with mean calibration slope `0.993`, but it did not produce
positive retrospective betting returns.

These returns use final result-page odds for model input and sizing. They are
useful diagnostics, not executable historical returns. No Kelly level or edge
filter should be selected from this reused development history.

### Model-design coverage experiment

`src/hkjc/suites/model_design_suite.py` tested whether low activity was specific to
the offset architecture. The design split used 2018-2021 as development and
2022-2026 as later evaluation, with identical settlement and final odds.

- Direct market-aware softmax produced 5,485 apparent positive-edge bets across
  80.0% of later races, but worsened market log loss by `+0.007069` and lost
  `-43.5%` per flat stake.
- Independent fundamental softmax produced bets in every race, but trailed the
  market by `+0.177288` log loss and lost `-27.6%` per flat stake.
- Scaling the offset correction by `2x` retained a `-0.002897` log-loss
  improvement, produced 738 bets across 721 later races, and lost `-15.8%` per
  flat stake.
- Scaling by `2.5x` covered 1,476 later races and was approximately neutral on
  log loss (`-0.000070` versus market), but still lost `-11.2%` per flat stake.
- At `3x`, coverage rose to 65.3% while probability quality became worse than
  market and flat-stake ROI remained `-11.5%`.

The controlled conclusion is that offset anchoring explains the original low
bet count, but removing or amplifying the anchor creates activity faster than
it creates incremental pricing power. With the current features, the `2x`
residual is a useful high-coverage research challenger, not a promoted betting
model. The next feature iteration should target market-independent information
that improves the ordering of model-market residuals; changing Kelly or merely
increasing probability amplitude cannot supply that information.

Artifacts:

- `results/model_design_suite/summary.csv`
- `results/model_design_suite/yearly.csv`
- `results/model_design_suite/edge_buckets.csv`
- `results/model_design_suite/kelly_race_paths.parquet`
- `results/model_design_suite/kelly_bets.parquet`
- `results/model_design_suite/model_design_report.html`

### Contextual feature batch 1

The first context-rich feature rebuild retained the promoted model as the
benchmark and expanded the audit dataset from 66 to 129 non-market candidates.
It added:

- full course/rail and surface-specific going state;
- raw, median-relative, rank, and field-dispersion rating/weight state;
- draw rank among actual runners;
- explicit debut, uncapped layoff, and career reliability state;
- separate position-derived and comment-derived pace systems;
- body-weight percentage change, prior median, surprise, and layoff interaction;
- shared-race formline comparisons with 84.6% runner coverage;
- recency- and distance-conditioned barrier-trial readiness and reliability.

Correctness repairs include deterministic tied market ranks, an unambiguous AWT
course category, removal of generic `held up` from the trouble regex, use of the
source barrier-trial race key, and elimination of the silent `pace_pressure` /
`closer_setup` overwrite. The prior comment-derived pace remains under explicit
legacy names so incumbent comparisons are reproducible.

The serious validation used the actual Offset model over 2018-2026, seeds
`[7, 17, 42]`, 100 rounds, and chronological calibration. No dividends or ROI
were loaded during selection.

- The full contextual card block was too broad and unstable.
- Course/going context had a small mean gain in an interim validation but did
  not improve opportunity coverage.
- Shared-race pairwise and transition blocks were strongly fundamental but not
  stable enough relative to the market for promotion.
- Contextual barrier-trial additions did not beat the already strong incumbent
  trial block.
- The compact rating/weight block increased `2x` race coverage from 24.6% to
  26.6% and `2.5x` coverage from 47.7% to 49.6%, but its mean paired log-loss
  change versus incumbent was `+0.000015` and only 13/27 runs improved.

No batch-one candidate is promoted. The useful conclusion is that decomposed
context can move coverage without probability amplification, but static card,
formline, and trial summaries still do not improve residual ordering reliably.
The next batch should target condition-matched sectional energy and pace setup,
where the existing generic features have strong fundamental signal but weak
incremental market value.

Final batch-one artifacts:

- `results/contextual_feature_suite/final_screen/`
- `results/contextual_feature_suite/final_rating_validation/`
- `results/feature_audit/contextual_batch1_screen/`

---

## TL;DR — Current best config

> **Superseded.** This configuration is not the production model. It describes 25 features,
> Kelly 0.02-0.03 and cumulative returns from a settlement that predates the current
> `min_overround` filter and the rebate policy. Production today is the 400-round, 5-seed,
> 28-input Offset model at Kelly 0.01; its settled numbers are in
> `results/reports/bagged_offset/report.html`, which places 339 bets across twelve years for a
> return within a hundredth of a percent of break-even. The caveat immediately below -- that
> bag_A was the luckiest of five meta-sets -- is the part of this section that has aged well,
> and it is why seed choice is now recorded in every run lock.


- **Model:** Offset (market as `base_margin`) + 5-seed bagging
- **Seeds:** `[7, 17, 42, 123, 256]`
- **Params:** `lr=0.01, depth=5, min_child_weight=20, subsample=0.7, colsample_bytree=0.7, gamma=0.2, reg_alpha=2.0, reg_lambda=10.0`, 400 rounds (deep_slow)
- **Features:** 19 edge + 4 race card + 2 market = **25 total** (see feature set below)
- **Kelly:** 0.02 for consistency / 0.03 for max return

### Results (2015-2026, compounded; 2026 partial)

**⚠️ Honest caveat first:** the bag_A seeds `[7, 17, 42, 123, 256]` that we use as production were retested against 4 other independent 5-seed sets. **Bag_A was the luckiest tail** — across 5 meta-sets the mean cum ROI at K=0.02 is **+107%**, with std 130%, min **+24%**, max +332% (bag_A). So production's bag_A numbers below represent the top of the distribution, not the expected case.

Headline numbers (bag_A, lucky tail):

| Kelly | Cum no_rebate | Cum with_rebate | Pos no_rebate | Pos with_rebate |
|---|---|---|---|---|
| 0.01 | +139.47% | +260.26% | 9/12 | 11/12 |
| **0.02** | **+332.53%** | +880.08% | **9/12** | **11/12** ← bag_A |
| 0.03 | +522.74% | +2027.26% | 7/12 | 10/12 |
| 0.05 | +644.80% | +5701.83% | 7/12 | 9/12 |

**Realistic production expectation (mean across 5 independent 5-seed bagging sets, K=0.02):**

| Stat | cum_no | cum_wr |
|---|---|---|
| MEAN | +107.18% | +366.46% |
| STD | 129.60% | 295.37% |
| **MIN** | **+23.99%** | **+178.64%** |
| MAX | +332.48% | +879.87% |

**All 5 bagging sets positive (min +24%).** This is the genuine win: bagging reliably eliminates catastrophic losing seeds (single-seed min was −19%), even if it doesn't reliably deliver the bag_A bonanza.

Out-of-sample: 2025 slight negative (~−2% at K02 averaged across meta-sets), 2026 partial positive across all 5 meta-sets (+1% to +7%).

### Horse form v2 (latest production update)

- **Production Offset form block:** `legacy_recent_form`, `legacy_career_beat_odds`, `form_surprise`, `form_stability`
- **Softmax best test block:** `form_fast`, `form_cycle`, `form_surprise`
- The horse-form redesign replaces the old block:
  - `recent_form`, `form_vs_career`, `prev_win`, `career_beat_odds`
  with a latent-state block plus two explicit legacy anchors.

**Offset, yearly 2015-2026 vs old form block:**
- average log loss improved from `0.237417` to `0.237407`
- cumulative ROI with rebate improved from `+183.88%` to `+589.65%`
- LL improved in `7/12` years, so the improvement is not perfectly uniform, but the aggregate gain is strong enough to adopt in production

**Softmax, yearly 2015-2026:**
- best LL candidate: `condensed_stability`, `model_weight=0.45`, `alpha=0.0075`
- best return candidate: `condensed_triplet`, `model_weight=0.50`, `alpha=0.015`
- both beat the old form block on average, but neither is consistent enough year-by-year to treat as a locked production default yet

### Trainer/track v2 (latest iteration) ❌ NOT PROMOTED

- Tested continuous shrunk trainer features:
  - `trainer_resid_shrunk`
  - `trainer_context_resid`
  - `trainer_context_delta`
- Goal: replace brittle thresholded trainer block:
  - `trainer_track_spec`, `tt_gap`, `tr_gap`

Result:
- Screened on `2024-2026`, then checked targeted full-year variants.
- No trainer-v2 candidate beat the legacy trainer block on both LL and returns strongly enough to justify promotion.
- Current production remains the legacy trainer block.

Takeaway:
- Shrinkage was directionally sensible, but the current implementation likely throws away too much useful track-specific asymmetry that the old market-gap features still capture.

### Draw/setup v2 (latest iteration) ⚠️ SOFTMAX ONLY IMPROVEMENT

- Re-engineered `draw_outside_ST` / setup block with field-size-aware features:
  - `draw_pct`
  - `draw_style_tension`
  - `draw_setup_resid`
  - `draw_field_resid`
  - `draw_field_pressure`
  - `setup_weight_resid`
  - `setup_weight_tension`

Key result:
- **Offset:** legacy block still wins. Keep `draw_outside_ST + setup_weight_z` in production.
- **Softmax:** `draw_field_resid + setup_weight_z` is the best-tested replacement.

Second-pass summary:
- Offset baseline: `avg_log_loss=0.237411`, `cum_roi_with_rebate=+895.61%`
- Offset best replacement (`draw_field_resid + setup_weight_z`): `avg_log_loss=0.237415`, `cum_roi_with_rebate=+449.67%`
- Softmax baseline: `avg_log_loss=0.237351`, `cum_roi_with_rebate=-27.82%`
- Softmax best replacement (`draw_field_resid + setup_weight_z`): `avg_log_loss=0.237335`, `cum_roi_with_rebate=+1.05%`

Takeaway:
- For **Offset**, the old hard ST-outside heuristic still dominates once full walk-forward returns are considered.
- For **Softmax**, replacing the binary draw flag with a **field-size-aware draw residual** is a real improvement in both LL and returns.

### Running it

```bash
cd model/
python -m hkjc.suites.production
```

Generates `results/bankroll_viz.html` — interactive visualization with Kelly tabs + year tabs.

---

## Feature set (final)

> **Superseded as a specification.** The authority is
> `configs/features/production.json`, which declares 28 inputs of which 27 are learned --
> the market column is consumed as the booster's base margin and is not a learned feature. Two
> columns named here (`log_implied_prob`, `Draw`) are no longer inputs. The grouping below is
> still the grouping the ablation uses, which is why the section is kept.


### Market (2)
- `Implied_Prob` — market win probability (normalized per race)
- `log_implied_prob` — log of implied

### Race card (4)
- `Racecourse` — ST=1, HV=0
- `Track_Turf` — Turf=1, AWT=0
- `Draw` — starting stall (1-14)
- `Act_Wt` — weight carried (lbs)

### Edge (19)

**Class (1):** `class_change` = `prev_class − current_class`

**Weight (2):**
- `wt_z` — z-score of `Act_Wt` within current field
- `setup_weight_z` — z-score vs horse's own weight history

**Form (4, production Offset subset from horse-form v2):**
- `legacy_recent_form` — legacy avg(1/place) over last 3 races, kept as an anchor
- `legacy_career_beat_odds` — legacy expanding mean of `(win − implied_prob)`
- `form_surprise` — exponentially weighted market-relative outperformance
- `form_stability` — rolling spread of recent finish quality

**Trainer (3):**
- `trainer_track_spec` — trainer's track-specific WR minus overall WR
- `tt_gap` — trainer × track actual WR minus market-implied WR
- `tr_gap` — trainer overall actual WR minus market-implied WR

**Draw/Field interactions (2):**
- `draw_outside_ST` — binary: Draw ≥ 10 AND at ST
- `fav_field_size` — is_fav × field_size

**Barrier trial (2):**
- `bt_avg_early` — expanding avg front-running in trials
- `bt_last_behind` — time behind winner in last trial

**Sectional (4):**
- `trail_sect_closing` — avg closing speed vs race (trailing per horse)
- `trail_sect_peak_at` — avg position of peak speed (0=early, 1=late)
- `trail_sect_avg` — avg overall sectional speed vs race
- `trail_sect_best_gain` — avg biggest in-race position gain

**Body weight (1) — NEW this session:**
- `body_wt_mkt_residual` — time-respecting market-failure residual bucketed by `(body_wt_change × implied_prob)`. Formula: `(past_wins − past_implied) / (past_implied + k)` with k=50, per bucket.

---

## What we tried — chronologically

### 1. Feature pruning (prune2) ✓ WIN

Dropped the 2 lowest-importance features: `top3_count_5` (rank 22.1) and `field_form_cv` (rank 22.6). 

**Result: +69pp paired delta over baseline, robust across 3 seeds, LL unchanged.**

Key lesson: low-importance LEAF features can be safely pruned. Low-importance BACKBONE features (Racecourse, Track_Turf) cannot — they enable downstream interactions even when their own gain is low.

### 2. Draw re-engineering (`draw_bias_loc`) ❌ FAILED

Built time-respecting bucketed lift feature at `(Racecourse, Dist, Draw)`. Correctly encoded empirical draw bias (HV 1200m Draw 1 = +41% win rate advantage; ST 1000m reversed).

**Result: −21pp paired delta.** Market already prices draw bias. Duplicating it added noise.

**Key lesson:** Don't encode features the market already prices. Target market-failure residuals instead.

### 3. prev_mkt_residual (prev_win swap) ⚠️ NEUTRAL

Replaced binary `prev_win` with bucketed residual `(prev_won × prev_impl)`. Empirical analysis showed market under-reacts to repeat favourites (+2.1% edge) and overprices fluke longshot winners (−0.4%).

**Result:** Single-seed showed +20pp, but across 3 seeds mean was +9pp with high variance. Not clearly better than noise.

### 4. class_change re-engineering ❌ FAILED

Built market-failure residual bucketed by `(RaceClass_ord × class_change)`. Empirical analysis showed strong patterns: class rises +1.30% edge, class drops −0.61%, Class 2 rises the strongest at +21% residual.

**Result: −38pp paired delta across seeds.**

**Key lesson:** Don't re-encode the model's top-importance features. `class_change` is consistently rank #1 — XGBoost already finds optimal splits. Adding a derived version wastes tree capacity.

### 5. Sectional consolidation ❌ FAILED

Tried reducing 4 sectional features to 2 or 1 composite (`sect_kick = closing − avg`).

**Result: −22pp to −47pp paired delta.** All 4 sectionals carry complementary information — not redundant.

### 6. Form normalization ❌ FAILED

Swapped `recent_form = avg(1/place)` for `avg((field_size − place + 1)/field_size)` to properly account for field size.

**Result: −32pp paired delta.** The concave `1/place` encoding implicitly weights top finishes more heavily — turns out this is the right shape for horse-quality signal. Linear normalization is too democratic.

### 7. Dropping odds-dependent features ❌ FAILED

Tested LOO on each of: `fav_field_size`, `career_beat_odds`, `tt_gap`, `tr_gap`. All have moderate corr with market; rationale was reducing deployment fragility.

**Result: Every LOO hurt.** fav_field_size: −40pp, career_beat_odds: −30pp, tt_gap: −21pp, tr_gap: −21pp. All 4 removed: −33pp. Every odds-dependent feature earns its keep.

### 8. body_wt_mkt_residual ✓ **WIN**

**First robust feature addition.** Built from previously-unused `Declar. Horse Wt.` field in horses JSON.

Bucketed market-failure residual at `(body_wt_change_bucket × impl_bucket)`. Feature is nearly orthogonal to market (corr = +0.01).

Empirical insights:
- Favourites (impl > 0.15) with weight changes outperform market by +2-3% edge
- Horses after long rest (>60 days): weight GAIN is positive (+1.5% edge)
- Volatile-weight horses with big drops: +0.7% edge
- Prev winners with weight drops: +1.3% edge

**Result (8-seed test):**
- Baseline mean: +14.42%, min −33.3%, max +66.4%
- With body_wt_residual: mean **+48.74%**, min **−3.0%**, max +91.0%
- Mean paired Δ: **+34.32%**, 7/8 seeds positive
- **Downside compressed dramatically (−33% → −3%).**

Also tested raw body_wt_z + body_wt_change (2 features) and `body_wt_after_win` targeted binary — both hurt. Only the bucketed residual works.

### 9. 5-seed Offset bagging ✓ **WIN**

Train 5 Offset models on same data with different XGBoost seeds. Average predictions per race, renormalize.

**Result (3 meta-seed sets × 10 years):**
- Mean cum ROI: +70.98% (vs +48.74% single-seed)
- Min cum: **+41.38%** (vs −3.0% single-seed)
- Best meta-set: +103.65% cum, **8/10 positive years** (vs 5-6/10 single-seed)
- **Flipped 2018 AND 2019 from negative to positive** — both were structural losers

**Why:** XGBoost has random elements (column subsampling, row subsampling, split randomization). Different seeds → different trees → averaging cancels noise. LL stays flat (~0.24520) — bagging doesn't improve *prediction accuracy*, it reduces *prediction variance* that Kelly was amplifying.

### 10. Calibration + Softmax ensemble ❌ FAILED

Tested:
- Bagged Offset + isotonic calibration: **−141pp**
- Softmax (with log_implied_prob + isotonic): **−91pp**
- Ensemble (0.5 × bagged_offset_cal + 0.5 × softmax_cal): **−62pp**

**Result: bagged Offset alone is the architectural ceiling.**

- Isotonic on Offset hurts because market anchor (`base_margin`) already provides calibration by construction.
- Softmax underperforms because it has to learn the market anchor from scratch (as a feature) whereas Offset hard-wires it.
- Ensembling a good model with a bad one produces a mediocre model.

### 11. Kelly sweep on bagged ✓ **WIN**

With bagging reducing variance, Kelly can safely be 2-3x higher than the old 0.01 standard.

| Kelly | Cum ROI | Pos |
|---|---|---|
| 0.01 | +103.66% | 8/10 |
| 0.02 | +218.63% | 8/10 |
| 0.03 | +303.92% | 7/10 |
| 0.05 | +290.71% | 6/10 |

Kelly 0.03 is the new sweet spot; Kelly 0.02 the conservative pick.

### 12. Truncated training (last 5 / last 3 years) ❌ FAILED

Hypothesis: if the market is pricing in our edges over time, training only on
recent years should outperform full-history training. Tested at K=0.02, 3 seeds:

| Window | Mean cum no_rebate | Seeds |
|---|---|---|
| Full (2010→$t$−1) | **+103.34%** | +26.6%, −18.9%, +302.4% |
| Last 5 years | −71.37% | −69.2%, −72.7%, −72.2% |
| Last 3 years | −85.06% | −67.2%, −91.1%, −96.9% |

**Truncation catastrophically hurt overall.** BUT year-by-year showed
partial recency gains: last_3 beat full history in 2024 (+7.92% vs −4.46%),
2025 (+21% vs −7%), and 2026 (+3% vs −0.3%). So the edge-decay story has
partial support, but simple truncation is too blunt — the cost in
2015-2021 dwarfs recency gains in 2024-2026.

**Next attempt (untested):** recency-weighted training
($w = e^{-\lambda\,\text{years\_ago}}$) instead of throwing away data.

### 13. 5-seed variance study: "was bag_A lucky?" ✅ YES

Ran 5 INDEPENDENT 5-seed bagging meta-sets at K=0.02 over 12 years:

| Meta-set | cum_no | cum_wr | pos_no |
|---|---|---|---|
| bag_A (production) | **+332.48%** | +879.87% | 9/12 |
| bag_B | +101.18% | +354.02% | 9/12 |
| bag_C | +29.07% | +188.39% | 7/12 |
| bag_D | +23.99% | +178.64% | 6/12 |
| bag_E | +49.19% | +231.36% | 6/12 |
| **MEAN** | **+107.18%** | +366.46% | 7.4/12 |
| **STD** | 129.60% | 295.37% | — |
| **MIN** | +23.99% | +178.64% | 6/12 |

**Bag_A was the luckiest of 5 independent bagging attempts.** Mean realistic
production is ~+107% cum (not +332%). But:

- **All 5 meta-sets positive** (min +24%) — bagging reliably removes catastrophic runs.
- Single-seed comparison: MEAN +131%, MIN **−18.9%**, MAX +302%.
- So bagging's actual benefit is *downside compression* (min −19% → min +24%), NOT raising the expected mean.
- **LL is identical across all 5 meta-sets** (0.23708-0.23709) — confirms the variance-reduction mechanism.

Years robust across meta-sets:
- **2016, 2022, 2023, 2026**: positive in all 5 meta-sets.
- **2020**: negative in all 5 meta-sets (structural COVID-era bad year).
- **2015**: chaotic (−24% to +82% depending on meta-seed) — the main source of
  production variance.

---

## Year-by-year (bagged Offset, Kelly 0.01, no rebate)

> **Superseded.** These per-year returns are from the same pre-filter settlement as the TL;DR
> above. The current per-year figures, with bet counts beside them, are in the Kelly section of
> `results/reports/bagged_offset/report.html`.


| Year | Single seed=42 | 5-seed bagged (bag_A) |
|---|---|---|
| 2015 | +39.12% | +35.48% |
| 2016 | +24.71% | +20.95% |
| 2017 | −0.96% | −2.02% |
| 2018 | −3.46% | **+2.46%** |
| 2019 | −7.71% | **+5.24%** |
| 2020 | −9.27% | −5.34% |
| 2021 | +15.31% | +11.45% |
| 2022 | +4.74% | +7.19% |
| 2023 | +3.24% | +3.08% |
| 2024 | −0.65% | +0.90% |
| **Cum** | **+72.04%** | **+103.66%** |

---

## Methodology lessons

### 1. LL-flat / ROI-variable is the norm

Most feature tweaks leave log-loss essentially unchanged (Δ ≤ 0.0001) but move ROI by ±30-100pp. This is Kelly amplifying tiny probability shifts. Single-seed ROI differences at this scale are noise-dominated.

**Rule:** Rank feature changes by paired-delta across 3+ seeds, not single-seed ROI.

### 2. Market-failure residuals > raw features > derived re-encodings

Pattern that worked: take a feature with market-orthogonal signal (body weight), discretize by meaningful buckets (weight change × implied), compute time-respecting `(actual − implied)/ (implied + k)` per bucket. This targets exactly where market is wrong.

Pattern that failed: re-encoding features already in the model (class_change_residual, draw_bias_loc). Adds correlated info → overfitting surface.

### 3. Importance rank ≠ marginal value

XGBoost gain-importance measures *this feature's contribution to splits*. Doesn't measure *this feature's value as interaction substrate*. Low-rank CATEGORICALS like Racecourse are essential backbones for other features' interactions; removing them crashes the model.

### 4. Bagging is cheap variance reduction

5-seed bagging: 5× training cost, +22pp mean ROI, eliminates downside. Almost certainly the best $ per effort ratio in the session.

### 5. Offset > Softmax for HKJC

The Offset model's `base_margin = logit(Implied_Prob)` hard-anchors predictions to market. Softmax trying to learn this from features is fundamentally weaker. Don't spend cycles on Softmax architectures.

---

## Reproducibility

> **Superseded.** `hkjc.suites.production` and `results/bankroll_viz.html` do not exist. The
> supported pipeline is in `docs/pipeline.md`; the canonical rebuild sequence is in
> `results/README.md`.


Pipeline:
1. `python -m hkjc.features.build` — builds `data/processed/race_features.parquet` with all 25 production features (including `body_wt_mkt_residual`). Regenerate after any data refresh or feature-code change.
2. `python -m hkjc.suites.production` — loads the parquet, trains the bagged Offset model across 2015-2024 at Kelly ∈ {0.01, 0.02, 0.03, 0.05}, writes `results/bankroll_viz.html`.

## Two compiled kernels that were correct and never wired in (2026-08-31)

Both were written during the Phase 2 iteration work, verified, deliberately left unused, and removed
on 2026-08-31 once it was clear that a kernel with no caller is dead code however well documented.
Retrieve either with `git show f08e0b0:src/hkjc/models/common/race.py`. The findings are here because
they are the reason not to repeat the attempt, and they are about the *callers*, which are still live.

**`tempered_softmax`.** Bit-identical to `scipy.special.softmax` per race in both float32 and
float64 -- verified over six seeds in each precision -- and wiring it into
`apply_temperature_binned` still moved the canonical probabilities by one float32 unit in the last
place. The cause is in the caller, not the kernel: that loop's arithmetic precision is decided by the
Python *type* of each temperature. A `np.float64` coming out of `scipy.optimize.minimize` promotes a
float32 margin to float64, and the Python float `1.0` used for the untempered stage does not. So two
stages of one pipeline compute in two precisions, and a single array of temperatures cannot carry
both behaviours. Fixing that is a scientific change with its own baseline, not a speed change.
`tests/unit/test_models.py::TemperedSoftmaxTests` still pins the promotion behaviour as a fact about
NumPy rather than as prose.

**`blend_nll`.** Correct in float64 and *not* bit-identical to the reference in float32, which is
what kept it out. The reference forms the whole blended array and then takes the winners' mean; a
per-winner accumulation differs by one unit in the last place on roughly four inputs in five. The
blend fitter's inputs are float32, one `minimize` over a single parameter is about 7% of a fit cell's
calibration, and a fitted blend weight that moves in its last bit moves every published probability.
So the cost of compiling it was never worth the risk. `learn_blend_weight` still carries the
"Deliberately *not* compiled" note, and a test asserts it does.

The general lesson, which is the transferable part: **a kernel proved bit-identical in isolation can
still move a published number, because the precision of the surrounding loop is not a property of the
kernel.** Both attempts failed at the boundary rather than in the arithmetic.

## Memory files

Individual findings preserved in `~/.claude/projects/.../memory/`:
- `production_state.md` — current config
- `body_wt_residual_finding.md` — the body weight feature
- `architecture_5seed_bagging.md` — bagging details
- `architecture_kelly_sweep.md` — Kelly 0.02-0.03 finding
- `architecture_cal_ensemble_rejected.md` — what NOT to retry
- `feedback_dont_duplicate_market.md`, `feedback_dont_reencode_top_features.md`, `feedback_pruning_strategy.md` — methodology guardrails
