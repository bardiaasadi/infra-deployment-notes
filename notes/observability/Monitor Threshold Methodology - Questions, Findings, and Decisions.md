# Monitor Threshold Methodology — Questions, Findings, and Decisions

> [!instruction] Ledger update rule
> When I paste this ledger, compare it with the current conversation and identify only durable findings or decisions not already recorded. Do not edit, rewrite, reorganize, or reconcile existing content unless explicitly asked. Return only append-ready Markdown matching the ledger’s structure, with no introduction or commentary. If nothing is worth adding, say **Nothing to add.**



**Status:** exhaustive working record; intentionally longer than a presentable methodology note.

**Purpose:** preserve the questions, assumptions, competing interpretations, findings, and decisions that shaped the monitoring methodology, especially the v2 → v3 transition. This is source material for future summaries and presentations. It is not the normative runbook.

**Normative implementation:** [[Monitor Threshold Analysis Runbook]]
**Conceptual companion:** [[Monitor Threshold Methodology - Concepts]]

## Reading guide

* **[SHOWCASE]** — strong candidate for a concise presentation or executive summary.
* **[OPEN]** — unresolved or deliberately deferred.
* **[TENSION]** — two statements or design choices that must remain visible together; this draft does not resolve them.
* **[DUPLICATE]** — substantially overlaps another item and may be merged during editing.

Each item records:

> **Question / assumption** — what was challenged.
> **Finding** — what the evidence or reasoning showed.
> **Decision** — what the methodology now does.
> **Why it matters** — the failure or misleading conclusion this prevents.

---

# 1. Monitoring objective and philosophy

## 1.1 [SHOWCASE] Should the methodology detect every statistically abnormal period?

**Question / assumption**
A sensitive monitor should report every period that differs from historical normal.

**Finding**
Across 30–50 series, detecting every abnormal period creates an operationally unmanageable monitor set. Statistical abnormality is also not the same as meaningful operational degradation.

**Decision**
The active first-round monitor set focuses on two conditions: sustained relative degradation (Monitor A) and severe absolute failure (Monitor C). The proposed accumulated-degradation Monitor B was deferred before any monitors were created.

**Why it matters**
This keeps the monitor set actionable and prevents a statistically sophisticated system from becoming an alert-noise generator. Drift control remains a valid future objective, but it is not part of the current Datadog monitor set.

## 1.2 [SHOWCASE] Is alert volume an input to threshold calibration?

**Question / assumption**
If historical backtesting produces too many events, the threshold should be raised until the event count is acceptable.

**Finding**
This turns a desired alert budget into a hidden definition of "normal." A frequently degraded service can therefore train the method to ignore its own degradation.

**Decision**
Alert volume is a review output, not a calibration input. Thresholds are not mechanically raised to make history quieter.

**Why it matters**
It prevents the method from normalizing known or chronic service problems.

## 1.3 Is quiet history proof that a threshold is correct?

**Question / assumption**
A threshold that never fires in backtesting is probably well calibrated.

**Finding**
Silence can mean the service is healthy, but it can also mean the metric is blind, the threshold is unreachable, or the model is too loose.

**Decision**
Quiet history is review evidence only. It is interpreted alongside metric mechanics, signal coverage, actual observed maxima, and known incidents.

**Why it matters**
It avoids mistaking missing visibility for healthy service behavior.

## 1.4 Is frequent degradation automatically normal for that service?

**Question / assumption**
If a service spends substantial time above the baseline, that behavior should be absorbed into the baseline.

**Finding**
Frequent elevation may represent a service that is genuinely unhealthy or a model that does not represent the healthy distribution correctly.

**Decision**
Frequent crossings trigger investigation. They do not automatically redefine normal.

**Why it matters**
A chronically unhealthy service must not be rewarded with weaker monitoring.

## 1.5 [SHOWCASE] What is the methodology trying to standardize?

**Question / assumption**
The work can be summarized as "a model that gives us thresholds."

**Finding**
The actual deliverable is broader: reproducible data selection, traffic validity, robust baseline estimation, persistence, absolute failure floors, backtesting, signal-scope documentation, and human review.

**Decision**
Describe the work as a repeatable, fleet-tested threshold-calibration methodology rather than a single threshold model.

**Why it matters**
It makes the engineering scope visible and avoids making the result sound opaque or arbitrary.

---

# 2. Calibration population: BUSY and PEAK

## 2.1 [SHOWCASE] Should quiet and peak traffic be modeled as one operating regime?

**Question / assumption**
All hours can be mixed into one historical baseline.

**Finding**
Quiet overnight traffic and event-time traffic can have materially different request volume and metric behavior. Mixing them can pull the baseline downward and make expected high-load behavior look abnormal.

**Decision**
Derive the statistical baseline from a fixed BUSY operating window.

**Why it matters**
The baseline represents the operating regime that matters most rather than an average of incompatible regimes.

## 2.2 How should BUSY be selected?

**Question / assumption**
BUSY hours can be chosen informally from team intuition.

**Finding**
A repeatable fleet method requires a mechanical rule.

**Decision**
Use hourly median request counts, identify the peak hour, retain the contiguous block at or above 50% of peak, and use the union when PEAK and recent differ. The platform result is 18:00–22:59 America/Toronto.

**Why it matters**
The calibration regime is explainable, reproducible, and not hand-picked per service.

## 2.3 Why not calibrate from the latest 30 days only?

**Question / assumption**
Recent history is always the most representative history.

**Finding**
For a seasonal sports platform, recent off-season traffic can be several times lower than the peak season. Thresholds derived only from recent traffic may be too sensitive when concurrency returns.

**Decision**
Use a high-load PEAK reference window for calibration and recent 30 days for contextual backtesting.

**Why it matters**
The monitor remains suitable for the period of highest operational risk.

## 2.4 Is off-season headroom a defect?

**Question / assumption**
A threshold that looks loose during off-season should be lowered.

**Finding**
The looseness is the accepted consequence of calibrating for peak-season operation.

**Decision**
Report off-season headroom rather than seasonally moving the threshold.

**Why it matters**
It preserves stability and avoids recalibration churn driven by predictable seasonal load changes.

## 2.5 [SHOWCASE] T1 is derived from BUSY. Where should its statistical validity be judged?

**Question / assumption**
A BUSY-derived T1 should be judged by how often it is crossed across the entire 24-hour day.

**Finding**
Fleet validation showed T1 generally sits above the BUSY historical bulk. Earlier Shain raises were driven mostly by off-BUSY behavior, especially denominator collapse overnight.

**Decision**
Judge T1's statistical placement primarily against the BUSY population from which it was derived. Off-BUSY crossings remain operationally interesting but are not evidence that BUSY T1 is misplaced.

**Why it matters**
It separates "the baseline is wrong" from "another operating regime behaves differently."

## 2.6 [TENSION] Production monitors run 24/7, but T1 is calibrated and validated in BUSY

**Question / assumption**
The derivation and live evaluation populations should be identical.

**Finding**
The methodology intentionally calibrates on the representative high-load regime while production monitors continue to evaluate all hours. Off-hours can therefore cross a threshold that is healthy for BUSY.

**Decision**
Keep BUSY calibration and 24/7 monitoring, but do not use off-BUSY alert volume to raise T1.

**Why it matters**
This design preserves peak-season relevance while preventing off-hours denominator effects from rewriting the baseline.

---

# 3. Request-count validity gate R

## 3.1 [SHOWCASE] Can every 5-minute bucket support a meaningful percentile or rate?

**Question / assumption**
If Datadog returns a percentile or ratio, it is statistically meaningful.

**Finding**
At low request volumes, p95/p99 may be determined by one request and a few errors can create an extreme rate.

**Decision**
Only buckets whose exact request-count series meets R participate in calibration and evaluation.

**Why it matters**
It prevents low-denominator artifacts from being interpreted as service-wide degradation.

## 3.2 How is R selected?

**Question / assumption**
R can be chosen manually for each service.

**Finding**
A fixed candidate menu provides a consistent minimum-confidence policy while allowing services to use the strongest gate their traffic supports.

**Decision**
Choose the largest value in `{50, 100, 200, 500}` that is at or below BUSY p1 in both PEAK and recent windows.

**Why it matters**
At least 99% of BUSY buckets in both windows satisfy the selected gate.

## 3.3 [SHOWCASE] Is R a property of the logical service?

**Question / assumption**
One service should have one shared R across all monitors.

**Finding**
APM and ALB request populations can differ materially for the same service. Shain and Reception demonstrated that a single service-level R can be incoherent.

**Decision**
R belongs to the exact request-count series paired with the monitored metric.

**Why it matters**
The confidence gate measures the same traffic population as the metric it validates.

## 3.4 Can an ALB-derived R be reused for an APM monitor?

**Question / assumption**
ALB and APM are two views of the same traffic, so their request gates are interchangeable.

**Finding**
Some service pairs match closely, while others show APM request volume materially above the public ALB target-group volume.

**Decision**
Derive and store ALB and APM R independently, even when they happen to select the same candidate.

**Why it matters**
Matching traffic must be demonstrated rather than assumed.

## 3.5 What does `NONE` mean?

**Question / assumption**
If no candidate qualifies, lower the minimum until the monitor can be built.

**Finding**
A series below the platform minimum does not support reliable service-level percentiles or rates.

**Decision**
Record `NONE` honestly and make a per-monitor human disposition.

**Why it matters**
The methodology does not manufacture statistical confidence where traffic does not provide it.

## 3.6 [SHOWCASE] Are missing Datadog count buckets missing data or zero traffic?

**Question / assumption**
Percentiles can be calculated from only the buckets Datadog returns.

**Finding**
Datadog omits empty count buckets. Ignoring them makes idle series appear busier than they are and inflates p1.

**Decision**
Densify count series to the full 5-minute grid and fill omitted buckets with zero before request-volume statistics.

**Why it matters**
The selected R reflects actual traffic, including true zero periods.

## 3.7 Should an invalid bucket be skipped inside a consecutive run?

**Question / assumption**
A threshold sequence can bridge a low-volume bucket because the metric values before and after it are elevated.

**Finding**
The low-volume bucket provides no valid service-level evidence, and Datadog cannot skip it inside a strict consecutive condition.

**Decision**
A sub-R bucket breaks the run.

**Why it matters**
Backtests match live monitor semantics and do not invent continuity through statistically invalid data.

---

# 4. Metric inputs and telemetry mechanics

## 4.1 Should the analysis discover the metric through trial and error?

**Question / assumption**
The script can explore Datadog until a plausible-looking series appears.

**Finding**
Metric variants, aggregations, operations, and scopes can produce superficially plausible but semantically different data.

**Decision**
Treat metric name, scope, and paired count series as explicit inputs. Probe the exact query before a long fetch.

**Why it matters**
The method derives thresholds from the intended signal rather than whatever query happens to return points.

