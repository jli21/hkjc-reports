---
title: Research memory
kind: research-memory
status: living-document
moved: 2026-08-31
moved_from: RESEARCH_MEMORY.md
moved_by: docs/plans/archive/2026-08-30-report-first-redesign-probit-optimization-and-feature-development.md
---

# Research Memory and Future Model Backlog

Last updated: 2026-08-24

This document preserves research questions, hypotheses, data-source leads, and
modeling concerns that are not part of the current production implementation.
It is intentionally separate from [findings.md](findings.md), which records validated
results, and from [backlog.md](backlog.md), which tracks what is open and why. Where this
document disagrees with a published report, the report is right.

Nothing in this file is a promoted result. Treat every proposal as a hypothesis
that must beat the incumbent under chronological racewise log-loss evaluation.

## Current research contract

- The promoted model is a racewise market Offset model.
- Final HKJC result-page odds are a retrospective benchmark, not an executable
  pre-race price.
- Feature selection uses paired racewise multinomial log loss, not ROI.
- Historical aggregates must use complete dates strictly before the prediction
  date.
- A new feature or architecture must improve the incumbent, not merely beat a
  uniform or weaker fundamental baseline.
- All historical years through 2026 have been used in development. Definitive
  confirmation now requires a frozen prospective shadow test.

## Topic 1: Conditional IIA and race interactions

### The concern

For an Offset member, the current model has the form:

```text
score_i = log(market_prob_i) + correction_i
prob_i = softmax(score_i / temperature)
```

For fixed scores and temperature:

```text
prob_A / prob_B = exp((score_A - score_B) / temperature)
```

Adding horse C changes the denominator and therefore the absolute probabilities
of A and B, but it does not change their relative odds unless C also changes A's
or B's modeled score or the calibration temperature. This is the conditional
independence-of-irrelevant-alternatives restriction.

The random-utility interpretation uses independent Gumbel shocks for each
runner. It does not imply that the events `A beats B` and `B beats C` are
independent. Those comparison events share B and are dependent. The restrictive
assumption is that omitted runner-level shocks do not create additional
substitution or interaction after conditioning on the modeled context.

### Why it may matter in racing

Horses do not perform in a vacuum. A third runner can change the relative chance
of A versus B through:

- pace competition;
- lead contests and energy expenditure;
- draw and position interactions;
- crowding and traffic;
- tactical jockey responses;
- shared track or weather shocks;
- style-specific responses to an unexpectedly fast or slow pace.

Example:

- A is a front-runner.
- B is a closer.
- C is another strong front-runner.

Without C, A may control the race. With C, A and C may contest the lead and help
B. A correct model should allow C to lower A's expected utility and raise B's,
not merely consume probability mass.

### How the current model partially addresses it

The production model is not strictly IIA with respect to the raw race card,
because adding or removing a runner can change contextual inputs:

- `field_size`;
- `market_rank`;
- `rating_z`;
- `wt_z`;
- `draw_pct`;
- `pace_pressure`;
- `closer_setup`;
- current market odds;
- field-size calibration bin.

The remaining assumption is narrower but still strong:

> Once the race has been summarized by the engineered context features and the
> market, no material omitted entrant interaction remains.

Current pace summaries are coarse. They count qualifying speed rivals but do not
fully represent rival identity, quality, draw, tactical uncertainty, or a latent
pace distribution.

### Diagnostics to revisit

1. Test whether pairwise model-market log-odds residuals depend on omitted rival
   characteristics after controlling for current `pace_pressure`.
2. Stratify residuals by leader, stalker, and closer styles and projected pace
   composition.
3. Test whether adding another projected leader changes leader-versus-closer
   residual ordering beyond the current summary features.
4. Report residuals by draw x style x course x distance cohorts.
5. Measure whether large member disagreement is concentrated in interaction-heavy
   race shapes.
6. Compare response stability across years before adding architecture complexity.

