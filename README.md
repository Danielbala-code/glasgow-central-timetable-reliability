# Glasgow Central timetable reliability

**Status — 9 November 2026:** exploratory data acquisition and inspection completed for one pilot day. The simulation and evaluation described below are planned. This repository currently contains the project brief and proposed methods; implementation will be added after review.

## Question and intended decision

For a defined group of services calling at Glasgow Central, how does recorded lateness change during the station stop, and which service patterns warrant closer timetable review?

The first cohort consists of services travelling from Partick through Glasgow Central. It is a subset of Central's services. Extending to other routes or Central-originating trains will require a verified service inventory and separate treatment of operationally different groups.

The intended output is a reproducible historical analysis, an empirical Monte Carlo baseline and a short technical note explaining possible timetable-review candidates, uncertainty and limitations.

## Why this problem

A late arrival does not establish that the station stop added delay. Comparing arrival and departure lateness helps distinguish inherited lateness from changes during the visit. This supports the preparation of historical event data for simulation and the assessment of timetable risks.

The study is intended to screen candidates for operational review. Physical feasibility, signalling conflicts and an operationally approved timetable require additional engineering inputs.

## Data inspected so far

Source: Rail Delivery Group's **Historical Service Performance (HSP)**, which contains historical Darwin-derived service information.

| Pilot inspection | Verified result |
|---|---:|
| Service date | 3 November 2026 |
| Discovery query | Partick → Glasgow Central, full departure day in six bounded windows |
| Distinct dated journeys returned | 101 |
| Location-visit records across their complete journeys | 2,268 |
| Duplicate journey/sequence keys in the extracted table | 0 |
| Central visits with all four scheduled/actual arrival/departure fields populated | 79 |
| Central visits requiring investigation of incomplete timings | 22 |
| Original journey JSON, excluding manifests and metrics responses | 269,974 bytes |
| Compressed Parquet containing the extracted location records | 22,210 bytes |

These counts describe the returned cohort, not an independently verified count of every scheduled service. Populated fields also require timing-consistency checks before modelling. Missing observations will be retained and investigated rather than automatically classified as cancellations.

An initial Python acquisition prototype and storage checks have been run locally. Source code, reproducible checks and results will be published as they are reviewed. API credentials and raw service responses are excluded from this repository.

## Proposed study window

The intended expansion is **1 May–31 October 2026**. Six-month acquisition has not yet been completed.

May–September is the proposed development period; October is reserved for chronological evaluation. Service-pattern changes, day type, disruption conditions and missingness will be reviewed before pooling dates. The final split will be recorded before model comparison.

## Understanding the records

HSP exposes two relevant operations:

1. **serviceMetrics** discovers matching services between a station pair over a date/time window and returns their RIDs.
2. **serviceDetails** retrieves one dated journey and its ordered location list.

| Field | Interpretation |
|---|---|
| rid | Identifier for a dated journey; retained as supplied |
| date_of_service | Operating date supplied by HSP |
| toc_code | Operator code |
| location | CRS station code in the inspected HSP records |
| gbtt_pta / gbtt_ptd | Public scheduled arrival/departure |
| actual_ta / actual_td | Reported actual arrival/departure |
| late_canc_reason | Source reason code, requiring interpretation |

The analytical grain is **one location visit within one dated journey**. Preserve the array order as stop sequence: a journey can revisit a location, so journey ID plus station code is not assumed to be a unique visit key.

Original time strings are preserved. Dated timestamps, midnight crossings, source timezone and timetable basis must be resolved before general duration calculations. The inspected HSP timings have minute precision, which limits interpretation of short dwell differences.

## Proposed mathematics

For one station visit, let:

- $a_s, d_s$: scheduled arrival and departure;
- $a_o, d_o$: reported actual arrival and departure.

Recorded lateness is:

$$
L_a = a_o-a_s,\qquad L_d=d_o-d_s.
$$

Scheduled and observed stop durations are:

$$
D_s=d_s-a_s,\qquad D_o=d_o-a_o.
$$

Their relationship is:

$$
\Delta L=L_d-L_a=D_o-D_s.
$$

Positive $\Delta L$ means recorded lateness increased during the visit; negative values mean it decreased. This is an accounting identity, not a causal attribution of delay to the station.

### One inspected example

| Glasgow Central visit | Scheduled | Reported actual |
|---|---|---|
| Arrival | 07:13 | 07:15 |
| Departure | 07:18 | 07:17 |
| Stop duration | 5 minutes | 2 minutes |

Here $L_a=2$, $L_d=-1$, and $\Delta L=-3$ minutes. The apparent early departure must be assessed against timing definitions and precision before imposing a departure-holding rule.

## Planned empirical simulation

The initial model will reproduce distributions of departure lateness for comparable service groups. A sampled visit satisfies:

$$
L_d^{(b)}=L_a^{(b)}+D_o^{(b)}-D_s.
$$

Arrival conditions and dwell observations will be sampled jointly, or through a justified conditional model, to preserve their dependence. Service pattern, scheduled stop duration and time band are candidate grouping variables, subject to sample size.

The baseline timetable will remain explicit and versioned. Any hypothetical change to scheduled allowance or departure rules will be reported as a sensitivity scenario with declared behavioural assumptions. A change in reported lateness under a changed schedule is not automatically a reduction in actual journey time.

This is an empirical station-performance model. It does not contain a train-physics engine, signalling headways, platform conflicts, crew or rolling-stock circulation constraints.

## Validation and uncertainty

- Audit coverage, repeated visits, scheduled/actual timing consistency and missing observations before fitting.
- Compare with simple baselines: inherited arrival lateness and historical departure-lateness summaries for comparable services.
- Develop on earlier dates and evaluate on later dates.
- Compare departure-lateness distributions, median/tail errors and explicitly defined punctuality thresholds.
- Report sample counts and uncertainty by service group; use day-level resampling where appropriate to preserve shared operating conditions.
- Keep simulation variability separate from uncertainty due to limited historical evidence.
- Treat incomplete and disrupted services explicitly so complete-case results are not presented as the whole service population.

No predictive accuracy, calibration result or timetable benefit has yet been established.

## Work sequence

1. Review the pilot records and agree the cohort and decision question.
2. Investigate the incomplete Central timing records and decode reason fields.
3. Extend historical collection with caching, manifests and a 5 GB project-data ceiling.
4. Produce the cleaned event table and descriptive analysis.
5. Implement and evaluate the empirical baseline.
6. Compare clearly specified scenarios and publish a concise technical note.

The current local tools are Python, Requests, DuckDB and PyArrow, with original JSON and compressed Parquet. Cloud infrastructure is not required for the initial study.

## Sources and attribution

- [Historical Service Performance — Rail Data Marketplace](https://raildata.org.uk/dataProduct/P-2dfd6e84-c3e7-460f-8b3c-ac201d259f51/overview)
- [National Rail Darwin data feeds](https://www.nationalrail.co.uk/developers/darwin-data-feeds/)

Rail data is provided by Rail Delivery Group/National Rail under the applicable product terms. This is an independent portfolio study.