## 4.2 Does zero points always mean no historical data?

**Question / assumption**
A zero-point probe means the service had no activity.

**Finding**
For latency, zero points often means the wrong Datadog metric variant, such as querying a gauge with a percentile aggregation.

**Decision**
Stop and verify the metric variant; never switch aggregation merely to make data appear.

**Why it matters**
Averaging a gauge is not a substitute for querying the actual trace distribution percentile.

## 4.3 [SHOWCASE] Does zero returned 5XX data mean the error query is wrong?

**Question / assumption**
The same zero-point probe rule should apply to sparse event-count numerators and traffic denominators.

**Finding**
A 5XX count series can legitimately return no points during an error-free probe period while the request denominator confirms healthy traffic and longer history contains errors.

**Decision**
The zero-point stop applies strictly to the traffic denominator. A silent sparse numerator may represent true zero events and is densified accordingly.

**Why it matters**
Healthy zero-error series are not incorrectly rejected as query failures.

## 4.4 How should rate metrics be fetched?

**Question / assumption**
A precomputed percentage query is sufficient.

**Finding**
Precomputed ratios can obscure rollup semantics and detach the result from the exact denominator used for R.

**Decision**
Fetch numerator and denominator separately at explicit 5-minute resolution, densify, and divide per bucket.

**Why it matters**
The unit of analysis remains transparent and reproducible.

## 4.5 Can percentiles be averaged across buckets?

**Question / assumption**
A coarse rollup of p95 or p99 can represent the original percentile series.

**Finding**
Percentiles are not additive or safely averageable across time buckets.

**Decision**
Fetch and analyze the percentile at the same 5-minute unit used by the live monitor.

**Why it matters**
The historical calculation and production monitor evaluate the same statistical object.

## 4.6 What if a latency value is missing while request volume is valid?

**Question / assumption**
Missing latency buckets can be silently discarded or filled with zero.

**Finding**
Doing so can make the baseline appear healthier and hide a telemetry failure.

**Decision**
Never zero-fill percentile values. If count ≥ R but latency is missing, stop and report a telemetry-integrity issue.

**Why it matters**
Missing observability is not misclassified as excellent performance.

## 4.7 Why must ALB queries be pinned to exact IDs?

**Question / assumption**
Names such as `production-webfacing` and `production-shain` uniquely identify the intended resources.

**Finding**
The Datadog organization contains colliding names across accounts and customers. Wildcarding previously inflated Shain analysis materially.

**Decision**
Pin the exact load balancer and target-group identifiers.

**Why it matters**
The threshold is calibrated on the intended service traffic rather than an accidental fleet aggregate.

---

# 5. Robust baseline and T1

## 5.1 [SHOWCASE] Is T1 "pure statistics"?

**Question / assumption**
Because T1 uses MED and MADn, it is a statistically proven definition of degradation.

**Finding**
MED and MADn are statistical estimators. The `1.5×` and `6×` choices and the interpretation of the result as degradation are engineering policy built on those estimators.

**Decision**
Describe T1 as a reproducible statistical heuristic for "outside the normal historical BUSY regime," not a universal law of health.

**Why it matters**
It preserves methodological honesty and keeps operational meaning separate from mathematical calculation.

## 5.2 Why MED instead of mean?

**Question / assumption**
Average behavior is the natural baseline center.

**Finding**
Incidents and elevated periods can materially pull the mean upward. The median remains stable under substantial contamination.

**Decision**
Use MED as the robust center.

**Why it matters**
The very incidents the method is trying to identify do not redefine the baseline.

## 5.3 Why MADn instead of standard deviation?

**Question / assumption**
Standard deviation is the obvious measure of spread.

**Finding**
Heavy tails and recurring elevated periods inflate standard deviation. MADn measures typical spread around the median more robustly.

**Decision**
Use normalized median absolute deviation, including the 1.4826 normalization.

**Why it matters**
The spread estimate survives the contaminated operational data on which it must operate.

## 5.4 Why does T1 have two arms?

**Question / assumption**
One center-plus-spread expression is sufficient for every metric shape.

**Finding**
Very tight distributions can produce a microscopic spread threshold, while noisy distributions need a spread-sensitive boundary.

**Decision**
Use `T1 = max(1.5 × MED, MED + 6 × MADn)` and record which arm wins.

**Why it matters**
The ratio arm protects tight nonzero baselines; the MADn arm adapts to natural variation.

## 5.5 [SHOWCASE] Should T1 be raised until historical event counts are acceptable?

**Question / assumption**
Repeated 15-minute events mean the initial T1 is too low.

**Finding**
The old raise loop often reacted to off-BUSY behavior, service problems, or alternate distribution shapes rather than a misplaced BUSY boundary.

**Decision**
T1 is fixed after formula calculation. Frequent crossings are review evidence.

**Why it matters**
The method no longer converts historical alert frequency into baseline inflation.

## 5.6 What does "contamination" mean?

**Question / assumption**
Contamination and the share of valid BUSY buckets above T1 are separate diagnostics.

**Finding**
They are the same quantity in this methodology.

**Decision**
Report one measure: percentage of valid BUSY buckets above T1.

**Why it matters**
It avoids duplicate metrics that appear to provide independent evidence.

## 5.7 Does contamination alone tell us whether T1 is well placed?

**Question / assumption**
A high crossing percentage proves T1 is wrong.

**Finding**
High contamination can come from shallow values crowding the line, deep isolated spikes, sustained episodes, or a second distribution mode.

**Decision**
Interpret contamination with exceedance depth and temporal run structure.

**Why it matters**
The method distinguishes a boundary problem from elevated behavior above a correctly placed boundary.

## 5.8 How should exceedance depth be characterized?

**Question / assumption**
Counting crossings is sufficient.

**Finding**
A value at `1.02×T1` and a value at `20×T1` have very different implications for threshold placement.

**Decision**
Report median/p90 `value ÷ T1`, shallow-exceedance share, and depth bands.

**Why it matters**
It reveals whether T1 cuts through ordinary variation or isolates a separate elevated tail.

## 5.9 Is a 15-minute T1 crossing still useful if v3 Monitor A changed?

**Question / assumption**
Diagnostics should only reproduce the current monitor definitions.

**Finding**
Repeated 15-minute T1 crossings were the original symptom that prompted T1 re-evaluation.

**Decision**
Retain raw three-bucket T1 runs as a diagnostic, clearly labeled as not projected A2 alerts.

**Why it matters**
The experiment directly answers the question that triggered it.

## 5.10 Should raw T1 validation runs be merged when separated by a short healthy gap?

**Question / assumption**
Operational episode-merging logic should also apply to formula characterization.

**Finding**
Merging would hide the number and duration of actual threshold crossings.

**Decision**
Use raw maximal consecutive runs for T1 validation; no gap-based merging.

**Why it matters**
The diagnostic measures the threshold's crossing behavior rather than a reporting abstraction.

## 5.11 [SHOWCASE] Did fleet testing show that T1 is broadly unhealthy?

**Question / assumption**
Because several early services required v2 raises, the fixed T1 formula may be systematically too low.

**Finding**
In the standalone BUSY validation, 20 of 26 analyzable series showed contamination around or below 1% with zero or near-zero sustained 15-minute crossings. Only a small subset showed concerning signatures, and those signatures were not uniform.

**Decision**
Treat T1 as broadly healthy for the BUSY calibration population, with per-series review where distribution shape or shallow persistence is unusual.

**Why it matters**
The evidence does not support a fleet-wide formula defect or a fleet-wide raise loop.

## 5.12 What did Shain's T1 validation show?

**Question / assumption**
Shain's earlier raises implied the initial T1 was poorly placed.

**Finding**
Shain error rate, p95, and p99 were quiet in BUSY, with very low contamination and no sustained 15- or 30-minute crossings.

**Decision**
Treat Shain's BUSY T1 as healthy. Interpret its earlier raises as consequences of off-BUSY behavior and the old raise loop.

**Why it matters**
It directly demonstrates the conceptual error in using all-hours event volume to tune a BUSY-derived boundary.

## 5.13 What did Rocket p95 show?

**Question / assumption**
Rocket's relatively high contamination means T1 cuts through normal variation.

**Finding**
Most crossings were isolated and many were extremely deep multiples of T1, with only a small number of sustained runs.

**Decision**
Treat Rocket as a deep-spike pattern rather than immediate evidence of a low T1.

**Why it matters**
The same contamination percentage can describe fundamentally different distributions.

## 5.14 What did ISL OData p99 show?

**Question / assumption**
All fleet exceptions are deep spikes or obvious incidents.

**Finding**
ISL OData p99 showed shallow depth, substantial density near the threshold, and sustained crossings.

**Decision**
Flag it as a genuine candidate where T1 may sit near the top edge of ordinary variation.

**Why it matters**
Fleet validation can localize potential model weaknesses without weakening the formula globally.

## 5.15 What did Nightwatch latency show?

**Question / assumption**
A very small absolute T1 is automatically invalid.

**Finding**
Nightwatch has an ultra-tight baseline and shallow-but-sustained elevated episodes. The data does not by itself distinguish mild real episodes from a threshold made operationally microscopic by distribution tightness.

**Decision**
Flag Nightwatch for service-level review; do not infer a fleet-wide T1 failure.

**Why it matters**
Absolute smallness and statistical misplacement are related concerns but not the same conclusion.

## 5.16 What did Populist latency show?

**Question / assumption**
High contamination always means the T1 multiplier needs adjustment.

**Finding**
Populist showed a very tight low mode plus frequent isolated seconds-level spikes. In PEAK BUSY, contamination was 12.95% for p95 and 15.71% for p99, with median exceedance depths of roughly 41× and 15× T1. Almost every exceedance run was one 5-minute bucket, so Monitor A remained relatively quiet while the floored Monitor C absorbed the deep spikes.

**Decision**
Do not treat the pattern as a simple multiplier problem or raise T1 to normalize it. The baseline-model question and the operational-severity question must be kept separate. Before wiring Populist latency monitors, obtain service-owner confirmation that the seconds-level behavior is expected, or investigate and stabilize it.

**Why it matters**
Changing `6×MADn` cannot necessarily fix a metric with more than one regime. At the same time, Monitor C is currently floor-controlled: it crosses because the observed values exceed the absolute 5-second p95 / 10-second p99 Critical policy, not merely because T1 is small.

## 5.17 How should a zero-baseline error rate be handled?

**Question / assumption**
When MED and initial MADn are zero, recomputing MADn from only the nonzero buckets can recover a useful normal boundary.

**Finding**
Conditioning on nonzero buckets uses the exceptional errors themselves to estimate normal variability. On Populist, seven of the ten nonzero BUSY buckets belonged to one incident; the fallback therefore produced a 57.04% T1, above both the 5% Warning and 20% Critical floors. A later review of binomial, Poisson, and beta-binomial alternatives found that their added complexity did not provide enough operational value for the very-low-MED/MADn case to justify replacing a transparent fleet policy floor.

