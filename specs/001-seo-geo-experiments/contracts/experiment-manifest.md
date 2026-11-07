# Contract: Experiment Manifest (`experiments/<id>/experiment.yaml`)

The manifest is the single source of truth for an experiment. The guardrail checker, the assignment code and the agent all read it. Field meanings are in [../data-model.md](../data-model.md).

```yaml
id: E2-ascii-header
hypothesis: >-
  An animated ASCII header raises the share of human pageviews with an
  engagement event, without lowering search impressions.
unit: time_window            # page | time_window | none | visitor (engagement-only)
urls: ["/"]
variants:
  - id: control
    description: Today's static header.
  - id: v1-ascii
    description: Animated ASCII header; indexable text unchanged.
    indexable_change: false
schedule:                    # time_window only
  window: 1d                 # UTC calendar days
  design: paired_switchback  # each pair of days holds both arms in random order
  seed: 4127                 # recorded so the schedule can be regenerated
  start: 2027-01-04
  planned_end: 2027-02-01
page_assignment: {}          # page unit only: {"/topics/a": control, ...}
primary_metric:
  name: engaged_share
  source: statsig
  definition: engaged pageviews / human pageviews
secondary_metrics:
  - {name: outbound_clicks_per_pv, source: statsig}
guardrail_metrics:
  - {name: gsc_impressions_root, source: gsc, stop_if: "drops >50% vs pre-period weekly median for 2 weeks"}
min_sample: {per_arm: 100, unit: human_pageviews}
planned_duration: 28d
success_rule: >-
  Ship v1-ascii if P(engaged_share v1 > control) >= 0.90 with >=100 human
  pageviews per arm and no guardrail stop; otherwise inconclusive.
statsig:
  kind: dynamic_config       # dynamic_config | experiment
  name: e2_ascii_header      # names only; never keys
status: draft
extended: false
```

## Validation rules (enforced by `checks/manifest`)

1. Every field above except `page_assignment`/`schedule` (one of them is required by `unit`) is present.
2. `unit: visitor` is rejected unless every metric has `source: statsig` and every variant has `indexable_change: false` (FR-004).
3. No path in `urls` belongs to another manifest with `status: running` (FR-006).
4. `success_rule`, `primary_metric`, `min_sample` and `planned_duration` are byte-identical to the version recorded at `approved`; any change after approval fails the check (FR-021).
5. `extended: true` is allowed once; `planned_end` may move by at most `planned_duration` (FR-022).
6. No value matches a secret pattern (FR-025).