### Architecture ladder

Use the smallest model that resolves a demonstrated residual problem.

| Rank | Candidate | Purpose | Main caveat |
|---:|---|---|---|
| 1 | Richer explicit field-interaction features | Preserve the current robust Offset model | Interactions remain hand-specified |
| 2 | Latent pace-factor Offset | Add a shared race shock with horse-specific style loadings | Requires a defensible pace observation model |
| 3 | Mixed logit | Allow race-varying coefficients and relax IIA | Simulation and interpretation cost |
| 4 | Structured multinomial probit | Model correlated performance shocks | Variable field size, identification, expensive integration |
| 5 | Market-Offset DeepSets/Set Transformer | Let each horse attend to all rivals | Flexible, less interpretable, overfit risk |

### Why generic multinomial probit is not automatically the answer

A multinomial probit can use correlated normal errors:

```text
utility = mean_utility + error
error ~ Normal(0, covariance)
```

This relaxes logit's exact IIA property, but an unrestricted covariance matrix is
not naturally shared across variable-size fields and is difficult to estimate
from one observed winner per race. A fixed covariance also does not explain why
the presence of a particular front-runner changes another runner's expected
performance.

A structured factor model is more useful:

```text
utility_ri = mean_ri + pace_loading_ri * latent_pace_r + residual_ri
```

Front-runners and closers can have opposite pace loadings. Winner probabilities
can then be evaluated by Monte Carlo. This is a structured probit-like model with
a racing interpretation, rather than a generic covariance matrix.

### Promising explicit interaction features

- expected number of leaders;
- sum and maximum rival early-speed probability;
- quality-weighted rival speed;
- inside/outside distribution of projected speed;
- pace concentration or entropy;
- horse-specific pace sensitivity from prior sectional races;
- draw x style x course interactions;
- projected pace mean and uncertainty;
- leader-versus-leader pair terms;
- closer benefit under high projected pace;
- trainer/jockey tactical tendencies, if historically vintaged.

Do not promote these based on intuitive signs. Evaluate paired loss against the
incumbent on the same race universe and seeds.

## Topic 2: Sparse local horse history and cold starts

### Measured local-history coverage

The local generated feature table available during this analysis had 149,309
runner rows. It was stale relative to the 152,046-row canonical build, so exact
counts must be refreshed before implementation. The broad coverage pattern was:

| Local history | Runner share |
|---|---:|
| First HK start | 5.36% |
| One or fewer HK starts | 10.45% |
| Three or fewer HK starts | 20.02% |
| Five or fewer HK starts | 28.69% |
| Ten or fewer HK starts | 47.13% |

Recent cohorts were colder:

| Year | First HK start | Three or fewer starts |
|---|---:|---:|
| 2022 | 4.58% | 17.19% |
| 2023 | 4.52% | 17.28% |
| 2024 | 5.52% | 20.69% |
| 2025 | 5.68% | 21.43% |
| Partial 2026 | 5.84% | 24.47% |

Other observed coverage:

| State | Coverage |
|---|---:|
| Prior HK run | 94.64% |
| At least one barrier trial | 92.90% |
| Prior body-weight history | 89.55% |
| Nonzero rank reliability | 94.63% |

The acute history problem is approximately the first zero to three HK starts,
not every runner.

### Import cohorts

For horse profiles from 2020 onward:

| Import type | Horses | Share |
|---|---:|---:|
| PPG | 1,545 | 57.9% |
| PP | 927 | 34.8% |
| VIS | 120 | 4.5% |
| ISG | 75 | 2.8% |

Interpretation:

- PP horses usually have overseas racing history.
- PPG horses are generally unraced before import.
- ISG horses are especially connected to sales and preparation evidence.
- VIS horses have international form but may need a separate treatment.

The current `debut` indicator means no previous HK run. It combines genuinely
unraced horses with experienced overseas imports. This distinction should be
foundational in future work.