**Decision**
Remove the nonzero-bucket MADn fallback. For error-rate series, use `T1 = max(formula T1, 1%)`. When MED = 0 and MADn = 0, the formula collapses to zero and the 1% floor supplies the degradation threshold. The earlier 2% direct zero-baseline default is superseded.

**Why it matters**
Rare incidents no longer define their own normal boundary. At the minimum valid request gate `R = 500`, 1% is approximately five errors per 5-minute bucket; Monitor A still requires three consecutive valid buckets above the line. The result is an intelligible severity ladder: sustained relative degradation from 1%, immediate Warning from 5%, and immediate Critical from 20%.

## 5.18 What if MED and MADn are both zero for another metric type?

**Question / assumption**
A threshold should still be created so every service has a monitor.

**Finding**
A zero-center, zero-spread population contains no empirical scale from which the generic T1 formula can derive a nonzero boundary. Error rate now has an explicit engineering default; other metric types do not automatically inherit it.

**Decision**
Use an already documented metric-specific zero-baseline rule. If none exists, mark the series `NOT ANALYZABLE`; do not invent or borrow a threshold.

**Why it matters**
The methodology remains honest about what the historical data can and cannot support.

---

# 6. Cleaning and the healthy-tail anchor

## 6.1 Why remove values above T1 before calculating the upper healthy tail?

**Question / assumption**
The historical p99.5 can be calculated directly from all valid data.

**Finding**
Recurring incidents and isolated spikes can occupy enough of the tail that p99.5 lands on elevated behavior.

**Decision**
Remove all valid buckets above T1 before deriving ANCHOR.

**Why it matters**
The failure threshold is anchored to the top of healthy behavior rather than the bottom of recurring incidents.

## 6.2 Should only sustained T1 events be removed?

**Question / assumption**
Isolated spikes are not degradation episodes and may remain in the healthy-tail dataset.

**Finding**
Even isolated spikes contain no information about the healthy cluster and can destabilize a high percentile.

**Decision**
Cleaning is point-wise: every bucket above T1 is excluded.

**Why it matters**
ANCHOR remains stable even when the service has many isolated spikes.

## 6.3 Why are T1 and ANCHOR different statistical ideas?

**Question / assumption**
One robust center-and-spread formula should derive all thresholds.

**Finding**
T1 must survive contaminated data and answer "how far from typical?" ANCHOR operates after cleaning and answers "how high can healthy behavior reach?"

**Decision**
Use center-based MED/MADn for T1 and tail-based p99.5 for ANCHOR.

**Why it matters**
Each statistic is matched to the question it is capable of answering.

## 6.4 Why must ANCHOR be at or below T1?

**Question / assumption**
The cleaned healthy tail can exceed the degradation boundary.

**Finding**
If values above T1 are removed correctly, the top of the retained healthy population cannot exceed T1.

**Decision**
Treat `ANCHOR ≤ T1` as a consistency condition.

**Why it matters**
Violations reveal calculation or masking errors.

---

# 7. Persistence and deferred drift control

## 7.1 Is one elevated 5-minute bucket enough to call relative degradation?

**Question / assumption**
Every T1 crossing should notify.

**Finding**
Operational metrics contain isolated spikes that may be statistically unusual but too brief to represent sustained service degradation.

**Decision**
Require persistence for the relative degradation tier.

**Why it matters**
It suppresses transient noise without moving the statistical boundary.

## 7.2 Are the persistence numbers statistically derived?

**Question / assumption**
Three or six consecutive buckets have the same statistical status as MED and MADn.

**Finding**
They are engineering policy choices about operational significance and response timing.

**Decision**
Document them as chosen constants, validate their behavior, and revisit only with evidence.

**Why it matters**
It distinguishes statistical estimation from operational design.

## 7.3 Why return Monitor A to three consecutive buckets above T1?

**Question / assumption**
Monitor A needed the interim dual `6×T1 / 3×2T1` paths because the fixed T1 formula might be broadly too low.

**Finding**
Fleet validation showed T1 is broadly healthy within BUSY. The dual-path design was compensating for uncertainty that the wider validation weakened.

**Decision**
Use one Monitor A rule: three consecutive valid 5-minute buckets above T1. Repeated backtest crossings are review evidence, especially when concentrated outside BUSY; service-specific issues or exclusions are handled explicitly rather than by weakening the fleet rule.

**Why it matters**
Monitor A remains simple, reproducible, and directly tied to the degradation boundary.

## 7.4 Why strict consecutive buckets instead of M-of-N?

**Question / assumption**
A looser M-of-N pattern would tolerate brief gaps and catch more events.

**Finding**
M-of-N adds complexity, can bridge invalid traffic periods, and is harder to reproduce consistently in Datadog.

**Decision**
Use strict consecutive buckets.

**Why it matters**
The backtest and live monitor remain simple and equivalent.

## 7.5 Why use recovery below the alert threshold?

**Question / assumption**
Recovery should occur the moment the metric falls below the alert line.

**Finding**
Values hovering around the line can repeatedly alert and recover.

**Decision**
Use 0.9× the relevant threshold for recovery.

**Why it matters**
It adds hysteresis without altering the degradation definition.

## 7.6 Why was Monitor B deferred?

**Question / assumption**
Accumulated time above T1 can serve as a useful monitor for silent drift that Monitor A misses.

**Finding**
Accumulated degraded time can identify a bad day, but the proposed fixed B threshold did not cleanly answer the desired question, "Is this service getting worse?" A PEAK-derived fixed baseline says only that the current period differs from PEAK; an A-event count mostly repeats Monitor A; and a genuine moving drift baseline would need frequent updates and seasonal awareness. In early Populist analysis, B either duplicated A's meaning or became inert because historical spiking inflated its allowance.

**Decision**
Do not create Monitor B for any service in the first round. Keep the active service-health set at A + C.

**Why it matters**
The first implementation does not claim to provide drift control through a monitor whose signal and operational response remain unclear.

## 7.7 What happened to the proposed accumulated-degradation formula?

**Question / assumption**
Use the single worst historical day or count Monitor A events.

**Finding**
The maximum is unstable, and event count loses duration information.

**Decision**
The candidate `max(2 × p95 daily degraded minutes, 60 minutes)` formula is retained only as design history. It is not part of the active methodology and must not be produced or wired by worker runs.

**Why it matters**
Future work can understand what was tested without mistaking a superseded candidate for a current requirement.

## 7.8 [OPEN] What should future drift control look like?

**Question / assumption**
A static Datadog threshold can provide meaningful long-term drift detection without maintenance.

**Finding**
Real drift control needs a moving comparison population, periodic recalculation, and protection against predictable seasonal transitions such as returning to PEAK. That may be better expressed as scheduled analysis rather than an always-on static monitor.

**Decision**
Defer the design. A possible future path is a CI/CD or scheduled pipeline that recalculates degradation-bucket distributions, compares recent periods with an appropriate moving or seasonal baseline, and proposes or validates changes under review.

**Why it matters**
This preserves the original goal—detecting where the service is heading—without forcing it into the wrong monitoring primitive.

---

# 8. Severe failure thresholds and floors

## 8.1 Is a relative multiplier enough to define operational failure?

**Question / assumption**
A value many times above a very low healthy baseline must be operationally serious.

**Finding**
On exceptionally healthy or tight services, relative multipliers can produce thresholds that are statistically unusual but operationally trivial.

**Decision**
Combine relative multipliers with absolute operational floors.

**Why it matters**
The failure monitor reflects actual paging appetite rather than only statistical surprise.

## 8.2 Why retain relative multipliers after adding floors?

**Question / assumption**
Fixed absolute thresholds can replace service-specific calibration.

**Finding**
Naturally slower or noisier services still need thresholds relative to their own healthy behavior.

**Decision**
Use the maximum of the relative threshold and the metric-type floor.

**Why it matters**
The method remains service-specific without notifying on trivial absolute values.

## 8.3 Are `4×` and `20×` statistically derived constants?

**Question / assumption**
The multipliers have the same evidentiary status as MED, MADn, or p99.5.

**Finding**
They are engineering severity choices informed by prior convention and backtesting.

**Decision**
Document them as constants and revisit only with operational evidence.

**Why it matters**
The methodology does not disguise policy choices as mathematical truths.

## 8.4 Why are floors lower bounds rather than caps?

**Question / assumption**
A floor should replace a high relative result or constrain thresholds to a fixed range.

**Finding**
The floor's purpose is only to prevent trivial low thresholds. A naturally high baseline may legitimately require a higher threshold.

**Decision**
Use `max(relative, floor)` with no ceiling.

**Why it matters**
The method avoids both under-alerting healthy services and over-constraining slow/noisy ones.

## 8.5 Should Critical be forced to appear in historical data?

**Question / assumption**
A Critical threshold that never fired in PEAK or recent history is not useful.

**Finding**
Healthy historical months are expected not to contain paging-level failures.

**Decision**
Remove the historical-max reachability adjustment and the automatic "Critical never reached" concern.

**Why it matters**
The method stops lowering paging thresholds merely to make backtests produce a Critical.

## 8.6 What if a rate formula produces an impossible threshold above 100%?

**Question / assumption**
Add a special cap or arithmetic rule for bounded metrics.

**Finding**
An impossible result is evidence about the baseline, signal, or model; silently capping it hides that evidence.

**Decision**
Flag the result for review rather than adding another automatic correction formula.

**Why it matters**
The method remains auditable and avoids accumulating ad hoc patches.

## 8.7 Why was the absolute error-count gate N removed?

**Question / assumption**
A rate alert is only meaningful if a second count threshold is also crossed.

**Finding**
A global count gate can silence real failures, adds a second calibration problem, and partly duplicates the confidence and impact already represented by R, persistence, severity floors, and mandatory absolute-impact reporting.

**Decision**
Remove N from the default methodology. Report absolute error impact in backtesting and allow a documented service-specific exception only if necessary.

**Why it matters**
The monitor does not discard valid high-rate failures solely because they occurred at a lower absolute count.

## 8.8 [SHOWCASE] Why did Shain's old N=100 fail conceptually?

**Question / assumption**
A mechanically selected large error-count gate would protect against small-denominator noise.

**Finding**
It would have silenced every historical Shain alert, including clearly real elevated events.

**Decision**
Use R for denominator validity and retain absolute counts as evidence rather than a mandatory global gate.

**Why it matters**
A safeguard must not remove the entire signal it is meant to make trustworthy.

---

# 9. Monitor architecture and routing

## 9.1 Why not use one monitor per metric?

**Question / assumption**
One threshold can represent all forms of degradation and failure.

**Finding**
Relative persistence and absolute severity answer different operational questions. Accumulated degradation was explored as a third question but deferred because it did not yet provide a clear drift signal.

