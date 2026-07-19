# Model-hardness predictor (ingest-time escalation switch)

Offline predictor of `P(part is model-hard)` from features available the moment a new
exam question part is ingested, before any solve has run. It is the deployment switch
for the two expensive interventions in the gate: injecting a worked exemplar into the
prompt, and escalating the part to a stronger solver configuration. Both cost budget,
so we only want to spend them on parts likely to defeat the cheap pool.

Script: `scripts/gate/hardness_predictor.py` (runnable end to end, standard library
only, zero API calls). Run with `PYTHONUTF8=1 python scripts/gate/hardness_predictor.py`.

## What "model-hard" means here

Two different notions of "hard" are in play, and they are only weakly related.

- **Labelled-hard** is the exam's own view: how many marks the part is worth, or
  Elevatar's 1 to 3 difficulty tag. This is what a human examiner thinks is hard.
- **Model-hard** is what actually defeats our solver pool. We define it empirically
  from the Stage-D pool sweep (`reports/gate/stage_d/sweep.jsonl`): for each of the 7
  pool configurations we compute a per-part score = the mean of the boolean `correct`
  over that configuration's passes, and call it a **miss** when the score is below 0.5.
  A part is **model-hard** (label `y = 1`) when **4 or more of the 7 configurations
  missed it**. That yields **57 model-hard parts out of 554** dev parts (10.3 %).

These are not the same thing. The campaign measured a Spearman rank correlation
(a monotone-association coefficient between minus 1 and plus 1) of only **rho = +0.148**
between exam marks and pool miss-count. So marks barely rank model-hardness, and a
predictor that just reads marks is close to useless. The job of this model is to learn
a better combination of cheap, observable signals.

Why predict it at all: at ingest there is no solve history, so we cannot read the
miss-count directly. If a cheap feature-only model can rank the incoming part's
hardness even moderately, we can pre-emptively route budget to the parts that need it.

## Features (observable at ingest only)

Every feature below is computable from the question text and its provenance metadata
alone. **Nothing is derived from solve logs, scores, or the sweep** (that would leak
the label). Numeric features are standardised (mean-centred, scaled to unit standard
deviation) on the training fold only.

| Feature | Type | Meaning and why it should matter |
|---|---|---|
| `mod_M2..mod_M5` | one-hot | Elevatar ontology module (M1 is the dropped reference). Modules differ sharply in model-hardness: M3 (calculus) and M5 are the hardest observed, M4 the easiest. |
| `at_*` | one-hot | Answer type (numeric is the dropped reference; the rare `coordinate`, `multi_value`, `prose` types are folded into `other`). Free-form or unusual answer shapes are graded more strictly and mis-solved more often. |
| `tech_active` | binary | Tech-active (calculator/CAS-permitted, Exam 2 style) vs tech-free (Exam 1). See the collinearity note below. |
| `is_official` | binary | Source is VCAA (official) vs a commercial trial vendor (Kilbaha, itute). Commercial parts are noisier and empirically harder for the pool. |
| `is_2_marks` | binary | The 2-mark band explicitly, requested as its own flag because 2-markers are the modal multi-step short-answer part. |
| `marks` | numeric | Raw mark value (1 to 5). Weak on its own (rho = +0.148) but retained so the model can combine it with the rest. |
| `difficulty` | numeric | Elevatar's own 1 to 3 difficulty label, median-filled when missing. The single most monotone human signal (see weights). |
| `qlen` | numeric | Question length in characters. Cheap text feature 1: a proxy for how many sub-conditions the stem carries. Longer stems hold more state for the solver to track, so they are more error-prone. |
| `nlines` | numeric | Number of lines in the question (newline count plus 1). Cheap text feature 2: multi-line stems typically define functions or set up cases, a structural-complexity signal distinct from raw character length. |
| `has_prob` | binary | Contains probability notation (`Pr(` or the word "probability"). Cheap text feature 3: marks the probability register. Empirically **protective** (the pool handles probability well), so it earns a negative weight rather than being noise. |
| `has_show` | binary | Contains a closed-target proof cue (`show that` / `show `). Cheap text feature 4: proof parts must land on a specified expression and are graded strictly. This overlaps the `show_that` answer type; it is kept as a correlated feature and flagged in Limitations. |

Each of the four text features is a single string operation (length, count, or
substring test), so all of them together add negligible ingest cost.

**Collinearity note (important).** In this dataset every tech-free part is an Exam 1
part and every tech-active part is an Exam 2 part: the counts match exactly (139
tech-free = 60 + 59 + 20 Exam 1 papers). So the requested "Exam 1 vs Exam 2" feature
is **identical** to the tech flag; encoding both would be perfect duplication. We keep
the single `tech_active` flag and treat it as carrying the Exam 1 / Exam 2 distinction.

## Models compared

1. **Logistic regression, all features.** Log-odds linear in the features, squashed to
   a probability by the logistic (sigmoid) function, trained by full-batch gradient
   descent with an L2 (ridge) penalty on the weights (the intercept is unpenalised).
   554 rows makes this a millisecond fit.
