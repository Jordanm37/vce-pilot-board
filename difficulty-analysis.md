# Difficulty analysis: empirical difficulty versus the exam-marks proxy

Scope and method. This report asks whether the difficulty axis used by the cost model, the
number of exam marks a question part is worth, actually predicts where the solver pool
struggles, and it characterises the hard tail so that routing, escalation, and gold-data
review can be aimed at the right parts. It reads only campaign artefacts already on disk (the per-solve log at
`reports/gate/stage_d/sweep.jsonl`, the gold rows, module tags, and the review queue),
with no new model calls. Every number below is emitted
by `scripts/gate/difficulty_analysis.py` (run command at the end); nothing is hand-computed.

The corpus is 554 question parts from VCE Mathematical Methods exams. VCE is the Victorian
Certificate of Education, the final secondary-school qualification in Victoria, Australia;
Mathematical Methods is its main calculus-and-statistics subject. A "part" is a single
scored sub-question (for example question 6 part b). Seven solver configurations (a
configuration is one model plus its settings, for example `opus-4.7-fast-low`) each
attempted all 554 parts under a hard 15-second wall. A part counts as "delivered" for a
configuration only if the configuration returned the correct answer inside 15 seconds; a
timeout counts as wrong. Two configurations ran the sweep twice (`gemini-3.5-0think` and
`gpt-oss-120b-paid`); a part's score for such a configuration is the mean over its two
passes, and a score of 0.5 or higher counts as solved.

## Definitions used throughout

- **Empirical difficulty of a part** = how many of the 7 configurations missed it, from 0
  to 7. This is the pool's own verdict on hardness, not an a-priori label. Buckets:
  empirically-easy (emp-easy) = 0 misses, empirically-medium (emp-med) = 1 to 3 misses,
  empirically-hard (emp-hard) = 4 or more misses.
- **Marks proxy** = the cost model's difficulty label from exam marks: easy = 1 mark, medium
  = 2 marks, hard = 3 or more marks.
- **Leave-one-out (LOO)** = when profiling a single configuration, its own difficulty axis is
  built from the other 6 configurations only (0 / 1 to 2 / 3 or more of 6 missed), so a
  configuration is never scored against a difficulty scale that its own result helped set.
- **Module** = the syllabus topic a part belongs to, tagged M1 to M5. The noun names, from
  the ontology (`data/gate/ontology.json`), are M1 = Algebra, functions and relations; M2 =
  Polynomials, exponentials and logarithms; M3 = Calculus; M4 = Probability, statistics and
  algorithms; M5 = Trigonometry.
- **tech-active** = calculator-allowed question; **tech-free** = no-calculator question.

## (a) Difficulty distribution

Most parts are easy for the whole pool, and a small, sharp tail is hard for everyone.

| Misses (of 7) | Parts | Share | Cumulative |
|---|---|---|---|
| 0 | 370 | 66.8% | 66.8% |
| 1 | 69 | 12.5% | 79.2% |
| 2 | 33 | 6.0% | 85.2% |
| 3 | 25 | 4.5% | 89.7% |
| 4 | 11 | 2.0% | 91.7% |
| 5 | 6 | 1.1% | 92.8% |
| 6 | 10 | 1.8% | 94.6% |
| 7 | 30 | 5.4% | 100.0% |

Headline: 370 of 554 parts (66.8%) are solved by all 7 configurations, and 30 of 554
(5.4%) are missed by all 7. The distribution is bimodal (two-peaked): a large mass at zero
misses and a second, smaller spike at the maximum of 7 misses (30 parts, larger than the
6-miss and 5-miss bins). Grouped into buckets, 370 parts are emp-easy, 127 are emp-med, and
57 are emp-hard. The 57 emp-hard parts carry the widest per-part differences between
configurations; the other 497 are either solved by everyone or missed by only one to three.

## (b) Does the marks proxy predict empirical difficulty?

Weakly, and non-monotonically (the relationship does not rise steadily with marks). The
Spearman rank correlation between marks (1 to 5) and miss-count (0 to 7) is +0.148 across
all 554 parts. Spearman rank correlation measures how well two orderings agree, from +1
(perfect agreement) through 0 (none) to -1 (reversed); +0.148 is a weak positive
association. With n = 554 it is statistically distinguishable from zero (t is about 3.5),
but the effect size is small: marks explain very little of where the pool actually fails.