**Decision**
Use Monitor A and Monitor C as the active first-round monitor set.

**Why it matters**
Each notification has a clear meaning and response expectation.

## 9.2 What does Monitor A mean?

**Question / assumption**
A is merely a lower-severity version of C.

**Finding**
A detects sustained distance from the service's normal BUSY regime, even below absolute failure levels.

**Decision**
Use A as relative early-warning degradation evidence.

**Why it matters**
The team can detect worsening before the service becomes absolutely broken.

## 9.3 What was Monitor B intended to mean?

**Question / assumption**
B is simply a recurrence count for A alerts.

**Finding**
Counting events ignores duration and can be distorted by event-splitting rules, so accumulated valid degraded minutes was the stronger candidate. However, a fixed historical allowance still did not reliably express whether a service was drifting, and could duplicate A or become inert.

**Decision**
Monitor B is deferred and is not created for any service. Its intended drift-control goal remains future work, potentially outside static Datadog monitors.

**Why it matters**
The ledger preserves the intent without presenting an unresolved design as an active monitor.

## 9.4 What does Monitor C mean?

**Question / assumption**
C is another relative anomaly detector.

**Finding**
C is intended to represent absolute operational trouble now.

**Decision**
Use one valid 5-minute bucket above floored Warning/Critical thresholds.

**Why it matters**
Severe failures do not have to wait for persistence.

## 9.5 How should Monitor C crossings recover and be counted?

**Question / assumption**
Each Warning or Critical crossing bucket can be treated as a separate alert, or divided by the number of days to estimate alert frequency.

**Finding**
Crossing buckets often cluster around one continuing problem. Bucket counts measure how many 5-minute intervals crossed a threshold; they do not measure monitor state transitions, notifications, or distinct operational episodes. Counting the same episode once as Warning and again as Critical also double-counts its severity progression.

**Decision**
For each series separately, Monitor C evaluates whether the threshold was crossed at least once over the last 30 minutes. One valid 5-minute bucket still triggers immediately, but the 30-minute window acts as the recovery hold. The backtest reproduces the same behavior: crossings separated by 30 minutes or less form one episode. If any crossing in an episode reaches Critical, classify the episode only as Critical; otherwise classify it as Warning. Do not merge p95 and p99 episodes. Report distinct Warning episodes and affected calendar days, and distinct Critical episodes and affected calendar days. Raw crossing-bucket counts may remain as diagnostic evidence, but never as alerts, pages, or a bucket-count-derived daily alert rate.

**Why it matters**
The report matches live Monitor C behavior, preserves Warning-to-Critical escalation as one episode, and describes clustered failures without overstating paging frequency.

## 9.6 Should every monitor page?

**Question / assumption**
If a condition deserves a monitor, it deserves on-call paging.

**Finding**
Relative degradation is valuable early-warning evidence but may not justify immediate paging. Absolute severity has distinct Warning and Critical responses.

**Decision**
Route A to SRE/monitoring review; route C Warning to the monitoring team and C Critical to paging.

**Why it matters**
Notification urgency matches the meaning of the condition.

## 9.7 Should A, C, and gates be combined into composites immediately?

**Question / assumption**
Composite monitors are necessary to express the full methodology.

**Finding**
Composites increase monitor count and wiring complexity, and some combinations can hide which condition actually triggered.

**Decision**
Defer composites unless they provide clear operational value after the base monitors are understood.

**Why it matters**
The first implementation remains explainable and within the fleet monitor budget.

## 9.8 What happens when a service is already clearly unhealthy?

**Question / assumption**
The threshold should be weakened so the monitor can be launched quietly.

**Finding**
That would encode the existing problem into the monitoring baseline.

**Decision**
Fix the service, use a temporary documented downtime with owner/review date, or revise the model only with evidence that it misrepresents healthy behavior.

**Why it matters**
Monitoring does not become a mechanism for hiding known debt.

---

# 10. APM, ALB, and signal scope

## 10.1 [SHOWCASE] Are APM and ALB interchangeable views of one service?

**Question / assumption**
A request or error rate from either layer can stand in for the other.

**Finding**
They observe different boundaries. APM records instrumented application activity; an ALB target group records traffic routed through that exact ingress leg.

**Decision**
Treat each as a distinct telemetry channel with an explicitly documented population.

**Why it matters**
The method no longer assumes population equivalence merely because the service name is the same.

## 10.2 Does APM request volume prove APM error visibility?

**Question / assumption**
If APM records trace hits, it must also be able to calculate an honest error rate.

**Finding**
A request may create a span while its final client-facing 5XX outcome is missing from the traced pipeline.

**Decision**
Evaluate instrumentation coverage separately for volume, latency, and error outcomes.

**Why it matters**
A populated APM request series does not conceal an error-observability blind spot.

## 10.3 [SHOWCASE] Why was Shain's assumed APM error baseline retracted?

**Question / assumption**
Zero APM errors meant Shain simply had not experienced enough failures, so an assumed bootstrap baseline was acceptable.

**Finding**
Across millions of requests, APM continued to show zero 5XX while the ALB recorded real target failures. The metric was structurally blind, not merely quiet.

**Decision**
Retract the assumed APM baseline and calibrate Shain's attributable error rate from the ALB target signal.

**Why it matters**
A threshold cannot compensate for a metric that cannot observe the failure class.

## 10.4 Can Shain have an honest error-rate monitor?

**Question / assumption**
Because APM misses errors and ALB coverage is partial, no error-rate monitor is honest.

**Finding**
`target_5xx ÷ request_count` is internally coherent for the exact Shain target-group population.

**Decision**
Monitor and label it as the Shain public-target-group error rate, not the complete whole-service error rate.

**Why it matters**
Useful partial coverage is retained without overstating completeness.

## 10.5 Do Shain latency and error monitors cover the same traffic?

**Question / assumption**
The two signals can be interpreted as matched views of identical requests.

**Finding**
APM request volume is materially higher than the pinned public ALB target-group volume.

**Decision**
Document that latency covers traced ingress while error rate covers the public ALB leg.

**Why it matters**
A flat error rate and elevated latency are not necessarily contradictory.

## 10.6 Is APM always the complete service view?

**Question / assumption**
Because APM often sees more traffic than one public ALB, it represents every request reaching the service.

**Finding**
APM is the fuller observed view for the compared services, but completeness still depends on instrumentation and tracing coverage.

**Decision**
Describe APM as "all traced ingress," not automatically "all service traffic."

**Why it matters**
The documentation remains precise about what is observed rather than making an unprovable completeness claim.

## 10.7 [TENSION] What causes APM > ALB request volume?

**Question / assumption**
The gap proves traffic arrives through a second ALB or source.

**Finding**
The gap proves the two populations differ, but the mechanism has not been attributed. Candidates include internal calls, another listener or ALB, direct service traffic, and scheduled/background work.

**Decision**
Record the population mismatch as fact and the ingress mechanism as open.

**Why it matters**
The methodology uses verified evidence without converting a plausible explanation into a conclusion.

## 10.8 Does ALB target 5XX include ALB-generated 504s?

**Question / assumption**
The target error-rate monitor represents every client-facing 5XX on that route.

**Finding**
ALB-generated 5XX can lack target-group attribution and are not part of `target_5xx`.

**Decision**
Document the residual blind spot. Use ALB access logs if per-request attribution becomes necessary.

**Why it matters**
The monitor's scope is explicit and does not imply perfect failure accounting.

## 10.9 Can ALB target errors plus APM latency be called complete failure coverage?

**Question / assumption**
The two complementary monitors together cover every Shain failure mode.

**Finding**
They improve coverage but can still miss non-public ingress errors, untraced requests, and unattributed ALB-generated failures.

**Decision**
Call them complementary, not complete.

**Why it matters**
The documentation preserves residual risk instead of presenting a false sense of completeness.

## 10.10 Should the first monitor round solve whole-service cross-ingress error accounting?

**Question / assumption**
A monitor is incomplete unless it covers every ingress path.

**Finding**
Whole-service accounting requires missing application instrumentation and ingress attribution that are not currently available.

**Decision**
Monitor per channel for the first round and record whole-service coverage as out of scope.

**Why it matters**
The team can improve monitoring now without pretending the observability architecture is already complete.

## 10.11 Can a negative cross-signal correlation rule out a failure relationship?

**Question / assumption**
If latency does not correlate with the selected 5XX series, then latency-related failures are not happening.

**Finding**
A correlation study only tests the exact telemetry populations included in it. If the error series represents one failure class or attribution layer, it cannot rule out failures that surface in another metric family.

Example pattern: latency was compared against target-generated 5XX. That can show whether latency co-moves with errors returned by the target, but it says nothing about ALB-generated 504s caused by timeout behavior, because those live in a different error series and may not have target-group attribution.

**Decision**
When using cross-signal analysis, explicitly state:

* which metric population was tested;
* which failure class it can observe;
* which failure classes remain outside the analysis.

A negative correlation is evidence only within that stated scope.

**Why it matters**
It prevents over-interpreting "no correlation" as "no relationship exists." The method remains honest about the boundary between what the selected telemetry can disprove and what remains untested.

---

# 11. Backtesting and review

## 11.1 What is backtesting for?

**Question / assumption**
Backtesting exists to tune thresholds until alert counts look acceptable.

**Finding**
That reproduces the v2 normalization problem.

**Decision**
Backtesting describes what the fixed rules would have done and surfaces review questions.

**Why it matters**
Historical evidence informs decisions without silently changing the formulas.

## 11.2 Why backtest PEAK and recent separately?

**Question / assumption**
One historical window is enough.

**Finding**
PEAK validates the high-load calibration regime; recent reveals current behavior and off-season headroom.

**Decision**
Report both without using recent quietness to lower the peak-derived thresholds.

**Why it matters**
The methodology separates calibration from current context.

## 11.3 What should an error-rate backtest include beyond percentages?

**Question / assumption**
Rate and duration are enough to judge operational importance.

**Finding**
A high rate can represent two errors or hundreds, depending on request volume.

**Decision**
Report absolute error counts per degraded bucket and event.

**Why it matters**
Human reviewers can judge practical impact without adding a mandatory global count gate.

## 11.4 Should AI or human review be allowed to change thresholds silently?

**Question / assumption**
A reviewer can correct obviously strange outputs during derivation.

**Finding**
Silent adjustment destroys reproducibility and makes the final threshold impossible to audit.

**Decision**
Review may flag, explain, or recommend a documented model change; it must not silently alter the result.

**Why it matters**
The mechanical derivation remains reproducible and disagreements remain visible.

## 11.5 What does a "critical never reached" result mean?

**Question / assumption**
The monitor is ineffective if history contains no Critical firing.

**Finding**
A paging threshold is not expected to appear in healthy months.

**Decision**
Report historical reachability as context where useful, but do not force Critical downward.

**Why it matters**
The monitor remains a failure detector instead of a guaranteed historical-event generator.

