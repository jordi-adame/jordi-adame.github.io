# Contract: Metrics Snapshot (`experiments/<id>/snapshots/<YYYY-MM-DD>.json`)

Written by the read step; never edited afterwards. One file per read, holding rows from every source.

```json
{
  "experiment": "E2-ascii-header",
  "read_at": "2027-01-12T07:05:00Z",
  "rows": [
    {"source": "statsig", "day": "2027-01-11", "variant": "v1-ascii", "url": "/",
     "metrics": {"human_pageviews": 9, "engaged_pageviews": 4, "outbound_clicks": 2},
     "complete": true, "no_data": false},
    {"source": "gsc", "day": "2027-01-09", "variant": "v1-ascii", "url": "/",
     "metrics": {"impressions": 14, "clicks": 1, "ctr": 0.071, "position": 6.2},
     "complete": true, "no_data": false},
    {"source": "statsig", "day": "2027-01-11", "variant": "v1-ascii", "url": "/",
     "metrics": {"crawler_hits": {"GPTBot": 1, "OAI-SearchBot": 0, "ClaudeBot": 2, "Googlebot": 3},
                 "ai_referrals": {"chatgpt": 0, "perplexity": 1}},
     "complete": true, "no_data": false}
  ],
  "sources_missing": ["bing_ai"],
  "status": "collecting"
}
```

## Rules

- `day` is the latest day the source marks complete (for Search Console: before `metadata.first_incomplete_date` with `dataState: final`).
- For time-window units, a source row's `variant` is the variant live on that `day`; days that straddle a window change are dropped from analysis.
- `no_data: true` means the source returned nothing; never written as zeros.
- `status` is one of `collecting`, `insufficient_data`, `ready_to_decide`, `guardrail_breach`.
- No row may contain secrets, raw IPs or full user-agent strings (bot name only).