Cross-tab of the marks bucket against the empirical bucket, as row percentages (each row
sums across the empirical buckets to 100%):

| Marks bucket | n | emp-easy | emp-med | emp-hard |
|---|---|---|---|---|
| easy (1 mark) | 334 | 72.5% | 20.1% | 7.5% |
| medium (2 marks) | 181 | 56.9% | 27.6% | 15.5% |
| hard (3+ marks) | 39 | 64.1% | 25.6% | 10.3% |

The proxy fails in the direction that matters. The 2-mark parts, not the 3-or-more-mark
parts, have the highest empirically-hard rate (15.5%). The parts the proxy calls "hard"
(3 or more marks) are emp-hard only 10.3% of the time, below the 2-mark rate and barely
above the 1-mark rate. Mean miss-count by marks confirms the non-monotonic shape: 1 mark
0.79, 2 marks 1.40, 3 marks 0.82, 4 marks 1.75 (only 4 such parts), 5 marks 3.00 (a single
part). The curve peaks at 2 marks and drops back at 3.

Read the other way, by where each empirical bucket's mass comes from: of the 57 emp-hard
parts, 25 (44%) are 1-mark, 28 (49%) are 2-mark, and only 4 (7%) are 3-or-more-mark. The
hard tail is almost entirely low-mark parts. A difficulty axis that expects the hard parts
to be the high-mark ones is looking in the wrong place: the 2-mark parts carry the most
emp-hard mass and the highest emp-hard rate, and 3-or-more-mark parts are a rounding error
in the tail.

Verdict: marks and empirical difficulty agree only weakly (rho +0.148), and the agreement
is non-monotonic. High-mark parts are not disproportionately hard for the pool; 2-mark
parts are. Marks are a poor predictor of solver accuracy.

## (c) Per-configuration difficulty profiles (leave-one-out)

Each configuration's delivered rate is shown against a difficulty axis built from the other
6 configurations only, so a configuration never appears on its own x-axis. "Delivered %"
equals the solved rate (correct within 15 seconds).

| Configuration | Overall | emp-easy | emp-med | emp-hard | n (easy/med/hard) |
|---|---|---|---|---|---|
| opus-4.7-fast-low | 89.2% | 98.7% | 86.9% | 43.1% | 375/107/72 |
| gemini-3.5-0think | 87.9% | 98.1% | 87.1% | 38.2% | 377/101/76 |
| gpt-5.4-none-seed | 86.6% | 99.5% | 82.6% | 27.4% | 372/109/73 |
| gpt-oss-120b-paid | 86.6% | 97.9% | 82.0% | 36.8% | 378/100/76 |
| opus-4.8-fast-low | 84.7% | 96.4% | 77.2% | 30.4% | 384/101/69 |
| gpt-5.5 | 84.1% | 96.4% | 78.6% | 22.4% | 384/103/67 |
| gpt-5.4-low | 80.9% | 95.1% | 67.7% | 16.7% | 389/99/66 |

Two facts stand out. First, the easy parts are saturated: every configuration delivers
between 95.1% and 99.5% on emp-easy parts, so the rate spread there is small (4.4 points).
Second, the hard tail is where they differ. On emp-hard parts the delivered rate spreads
from 43.1% (`opus-4.7-fast-low`) down to 16.7% (`gpt-5.4-low`), a range of 26.4 percentage
points. The emp-hard column carries the widest rate spread, though in absolute parts each
bucket contributes comparably to the leaderboard gap (easy: 4.4 points across ~380 parts,
med: 19.4 across ~100, hard: 26.4 across ~70 - roughly 17-19 parts each). The other ~480 parts are near-universally solved or near-universally
missed.

## (d) Anatomy of the hard tail

There are 57 emp-hard parts (4 or more misses), of which 30 are missed by all 7
configurations. The 30 missed-by-all parts, with marks, answer type, module,
and the first line of the question:

Answer-type abbreviations: numeric = a single number; mc_letter = a multiple-choice option
letter; expression = an algebraic expression; set = a set of values; interval = a range;
show_that = prove a stated result (the target is given); multi_value = several required
values.