## 11.6 Should unusual endpoint, hour, or regime patterns be part of the T1 formula test?

**Question / assumption**
A complete validation should explain every crossing during the fleet formula experiment.

**Finding**
Those analyses help diagnose individual services but dilute the narrow question of where T1 sits relative to its own BUSY distribution.

**Decision**
Keep the standalone T1 experiment focused. Investigate contextual patterns separately for flagged series.

**Why it matters**
The experiment remains answerable and does not turn into a full service RCA.

## 11.7 Why keep validation artifacts separate from official threshold notes?

**Question / assumption**
Pilot and fleet-test outputs can be promoted directly into service KB records.

**Finding**
Validation windows and objectives may differ from official derivation runs.

**Decision**
Label fleet-test outputs as validation artifacts until a formal per-service derivation is completed.

**Why it matters**
Experimental evidence does not accidentally become production configuration of record.

## 11.8 Why must every worker report begin with an Executive Summary?

**Question / assumption**
The detailed runbook output is sufficient because it contains all of the evidence.

**Finding**
The original Populist reports were complete reference documents but did not surface the outcome, expected alert behavior, major findings, or required human decisions quickly enough to support review.

**Decision**
Every service note begins with a compact Executive Summary containing deterministic analysis status, what was analyzed, the normal/threshold table, PEAK and recent backtest results, no more than five evidence-bearing findings, and the unresolved human questions. When threshold derivation and backtesting completed, the required threshold and backtest tables must include every derived monitor, series, and window even when a result is not operationally remarkable. Selectivity applies only to the prose findings, not to table completeness. If the run stopped before backtesting, the table is not required. The detailed report remains below as the evidence layer. "Analysis complete" must never be described as "ready to deploy."

**Why it matters**
The same worker output can serve both decision-making and audit/reference use without creating a separate hand-written summary note.

## 11.9 How should cross-signal claims be kept consistent?

**Question / assumption**
A report can safely reuse a cross-signal conclusion from an earlier companion report.

**Finding**
The first Populist error report called the recent window error-free. A refreshed run found 1,617 5XX on 2026-07-01, overlapping the recent latency cluster, but the latency summary retained the stale "not 5XX-correlated" conclusion.

**Decision**
Before finalizing summaries, reconcile every cross-note claim against the latest companion report and current backtest. When evidence differs in severity, state it precisely: 2026-04-22 had severe latency/5XX correlation; 2026-07-01 had milder, sub-Warning 5XX correlation.

**Why it matters**
Independent report revisions must not leave mutually contradictory service conclusions.

## 11.10 How should isolated periodic crossings be detected and characterized?

**Question / assumption**
Existing contamination, bimodality, clustering, and persistence review flags are sufficient to surface a recurring timed problem.

**Finding**
A strictly periodic single-bucket spike can evade those routes: it may remain below the contamination threshold, appear as unrelated isolated spikes, be removed from the healthy-tail dataset during cleaning, and never satisfy Monitor A's three-consecutive-bucket rule. Populist's `sportschedules` pattern demonstrated the gap: almost every spike occurred at `:00` or `:30`, but the service-level report initially described only bimodality and isolated spikes rather than the half-hour mechanism.

**Decision**
Before cleaning, test all valid-bucket T1 crossings, including isolated spikes, for periodicity separately in PEAK and recent. With at least ten crossings, flag a periodic/timed pattern when either: one of the twelve possible 5-minute minute-of-hour slots contains at least three times its uniform share and at least five crossings; or the modal inter-arrival time accounts for at least 50% of intervals. These are fleet engineering defaults and are never tuned per service. A periodicity flag, as well as bimodality, contamination above 10%, or clustered/frequent crossings, requires report-only anomaly characterization before review completes: identify where the crossings originate, when they recur, and whether unrelated or canary endpoints share the signature. APM uses operation/resource attribution; ALB uses target-group or URL evidence where available and otherwise records that attribution is unavailable at that telemetry layer. Characterization never changes thresholds or creates resource-specific monitors automatically.

**Why it matters**
The methodology can now reveal a timed endpoint-specific mechanism that persistence and cleaning structurally hide, while keeping threshold derivation deterministic and monitor count unchanged.

---

# 12. Rejected or superseded approaches

## 12.1 [SHOWCASE] Raise T1 by 25% until event volume is acceptable

**Finding**
This normalized known degradation and off-BUSY behavior.

**Decision**
Rejected in v3; T1 remains fixed.

**Why it matters**
Alert-volume preference no longer defines health.

## 12.2 Use one R per logical service

**Finding**
APM and ALB populations differ for some services.

**Decision**
Rejected; R is per exact count series.

**Why it matters**
Metric validity follows the traffic actually measured.

## 12.3 Assume ALB and APM are interchangeable

**Finding**
They can differ in both traffic population and failure visibility.

**Decision**
Rejected; each channel is scoped explicitly.

**Why it matters**
The monitor does not claim coverage it does not possess.

## 12.4 Treat APM hits as proof that APM errors are instrumented

**Finding**
Shain produces trace hits while its client-facing 5XX class remains invisible to APM.

**Decision**
Rejected; instrumentation is assessed per signal type.

**Why it matters**
Volume visibility cannot hide outcome blindness.

## 12.5 Assume missing count buckets are absent data

**Finding**
Datadog omits true-zero count buckets.

**Decision**
Rejected; densify to zero.

**Why it matters**
Low-traffic services do not receive artificially high R values.

## 12.6 Recompute MADn using only nonzero error buckets

**Finding**
This conditions the baseline on exceptional buckets and can allow one incident to define "normal." Populist produced a 57.04% T1 through this path.

**Decision**
Rejected. Apply the 1% error-rate T1 floor; when MED = MADn = 0, the floor supplies T1. Other metric types still require their own documented rule or a `NOT ANALYZABLE` disposition.

**Why it matters**
The method uses an explicit operational policy instead of disguising incident severity as statistical spread.

## 12.7 Use a global absolute count gate N

**Finding**
It can suppress valid failures and duplicates other controls.

**Decision**
Rejected as a default; retain absolute-count reporting.

**Why it matters**
Impact remains visible without a second brittle threshold system.

## 12.8 Cap Critical using historical maximum so it becomes reachable

**Finding**
This lowers paging severity merely because healthy history lacks failures.

**Decision**
Rejected; no reachability clause.

**Why it matters**
Critical remains an operational failure threshold.

## 12.9 Treat frequent degradation as automatic evidence to weaken persistence

**Finding**
This can hide a genuinely unhealthy service.

**Decision**
Rejected; investigate or use documented temporary operational handling.

**Why it matters**
The monitor does not adapt itself into silence.

## 12.10 Merge nearby raw runs during T1 formula validation

**Finding**
Merging obscures the original repeated-crossing symptom.

**Decision**
Rejected for formula characterization; raw consecutive runs remain separate.

**Why it matters**
The diagnostic measures what T1 actually did.

## 12.11 Claim that complementary monitors provide complete coverage

**Finding**
Different channels and instrumentation gaps leave residual blind spots.

**Decision**
Rejected; document complementary coverage and exclusions.

**Why it matters**
The monitoring story remains credible under scrutiny.

## 12.12 Use APDEX as the primary service monitor

**Finding**
A single APDEX score is opaque and collapses latency distribution and user impact into one less-explainable number.

**Decision**
De-prioritized in favor of explicit percentile latency and error-rate monitors.

**Why it matters**
Operators can understand exactly what crossed and why.

---

# 13. Open questions and deliberate scope boundaries

## 13.1 [OPEN] What produces APM > public-ALB traffic for Shain, Reception, Populist, and Nightwatch?

The population difference is established; the ingress mechanism is not. Candidates include internal calls, another listener or ALB, direct service access, or background work.

## 13.2 [RESOLVED] How should zero-baseline error rates be monitored?

Use `T1 = max(formula T1, 1%)` for error rates. When MED and initial MADn are both zero, the 1% floor supplies T1; do not condition MADn on nonzero buckets. This resolves error-rate T1. Zero-baseline metrics of other types remain analyzable only when they have their own documented rule.

## 13.3 [OPEN] Does ISL OData p99 require a different healthy-distribution model?

Its shallow sustained BUSY crossings make it the strongest current candidate for T1 misplacement or multi-regime behavior.

## 13.4 [OPEN] Is Nightwatch's microscopic latency baseline operationally useful?

The statistical result may be correct while the absolute difference remains too small to matter. This requires service context and possibly absolute floors outside Monitor C.

## 13.5 [OPEN] How should deliberately bimodal services such as Populist be modeled?

One MED/MADn baseline may be too simple, but frequency alone does not prove the elevated mode is healthy. For Populist latency, the immediate action is service investigation or owner confirmation before monitor wiring. Potential multi-regime approaches must be evaluated deliberately rather than handled through per-service multiplier changes.

## 13.6 [OPEN] Should ALB access logs be enabled for per-request failure attribution?

Aggregate metrics cannot attribute shared-ALB generated 5XX to a target service. Access logs would close part of that observability gap.

## 13.7 [OPEN] When should a service-specific absolute error-count exception be allowed?

v3 removes N by default but permits documented exceptions if valid rate alerts prove operationally trivial. The evidence standard and governance are not yet formalized.

## 13.8 [OPEN] What belongs to service health versus operations health?

This methodology intentionally excludes MTTA, acknowledgement latency, unresolved-alert age, ticket conversion, and similar human/process state. A separate operations-health methodology remains future work.

## 13.9 [OPEN] How should fleet drift control be implemented?

Monitor B is deferred. A future scheduled or CI/CD mechanism may compare recent degradation-bucket behavior with moving and seasonal baselines, but its cadence, reference population, change controls, and operational response remain undesigned.

---

# 14. Flagged duplicates and tensions for later editing

This section does not resolve or merge anything. It marks likely consolidation points.

## 14.1 [DUPLICATE] Alert volume must not tune T1

Overlaps: 1.2, 1.4, 5.5, 11.1, 12.1, 12.9.
Likely future canonical item: **"Alert volume is evidence, not a calibration input."**

## 14.2 [DUPLICATE] BUSY calibration versus off-BUSY crossings

Overlaps: 2.1, 2.5, 2.6, 5.12.
Likely future canonical item: **"Judge a BUSY-derived boundary in BUSY; monitor off-hours without letting them rewrite it."**

## 14.3 [DUPLICATE] R follows the exact series

Overlaps: 3.3, 3.4, 10.1, 12.2, 12.3.
Likely future canonical item: **"Traffic confidence and signal scope are channel-specific."**

## 14.4 [DUPLICATE] APM hits do not prove error coverage

Overlaps: 10.2, 10.3, 12.4.
Likely future canonical item: **"Instrumentation must be verified separately for volume, latency, and outcomes."**

## 14.5 [DUPLICATE] Complementary does not mean complete

