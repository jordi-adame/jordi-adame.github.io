# Data Model: Agent-Run SEO/GEO Experiments

All records are plain files in the repository under `experiments/` (excluded from the Jekyll build and from the deployed site). Statsig holds assignment config and raw events; the repo holds the agent's durable record. Files are append-only where noted, so history is auditable in git.

## Experiment

File: `experiments/<id>/experiment.yaml` (schema: [contracts/experiment-manifest.md](contracts/experiment-manifest.md))

| Field | Type | Rules |
|-------|------|-------|
| `id` | string | `E<n>-<slug>`, unique, never reused |
| `hypothesis` | string | one sentence, falsifiable |
| `unit` | enum | `page`, `time_window`, `none` (observational probe); `visitor` only if every metric is `engagement` (FR-004) |
| `urls` | list of paths | each path in at most one experiment with status `running` (FR-006) |
| `variants` | list of Variant ids | exactly one `control` |
| `schedule` | object | for `time_window`: window length (`1d`, `7d`), randomized order seed, start, planned end |
| `page_assignment` | map path to variant | for `page`: fixed before launch, produced by a recorded random seed |
| `primary_metric` | Metric ref | exactly one |
| `secondary_metrics`, `guardrail_metrics` | list of Metric refs | guardrails may stop the experiment early |
| `min_sample` | object | per arm, in the primary metric's unit |
| `planned_duration` | duration | at least 14 days for engagement, 28 days for search/AI metrics |
| `success_rule` | string | pre-registered; frozen once status leaves `approved` |
| `statsig` | object | experiment or dynamic-config name in Statsig; no keys |
| `status` | enum | see state machine |
| `extended` | bool | at most one extension (FR-022) |

State machine:

```text
draft -> checked -> approved -> running -> stopped -> decided
          |            |           |                     ^
          v            v           +--(emergency stop)---+
       rejected     withdrawn
```

- `draft -> checked`: all guardrail checks pass for all variants.
- `checked -> approved`: Jordi's approval recorded for every variant and for the start.
- `running -> stopped`: planned end reached, guardrail metric breached, or stop by Jordi/agent.
- `stopped -> decided`: decision written and (for ship) approved by Jordi.
- `running -> running` with `extended: true`: once, by at most `planned_duration`.

## Variant

File: `experiments/<id>/variants/<variant-id>/` holding the rendered content (HTML fragment or full page) plus `variant.yaml`.

| Field | Type | Rules |
|-------|------|-------|
| `id` | string | `control` or `v<n>-<slug>` |
| `description` | string | what changes, in one sentence |
| `content_ref` | path | files inside the variant folder |
| `indexable_change` | bool | true if visible text, links, title, meta or structured data change |
| `outbound_links` | list of URLs | each domain must be on the allowed list (FR-010) |
| `new_pages` | list of paths | at most 5 per experiment by default |

## CheckResult

File: `experiments/<id>/checks/<timestamp>.json` (append-only)

| Field | Type |
|-------|------|
| `variant` | Variant id |
| `check` | enum: `impersonation`, `unsourced_claim`, `outbound_domain`, `hidden_content`, `crawler_parity`, `page_count`, `render`, `secret_scan`, `copied_text` |
| `result` | `pass` / `fail` |
| `reason` | string, required on fail |
| `tool` | which checker produced it (script name or `agent-review`) |

## Approval

File: `experiments/<id>/approvals.md` (append-only)

| Field | Type | Rules |
|-------|------|-------|
| `what` | enum: `publish_variant`, `start`, `extend`, `ship`, `revert` | |
| `target` | variant id or experiment id | |
| `by` | `Jordi` | only Jordi's own typed words count; never inferred |
| `when` | ISO timestamp | |
| `words` | quoted text | Jordi's words, verbatim |
| `where` | `PR review`, `chat window`, `issue comment` | |

## AssignmentWindow (time-window units)

File: `experiments/<id>/schedule.json` (generated before launch, frozen)

| Field | Type |
|-------|------|
| `start`, `end` | ISO timestamp, UTC |
| `variant` | Variant id |

## ObservationSnapshot

File: `experiments/<id>/snapshots/<YYYY-MM-DD>.json` (one per read, never edited; schema: [contracts/metrics-snapshot.md](contracts/metrics-snapshot.md))

| Field | Type | Rules |
|-------|------|-------|
| `read_at` | ISO timestamp | |
| `source` | enum: `statsig`, `gsc`, `gsc_ai_report`, `bing`, `bing_ai`, `request_logs` | |
| `day` | date | latest complete day for the source |
| `variant` | Variant id | |
| `url` | path | |
| `metrics` | map | metric name to value |
| `complete` | bool | false while the source may still revise the day |
| `no_data` | bool | true when the source returned nothing (distinct from zero) |

## Decision

File: `experiments/<id>/decision.md`

| Field | Type |
|-------|------|
| `outcome` | `ship`, `revert`, `inconclusive` |
| `rule` | the frozen success rule, copied |
| `numbers` | table of the primary and guardrail metrics per arm with completeness |
| `reasoning` | short text |
| `approval` | Approval ref (required for `ship`) |

## AllowedDomains

File: `experiments/allowed-domains.txt`, one domain per line, edited only by Jordi (enforced by CODEOWNERS and by the check refusing a list changed in the same PR as a variant).

## PageviewEvent (in Statsig, not in the repo)

Logged server-side for every HTML request to an experiment URL, and for every crawler request to any URL.

| Field | Notes |
|-------|-------|
| event name | `pageview` |
| unit ID | the assignment unit: URL path (page unit) or window id (time unit); never a visitor ID |
| `path`, `variant`, `window_id` | |
| `client_class` | `human`, `crawler_verified`, `crawler_unverified` |
| `bot_name`, `bot_purpose` | e.g. `GPTBot`/`training`, `OAI-SearchBot`/`ai_search`, `ChatGPT-User`/`user_fetch` |
| `ref_class` | `search`, `ai:<engine>`, `social`, `direct`, `other` |

Engagement events (`outbound_click`, `engaged_15s`, `scroll_75`) are logged from the browser with the same unit ID and `variant`, no cookie.