2. **Marks-only logistic.** The "just use exam marks" strawman: the same fitting
   procedure on the single `marks` feature.
3. **Naive rule.** A hand rule with no fitting: flag as hard when the module is M3 or
   M5 **and** the part is tech-active.

## Evaluation protocol

5-fold cross-validation over the **existing stem-grouped folds** from
`folds.make_folds`. The unit of independence is the parent stem (exam plus question
number); every part of one question lands in the same fold. A random split would leak,
because sibling parts of one question share phrasing and difficulty and would appear on
both sides of the split. Standardisation statistics are fit on the training folds only.

Metrics (all on pooled out-of-fold predictions unless stated):

- **ROC AUC** (area under the receiver-operating curve): computed rank-wise as the
  Mann-Whitney statistic = the probability that a random model-hard part scores above a
  random easy part. 0.5 is chance, 1.0 is perfect ranking.
- **Average precision** (area under the precision-recall curve): the more informative
  summary when positives are rare (here the base rate is 0.103, so random AP = 0.103).
- **Precision at fixed recall** (20 %, 30 %, 50 %): if we tune the flag threshold to
  catch that fraction of the model-hard parts, what fraction of the parts we flag are
  actually hard, and how many parts get flagged.
- **Calibration-in-the-large**: mean predicted probability minus observed hard-rate. A
  gap near zero means the probabilities are usable as probabilities, not just as ranks.
- **Per-fold AUC spread**: reported explicitly, not just the mean, because 57 positives
  split across 5 folds leaves only 6 to 17 positives per fold and the per-fold numbers
  are genuinely noisy.

## Results

Full program output is reproduced below (tail truncated) at the end of this report. Headline figures:

| Model | OOF ROC AUC | OOF avg precision | Calibration gap | Notes |
|---|---|---|---|---|
| Logistic, all features | **0.713** | **0.257** | +0.001 | Best ranker; well calibrated |
| Marks-only logistic | 0.530 | 0.114 | -0.000 | Barely above chance (base rate 0.103); confirms marks are weak |
| Naive rule (M3/M5 AND tech-active) | 0.647 | n/a | n/a | Fixed operating point: precision 0.185, recall 0.596, flags 184/554 |

Average precision of 0.257 is **2.5 times the 0.103 base rate**: ranking by the model
concentrates the hard parts far above chance. Calibration is effectively exact
(predicted mean 0.104 vs observed 0.103), so the output can be read as a probability.

**Operating points (full model, pooled out-of-fold):**

| Target recall | Precision | Parts flagged | Threshold |
|---|---|---|---|
| 20 % | 0.211 | 57 / 554 (10 %) | 0.25 |
| 30 % | 0.240 | 75 / 554 (14 %) | 0.22 |
| 50 % | 0.266 | 109 / 554 (20 %) | 0.18 |

**Per-fold AUC:** 0.879, 0.836, 0.640, 0.551, 0.681 (mean 0.717, standard deviation
0.123, min 0.551, max 0.879 on the individual folds). The spread is wide because folds
3 and 1 hold only 6 and 7 positives. The mean is trustworthy; any single fold is not.

**Feature weights** (full-data fit, standardised, log-odds units, largest first):

```
tech_active   +1.223  (harder)     mod_M4        -0.989  (easier)
at_other      +0.871  (harder)     is_official   -0.850  (easier)
difficulty    +0.589  (harder)     mod_M3        +0.458  (harder)
qlen          +0.375  (harder)     is_2_marks    +0.280  (harder)
has_prob      -0.236  (easier)     at_show_that  -0.200  (easier)
marks         +0.126  (harder)     ...
```

The signal is carried by a handful of features and they are all mechanistically
sensible: tech-active parts are much harder (+1.22), the M4 module is protective
(-0.99), commercial-source parts are harder than official VCAA (`is_official` -0.85),
Elevatar difficulty ranks monotonically (+0.59), and longer questions are harder
(`qlen` +0.375). Raw `marks` carries almost nothing once the rest are present (+0.126),
which is exactly what the rho = +0.148 correlation predicted.

## Verdict (plain language)

**Deployable as a coarse escalation switch, not as a precise gate.** The predictor
ranks model-hardness roughly 2.5 times better than chance and its probabilities are
well calibrated, but at any useful recall the flagged set is still mostly easy parts
(precision 0.21 to 0.27). The sensible operating point depends on the cost asymmetry: a
worked-exemplar injection (NOTE 2026-07-19: the hard-exemplar experiment found injection does not help capable models and is directionally harmful - see HARD_EXEMPLAR_RESULTS.md; use this operating point for ESCALATION/ROUTING, or injection on weak-tier models only) is cheap, so run it at the **50 % recall point** (threshold
about 0.18), which flags 20 % of incoming parts and catches half the genuinely hard
ones at 2.6 times base precision. A full model escalation is expensive, so gate it at
the **20 to 30 % recall point** (threshold about 0.22 to 0.25), flagging only 10 to
14 % of parts. Do **not** use marks alone (AUC 0.53) or the naive rule as the primary
switch; the naive rule's recall of 0.60 comes at the cost of flagging a third of all
parts. Recommend shipping the logistic model behind a configurable threshold and
revisiting once more sweep windows exist to shrink the per-fold variance.