Overlaps: 10.4, 10.5, 10.8, 10.9, 12.11.
Likely future canonical item: **"Every monitor carries an explicit population and residual blind spots."**

## 14.6 [TENSION] T1 is a BUSY-relative degradation boundary, while Monitor A evaluates 24/7

Keep both visible. The methodology accepts this asymmetry but must never use off-BUSY event volume to recalibrate T1.

## 14.7 [TENSION] APM is the fuller observed ingress view, but not necessarily complete

The traffic comparison supports `APM > one public ALB leg`; it does not prove every service request is traced.

## 14.8 [TENSION] A high-contamination series can mean formula weakness, service elevation, or multi-regime behavior

The methodology intentionally refuses to infer the cause from contamination alone. Depth, run structure, and service review remain necessary.

## 14.9 [TENSION] Statistical correctness versus operational usefulness

A threshold can be correctly placed relative to a very tight historical distribution yet represent a trivial absolute change. Relative degradation monitors and absolute failure floors answer different parts of this problem.

## 14.10 [TENSION] Fixed fleet constants versus service-specific evidence

The method needs consistent constants to be reproducible, but exceptions may be necessary for structural metric differences. Exceptions must be explicit and documented, never hidden in per-service tuning.

---

# 15. Presentation-worthy decisions shortlist

These items are tagged `[SHOWCASE]` above and are the strongest candidates for a future concise narrative:

1. **From alert-volume tuning to evidence-based review:** thresholds are no longer raised merely to make history quiet.
2. **BUSY-derived baselines are judged in BUSY:** off-hours behavior is monitored but does not redefine peak-operation normality.
3. **Traffic confidence is part of the model:** a metric is only interpreted when its exact request population supports it.
4. **R is per telemetry channel, not per service name:** ALB and APM can represent different traffic populations.
5. **T1 is statistics-informed, not statistical truth:** robust estimators produce a reproducible relative boundary; operational policy still determines persistence and severity.
6. **Fleet validation challenged the methodology itself:** T1 was tested across 29 series, not accepted because it looked plausible on one service.
7. **The test separated formula problems from service patterns:** shallow sustained crossings, deep spikes, and bimodality were not treated as the same phenomenon.
8. **Metric visibility was challenged before thresholding:** Shain's APM error baseline was retracted when evidence showed structural blindness.
9. **Partial monitoring is labeled honestly:** Shain's ALB error rate is valid for its public target-group population, not advertised as whole-service coverage.
10. **Relative degradation and absolute failure are separate:** A detects sustained worsening from normal; C uses floors to represent operational trouble. The proposed drift/accumulation monitor is deferred.
11. **Historical silence does not force a lower Critical:** healthy months without paging-level failure are expected.
12. **Known unhealthy behavior is never normalized automatically:** fix, temporarily suppress with ownership, or revise the model with evidence.
13. **Evidence acquisition is separate from interpretation:** preserved inputs and deterministic outputs can be re-analyzed without refetching, while a gate—not model confidence—establishes numerical conformance.

---

# 16. Current overall conclusion

The methodology did not emerge from one formula or one round of threshold selection. It evolved by repeatedly challenging assumptions about:

* which historical population represents normal operation;
* when a percentile or rate is statistically interpretable;
* whether the monitored telemetry can actually observe the failure;
* whether repeated crossings indicate a weak threshold, a different regime, or a struggling service;
* how relative abnormality differs from absolute operational failure;
* and where automatic calibration must stop and explicit engineering judgment must begin.

The central v3 change is not a new multiplier. It is a change in discipline:

> **Mechanical derivation produces evidence. Backtesting and review may challenge the model, but alert volume and historical inconvenience do not silently rewrite the definition of health.**

The later Part 2A/2B execution design extends the same discipline to AI-assisted work: acquisition and deterministic derivation create a preserved, gated evidence package; interpretation and report creation consume that package without silently repairing or reconstructing missing results.

---

# 17. Populist first-round disposition — 2026-07-19

## 17.1 What did the Populist error-rate run establish?

**Finding**
The ALB target-group error rate has a genuine zero baseline. Nearly the entire PEAK error history was one 2026-04-22 incident: roughly 20,580 of 20,588 5XX, peaking at 16.26% and coinciding with 20-second-plus latency. A refreshed recent window also found a smaller 2026-07-01 cluster: 1,617 5XX at 1.31–3.15%, overlapping the recent latency cluster. The 5XX buckets were separated by healthy buckets, so the three-consecutive-bucket Monitor A did not fire in either window.

**Decision**
At the time of this disposition, use the then-current zero-baseline error-rate default T1 = 2%, Warning = 5%, and Critical = 20%. Warning correctly gives sub-20% incidents attention without waking on-call; Critical remains the paging line. Keep Monitor A's fleet persistence rule unchanged for now even though it is structurally quiet on Populist's observed pattern; reconsider persistence only case by case with broader evidence. Do not create B. This 2% decision was later superseded by the fleet-wide 1% error-rate T1 floor; Populist's official report must use the current rule when regenerated.

**Why it matters**
The monitor set distinguishes attention from paging, avoids letting one incident produce a 57% "normal" threshold, and records A's blind spot without redesigning the fleet rule around one service.

## 17.2 What did the Populist latency run establish?

**Finding**
APM p95/p99 latency is strongly bimodal: a tight normal cluster near 25 ms p95, plus isolated seconds-level spikes. Monitor A is relatively quiet because nearly all spike runs are one bucket. Monitor C is floor-controlled and produced substantial severe crossing-bucket counts: in PEAK, p95 recorded 75 Critical buckets in 66 days and p99 recorded 27. These are not page counts and cannot be converted into a daily paging rate because the original backtest did not simulate Monitor C alert and recovery state. The crossings were heavily clustered: the deepest PEAK cluster on 2026-04-22 correlated with the severe 5XX incident, while the 2026-07-01 cluster correlated with milder sub-Warning 5XX.

**Decision**
Do not raise T1 or weaken persistence to absorb the spikes. Also do not assume that bimodality alone makes the spikes healthy. Regenerate the Populist latency backtest using the 30-minute Monitor C recovery and episode rules before interpreting its alert frequency. Hold Populist latency monitor wiring until the development/service team explains the behavior, confirms it as expected, or stabilizes it. More service analyses will show whether the fleet latency floors need broader reconsideration.

**Why it matters**
The raw C crossings may be correct evidence of recurring severe latency, but they do not establish how often the monitor would notify. The baseline-model question, episode frequency, and operational paging policy remain distinct.

## 17.3 Final first-round monitor-set decision

**Decision**
The active methodology is Monitor A + Monitor C only. Populist error rate is methodologically settled pending a mechanical rerun under the current 1% error-rate T1 floor. Populist latency remains investigation-blocked. Monitor B is deferred fleet-wide; future drift control may be implemented through scheduled analysis or CI/CD rather than a static Datadog monitor.

## 17.4 What did the follow-up Populist latency characterization reveal?

**Finding**
The apparent service-level bimodality contained at least two distinct patterns. Approximately 99% of `sportschedules` spikes landed exactly at `:00` or `:30`, lasted one 5-minute bucket, and returned to an approximately 50 ms baseline between pulses. Their depth increased with load, from roughly 2 seconds off-season to roughly 20 seconds during playoffs. This is the signature of a 30-minute cache-expiry/first-request rebuild cycle rather than randomly scattered latency. A separate daily event near 11:00 also affected the ping endpoint and therefore has broader service scope.

**Decision**
Record the half-hour `sportschedules` pulse and the 11:00 broader stall as separate phenomena in the Populist latency note. Use this miss to add the pre-cleaning periodicity rule and mandatory where/when/how-broad characterization to the runbook. The finding does not alter T1, the Monitor C floors, or the default service-level monitor set, and does not automatically create a `sportschedules`-specific monitor.

**Why it matters**
The original labels "bimodality" and "isolated spikes" were statistically accurate but operationally incomplete. Endpoint attribution and timing expose the mechanism class and prevent two unrelated behaviours from being treated as one distribution problem.

---

# 18. Shain v3 re-derivation findings — 2026-07-24

## 18.1 What did the Shain latency rerun reveal about the removed v2 raise loop?

**Question / assumption**
If v2 had raised Shain's latency T1, the higher value may have represented a better estimate of normal service behavior.

**Finding**
The v3 rerun restored the original unraised thresholds: Monitor A at 465 ms p95 and 809 ms p99; Monitor C Warning at 2.00 s p95 and 5.00 s p99; and Monitor C Critical at 7.80 s p95 and 14.67 s p99. The statistical checks passed, but the recent 30-day backtest produced 19 p95 and 26 p99 Monitor A events. Every A event occurred approximately 04:00–11:00 Toronto time, with none during the BUSY calibration window. The timing and endpoint attribution align with the previously investigated `GET /shain/v1/dataservice/resizeimage/$value` morning pattern; that causal attribution was reused from [[rogers-diva-shain-api Latency - Spike Analysis]] rather than independently re-proven by the threshold run. Neither series reached Critical.

**Decision**
Keep the mathematically derived v3 thresholds unchanged and require human review before monitor creation. The reviewer must decide whether the known recurring morning pattern is currently actionable service degradation. Do not let the old raise loop silently absorb it into a higher threshold.

**Why it matters**
This is a concrete demonstration of v2's failure mode: an operational pattern outside BUSY was converted into a looser definition of normal. v3 exposes the decision instead of hiding it in calibration.

## 18.2 What did the Shain error-rate rerun establish?

**Question / assumption**
Frequent Shain error-rate events may indicate worsening service behavior or an incorrectly applied failure floor.

**Finding**
The exact-pinned ALB target 5XX/request-count series has MED = 0%, MADn = 0, and ANCHOR = 0.86%. The current floors therefore supply all three thresholds: Monitor A = 1%, Monitor C Warning = 5%, and Monitor C Critical = 20%. PEAK produced 38 Monitor A events, or 17.3 normalized per 30 days, plus 11 Warning episodes across 9 days and no Critical episodes. Recent produced 3 Monitor A events and no C episodes. Roughly 95% of PEAK T1 crossings and 90% of recent crossings occurred outside BUSY, concentrated approximately 01:00–05:00. The current run independently confirmed the timing and low overnight denominators; the interpretation that a relatively steady all-hours error trickle is divided by much lower overnight traffic comes from the earlier [[rogers-diva-shain-api Error Rate]] investigation. The periodicity rule did not fire in either window. Endpoint attribution remains unavailable at the ALB metric layer, and the known APM blindness means trace data cannot close that gap.

**Decision**
Treat the thresholds as correctly derived but require a human decision on whether the Slack-only Monitor A frequency is acceptable. Keep the C floors: PEAK genuinely crossed 5%, while neither window approached 20%.

**Why it matters**
The result separates three claims that must not be collapsed: the BUSY-derived statistical boundary is valid; the 24/7 monitor will still see an overnight denominator effect; and the failure tier is behaving correctly rather than being bypassed by an unfloored backtest.