| Part id | Marks | Answer type | Module | Question (first ~90 chars) |
|---|---|---|---|---|
| 2023-MM2/QB5/e | 1 | numeric | M1 Algebra, functions, and relations | f(x) = e^x + e^(-x); g(x) = (1/2)f(2 - x); smallest k so h meets inverse of h_1 |
| 2024NHT-MM2/B4/c | 1 | expression | M1 Algebra | f(x) = a(x - b)^3 + c on [-4, 0]; Timothy's tiling pattern |
| 2024NHT-MM2/B5/e | 2 | set | M1 Algebra | f(x) = e^(-ex) on R; value of a determined by... |
| 2024NHT-MM2/B4/d | 2 | expression | M2 Poly/exp/log | f(x) = a(x - b)^3 + c on [-4, 0]; Timothy's tiling pattern |
| itute-2024-MM-T2/SA-Q1 | 1 | mc_letter | M2 Poly/exp/log | On [0,1], f(x) = 12x^3 - 14x^2 - 1 has... (describe the roots) |
| 2023-MM2/QB5/f | 2 | numeric | M3 Calculus | f(x) = e^x + e^(-x); g(x) = (1/2)f(2 - x) |
| 2023NHT-MM2/B2/h | 2 | interval | M3 Calculus | C(t) = 65e^(-t/8), t >= 0; C_2(t) = 65e^(-t/8) for 0 <= ... |
| 2023NHT-MM2/Q11 | 1 | mc_letter | M3 Calculus | u = g(x), v = e^(g(x)); d/dx (uv) equals... |
| 2024-MM2/SA-Q2 | 1 | mc_letter | M3 Calculus | g'(x) = x^3 - x, g(0) = 5; find a value of g |
| 2024-MM2/SB-Q2/f.iii | 2 | numeric | M3 Calculus | f(t) = 12 + 30t on 0 <= t <= 1/3; f(t) = 22 for t > 1/3 |
| 2024NHT-MM2/B1/d.iii | 1 | set | M3 Calculus | f(x) = x^3 - px on R; graph passes through the origin... |
| 2024NHT-MM2/B2/g.iii | 2 | numeric | M3 Calculus | h(theta) = 3/2 - 1/2 cos(theta) on [-pi/3, pi/3]; height of... |
| Kilbaha2022-Trial-MM2/QB1/b | 1 | numeric | M3 Calculus | f(x) = x^3 + bx^2 + cx on R |
| Kilbaha2022-Trial-MM2/QB1/c | 1 | expression | M3 Calculus | f(x) = x^3 + bx^2 + cx on R |
| Kilbaha2022-Trial-MM2/QB1/e | 2 | numeric | M3 Calculus | f(x) = x^3 + bx^2 + cx on R |
| Kilbaha2022-Trial-MM2/QB2/b.iv | 1 | numeric | M3 Calculus | while constructing the tent, Ray suggested the tent would be more stable... |
| itute-2024-MM-T2/SA-Q12 | 1 | mc_letter | M3 Calculus | h(x) = f(x)g(x), f(x) = 1 - sin/cos, g(x) = 1 + cos/sin; h'(x)... |
| itute-2024-MM-T2/SB-Q1/h | 2 | show_that | M3 Calculus | A: y = sqrt(x); B: y = (sqrt2/16)(x^2)... |
| 2023-MM2/QB4/d | 2 | numeric | M4 Prob/stats | D ~ Normal(mean 6.7, sd 0.1) cm; a manufacturer produces... |
| 2023-MM2/QB4/e | 2 | numeric | M4 Prob/stats | D ~ Normal(mean 6.7, sd 0.1) cm; a manufacturer produces... |
| 2023-MM2/QB4/j | 2 | numeric | M4 Prob/stats | D ~ Normal(mean 6.7, sd 0.1) cm; a manufacturer produces... |
| 2024-MM2/SB-Q4/e.ii | 1 | multi_value | M4 Prob/stats | f(x) = (1/67500) x^2 (30 - x) on 0 <= x <= 30; f(x) = 0 otherwise |
| 2024NHT-MM2/A12 | 1 | mc_letter | M4 Prob/stats | Newton's method to estimate the x-intercept of f: [0, inf) -> R |
| itute-2024-MM-T2/SB-Q5/c | 3 | multi_value | M4 Prob/stats | survey: 25% of families lived below the poverty line... |
| 2024NHT-MM2/B2/d | 1 | numeric | M5 Trigonometry | h(theta) = 3/2 - 1/2 cos(theta) on [-pi/3, pi/3]; a simple pendulum |
| 2024NHT-MM2/B2/g.i | 2 | numeric | M5 Trigonometry | h(theta) = 3/2 - 1/2 cos(theta); height of... |
| Kilbaha2022-Trial-MM2/QB2/a.iii | 2 | expression | M5 Trigonometry | assume q = 60 degrees; the diagram shows... |
| itute-2024-MM-T2/SA-Q2 | 1 | mc_letter | M5 Trigonometry | average gradient of the increasing section of y = (pi/5) sin(2...) |
| itute-2024-MM-T2/SB-Q2/f | 1 | numeric | M5 Trigonometry | wheels A (r=3), B (r=2), C (r=1) in contact, resting... |
| itute-2024-MM-T2/SB-Q2/g | 1 | numeric | M5 Trigonometry | wheels A (r=3), B (r=2), C (r=1) in contact, resting... |

