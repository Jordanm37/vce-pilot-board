# VCE solver optimization campaign — complete dossier

Purpose: a single self-contained document covering the entire evaluation-gate pilot for
the VCE Mathematical Methods solver — the task, the data, every design choice and why,
every optimization run and its verdict, the routing investigation, the statistics and
theory behind every estimate, and the production decision. It is written to be handed
to a language model (or a person) so they can answer questions about the results without
access to the repository. Every number comes from the campaign's own logs. Compiled
2026-07-10 from the canonical sources: `reports/gate/stage_c/FINDINGS.md`,
`reports/gate/stage_c/RESULTS_BOARD.md`, `docs/architecture-review/16` (full-dataset
plan) and `18` (Stage D routing plan), `reports/gate/stage_d/route_table.json`, and the
universal usage ledger.

---

## 1. The task and the single objective

The system solves Victorian Certificate of Education (VCE) Mathematical Methods exam
questions — one question PART at a time (an exam question "3(b)(ii)" is one part) — for
a tutoring product. The product constraint is a hard 15-second wall per part: an answer
that arrives after 15 seconds is worth nothing to the user.

**The sole optimization objective is DELIVERED ACCURACY: the fraction of parts answered
correctly within 15 seconds.** A timeout scores zero, identical to a wrong answer, in
the headline metric (they are tracked separately because they have different causes and
different fixes). Latency below the wall is not an objective — it is reported, and used
only to break ties between accuracy-equivalent configurations. Cost is reported, never
selected on. This was fixed by maintainer decision on 2026-07-09 after the routing plan
briefly drifted toward multi-objective selection.