Recent origin concentration:

- Australia and New Zealand: 78.8%.
- Australia, New Zealand, Britain, Ireland, and France: 94.8%.
- Adding Japan: 96.9%.

This supports a focused source strategy rather than solving every jurisdiction.

### Why the current history block may look weak

The production correction uses preparation changes, trainer/jockey historical
market residuals, running style, pace summaries, trip comments, barrier trials,
and body-weight change. It excludes richer dynamic rank, speed, margin, market
memory, sectional state, and formline blocks because those did not consistently
improve the market Offset in prior tests.

Weak incremental value can mean several different things:

- final odds already price conventional local history;
- cold and experienced runners require different priors;
- history summaries are too coarse;
- reliability is not sufficiently modeled;
- missing pre-import information matters more than another local-form transform;
- historical effects are diluted when fitted over all runners.

The best stable historical addition found so far was a small early-sectional
residual tilt, approximately `-0.000211` log loss versus the incumbent. More data
must still demonstrate incremental market-relative value.

## Topic 3: External identity and history sources

Public availability does not imply permission for automated collection, model
training, retention, or redistribution. Prefer official feeds or negotiated
licenses. Public pages are useful for manual validation and proof-of-concept
matching.

### Hong Kong identity anchor

| Source | URL | Potential fields |
|---|---|---|
| HKJC former name and pedigree | https://racing.hkjc.com/en-us/local/info/horse-former-name | Former registered name, HK identity, pedigree |
| HKJC horse search | https://racing.hkjc.com/en-us/local/information/selecthorse | Horse ID, brand, origin, import type, pedigree, local history |
| HKJC new horses | https://racing.hkjc.com/en-us/local/page/new-horse | New-horse profiles and preparation information |
| HKJC PP pre-import performance | https://racing.hkjc.com/en-us/local/page/ppo-performance | Selected overseas form and media |
| HKJC International Sale | https://racing.hkjc.com/en-us/international-sale/index | Sale lot, pedigree, source sale, price |

The first implementation dependency is a former-name and external-identity
crosswalk. The current profile collector does not reliably preserve former name,
exact foaling date, overseas registry ID, sale lot, or match evidence.

### Australia and New Zealand

| Source | URL | Potential fields | Access concern |
|---|---|---|---|
| Racing Australia | https://www.racingaustralia.horse/ | National form, ratings, renamed horses, pedigree | Public terms restrict scraping/commercial reuse |
| Australian Stud Book | https://www.studbook.org.au/ | DOB, sex, parentage, breeder, registration | Subscription/licensing |
| NZTR Stud Book | https://loveracing.nz/stud-book/search | Pedigree, microchip search, exported/unnamed horses | No advertised open bulk license |
| NZTR profiles/results | https://loveracing.nz/raceinfo/Horses.aspx | Official form, ratings, trials | Dynamic copyrighted site |
| Inglis | https://inglis.com.au/sales/ | Lots, price, vendor, purchaser, pedigree | Copyrighted sales database |
| Magic Millions | https://www.magicmillions.com.au/sales/ | Sales, photos, videos, pedigree | Copyrighted sales database |
| New Zealand Bloodstock | https://www.nzb.co.nz/sales/results | Sales, Ready to Run, breeze videos | Copyrighted sales database |

Australia and New Zealand are the first foreign procurement priority.

### Britain, Ireland, and France