Breakdown of all 57 emp-hard parts, with the concentrations named after each cut.

By module. M3 Calculus dominates: 27 of 57 (47.4%). Then M5 Trigonometry 9 (15.8%), M1
Algebra 8 (14.0%), M2 Polynomials/exponentials/logarithms 7 (12.3%), M4
Probability/statistics 6 (10.5%). Relative to each module's own size, the emp-hard rate is
15.3% for M3 Calculus (27 of 176) and 14.8% for M5 Trigonometry (9 of 61), both above the
corpus base rate of 10.3%, while M4 Probability/statistics is the safe module at 3.8% (6 of
157), well below base. Calculus and trigonometry are the risk topics; probability and
statistics is the reliable one.

By answer type. numeric 20 (35.1%), mc_letter 10 (17.5%), expression 9 (15.8%), show_that 6
(10.5%), set 6 (10.5%), coordinate 3, multi_value 2, interval 1. No single answer format
owns the tail; it is spread, led by numeric answers.

By marks. 2-mark 28 (49.1%), 1-mark 25 (43.9%), 3-mark 3 (5.3%), 4-mark 1 (1.8%). This is
the same message as section (b) seen inside the tail: 93% of emp-hard parts are worth 1 or 2
marks.

By calculator status. tech-active (calculator allowed) 54 of 57 (94.7%); tech-free 3 (5.3%).
Within each split, the emp-hard rate is 13.0% for tech-active (54 of 415) against 2.2% for
tech-free (3 of 139). The hard tail is almost entirely calculator-allowed questions. These
are the Exam 2 application-and-modelling parts, which wrap the mathematics in a real-world
scenario with multi-step set-up, and that context, not the raw calculation, is where the
solvers fail.

By source paper. The corpus mixes official and commercial papers: Exam 1 and Exam 2 are the
official VCAA (Victorian Curriculum and Assessment Authority) papers, Exam 1 being tech-free
and Exam 2 tech-active; NHT is the VCAA Northern Hemisphere Timetable paper, an official
alternate sitting; Trial Exam papers are commercial practice exams from third-party
publishers such as Kilbaha and itute. The emp-hard rate within each source is: Trial Exam 2
21.5% (20 of 93), Exam 2 11.9% (19 of 159), NHT Exam 2 9.2% (15 of 163), Trial Exam 1 5.0%
(1 of 20), Exam 1 3.3% (2 of 60), NHT Exam 1 0.0% (0 of 59). The commercial Trial Exam 2
parts are about twice as hard for the pool as the official Exam 2 parts. This is a concrete
lead for gold-quality review, because third-party transcriptions are more likely than
official ones to contain scanning or formatting errors (see below).

By year. Raw counts are 2024 27, 2023 15, 2022 11, 2025 4, but raw counts are confounded by
how many parts each year contributes. On a within-year rate the picture changes: 2022 24.4%
(11 of 45), 2024 12.2% (27 of 221), 2023 10.4% (15 of 144), 2025 2.8% (4 of 144). The 2022
spike is not a year effect; every 2022 part in this corpus is a Kilbaha commercial trial
paper, so the 2022 rate is the Trial Exam source effect wearing a date. Year on its own
carries no independent difficulty signal here.