## 18.3 What reporting defect did the Shain error-rate rerun expose?

**Question / assumption**
A report-writing model can safely reconstruct a missing PEAK statistic from nearby analysis outputs.

**Finding**
The results structure exposed a conveniently labeled `max_observed_recent` scalar but no equivalent canonical PEAK field. The PEAK maximum was available only in a separate tail structure. The first report therefore reused the recent maximum, 3.571% at 2026-06-28 01:40, as if it also described PEAK. The actual PEAK maximum was 9.76% at 2026-05-09 01:35. That isolated bucket crossed the 5% C-Warning line but was not a three-bucket A event. The maximum A-event peaks were 7.07% in PEAK and 2.53% recent. The incorrect 3.571% statement contradicted the same report's correct count of 11 PEAK Warning episodes above 5%.

**Decision**
Correct the Shain report and treat the incident as a schema and conformance failure, not only a prose mistake. Every reportable figure must be emitted canonically per window, validated against the correct window, and covered by the gate. A model must report a missing required field rather than silently reconstructing it from an asymmetric structure.

**Why it matters**
Plausible prose can conceal an impossible internal relationship. Stronger model review reduces the risk but cannot replace a complete data contract and deterministic consistency checks.

---

# 19. Reproducible execution architecture — 2026-07-25

## 19.1 [SHOWCASE] Why split data acquisition from analysis and report creation?

**Question / assumption**
The Part 2A/2B split exists mainly to reduce model cost.

**Finding**
Cost is secondary. The important boundary is between acquiring time-bounded evidence and interpreting it. Preserving the fetched data and deterministic outputs allows the methodology to be rerun, revised, audited, or reviewed by a different model without repeating the original collection. It also allows two versions of the analysis to be compared against identical evidence.

**Decision**
Use two explicit stages:

* **Part 2A — Data and deterministic derivation:** a Sonnet-class executor performs query validation and pinning, data retrieval, densification, script execution, mechanical detectors, Step 11 evidence fetching/tabulation, artifact packaging, and the conformance gate. It must not interpret flags, adjust thresholds, choose a different metric, or write the final report.
* **Part 2B — Review and report:** an Opus-class reviewer reruns the gate, reconciles the canonical outputs, interprets anomalies and prior investigations, frames human decisions, and writes the entire report including the Executive Summary.

The future multi-agent form may let Part 2B delegate Part 2A automatically. Initial runs remain manual so the boundary and artifacts can be tested.

**Why it matters**
The report becomes a repeatable interpretation of preserved evidence rather than an inseparable product of one model session. A cheaper executor is useful, but reproducibility and auditability are the governing reasons for the split.

## 19.2 Why is Sonnet acceptable for Part 2A but not the sole report authority?

**Question / assumption**
Trust can be assigned by report section: Sonnet writes the detailed report, while Opus handles only anomaly characterization and the Executive Summary.

**Finding**
The Shain maximum defect occurred in an ordinary detailed-report section. Sonnet correctly handled many explicit fields but made a consequential false claim when it had to join asymmetric structures, then failed to notice the claim was incompatible with 11 episodes above the stated 5% threshold. The practical trust boundary is therefore not "summary versus detail"; it is "explicit canonical output versus reconstruction and interpretation." Haiku is also not the preferred default for Part 2A because query pinning, probe recovery, and telemetry-integrity edge cases are not reliably trivial.

**Decision**
Use Sonnet as an executor and renderer of deterministic artifacts, with hard STOP conditions and a gate. Use Opus for all synthesis, cross-window reconciliation, anomaly meaning, attribution discipline, and report prose. Neither model is a correctness mechanism: code must validate reportable numbers and internal invariants.

**Why it matters**
Changing the prose model alone would reduce risk but leave the asymmetric-schema failure mode intact. The combination of canonical outputs, deterministic checks, and model-role separation is what makes the workflow trustworthy.

## 19.3 Which run artifacts are permanent?

**Question / assumption**
Every input and intermediate should be copied into each run package so the report is self-contained.

**Finding**
That would duplicate canonical PEAK data and create multiple copies that can diverge. Conversely, several important items previously existed only in terminal output or model memory: probe outcomes, telemetry-integrity warnings, query decisions, script state, and Step 11 span evidence that can age out quickly.

**Decision**
Retain the following:

* one immutable canonical PEAK raw file per `(series, exact window bounds or vintage)`, stored once outside individual run packages;
* permanent recent-window raw data with exact bounds;
* permanent deterministic results;
* permanent Step 11 anomaly evidence whenever characterization ran, because raw span retention is short;
* `manifest.json`;
* `checks.json`;
* `run.log` while it still contains unique evidence not represented structurally.

The manifest records PEAK slug and content hash rather than copying PEAK raw into every package. A newly fetched PEAK window becomes a new canonical immutable file first and is then referenced exactly like reused PEAK data. Canonical naming includes window bounds or vintage so seasonal refreshes never overwrite an older immutable file.

Do not retain densified grids, reproducible plots, temporary probe payloads, failed-query payloads, or duplicate script copies. Retain probe summaries such as point/non-null counts and a one-line record of any failed query and resulting correction. If a plot is directly cited and not cheaply reproducible, it can be promoted deliberately.

**Why it matters**
The permanent package preserves time-bounded evidence and every reportable result without multiplying large canonical inputs or retaining disposable execution noise.

## 19.4 What must `manifest.json`, `checks.json`, and `run.log` prove?

**Question / assumption**
A results JSON and final note are enough to reproduce a run.

**Finding**
Results can become unexplainable if the query, request gate, registry state, script revision, or uncommitted code differed. A Git commit alone is also false provenance when the script ran with local changes.

**Decision**
`manifest.json` pins:

* exact PEAK and recent bounds, source slugs/files, and hashes;
* exact metric identity, query, and scope;
* script path, commit SHA, exact argv, and dirty flag;
* the diff when the executed script was dirty;
* selected R and Gate-R, their selection basis, and the relevant registry state/version;
* probe point/non-null summaries;
* allowed query corrections and the reason for each;
* the linked vault note and publication commit when the vault copy is canonical.

`checks.json` records the conformance and invariant gate result. `run.log` remains permanent for now because warnings, tracebacks, probe counts, and rule-version context are not yet all emitted structurally. Once all meaningful evidence is structured, the log may become optional.

**Why it matters**
The package records not only what number was produced, but which evidence, code state, traffic gate, and execution decisions produced it.

## 19.5 Is the published report duplicated inside the run package?

**Question / assumption**
Every run package should contain `report.md`.

**Finding**
If the vault note is the maintained canonical report, a second editable copy will diverge. If the vault is not reliably versioned and committed at publication, omitting the snapshot makes the original interpretation unrecoverable.

**Decision**
Keep this conditional until the publication workflow is confirmed:

* if the vault report is Git-versioned and reliably committed when published, do not duplicate `report.md`; record the note name and publication commit in the manifest;
* otherwise retain a frozen `report.md` snapshot in the package.

**Why it matters**
The design avoids accidental duplicate sources of truth without losing the exact report that accompanied a historical run.

## 19.6 How is the report schema kept synchronized with the gate?

**Question / assumption**
A hand-maintained Part 2A required-field list and a separate Output-section list are sufficient if both are updated carefully.

**Finding**
Two independent lists reproduce the same asymmetric-schema risk one level higher. A new figure can become required by the report but remain invisible to the gate. Quiet windows also expose a special case: when there are no T1 crossings or A events, the derived tail arrays are empty, so a canonical per-window scalar may be the only evidence for the observed maximum.

**Decision**
Define one explicit master conformance schema for every reportable field. The Part 2A output contract, Output specification, and gate must reference or verify coverage against that master. A coverage mismatch is a `GATE-COVERAGE DEFECT`.

Part 2B normally derives stated extremes from the canonical per-window arrays. When the corresponding arrays are empty, the canonical window-specific scalar plus a passing gate is the accepted evidence path and the report identifies that sourcing.

**Why it matters**
Every figure required by prose or tables is guaranteed to exist in a window-specific, checkable form, including quiet-window cases where list-based validation would otherwise become vacuous.

## 19.7 Which cross-window invariants are hard failures?

**Question / assumption**
If PEAK and recent have identical maximum values, they must have identical timestamps.

**Finding**
Error rates are quantized at modest request volumes, so the same exact rate can legitimately occur at different timestamps. For disjoint windows, matching timestamps are impossible. Equality is therefore not evidence of back-fill by itself.

**Decision**
Validate each value and timestamp independently against its own window. A timestamp outside its named window is a hard failure. Identical values across disjoint windows are at most a warning, not a failure. Gate findings use explicit `FAIL`, `WARN`, or `VACUOUS` states so an empty comparison is not misrepresented as a passed substantive check.

**Why it matters**
The gate catches actual cross-window contamination without rejecting legitimate quantized coincidences.

## 19.8 What happens when Part 2B finds a Part 2A defect?

**Question / assumption**
Part 2B can repair a suspicious number or continue using the packaged `checks.json`.

**Finding**
That would make the reviewer another silent derivation layer. It would also trust a gate result produced before the reviewer's own consistency checks.

**Decision**
Part 2B reruns the gate itself. On a Part 2A or gate-coverage defect, stop report generation, correct the deterministic recipe/schema/gate, and rerun Part 2A from the retained recent raw, canonical PEAK raw, and retained anomaly evidence where applicable. Do not refetch the threshold time series merely to repair analysis logic. Preserve Step 11 extracts because their source retention may expire before remediation.

**Why it matters**
Defects are fixed at the deterministic evidence layer and the corrected report remains reproducible from the original inputs.

## 19.9 What is an allowed query correction versus a forbidden metric change?

**Question / assumption**
The Part 2A executor may switch queries whenever the first probe is empty.

**Finding**
Some corrections preserve the approved metric identity—for example fixing exact pinning or syntax after a probe reveals an empty or malformed response. Other changes select a different metric variant, aggregation, or traffic population and therefore change what is being analyzed.

**Decision**
Part 2A may make and log a query correction only when it preserves the pre-approved metric identity and population. It retains the probe summary and one-line decision record. Changing the metric variant, aggregation semantics, or monitored traffic population is a STOP condition requiring human resolution.

**Why it matters**
The mechanical executor can recover from ordinary API/query issues without gaining authority to redefine the signal.

## 19.10 Where are the actual model-cost savings?

**Question / assumption**
Most token cost comes from processing the raw five-minute series.

**Finding**
The recipe already processes large arrays in code without loading them into model context. Basic derivation is therefore comparatively token-light. The larger mechanical cost comes from query probing, pinning, retries, registry lookup, telemetry-integrity handling, and especially Step 11's high-volume span fetches and tabulation.