## Limitations

- **Pool-relative label.** "Model-hard" is defined against *this* 7-configuration pool
  and the >= 4/7 cutoff. Change the pool (swap a config, add a stronger model) and the
  label, and hence the target, shifts. The predictor learns the current pool's blind
  spots, not an absolute property of the question.
- **Single-window realisations.** Each per-config score is a mean over only 1 or 2
  passes. A part sitting near the 0.5 miss boundary can flip label on solver
  stochasticity alone, adding label noise that caps achievable AUC.
- **n = 57 positives.** Small. Per-fold AUC ranges from 0.55 to 0.88; treat the pooled
  and mean figures as the estimate and the spread as the honest uncertainty. Confidence
  intervals would be wide.
- **Features correlated with source mix.** The strongest signals (`tech_active`,
  `is_official`, module) are entangled with which vendor and paper a part came from.
  The model may be partly learning "commercial trial Exam 2 calculus part" as a bundle
  rather than independent causal difficulty. `has_show` also overlaps the `show_that`
  answer type. If the ingest stream's source mix changes, recalibrate.
- **Exam 1 / Exam 2 is not an independent feature** here: it is identical to the tech
  flag in this dataset (see the collinearity note), so it contributes no signal beyond
  `tech_active`.
- **No temporal validation.** Folds are stem-grouped, not chronological. Whether the
  ranking holds on a future exam year is untested; the campaign's leave-one-year-out
  device (`folds.loyo_splits`) would be the next check before trusting it on new years.

## Full program output

```
====================================================================
MODEL-HARDNESS PREDICTOR  (offline, ingest-time features only)
====================================================================
parts: 554   model-hard positives (>= 4/7 configs missed): 57  (0.103)
folds: stem-grouped 5-fold (folds.make_folds); CV never splits a stem

=== MODEL 1  logistic regression (all ingest features, L2) ===
  pooled OOF ROC AUC       : 0.713
  pooled OOF avg precision : 0.257   (base rate 0.103)
  calibration-in-the-large : mean-pred 0.104 vs observed 0.103  (gap 0.001)
  Brier score              : 0.088
  @recall>=20% : precision 0.211  (recall 0.211, thr 0.253, flags 57/554)
  @recall>=30% : precision 0.240  (recall 0.316, thr 0.223, flags 75/554)
  @recall>=50% : precision 0.266  (recall 0.509, thr 0.184, flags 109/554)
  per-fold AUC (fold:n,pos -> auc):
      fold 0: n=110 pos=14 -> 0.879
      fold 1: n=108 pos=7 -> 0.836
      fold 2: n=108 pos=13 -> 0.640
      fold 3: n=111 pos=6 -> 0.551
      fold 4: n=117 pos=17 -> 0.681
    mean 0.717  sd 0.123  min 0.551  max 0.879

=== MODEL 2  marks-only logistic (baseline: 'just use exam marks') ===
  pooled OOF ROC AUC       : 0.530
  pooled OOF avg precision : 0.114   (base rate 0.103)
  calibration-in-the-large : mean-pred 0.103 vs observed 0.103  (gap -0.000)
  Brier score              : 0.093
  @recall>=20% : precision 0.092  (recall 0.211, thr 0.116, flags 130/554)
  @recall>=30% : precision 0.118  (recall 0.316, thr 0.114, flags 153/554)
  @recall>=50% : precision 0.141  (recall 0.509, thr 0.109, flags 205/554)
  per-fold AUC (fold:n,pos -> auc):
      fold 0: n=110 pos=14 -> 0.543
      fold 1: n=108 pos=7 -> 0.561
      fold 2: n=108 pos=13 -> 0.636
      fold 3: n=111 pos=6 -> 0.681
      fold 4: n=117 pos=17 -> 0.551
    mean 0.594  sd 0.054  min 0.543  max 0.879

=== MODEL 3  naive rule: (M3 or M5) AND tech-active ===
  flags 184/554  precision 0.185  recall 0.596
  rank AUC (binary predictor): 0.647

=== feature weights (full-data fit, standardised; log-odds units) ===
  intercept: -2.836   (base log-odds ~ hard-rate 0.103)
    tech_active    +1.223   (harder)
    mod_M4         -0.989   (easier)
    at_other       +0.871   (harder)
    is_official    -0.850   (easier)
    difficulty     +0.589   (harder)
    mod_M3         +0.458   (harder)
    qlen           +0.375   (harder)
    at_interval    -0.288   (easier)
    is_2_marks     +0.280   (harder)
    at_mc_letter   -0.244   (easier)
    has_prob       -0.236   (easier)
    at_show_that   -0.200   (easier)
    at_set         +0.126   (harder)
    marks          +0.126   (harder)
    at_expression  -0.095   (easier)
    mod_M2         -0.077   (easier)
    mod_M5         -0.074   (easier)
    has_show       +0.022   (harder)
    nlines         +0.018   (harder)

====================================================================
VERDICT
====================================================================
```
```