Review-priority flags. A part missed by all 7 configurations has raised odds that its
recorded gold answer or its transcribed question stem is itself wrong, because seven
independent solvers converging on failure is as consistent with a broken item as with a hard
one. Of the 30 missed-by-all parts, 7 carry a mechanical text smell from the ingestion
template: a doubled token such as `f(x)(x)`, `D(D)`, or `f(x) = f(x) = ...`, produced when a
function name already carried its argument and the "DEFINITION: name(args) = rule" template
duplicated it. Those 7 are `2023NHT-MM2/B2/h`, `Kilbaha2022-Trial-MM2/QB1/b`,
`Kilbaha2022-Trial-MM2/QB1/c`, `Kilbaha2022-Trial-MM2/QB1/e`, `2023-MM2/QB4/d`,
`2023-MM2/QB4/e`, and `2023-MM2/QB4/j`. This is a smell, not proof, but it is the right
human-review shortlist. Independently, one missed-by-all part, `2023-MM2/QB5/e`, is already
in `data/gate/review_queue.jsonl` (tagged as a dropped timeout), which shows the
missed-by-all set is already catching known-bad items.

## (e) In the hard tail, is it "cannot do it" or "cannot do it in 15 seconds"?

For the emp-hard parts, each failed solve-attempt is either a timeout (hit the 15-second
wall), a wrong answer (returned in time but incorrect), or an error (the run crashed). The
split separates a capability gap from a speed gap.

| Configuration | Fail-rows | Timeout | Wrong | Error | % timeout |
|---|---|---|---|---|---|
| gemini-3.5-0think | 97 | 64 | 33 | 0 | 66.0% |
| gpt-5.4-low | 55 | 28 | 27 | 0 | 50.9% |
| gpt-5.5 | 52 | 17 | 35 | 0 | 32.7% |
| opus-4.8-fast-low | 48 | 13 | 35 | 0 | 27.1% |
| opus-4.7-fast-low | 41 | 11 | 30 | 0 | 26.8% |
| gpt-5.4-none-seed | 53 | 14 | 39 | 0 | 26.4% |
| gpt-oss-120b-paid | 102 | 12 | 71 | 19 | 11.8% |

Pooled over all 7 configurations there are 448 failing solve-rows on emp-hard parts: 159
timeouts (35.5%), 270 wrong answers (60.3%), 19 errors (4.2%). So most hard-tail failures
are genuine wrong answers, not clock losses. But the mix depends heavily on the
configuration. `gemini-3.5-0think` fails two-thirds of the time by timeout (66.0%), so a
large share of its hard-tail losses are speed, not capability; a longer wall could
plausibly recover some of them (untested - a timed-out attempt may still be wrong given
more time). At the other end, `gpt-oss-120b-paid` almost never times out (11.8%)
and instead returns wrong answers, so its hard-tail losses are a capability ceiling that
more time would not fix. The board leader `opus-4.7-fast-low` sits in between at 26.8%
timeout. The implication for the wall: relaxing the 15-second limit helps the slow-thinking
configurations most and the fast-but-wrong ones not at all.

## (f) Does escalating hard parts to the board leader recover them?

Escalation means running a cheap configuration first and sending only its misses to a
stronger one. The strongest configuration here is the board leader `opus-4.7-fast-low`.

When the escalation trigger is a single cheap configuration's miss, recovery is moderate.
Of the parts `gpt-oss-120b-paid` missed, `opus-4.7-fast-low` solves 47.3% (n = 74); of the
parts `gemini-3.5-0think` missed, it solves 40.3% (n = 67). So escalating a cheap
configuration's failures to the leader recovers roughly 4 in 10.

But recovery collapses as the part gets harder for the pool. Restricting to parts that are
emp-hard from the leave-one-out view (3 or more of the other 6 configurations missed them),
`opus-4.7-fast-low` solves only 43.1% (n = 72). And on the parts that all 6 other
configurations missed, it solves just 14.3% (n = 35). The leader is not a recovery path for
the consensus-hard core; those parts are hard for it too.