**Timeout (definition used throughout):** the solver failed to return a final answer
within 15 seconds of solve start. Provider throttling and transport retries for API
error codes are excluded from solve time (they are availability events, not solver
behaviour); multi-turn iterations by the solver itself ARE counted (they are the
solver's own time management).

## 2. The system under test

A forced-tool solver (`src/vce_solver/gate/tool_solver.py`): the model is given two
tools — a Python execution tool for computation and an answer-submission tool — and a
frozen "tool contract" prompt that mandates tool use. The solver loops turns (write
code, read result, submit) until it submits or the wall expires. The prompt has five
optimizable components on top of the frozen contract: CODE_STRATEGY (how to write the
one-shot solve script), TURN_DISCIPLINE (time management across turns), ANSWER_FORMAT
(required final-answer form), METHOD_NOTES (per-module solving heuristics), and
WORKED_DEMOS (exemplars). Prompt optimization edits only these five blocks; the
contract is never edited because breaking forced tool-calling silently zeroes a model.

Provider endpoints: OpenRouter by default; Cerebras direct (paid) for gpt-oss-120b
(the free tier's throttling corrupted early measurements — see finding 5); Google's
direct API for gemini zero-thinking (OpenRouter cannot switch gemini's thinking fully
off). The OpenRouter ":nitro" throughput-priority option was never used anywhere in
the campaign (verified in code: no config ever set it), and a later A/B showed no
benefit from it.

## 3. Data assets (all built and verified during the campaign)

- **Dev corpus:** 554 question parts from official VCAA exams and commercial trial
  papers, transcribed to an eval-ready JSONL (one row per part, carrying gold answer,
  answer type, parent stem id).
- **Folds:** 5 development folds {110, 108, 108, 111, 117 parts}, grouped by parent
  stem so sibling parts (a), (b), (c) of one question never straddle a fold boundary
  (they share information; splitting them leaks). Fold 1 (108 parts) is the held-out
  confirmation fold: the optimizer never sees it; every promotion decision is made on
  it. Folds {0,2,3,4} are the fit folds.
- **Verified reference bank v3 (527 rows):** worked solutions for the dev parts,
  verified by a three-model ensemble (gpt-5.5-high + gemini-3.1-pro-high +
  opus-4.8-high, each independently re-deriving the solution with a Python tool).
  Outcome: 5.9% of source solutions had a flawed METHOD despite a correct final answer
  — 29 corrected, 2 dropped (the 2 drops were verifier process failures — one timeout,
  one no-tool-call — not confirmed flaws; they sit in a human-review queue). Lesson: a
  correct final answer does not certify the working.
- **Code-exemplar bank v2 (521 rows):** verified solve SCRIPTS paired row-for-row with
  the verified working. Authored exclusively by capable models (gpt-5.5,
  gemini-3.1-pro, opus-4.8) — never by the student models under optimization. (A first
  version harvested from the students' own trajectories was scrapped for circularity:
  a student teaching itself its own habits.)
- **Marking schemes (1,036 rows):** official award criteria extracted from VCAA marking
  guides (118 parts with explicit award tables) and examiner reports (918 criteria
  rows, of which 285 matched to dev parts). Award types: A (answer) and H (handling)
  marks feed the judge; M (method) marks are extracted but DISABLED in the judge by
  maintainer directive. Where a part has both guide and report criteria, guide rows are
  authoritative and report rows supplement (a precedence bug that let report rows
  overwrite guide rows on 125 parts was found and fixed 2026-07-10). A further 1,421
  "common student errors" rows from examiner reports are banked, unused, reserved as
  future teacher-feedback material.
- **Syllabus:** an equation-complete extraction of the VCE Mathematical Methods study
  design, used by the syllabus-alignment judge.

## 4. Measurement methodology — the statistics and why each piece exists

Each element below was added because its absence produced a specific false positive
during the campaign.

- **Interleaved paired arms.** To compare prompt A vs prompt B, every part is solved
  with BOTH prompts within the same seconds, in randomized order. Sequential comparison
  (all of A, then all of B) was tried first and produced a "significant" +10pp that
  shrank to noise when re-measured interleaved — the two arms had experienced different
  provider-throttle conditions. Provider load varies minute to minute; only same-window
  arms cancel it.
- **McNemar's test on discordant pairs.** The significance test for paired binary
  outcomes. Only parts where the two arms DISAGREE (one right, one wrong — "fixed" and
  "broke" counts) carry information; McNemar computes an exact binomial p-value on that
  discordant set. Reported everywhere as "fixed:broke, p=".
- **A/A calibration.** Run the paired harness with the SAME prompt in both arms; any
  measured delta is pure pipeline noise. The measured noise floor on a 110-part fold
  was ~2.7pp — which retroactively invalidated two earlier "accepted" improvements of
  +1.8pp and +3.6pp (winner's curse: selecting the best of several noisy candidates
  biases the winner's measured score upward).
- **Held-out confirmation only.** GEPA's internal validation score is in-sample to its
  own candidate selection and ALWAYS flatters: the campaign saw +13.6pp and +15pp
  validation gains evaporate completely on the untouched test fold. No promotion is
  ever made from validation numbers.
- **Statistical power at n=108.** With 108 paired parts and campaign-typical
  discordance rates, McNemar at 95% confidence detects only effects of roughly 7-9
  percentage points or larger. Smaller true effects show as "directionally positive,
  not significant". Consequence: a non-significant p does NOT mean "no effect", and the
  strongest evidence at pilot scale is the PATTERN — 28 of 32 objective deltas positive
  across 8 runs (a null lever does not produce that). Per-objective certification is
  deferred to the full corpus (~1,400-part test fold, ~2pp detectable at 80% power).
- **Window acceptance.** A measurement window is discarded and re-run if the transport
  timeout/error rate exceeds 20% (judged on fit-fold parts only, so the evaluation fold
  is never selected on). Providers have real availability incidents — opus-4.7's fast
  endpoint once timed out on 49 of 108 parts while its completed solves stayed fast;
  gemini breached windows on consecutive days; terra's first confirmation window
  breached at 27/108. Paired deltas survive a degraded window (same-window arms);
  absolute levels do not.
- **Multiple-realization noise.** Cheap configs whose routing decisions sit near
  decision boundaries were solved twice in the routing sweep; a part counts at rate
  {0, 0.5, 1}. Documented day-to-day swing for gemini is ±4pp; for gpt-5.6-terra ~11pp
  (its ~10s medians sit close to the wall, so load swings flip timeouts).

## 5. The optimization method — multi-objective GEPA

GEPA (Genetic-Pareto prompt evolution, the `gepa` Python library) is a reflective
prompt optimizer: a strong "teacher" model reads batches of the student's failures
(question, the student's full code trajectory and answer, the verified reference
working, the verified reference script, and the official marking criteria) and proposes
edits to the five prompt components; candidates are kept on a Pareto frontier over
(instance × objective) scores and selected against a validation fold.

**Why multi-objective.** The first attempts scored only binary answer-match and
concluded prompt optimization was dead. A maintainer-directed log review found the
scoring blind to everything a prompt can actually improve: answer FORM (exact vs
decimal, multiple-choice letter only, set/interval notation, contextual units judged
against the reference answer), METHOD soundness of the working, SYLLABUS alignment of
the method, and FIGURE correctness. The rebuilt score is a 5-vector per part —
correctness, form (units folded in, judged only where the reference carries units),
method, syllabus (ALIGNED=1 / PARTIAL=0.5 / OFF=0), figure — with a composite scalar
(weights 0.40 / 0.20 / 0.15 / 0.15 / 0.10) so a correct-but-wrong-form part is not
"perfect" and reflection fires on it. Under this scoring, gains appeared immediately
and held on the test fold. Verdict: the earlier negative was a measurement artifact.

**Role separation rules (hard):**
- The teacher's model family never equals the student's (independence of the
  improvement signal). Teachers used: gemini-3.1-pro-high for most students,
  opus-4.8-high for the gemini student.
- The rescue judge (gpt-5.5, used when rule-grading can't parse an answer) is never a
  model being compared; when gpt-5.5 itself is the student, its solves are rescued by
  gemini-3.1-pro instead.
- The in-loop facet judge (gpt-5.4, no reasoning, structured outputs) returns
  form/units/method/syllabus verdicts fast enough to score every rollout.
- Anti-memorization constraint on the teacher: general method guidance only, never a
  specific question's answer (an early champion had memorized "the answer is D").

## 6. Chronology of attempts — including everything that failed, with full numbers

**6.1 Retrieval of worked exemplars (Stage C.1).** Inject k=2 verified worked solutions
(leave-stem-out retrieval) into the gpt-oss prompt. Paired over all 554 dev parts:
baseline 286 correct (51.6%) vs +exemplars 310 (56.0%), net +4.3pp — but McNemar shows
fixed 128 / broke 104, p=0.13. Verdict: marginal with HIGH CHURN — exemplars fix many
parts and distract nearly as many easy ones. Always-on injection is not a win; the
lever needs targeting (inject only when retrieval is confident or the part is hard).
Shelved as a conditional feature. (The depressed absolute baseline was free-tier
throttle; both arms shared it, so the paired delta stands.)

**6.2 Hand-rolled optimizer v1, single-objective.** Teacher gpt-5.5-low, failure traces
fed back, validation-gated acceptance. Validation composite 0.291 → 0.445 (looked
+15pp). Interleaved paired confirmation on 108 parts: 33.3% → 38.0% (+4.7pp), fixed 25
/ broke 20, p=0.55 — not significant. Root cause of the inflation: the harness scored
the baseline ONCE up front and candidates later, so Cerebras free-tier throttle drift
(which depressed the baseline from a clean-run 74% to 33%) flattered every later
candidate. Two rules came out of this: optimize on paid, stable substrates only, and
validation scoring must be paired-in-time.

**6.3 v2.1 structured single-edit + the A/A calibration.** A/A run (identical prompt in
both arms, 110 parts): null delta −2.7pp, p=0.55 — the pipeline's own noise floor. The
two v2.1 "accepted" improvements (+1.8pp, +3.6pp) sat AT that floor: winner's curse,
withdrawn. A 554-part paired confirmation of the turn-discipline block on gpt-5.4:
accuracy 85.2% → 84.8% (−0.4pp, p=0.89 — neutral) but timeouts 27 → 10 (−63%) and mean
compute 8.15s → 7.74s. The block was kept for robustness, not accuracy. This run also
located the bottleneck precisely: gpt-5.4's 14.8% misses decompose to ~4.9pp timeouts +
~9.9pp wrong answers — for a fast, disciplined model the binding constraint is
correctness, not delivery.

**6.4 Real `gepa` library, single-objective, every known defect fixed (v3).** Pareto
frontier, reflective mutation, reference working fed to the reflector, prompt growth
uncapped, verified v3 bank, equation-complete syllabus, and the Windows cp1252 encoding
bug fixed (PYTHONUTF8=1 — it had been silently discarding reflection proposals
containing math symbols). Validation 0.345 → 0.482 (+13.6pp). Test fold: seed 69.4% vs
champion 67.6% (−1.9pp, fixed 17 / broke 19, p=0.87) — no gain, and the grown
7,318-character prompt raised timeouts 19 → 27. At this point the campaign held a
seemingly robust negative: three fully-correct attempts, all validation gains
evaporating on held-out parts.

**6.5 The maintainer-directed reversal — multi-objective rebuild.** A log review found
the negative was a measurement artifact: (a) scoring was blind to form, units, method,
syllabus, and figures — the objectives a VCE prompt actually improves; (b) the teacher
(gpt-5.5-low) was underpowered and near the judge family; (c) one champion had
memorized answers ("the answer is D"); (d) the "prompt growth causes timeouts" claim
was false — a clean probe showed the champion FASTER (5.1s vs 5.6s, fewer turns, zero
timeouts); the earlier timeout excess was throttle noise. Rebuilt multi-objective (the
5-vector of section 5, gemini-3.1-pro-high teacher, anti-memorization constraint):
validation composite 0.806 → 0.856, and — the decisive test — held-out confirmation on
gpt-oss: correctness 78.7% → 88.9% (+10.2pp, fixed 15 / broke 4, p=0.019), form 66.7% →
80.6% (+13.9pp, fixed 21 / broke 6, p=0.006), method +8.3pp, syllabus +6.9pp, composite
0.795 → 0.887. Both headline objectives SIGNIFICANT on parts the optimizer never saw.
Prompt optimization works when you score what prompts can change.

**6.6 The strong-model check.** The same harness run natively on gpt-5.4-none: val
0.835 → 0.862 but the test fold showed correctness −1.9pp, form +2.8pp, method −1.9pp,
syllabus −0.9pp — all not significant. Combined with the earlier single-objective null
and a cross-model transfer null, this established the two-sided regime claim: cheap or
misbehaving students gain; already-disciplined students do not.

**6.7 Runs 1-8, the production campaign.** Final harness additions before these runs:
the capable-model code-exemplar bank v2 (the reflection teacher now sees the correct
METHOD and the correct CODE side by side), and official marking criteria in the facet
judge. Full results in section 7.

**The unifying mechanism (the campaign's central finding):** prompt optimization fixes
BEHAVIOUR under the 15-second wall — turn discipline, timeout avoidance, answer form,
syllabus-style working — never mathematical skill. Gains scale with how much
prompt-shaped misbehaviour a model has: gpt-5.5 (15 timeouts) and gpt-5.6-terra (20
timeouts) gained most; gpt-4.8-fast-low gained on form/method style; gpt-5.4 with no
reasoning, already disciplined, gained nothing in four independent attempts and runs at
seed. This single account reconciles every positive and null result in the campaign.

## 7. The final board — eight confirmed runs

Held-out fold (108 parts), interleaved paired seed-vs-champion, McNemar p. "Delivered"
= correct within 15s.

| # | Config | Seed | Champion | Delta | Timeouts s→c | $/part | Champion used? |
|---|--------|------|----------|-------|--------------|--------|----------------|
| 1 | opus-4.7-fast-low | 92.6% | **94.4%** | +1.9pp | 2→1 | $0.162 | yes |
| 2 | gemini-3.5-0think (direct) | 88.9%¹ | **89.8%** | +0.9pp | 8→6 | ~$0.005 | yes |
| 3 | gpt-5.4-low | 85.2% | **88.0%** | +2.8pp | 11→11 | $0.011 | yes |
| 4 | opus-4.8-fast-low | 81.5% | **87.0%** | +5.6pp | 5→3 | $0.043 | yes |
| 5 | gpt-5.5-none | 81.5% | **86.1%** | +4.6pp | 15→4 | $0.024 | yes |
| 6 | gpt-oss-120b paid (medium) | 80.6% | **85.2%** | +4.6pp | 0→0 | ~$0.002 | yes |
| 7 | gpt-5.4-none | 85.2% | 86.1% | +0.9pp (null ×4) | 8→5 | ~$0.015 | **no — seed** |
| 8 | gpt-5.6-terra-none | 74.1%² | **77.8%** | +3.7pp | 20→11 | $0.0064 seed | yes |

¹ gemini seed variance ±4pp with provider load (probe 81.5%/16 timeouts vs
confirmation-day 88.9%/8). ² terra variance ~11pp (probe 85.2%/8 timeouts vs confirm
windows 68.5-74.1%; first window breached transport acceptance and was discarded) —
near-wall ~10s medians make it acutely load-sensitive. Paired deltas are same-window
and unaffected by either.

### Per-run rubric tables (held-out, interleaved, 108 parts; seed → champion, fixed:broke, McNemar p)

"val" figures are the optimizer's internal composite trajectory — in-sample, context
only. Objectives: correctness (rule grade + judge rescue), form (required form +
contextual units, official criteria authoritative), method (working soundness),
syllabus (ALIGNED=1 / PARTIAL=0.5 / OFF=0).

**Run 1 — gpt-oss-120b paid, teacher gemini-3.1-pro (val 0.774 → 0.870)**

| Objective | seed → champ | fixed:broke | p |
|---|---|---|---|
| correctness | 0.806 → 0.852 (+4.6) | 12:7 | 0.359 |
| form | 0.685 → 0.769 (+8.3) | 18:9 | 0.122 |
| method | 0.806 → 0.833 (+2.8) | 11:8 | 0.648 |
| syllabus | 0.866 → 0.931 (+6.5) | 11:3 | 0.057 |

Third consecutive all-positive replication on this student. Timeouts 0 → 0.

**Run 2 — gpt-5.5-none, teacher gemini-3.1-pro (val 0.640 → 0.865)**

| Objective | seed → champ | fixed:broke | p |
|---|---|---|---|
| correctness | 0.815 → 0.861 (+4.6) | 9:4 | 0.267 |
| form | 0.759 → 0.843 (+8.3) | 14:5 | 0.064 |
| method | 0.833 → 0.889 (+5.6) | 10:4 | 0.180 |
| syllabus | 0.824 → 0.921 (+9.7) | 14:3 | **0.013** |

Timeouts 15 → 4 (−73%): the low val baseline was wall misbehaviour; the champion fixed
the behaviour. The only single-cell certification at pilot power.

**Run 3 — gpt-5.4-none, teacher gemini-3.1-pro (val 0.836 → 0.879)**

| Objective | seed → champ | fixed:broke | p |
|---|---|---|---|
| correctness | 0.852 → 0.861 (+0.9) | 6:5 | 1.000 |
| form | 0.787 → 0.806 (+1.9) | 8:6 | 0.791 |
| method | 0.880 → 0.852 (−2.8) | 6:9 | 0.607 |
| syllabus | 0.884 → 0.907 (+2.3) | 7:5 | 0.774 |

Fourth consecutive null on this model (single-objective, cross-model transfer, native
multi-objective, code-exemplar re-try). Verdict FINAL at pilot scale: routes at seed.
Its −2.8pp method is the single negative cell of the campaign's 32.

**Run 4 — gpt-5.4-low, teacher gemini-3.1-pro (val 0.739 → 0.819)**

| Objective | seed → champ | fixed:broke | p |
|---|---|---|---|
| correctness | 0.852 → 0.880 (+2.8) | 5:2 | 0.453 |
| form | 0.796 → 0.824 (+2.8) | 8:5 | 0.581 |
| method | 0.861 → 0.880 (+1.9) | 6:4 | 0.754 |
| syllabus | 0.875 → 0.875 (0.0) | 5:5 | 1.000 |

Timeouts unchanged 11 → 11 — unlike gpt-5.5, this champion did not fix the wall.

**Run 5 — gemini-3.5 zero-thinking direct, teacher opus-4.8 (val 0.745 → 0.866)**

| Objective | seed → champ | fixed:broke | p |
|---|---|---|---|
| correctness | 0.889 → 0.898 (+0.9) | 3:2 | 1.000 |
| form | 0.852 → 0.870 (+1.9) | 5:3 | 0.727 |
| method | 0.917 → 0.917 (0.0) | 1:1 | 1.000 |
| syllabus | 0.903 → 0.921 (+1.9) | 3:1 | 0.625 |

Timeouts 8 → 6. Seed already strong on confirmation day (variance note ¹ above).

**Run 6 — opus-4.8-fast-low, teacher gemini-3.1-pro (val 0.822 → 0.863)**

| Objective | seed → champ | fixed:broke | p |
|---|---|---|---|
| correctness | 0.815 → 0.870 (+5.6) | 12:6 | 0.238 |
| form | 0.741 → 0.815 (+7.4) | 14:6 | 0.115 |
| method | 0.806 → 0.861 (+5.6) | 13:7 | 0.263 |
| syllabus | 0.838 → 0.875 (+3.7) | 10:6 | 0.454 |

Timeouts 5 → 3. Largest opus gain — every objective positive.

**Run 7 — opus-4.7-fast-low, teacher gemini-3.1-pro (val 0.870 → 0.891)**

| Objective | seed → champ | fixed:broke | p |
|---|---|---|---|
| correctness | 0.926 → 0.944 (+1.9) | 3:1 | 0.625 |
| form | 0.880 → 0.880 (0.0) | 4:4 | 1.000 |
| method | 0.935 → 0.954 (+1.9) | 5:3 | 0.727 |
| syllabus | 0.958 → 0.968 (+0.9) | 2:1 | 1.000 |

Timeouts 2 → 1. Board ceiling: 94.4% delivered, method 95.4%, syllabus 96.8%.

**Run 8 — gpt-5.6-terra-none, teacher gemini-3.1-pro (val 0.597 → 0.850)**

| Objective | seed → champ | fixed:broke | p |
|---|---|---|---|
| correctness | 0.741 → 0.778 (+3.7) | 15:11 | 0.557 |
| form | 0.630 → 0.731 (+10.2) | 23:12 | 0.090 |
| method | 0.731 → 0.806 (+7.4) | 20:12 | 0.215 |
| syllabus | 0.787 → 0.843 (+5.6) | 15:9 | 0.307 |

Timeouts 20 → 11 (−45%). First confirm window breached transport acceptance (27/108
seed timeouts) and was discarded; both windows agreed directionally (all 8 deltas
positive across the two). Run cost ≈ $37 all-in per the ledger.

### Latency distributions — full pool, one shared window (2026-07-09, 108 parts each)

Statistics over completed solves; timeouts censored at the 15s wall, counted
separately. All configs measured back-to-back in the SAME window so they compare with
each other (the confirmation tables above are each their own window).

| Config | Min | Median | Mean | p95 | Max | Timeouts /108 | $/part (window) |
|---|---|---|---|---|---|---|---|
| opus-4.7-fast-low+champ ⚠ | 3.4s | 6.2s | 6.6s | 11.6s | 14.0s | 49 | $0.1402 |
| gemini-3.5-0think+champ | 3.6s | 7.2s | 7.9s | 12.7s | 13.8s | 5 | $0.0037 |
| gpt-5.4-low+champ | 5.1s | 8.2s | 8.6s | 11.9s | 14.4s | 10 | $0.0115 |
| opus-4.8-fast-low+champ | 3.5s | 5.7s | 6.2s | 10.0s | 13.5s | 2 | $0.0574 |
| gpt-5.5+champ | 4.5s | 7.8s | 8.2s | 11.9s | 15.0s | 3 | $0.0190 |
| gpt-oss-paid+champ | 1.6s | 3.6s | 4.2s | 8.5s | 14.2s | 2 | ~$0.002 est |
| gpt-5.4 seed | 4.9s | 8.0s | 8.1s | 12.8s | 15.0s | 5 | $0.0081 |
| gpt-5.6-terra-none seed (separate window, same fold) | 7.7s | 9.9s | — | 11.8s | 14.8s | 8 | $0.0064 |

⚠ The 49-timeout opus-4.7 figure is the availability incident of finding 15: in this
window the provider's fast endpoint timed out on 49 of 108 parts while completed solves
stayed fast (median 6.2s) — provider congestion, not the prompt or the model. It is why
the production configuration carries a fallback chain.

**Champion latency effect:** champions are FASTER than seeds despite longer prompts
(gpt-5.5 mean 9.2→8.2s) because turn discipline dominates added input tokens; they cost
more per part (added input tokens: gpt-5.5 $0.013→$0.024).

**Probed and excluded configs:** sonnet-5 (best 75.0%, 24 timeouts — a ~10.6s thinking
floor makes it wall-hostile; excluded, assigned to the un-walled verification band);
gemini via OpenRouter (dominated by the direct zero-thinking endpoint; gemini's limit
is generation speed, not thinking — zero-thinking still ran 11s); gpt-oss at low/high
effort (medium, the provider default, beats both — high is actively worse, -4.6pp);
gpt-5.6-sol-none 84.3% (dominated by terra at half its price); gpt-5.6-luna-none 79.6%.

## 8. Stage D — the routing investigation (verdict: does not pay at pilot scale)

**What the router is:** a static lookup table {cell → config + fallback chain}. A
part's cell is assigned at ingestion time, offline, before any clock starts; production
does one table lookup and ONE solve under the wall. No runtime intelligence, no added
latency. Cells are labels applied AFTER the fact to solve logs — nothing was "routed"
during measurement; the first genuinely routed execution would have been the live
validation run (which was never warranted — below).

**Plan discipline — the adversarial review.** The v1 plan was reviewed by an
independent fresh-context critic (a separate model instance starting from the docs and
code, not the planner's reasoning); verdict "execute with fixes", 14 findings, all
incorporated into v2 before any spend. The material ones:

- **Cell granularity is statistical theatre at pilot n.** Module × answer-type gives 45
  nominal cells, 34 populated, 19 after sparse-merge — median fit-cell size ~17 parts,
  posterior standard deviation ~6pp, far above any routing epsilon. Fix: route on 4
  answer-type cells (n 85-190 each, posterior SD 2.5-3.5pp) or 5 module cells; keep the
  cross as descriptive reporting only; the fine grain returns at full-corpus n.
- **The v1 promotion rule was mathematically unpassable.** "CI lower bound > −1pp at
  n=108" cannot be met at campaign discordance rates (CI half-width ~6pp). Fix: a
  pilot-honest non-inferiority gate (lower bound > −5pp), with the −1pp certification
  explicitly deferred to full corpus.
- **Prior leakage.** The v1 Beta-Binomial prior used dev-wide rates, leaking fold 1
  into every posterior. Fix: priors from fit folds {0,2,3,4} only.
- **GEPA contamination.** Champions were selected on exactly the fit folds, so fit-fold
  champion rates are in-sample-optimistic (campaign history: 5-15pp val optimism). Fix:
  champion priors anchor to their HELD-OUT board rates; a live paired run — not the
  replay — would be the promotion decision-maker.
- **Cheap-model realization noise.** Documented day-to-day swing (gemini ±4pp) exceeds
  any epsilon at the cheap-vs-strong boundary. Fix: double-pass the cheap configs; a
  part counts at rate {0, 0.5, 1}.
- **Timeouts modelled separately from wrongness** (a timeout is a re-drawable
  availability event; a wrong answer at temperature 0 mostly is not), **window
  acceptance keyed to fit-fold transport metrics only** (no realization selection on
  the evaluation fold), and **escalation triggers redesigned** (the v1 "posterior <80%
  escalates the cell" was a route-time-known permanent double-solve — deleted).

**D0 budget honesty:** the plan costed the sweep at $150-240 including one contingency
re-sweep of opus-4.7 (the 49-timeout incident showed availability failure is not a
tail case). Actual: ~$149 plus one gemini window re-run.

**D0 sweep (the reusable asset):** all 7 configs solved all 554 dev parts once (cheap
configs twice) in accepted transport windows — 4,986 graded solves logged per-part.

| Config | Delivered (554 parts, one window) | Timeouts |
|---|---|---|
| opus-4.7-fast-low champion | 89.2% | 24 |
| gpt-5.4-none seed | 86.6% | 26 |
| opus-4.8-fast-low champion | 84.7% | 20 |
| gemini-3.5-0think champion | 84.3% (2 passes) | 125 |
| gpt-5.5 champion | 84.1% | 35 |
| gpt-oss-120b champion | 81.5% (2 passes) | 32 |
| gpt-5.4-low champion | 80.9% | 69 (window artifact) |

**Oracle-ceiling gate (run before any policy fitting):** best single config 89.2%;
oracle (a part counts if ANY config solved it) 94.6% → +5.4pp headroom exists, so
fitting proceeded. But the headroom is SCATTERED, not cell-shaped. Pairwise error
correlation (phi coefficient on wrongness — the binary-variable analogue of a Pearson
correlation; +1 would mean the two models miss exactly the same parts):

| Pair | phi |
|---|---|
| gpt-oss ~ opus-4.7 | +0.53 |
| gemini-0think ~ opus-4.7 | +0.58 |
| gpt-5.5 ~ opus-4.7 | +0.47 |
| gpt-oss ~ gemini-0think | +0.57 |

Escalation recovery — the probability a second opinion rescues a miss:

| Conditional | Value |
|---|---|
| P(opus-4.7 correct \| gpt-oss wrong) | 0.47 (n=74) |
| P(opus-4.7 correct \| gemini wrong) | 0.40 (n=67) |
| P(gpt-5.5 correct \| gpt-oss wrong) | 0.28 (n=74) |
| P(gpt-5.5 correct \| gemini wrong) | 0.31 (n=67) |

The models miss the SAME hard parts; escalation rescues under half of cheap-config
misses even to the board leader.

**Posterior estimation (the theory):** per config per cell, delivered-correct is
modelled Beta-Binomial — a Beta(κ·p_g, κ·(1-p_g)) prior (p_g = the config's global
rate; champions anchored to their HELD-OUT board rates to avoid in-sample optimism)
updated with the cell's fit-fold counts. κ (prior strength, swept {5,10,20}) shrinks
thin-cell estimates toward the global rate — the standard guard against winner's-curse
routing, where a cell leader chosen from noisy small-sample estimates is
systematically overrated.

**Replay and verdict:** an offline replay grid (grain ∈ {answer-type, module} × κ ∈
{5,10,20} × epsilon ∈ {0, 1, 2, 3, 5, 8}pp — 36 policies) was FLAT: fit accuracy 0.886
for every single policy, because opus-4.7 wins almost every cell under every setting.
The knee rule (simplest grain, most shrinkage, within 1 standard error ≈ 1.5pp of the
maximum) chose grain=answer-type, κ=20, epsilon=0, yielding the fitted table:

| Cell (answer type) | Routed config |
|---|---|
| mc_letter (multiple-choice letter) | opus-4.7-fast-low |
| numeric | opus-4.7-fast-low |
| other (prose / show-that) | opus-4.7-fast-low |
| expression (algebraic form) | gpt-5.5 ← the one non-opus cell |

Frozen-policy replay on fold 1 (108 held-out parts, outcomes frozen at first accepted
window): routed 93.5% vs best-single 94.4% vs oracle ceiling 96.3%. The expression →
gpt-5.5 assignment was coarse-grain winner's curse and LOST 0.9pp — exactly the failure
mode the adversarial review predicted. The live paired validation (the planned
decision-maker) was skipped: nothing to promote. Cell routing returns at full-corpus
n≈1,400, where per-cell power is real. The D0 sweep log remains the reusable asset.

## 9. Production configuration (the pilot's shipped decision)

**opus-4.7-fast-low with its GEPA champion prompt, for all parts** (94.4% delivered,
method 95.4%, syllabus 96.8%), plus an availability fallback chain — opus-4.8-fast-low
champion (87.0%, one quarter the price, 2-timeout window) → gpt-5.4-none seed (86.1%,
stable anchor) — plus timeout-triggered escalation. The fallback chain exists because
availability is a first-class variable: the board leader had a 49/108-timeout provider
incident. gpt-5.6-terra's champion is a CANDIDATE third fallback (family diversity)
pending availability data; its ~11pp load swing currently argues against it.

**Cost picture (universal ledger, every API call logs model, role, tokens, cost,
duration):** production $/part spans $0.002 (gpt-oss) to $0.162 (opus-4.7 champion).
GEPA run costs: cheap students $15-37, opus-4.8 ≈$90, opus-4.7 ≈$260 across restarts.
Confirmations $6-10 each. The ledger enables run forecasting: token totals per role ×
model, per run, are queryable (`scripts/gate/usage_report.py`).

## 10. The sixteen campaign findings — full statements

**What works (confirmed on held-out data):**

1. **Multi-objective prompt optimization (GEPA) lifts models with prompt-shaped
   misbehaviour.** gpt-5.5: syllabus +9.7pp (p=0.013), timeouts 15→4. gpt-oss: form
   +8-12pp (p=0.007 on the clean substrate), all objectives positive across three
   independent replications. opus-4.8-fast-low: +5.6pp with every objective positive.
   gpt-5.6-terra: timeouts 20→11, form +10.2pp (p=0.090).
2. **The mechanism: prompts fix behaviour under the 15-second wall** — turn discipline,
   timeout avoidance, answer form, syllabus style — never mathematical skill. Gains
   scale with how much misbehaviour a model has.
3. **Best single configuration: opus-4.7-fast-low + champion = 94.4% delivered**
   (method 95.4%, syllabus 96.8%).
4. **Champions are faster despite longer prompts** (turn discipline dominates: gpt-5.5
   mean 9.2→8.2s) but cost more per part (added input tokens: $0.013→$0.024). Routing
   economics must use champion prices.
5. **The paid substrate transformed the cheap tier:** free-tier throttling alone cost
   ~14pp delivered and tripled measurement churn (38 vs 16 discordant of 108); gpt-oss
   paid = 85.2% at ~$0.002/part.

**What does not work (closed with clean evidence):**

6. **Prompt tuning on already-disciplined models: null.** gpt-5.4-none — four
   independent attempts (single-objective, cross-model transfer, native
   multi-objective, code-exemplar re-try), all null on held-out parts. Routes at seed.
7. **Cell-level routing at pilot scale cannot beat best-single.** Oracle headroom
   exists (+5.4pp) but is scattered (error phi +0.47-0.58 — the models miss the same
   hard parts); the one non-opus cell the fit chose was winner's curse and lost on
   fold 1 (93.5% vs 94.4%). Returns at full-corpus power.
8. **sonnet-5 and gemini-via-OpenRouter are wall-hostile in every configuration**
   (sonnet's thinking floor ~10.6s, best 75.0% with 24 timeouts, reasoning-none
   collapses to 41.7%; gemini's limit is generation speed, not thinking —
   zero-thinking still ran 11s via OpenRouter).
9. **Cross-model prompt transfer: null.** Champions are per-model artifacts; each
   deployment model needs its own optimization pass.

**Data quality:**

10. **5.9% of reference worked solutions had a flawed method despite a correct final
    answer** — caught by the three-model REPL ensemble; 29 corrected, 2 dropped (the 2
    were verifier process failures awaiting human review, not confirmed flaws).
11. **~2-4% transcription-error rate expected in digitized questions** —
    ensemble-disagreement gate plus a local human-review interface built for the queue
    (`review_app.py`: approve / edit / reject / transcription-error decisions to an
    append-only log).
12. **Official marking criteria are extractable at scale:** 118 parts with explicit
    A/M/H award tables (method marks retained but judge-disabled), 918 examiner-report
    criteria rows (285 gold-matched), formulas recovered via WMF→vision-OCR.

**Measurement lessons — each caught a would-be false positive:**

13. **In-sample validation always flatters:** +13-15pp validation gains evaporated on
    held-out parts twice before the discipline was fixed.
14. **Interleaved paired arms are mandatory:** sequential comparison was
    throttle-confounded — a "significant" +10pp shrank to honest error bars once both
    arms ran in the same seconds.
15. **Provider availability is a first-class variable:** opus-4.7 had a 49/108-timeout
    window; gemini breached three windows in two days; terra's first confirmation
    window breached at 27/108. Fallback chains and a ledger-driven monitor (every call
    logs duration) exist for this.
16. **n=108 certifies only effects ≥7-9pp:** the pilot's +2-8pp objective gains are
    directionally consistent (28/32 positive deltas) and await full-corpus n≈1,400
    (~2pp power) for per-objective certification.

## 10b. The gpt-5.6 series episode (2026-07-10) — a small case study in the discipline

Three new models (openai/gpt-5.6-sol, -terra, -luna) were smoke-tested (14 parts, no
reasoning and low reasoning both under the wall — sol and terra went 14/14). The
perfect smokes did NOT hold at n=108: terra-none 85.2% (8 timeouts, median 9.9s,
$0.0064/part), sol-none 84.3% (8, 11.3s, $0.0130), luna-none 79.6% (8, 10.0s, $0.0027)
— small-n flattery, the same lesson as finding 13 in miniature. Disposition: terra
recorded (best of series; family diversity for the fallback chain), sol and luna
dominated. A same-day GEPA pass on terra (run 8, ≈$37) confirmed the behaviour
mechanism once more: timeouts 20→11, all objectives positive, form +10.2pp (p=0.090).
Terra's near-wall medians make it the most availability-fragile config on the board
(~11pp absolute swing with provider load). During this episode the OpenRouter ":nitro"
provider option was audited: never used anywhere in the campaign, and an A/B showed no
benefit.

## 11. The full-corpus plan (next phase, briefly)

Corpus: ~7,820 parts — 115 raw exam files already local, plus the ~3,748-question
Elevatar bank (~2.3 parts per question), which is blocked on its export (a structured
dump beats re-extraction because it introduces no new transcription errors; the export
format specification and a validator exist).

Pipeline, in order: part-splitting (one row per part, stem id carried) → offline
module/subtopic classification (gemini-3.5-flash, 95% module / 86% subtopic accuracy,
never inside the solve wall) → transcription-error gate (solver disagrees with gold +
ensemble unanimity against gold = probable transcription error → human queue; ensemble
split = hard part, gold stands) → figure-criteria derivation (marking-scheme features
verbatim where available; otherwise opus-4.8 derives 3-5 objective criteria with a 10%
human spot-check) → reference-working generation and verification (capability ladder;
tier-1 gpt-5.5-high + REPL over all, three-model ensemble over flags, human interface
for the contested) → per-model GEPA concentrated on the cheap tier (6,000 metric calls
each; pilot improvements saturated by ~40% of budget), with module-stratified
reflection minibatches → per-objective McNemar with stem-clustered bootstrap confidence
intervals (resample stems, not parts) and Benjamini-Hochberg false-discovery-rate
correction across the 5-objective × 3-model grid → iterative cell routing with shrunk
posteriors (route to the cheapest model within epsilon of best; weak cells get targeted
GEPA passes; a route change ships only after beating the incumbent paired at p<0.05) →
one-shot final evaluation on an untouched holdout year, after which nothing changes.

Power: at n≈1,400 paired parts and pilot-level discordance, minimum detectable
correctness effect ≈ 2pp at 80% power. Promotion rule: correctness lower confidence
bound > −1pp AND at least one objective significant after FDR correction.

Cost and schedule estimate (from the plan of record):

| Phase | Item | Estimate |
|---|---|---|
| 0 | Gold verification sweep (7,820 parts, ensemble on ~10% flags) | $80-150 |
| 0 | Reference bank generation + verification | $60-120 |
| 0 | Figure criteria derivation + spot-check | $15-30 |
| 2 | GEPA × 3 models (6,000 calls each, facet judge included) | $60-150 |
| 3 | Interleaved confirmations × 3 models × ~1,400 parts × 2 arms | $50-120 |
| 4 | Routing iterations (2-3 passes over dev) | $40-90 |
| 5 | Holdout final (routed + best-single, ~700 parts) | $10-25 |
| | **Total** | **~$315-685** |

Wall-clock: Phase 0 is the long pole (~1-2 days of unattended verification runs);
Phases 2-5 fit in 2-3 working days. All long runs are crash-safe (append-only part
logs, checkpointed optimizer state, resume flags).

## 11b. Glossary

- **Delivered accuracy:** fraction of parts answered correctly within the 15-second
  wall. Timeout = zero.
- **Part:** the atomic eval unit — one sub-question of an exam question. **Stem:** the
  parent question; sibling parts share a stem and are never split across folds.
- **Seed:** a model's baseline prompt (frozen contract + default blocks).
  **Champion:** the best GEPA-optimized prompt for that model.
- **GEPA:** Genetic-Pareto reflective prompt evolution (the `gepa` library) — a teacher
  model reads failures and proposes prompt edits; candidates survive on a Pareto
  frontier.
- **Facet judge:** the fast in-loop model (gpt-5.4, structured outputs) scoring form /
  units / method / syllabus per rollout. **Rescue judge:** the model consulted when
  rule-grading cannot parse an answer.
- **McNemar's test:** exact significance test for paired binary outcomes, computed on
  discordant pairs only (fixed = seed wrong→champion right; broke = the reverse).
- **A/A run:** both arms use the identical prompt; the measured delta is the pipeline's
  noise floor.
- **Winner's curse:** selecting the best of several noisy estimates biases the winner's
  measured value upward; the guard is shrinkage (Beta-Binomial priors) and held-out
  confirmation.
- **Beta-Binomial shrinkage:** estimating a cell's rate as a weighted blend of the
  observed cell data and a prior centred on the config's global rate, with prior weight
  κ; thin cells get pulled toward the global rate.
- **Oracle ceiling:** delivered rate if every part were routed to a config that solves
  it — the upper bound any router could reach.
- **Phi coefficient:** correlation for two binary variables; used here on "model missed
  the part" indicators.
- **Window:** one contiguous measurement session against a provider. **Window
  acceptance:** discard and re-run if transport timeout/error rate exceeds 20% on
  fit-fold parts.
- **Transport failure:** timeout or API error attributable to the provider connection,
  as opposed to the solver's own behaviour.
- **Interleaved paired arms:** both prompts solve each part within the same seconds,
  order randomized — cancels provider-load drift.
- **Delivered vs composite:** "delivered" is correctness-only under the wall (the
  headline); "composite" is the weighted 5-objective scalar (optimizer-internal).

## 12. Reading guide for interrogating these results

- Absolute delivered levels carry window noise (gemini ±4pp, terra ~11pp); paired
  deltas within a run are the trustworthy quantity.
- "Not significant" at n=108 means "below ~7-9pp detection", not "zero effect".
- Validation-fold numbers (e.g. "val 0.597→0.850") are optimizer-internal and
  in-sample; never compare them across models or treat them as outcome claims.
- Costs marked "~" are price-sheet estimates (Cerebras returns no usage cost);
  unmarked costs are ledger-measured.
- The D0 sweep table and the board table disagree on absolute levels for the same
  configs because they are different windows over different part sets (554 dev vs 108
  held-out) — both are correct measurements of what they measure.
- Two dropped bank references and the transcription-flag queue await human decisions;
  nothing downstream currently depends on them.