**Decision**
Delegate those mechanical acquisition and tabulation tasks to Part 2A, but evaluate the architecture on reproducibility and correctness first. Opus still owns interpretation and the entire report even if that limits the maximum cost reduction.

**Why it matters**
The split does not compromise analytical quality to chase savings that may be smaller than expected. It places cheaper execution where the work is genuinely mechanical and preserves stronger review where judgment is unavoidable.

## 19.11 What is implemented versus still pending?

**Finding**
The Part 2A/2B split was added in commit `7b6df63`, and the initial `v3-gate.py` demonstrated that a gate could catch the Shain cross-window maximum defect. A subsequent draft revision addressed the master-schema coupling, quiet-window evidence path, invariant states, PEAK naming, 2B gate rerun, defect remediation loop, and query-change boundary. At the end of this discussion, that revised runbook was still awaiting placement over the normative copy. Matching `v3-gate.py` changes, the canonical PEAK filename migration, and an update to the older Concepts companion were also pending follow-up; the PEAK rename was explicitly confirm-first.

**Decision**
Treat Section 19 as the accepted execution design, but do not claim the revised gate and storage migration are operational until the normative runbook, code, and existing artifacts have been updated and tested together on fresh services.

**Why it matters**
The ledger distinguishes a settled design from a completed implementation. The first useful validation is now an end-to-end manual Part 2A → Part 2B run, not more theoretical prompt editing.

# 20. ISL OData findings and methodology revisions — 2026-08-08 to 2026-08-09

## 20.1 [SUPERSEDES 2.2] How is BUSY now handled?

**Finding**
The previous fixed `18:00–22:59` window mixed the stable historical PEAK calibration regime with a recent, seasonally shifting traffic profile.

**Decision**
Fix **PEAK BUSY at `19:00–22:59` America/Toronto**. Derive weekday and weekend **NOW BUSY** from each recent run for reporting only; NOW BUSY does not alter calibration or Gate R.

**Why it matters**
The calibration population remains stable while current traffic shifts remain visible without silently changing thresholds.

## 20.2 [SUPERSEDES 3.2] How is Gate R selected?

**Finding**
The fixed candidate menu did not directly establish percentile reliability and could apply one confidence assumption too broadly.

**Decision**
Select R from a **per-count-series, per-percentile reliability/survival curve**, using the house survival appetite and explicit human approval. Do not reuse an R across endpoint slices or percentiles without supporting evidence.

**Why it matters**
R reflects the actual stability of the exact percentile and request population being monitored.

## 20.3 [SHOWCASE] What did the ISL OData endpoint separation establish?

**Finding**
Whole-service p99 was misleading because `searchterms` represented very little traffic yet dominated the tail when higher-volume browse endpoints subsided. `itemlists/items` showed a separate two-population p95 pattern. A single aggregate latency monitor therefore mixed materially different populations.

**Decision**
Use separate latency calibration for:

* `itemlists/items`;
* the remainder of the service excluding both `itemlists/items` and `searchterms`.

Leave `searchterms` intentionally uncalibrated unless future evidence shows sufficient operational importance or user impact to justify a dedicated monitor.

**Why it matters**
A tiny endpoint cannot distort the service-wide percentile, while monitor separation remains evidence-driven rather than automatically creating a monitor for every endpoint.

## 20.4 [CLARIFIES 20.1] How is NOW BUSY derived and retained?

**Finding**
NOW BUSY is contextual: it affects recent headroom and BUSY/off-BUSY classification, but not threshold calibration or 24/7 event counting. Weekend traffic can differ materially from weekdays.

**Decision**
For each new run, Part 2A fetches the recent 30-day ALB traffic and derives weekday and weekend NOW BUSY separately using hourly medians and the 50%-of-peak rule. Select the contiguous qualifying block containing the peak hour; record but exclude detached qualifying hours. Package the derived windows and source dates. Remediation reruns reuse the packaged values rather than refetching.

**Why it matters**
Recent backtesting reflects the traffic regime of that period while remaining reproducible.

## 20.5 [CLARIFIES 3.2] How is error-rate R selected?

**Finding**
Requiring an error-rate R candidate to fit BUSY p1 in both PEAK and recent windows allowed a changing contextual window to affect an approved calibration input.

**Decision**
For error rates only, choose the largest value in `{50, 100, 200, 500}` at or below **PEAK BUSY p1**. NOW BUSY has no role in selecting error-rate R.

**Why it matters**
PEAK owns calibration; NOW remains contextual. A seasonally changing recent window cannot cause error-rate R to drift.

## 20.6 Are earlier packages recalculated after the BUSY and error-rate R changes?

**Decision**
Effective **2026-08-08**, the new PEAK BUSY, per-run NOW BUSY, and PEAK-only error-rate R rules apply to new runs. Existing packages and approved registry values remain untouched. Record that earlier results used the previous methodology and may differ slightly; revisit them only when the service is analyzed again or a concrete concern arises.

**Why it matters**
The methodology improves prospectively without creating a low-value fleet-wide rerun.

# 21. Execution scope discipline

## 21.1 How should analytical work avoid unnecessary expansion?

**Finding**
The execution process repeatedly promoted adjacent imperfections and optional refinements into new investigations, creating substantial review and decision overhead without proportional operational value.

**Decision**
Complete only the requested task using the simplest approach that fully satisfies it and preserves the established methodology. Make minor, reversible implementation decisions without escalation. Do not investigate adjacent issues, redesign settled methodology, or reopen prior decisions. Add work beyond scope only when something material would otherwise fail, and state that justification in one sentence. Defer non-blocking concerns without asking.

**Why it matters**
The human reviewer can evaluate the completed result instead of supervising every branch of the investigation.

# 22. Baseline health and Reception monitoring — 2026-08-09

## 22.1 [SHOWCASE] Can a statistically stable baseline be assumed healthy?

**Question / assumption**
A clean, low-contamination baseline with few historical alerts represents an acceptable service state.

**Finding**
Baseline-relative analysis detects deviation from ordinary behaviour; it cannot establish that ordinary behaviour is operationally acceptable. Reception demonstrated the failure mode: p99 was typically about 2 seconds, yet its broad distribution produced T1 ≈ 12.79 seconds and relative Monitor C thresholds near 40/201 seconds. Zero historical firings therefore reflected unusable or unreachable thresholds, not proven health.

**Decision**
Add a mandatory baseline-health gate. If the observed baseline appears materially unacceptable for the service type and exposure—or its acceptability cannot be established—stop calibration. Do not recommend baseline-relative thresholds until an acceptable operating state has been established through product expectations, dependency commitments, comparable systems, or explicit owner validation.

**Why it matters**
The methodology must not convert chronically poor but stable performance into the definition of health.

## 22.2 Why was aggregate Reception latency unsuitable as the primary monitor?

**Finding**
Reception’s aggregate latency mixes fast endpoints with user-facing token operations whose latency is dominated by Adobe Pass `preauthorize` calls. The aggregate percentile therefore changes with endpoint composition and does not cleanly represent either user experience or dependency health.

**Decision**
Stop the aggregate Reception calibration and separate the signals:

* one user-facing latency monitor combining `POST /api/tve/token` and `POST /api/tve/v2/token`;
* one diagnostic dependency-latency monitor for successful Adobe `preauthorize` spans to `sp.auth.adobe.com`.

Split v1 and v2 further only if their distributions or acceptable targets prove materially different. Retain aggregate service latency as context, not the primary alert.

**Why it matters**
The user-facing monitor answers whether authentication completes acceptably; the dependency monitor explains whether Adobe is responsible for degradation.

## 22.3 [OPEN] Is the observed Adobe latency expected, and how should its errors be handled?

**Finding**
Successful Adobe calls commonly take approximately 1–5 seconds and sometimes approach 10 seconds. The developer did not confirm that this is expected for Adobe Pass; the response only proposed short timeouts and limited retries as possible future resilience work. Therefore the latency’s acceptability remains unresolved.

Handled Adobe HTTP 400 responses are a separate concern. They may indicate invalid requests and should not be normalized as part of the latency baseline or retried generically.

**Decision**
Raise the chronic multi-second token latency and Adobe 400s to the application team for validation and prioritization. If the successful-response latency is confirmed acceptable, calibrate the Adobe dependency monitor against that established state and focus defect work on the 400s. Monitor dependency errors separately from successful-response latency.

**Why it matters**
A slow-but-expected dependency and a malformed or rejected request require different operational responses; combining them obscures both.

# 23. Production webfacing 5XX monitor disposition — 2026-08-09

## 23.1 How should the methodology be explained outside the SRE runbook?

**Finding**
Implementation tickets cannot show the full calibration process without becoming difficult to review, while listing only the final round-number thresholds makes them appear arbitrary.

**Decision**
Maintain a company-facing methodology companion at [[Thresholds/_Methodology/How SRE Calibrates Service Monitors]]. Tickets should link to it and explain only which part of the methodology controlled the final result.

**Why it matters**
The detailed methodology remains visible and credible without overwhelming each implementation ticket.

## 23.2 Why do the standard floors control the `production-webfacing` 5XX thresholds?

**Finding**
The ALB’s normal 5XX rate is effectively zero. Baseline-relative calculations therefore produce thresholds that are statistically unusual but operationally insignificant.

**Decision**
Use the standard error-rate floors as the effective thresholds:

* Monitor A: 1% sustained for three valid 5-minute buckets;
* Monitor C Warning: 5% in one valid 5-minute bucket;
* Monitor C Critical: 20% in one valid 5-minute bucket.

Apply the SRE-defined minimum-traffic gate.

**Why it matters**
The monitor ignores trivial deviations from a near-zero baseline while preserving clear severity for genuine user impact.

## 23.3 Should the existing anomaly monitor be retired or updated?

**Finding**
Existing monitor `ALB_EXTERNAL_5XX_HIGH` (`140515163`) can represent the five-minute Warning/Critical rule after conversion from raw-count anomaly detection to 5XX error rate. The 1% degradation condition requires different persistence and routing.

**Decision**
Prefer updating `140515163` to become the 5% Warning / 20% Critical monitor, unless its implementation makes conversion unsuitable. Create the 1% sustained-degradation signal as a separate SRE-only monitor.

**Why it matters**
The existing monitor can be reused without forcing two operationally different conditions into one evaluation model.

## 23.4 Are false alarms and real peak-load failures the same problem?

**Finding**
The current anomaly monitor catches both trivial deviations from the near-zero baseline and genuine peak-load bursts of approximately 5–27%. The baseline noise makes the real failures difficult to distinguish and encourages the alert to be ignored.

The real bursts are primarily 504s associated with the `rocket-api` → ISL connection-reuse defect.

**Decision**
Treat the threshold change as a monitoring correction only. Keep RD-7159 as the separate application fix for the real peak-load failures.

**Why it matters**
Better alerting preserves visibility and assigns meaningful severity, but it must not be presented as remediation of the underlying failure.