This is the pool's known scattered-headroom finding seen from the part level. Pairwise error
correlation, measured by the phi coefficient (a correlation between two yes/no variables,
here whether two configurations both miss the same part; +1 means identical mistakes, 0
means independent), ranges +0.47 to +0.69 across the 21 configuration pairs, mean +0.55. The
leader against the two cheap configurations is +0.53 (`gpt-oss-120b-paid`) and +0.58
(`gemini-3.5-0think`), squarely in the campaign's cited +0.47 to +0.58 band. Positive but
well short of +1 means misses overlap heavily yet not completely: there is some headroom
from disagreement, which is why escalating a single config's misses recovers about 40%, but
the overlap is strong enough that the all-miss core (14.3% recovery) stays largely
unrecovered. The headroom is real but scattered across moderately-hard parts, not
concentrated where a single escalation hop would clean up the tail.

## (g) Implications

For the cost model's marks-based difficulty axis. Marks predict empirical difficulty only
weakly (Spearman +0.148) and non-monotonically, with 2-mark parts harder for the pool than
3-or-more-mark parts. Marks remain fine as a cost axis, because a part's marks do track the
work and therefore the token and time budget a solver spends. They should not be reused as
an accuracy-risk axis. Anywhere the pipeline currently assumes "more marks means the model
is more likely to fail", that assumption is not supported: the failure mass sits on 1-mark
and 2-mark parts.

For routing and escalation design at full corpus. Easy parts are saturated (95.1% to 99.5%
delivered by every configuration), so routing has no accuracy lever there and should route
those to the cheapest or fastest configuration purely on cost and latency. All the accuracy
differentiation lives in the roughly 10% emp-hard tail, where the leader delivers 43.1% and
the weakest 16.7%. Escalating a cheap config's misses to `opus-4.7-fast-low` recovers about
40 to 47%, which is worth doing, but it does not rescue the consensus-hard core (14.3%
recovery when all others miss). A single escalation hop to one strong model will leave a
residual floor; closing more of it needs either genuine model diversity (a configuration
whose errors are less correlated than the current +0.55 mean) or a different mechanism
(tools, longer wall for the timeout-heavy configurations), not more of the same leader.

For reference-bank and transcription-review priorities. The missed-by-all set is the first
place to look for bad gold, because a part all 7 solvers miss is as likely broken as hard.
Priority order: the 7 template-doubling parts listed in section (d) first (`2023-MM2/QB4/d`,
`/e`, `/j`; `2023NHT-MM2/B2/h`; `Kilbaha2022-Trial-MM2/QB1/b`, `/c`, `/e`), then the rest of
the 30 missed-by-all parts, with the commercial Trial Exam 2 parts weighted up because that
source is about twice as hard as the official papers (21.5% versus 11.9% emp-hard) and
third-party transcriptions are the likeliest to carry scanning errors. `2023-MM2/QB5/e` is
already in the review queue, which is a check that this shortlist points at the right items.
Fixing or dropping broken items here directly raises the measured ceiling, because these
parts currently count against every configuration.

## (h) Limitations

- Single-window realisations. Each single-pass configuration attempted each part once under
  one 15-second window. A borderline part can flip on a re-run, so a part's miss-count is a
  point estimate, not a stable property.
- The metric is pool-relative, not absolute. Empirical difficulty is defined by this
  particular 7-configuration pool. "Missed by all 7" means hard for these solvers, not hard
  in any absolute sense; a different or larger pool would redraw the buckets. All difficulty
  claims are relative to the pool.
- Thin per-configuration hard-tail samples. Each configuration's emp-hard leave-one-out
  bucket holds only about 66 to 76 parts, so the emp-hard delivered rates in section (c)
  carry wide uncertainty (a few parts move a rate by several points); read the 26.4-point
  spread as a real ordering but not as precise per-configuration percentages.
- Double-pass configurations are half-weighted per attempt. `gemini-3.5-0think` and
  `gpt-oss-120b-paid` ran twice, and their part score is the mean of two passes, so a
  one-of-two split scores 0.5 and counts as solved. This makes their solved set slightly more
  generous than the single-pass configurations at the exact 0.5 boundary, and it is why the
  emp-hard count is 57 under the stated "score of 0.5 or higher is solved" rule.
- Text-artifact flags are a heuristic. The doubled-token detector is a smell for transcription
  problems, not a proof; each flagged part still needs a human to confirm the gold answer.

## Reproduce

```
PYTHONUTF8=1 python scripts/gate/difficulty_analysis.py
```

Source log: `reports/gate/stage_d/sweep.jsonl`. Gold and tags load through
`src/vce_solver/gate/dataset.py`. No network calls; stdlib only.