| Source | URL | Potential fields | Access concern |
|---|---|---|---|
| Weatherbys data supply | https://www.weatherbys.co.uk/commercial/data-supply | Licensed identity, breeding, and racing feeds | Commercial contract |
| BHA horse search | https://www.britishhorseracing.com/racing/horses/ | British form and breeding | Commercial scraping prohibited |
| BHA ratings | https://www.britishhorseracing.com/regulation/official-ratings/ratings-database/ | Official ratings | Licensing required for reuse |
| HRI horse statistics | https://www.hri.ie/statistics/horse | Irish horse profiles and form | No documented open bulk API |
| HRI RAS | https://www.hri-ras.ie/ | Results, registrations, supplements | Database rights apply |
| France Galop | https://www.france-galop.com/en/horses-and-people/horses | Form, pedigree, ratings | Extraction/reuse restricted |
| IFCE Info Chevaux | https://infochevaux.ifce.fr/en/info-chevaux | Legal identity, SIRE/UELN, DOB, pedigree | Some paid/private fields |
| Tattersalls | https://www.tattersalls.com/sales/ | Sale lots and horses-in-training | Copyrighted database |
| Goffs | https://www.goffs.com/sales | Sale lots and prices | Anti-bot and licensing controls |
| ARQANA | https://www.arqana.com/catalogues_resultats.html | French/international sales | Copyrighted database |

Weatherbys is a promising European route because it explicitly offers licensed
commercial data supply.

### Japan

| Source | URL | Potential fields | Access concern |
|---|---|---|---|
| JAIRS Stud Book | https://www.studbook.jp/users/en/UserMenu | Official pedigree and identity | No open bulk API |
| JRA database | https://www.jra.go.jp/JRADB/ | Central racing form and results | Dynamic Japanese interface |
| NAR data room | https://www.keiba.go.jp/KeibaWeb/DataRoom/DataRoomTop | Local racing history | Dynamic interface |
| JBIS | https://www.jbis.jp/horse/ | Integrated pedigree, form, and sales | Business reuse restricted |
| JRHA Select Sale | https://www.jrha.or.jp/selectsale/result | Major sale results | Copyrighted |
| HBA sales | https://www.hba.or.jp/english/ | Hokkaido and training sales | Copyrighted |

Japan is a second-stage integration after Australasian and European coverage.

### Commercial aggregators to evaluate

- Racing & Sports.
- Podium.
- Timeform.
- Weatherbys Data Supply.
- Arion Pedigrees.
- Equineline/TJCIS.
- Racing Post, subject to explicit licensing.

Send the same stratified proof-of-concept horse list to competing vendors and
measure identity and event coverage rather than accepting broad coverage claims.

## Topic 4: Identity resolution

Names alone are unsafe because HK horses are frequently renamed and names can be
reused across jurisdictions.

### Minimum external identity model

```text
horse_id
source
source_horse_id
registry
registry_id
registered_name
former_name
country_suffix
foaling_date
sex
colour
sire
dam
damsire
match_method
match_confidence
evidence
effective_from
effective_to
matched_at
```

### Match rules

| Evidence | Treatment |
|---|---|
| Registry/life number or microchip | Normally definitive |
| Exact DOB plus sire and dam | Very strong |
| Sire, dam, sex, YOB, and country agree | Strong |
| Former name plus complete race chronology | Strong |
| Name and sire only | Insufficient |
| Name alone | Never auto-match |
| Different dam or sex | Hard reject unless authoritatively corrected |

Normalize punctuation, spacing, case, and accents only for candidate generation.
Preserve original names, country suffixes, scripts, and Roman numerals.

Keep race, trial, sale, name-history, and identity-crosswalk records separate.
Do not force repeatable events into the one-row-per-horse profile table.

## Topic 5: Cold-start model ladder

Keep the incumbent market Offset and add a heavily regularized correction that
is active for early HK starts:

```text
prob_i proportional to market_prob_i * exp(incumbent_i + cold_i * cold_prior_i)
```

### Baselines

| Model | Description |
|---|---|
| M0 | Normalized market |
| M1 | Market calibrated by field size and cold-start composition |
| M2 | Current production Offset |
| C0 | M2 plus first-HK-start, import-type, and origin fixed effects |
| C1 | C0 plus hierarchical trainer, sire, and damsire priors |
| C2 | C1 plus cold-specific trial and body-weight translation |
| C3 | C2 plus translated overseas form for PP horses |
| C4 | C3 plus sales and breeze-up evidence |

Every candidate is compared with M2 on the same folds, seeds, and race universe.

### Hierarchical cold prior

Candidate structure:

```text
cold_prior =
    import_type_x_origin_prior
  + trainer_x_import_type_prior
  + sire_x_surface_distance_prior
  + damsire_x_surface_distance_prior
  + static_attributes
```

Use partial pooling so unseen or low-count groups revert toward zero, which means
trusting the market. Expose posterior mean, uncertainty, effective count, and
recency. Do not use raw unshrunk win rates.

Trainer effects are predictive, not causal. They can capture preparation,
purchasing, owner relationships, horse allocation, and stable quality.

### Pedigree ladder

1. Chronological sire and damsire market-relative posteriors.
2. Context-specific random slopes for distance and surface.
3. Pedigree relationship kernel or animal-model prior.
4. Inductive graph model only if simpler pedigree models prove incremental value.

An unpooled dam effect is not useful because most dams have very few HK offspring.
Future-registered siblings must not appear in historical pedigree aggregates.

### Cold-specific barrier trials

First-appearance trial coverage was approximately 79.4%, making this a practical
near-term source. Build a trial-to-first-race translation rather than treating a
trial as an ordinary race.

Candidate fields:

- session-standardized trial time;
- finish percentile and time behind;
- early position and position gain;
- trial count, recency, and trend;
- distance and surface match;
- trial after arrival;
- trial since overseas start;
- trainer x debut trial-to-race conversion;
- effort/restraint indicator if it can be observed without future information.

Trial participation and effort are selected and not fully competitive. Keep
availability and reliability explicit.

### Cold-specific body weight

`body_wt_change_z` is weak for a horse without personal history. Candidate
first-start features:

- raw log body weight;
- sex x age x origin x import-type cohort z-score;
- field-relative body-weight percentile;
- body weight relative to sire progeny;
- trial-to-race change where available;
- days since measurement and source reliability.

Use nonlinear effects. Weight gain is not universally positive or negative.

### Overseas-to-HK bridge for PP horses

Collect point-in-time overseas starts with:

- source horse/race IDs;
- date and jurisdiction;
- track, distance, surface, and going;
- race class and official/provider ratings;
- carried weight and age;
- finish, margin, time, and sectionals where available;
- trainer, equipment, and layoff;
- rating publication timestamp.

Do not compare raw ratings or times across jurisdictions. Normalize by provider,
jurisdiction, season, surface, and distance band, then learn a supervised mapping
from overseas evidence to early HK market-relative performance.

Expose translated ability mean, translation uncertainty, effective starts,
recency, distance/surface match, and jurisdiction reliability. Decay the overseas
prior as HK evidence accumulates.

### Sales and breeze-up features

- inflation and FX-adjusted log sale price;
- sale-cohort percentile;
- price residual versus pedigree, sex, sale, and year;
- breeze-up time relative to session;
- buyer, vendor, consignor, and agent effects;
- pass-in, withdrawal, private-sale, and missing-reason indicators;
- days from sale to import and first HK race.

Raw price is public valuation likely reflected in odds. Price residual versus
pedigree expectations is a more plausible incremental signal.

## Topic 6: Reliability and missingness

Every external or hierarchical block should expose:

```text
*_mean
*_sd
*_n_eff
*_days_since
*_available
*_missing_reason
*_source_quality
*_match_confidence
```

For time-decayed weights:

```text
n_eff = sum(w)^2 / sum(w^2)
```

Missing information should imply a correction near zero and high uncertainty,
not a confident numerical zero. Keep missing reasons separate: not applicable,
not published, failed match, unavailable source, and scraper failure are not the
same state.

Missingness can encode selection, but it can also encode website eras and source
failures. Test stability by season and provider.

## Topic 7: Evaluation and leakage controls

### Primary estimand

For each race:

```text
candidate_minus_incumbent_loss = candidate_race_loss - incumbent_race_loss
```

Negative is better.

Report:

- all races;
- races containing an early-career runner;
- first HK starts;
- starts 1-3 and 4-5;
- PP, PPG, ISG, and VIS;
- overseas-form available/unavailable;
- pedigree and identity confidence;
- trial available/unavailable;
- market-probability deciles.

Score every affected race, not only races won by a cold starter. Conditioning on
cold winners would select on the outcome.

### Required controls

- Use complete-date, point-in-time aggregates.
- Generate chronological OOF hierarchical features for downstream training.
- Freeze source records and ratings as observed at the historical cutoff.
- Do not join current profile snapshots such as current trainer, rating, age, or
  career totals into historical races without vintages.
- Do not reveal future siblings in historical pedigree topology.
- Preserve the same model parameters, seeds, and race universe for ablations.
- Use race-day block bootstrap intervals.
- Run leave-sire-family-out and leave-trainer-out diagnostics.
- Use pedigree-label permutations as a placebo.
- Do not select features using final-odds betting ROI.

### Vendor proof-of-concept gates

Use a stratified 300-400-horse sample covering PP and PPG horses from the main
source jurisdictions.

Minimum acceptance targets:

- at least 98% automatic identity match on the selected sample;
- zero unresolved pedigree contradictions among accepted matches;
- nearly complete official pre-import starts for PP horses;
- high official trial recovery for Australia/New Zealand;
- effective/observation dates on events and ratings;
- stable source IDs and correction handling;
- deterministic bulk backfill and incremental updates;
- explicit contractual rights for historical retention, internal model training,
  derived features, and required post-subscription use.

Legal training and retention rights are a pass/fail gate, not a weighted vendor
score.

## Prioritized revisit list

### Near term, existing data

1. Refresh the cold-start coverage measurements on the canonical 152,046-row
   feature table.
2. Extend the HK profile identity model with former name, foaling date, and match
   evidence.
3. Split first HK start into PP, PPG, ISG, VIS, and true career debut.
4. Build the C0 fixed-effect cold baseline.
5. Build the C1 hierarchical trainer/sire/damsire prior.
6. Build cold-specific trial and body-weight translation.
7. Use Phase 5 reporting to inspect cold-cohort calibration and year stability.

### External-data proof of concept

1. Assemble a manually verified stratified imported-horse identity set.
2. Request identical extracts from Racing & Sports and Podium.
3. Evaluate licensed Racing Australia/NZTR or equivalent feeds.
4. Evaluate Weatherbys/Timeform for European PP horses.
5. Evaluate Inglis, Magic Millions, and NZB sales and breeze evidence.
6. Add Japan only after the first source stack proves value.

### Architecture research

1. Diagnose conditional IIA failures using pace/style residuals.
2. Add explicit rival-interaction features.
3. Test a latent pace-factor Offset.
4. Test mixed logit or structured probit only if residual dependence remains.
5. Test a market-Offset set model only after simpler interaction models.

## Decisions deliberately not made

- No external website has been approved for automated scraping.
- No vendor has been selected.
- No pedigree or overseas feature is promoted.
- No evidence yet justifies replacing the production Offset with multinomial
  probit or a neural set model.
- No retrospective return claim should be interpreted as executable profit.
- No current profile snapshot should be treated as historically available without
  a vintage.

## Related project documents

- `FINDINGS.md`: validated research results and superseded findings.
- `PHASE5_REPORTING_PLAN.md`: production reporting implementation plan.
- `PHASE6_CHOICE_MODELS_PLAN.md`: probit, mixed-logit, and latent-regime
  architecture plan.
- `REFACTOR_PLAN.md`: package, performance, and scraping architecture.
- `REBATE_KELLY_PLAN.md`: rebate-aware sizing and settlement design.
- `docs/dataset-keys.md`: Phase 4 source identity and duplicate-key evidence.
