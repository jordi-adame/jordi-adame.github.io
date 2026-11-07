# Contract: Request-Time Assignment

Runs in Vercel Routing Middleware (hosting decided: OQ-2 A, Vercel Hobby) for every HTML request.

## Input

- Request path, method, `User-Agent`, `Referer`, client IP (IP is used only for crawler verification and is never stored or sent to Statsig).
- Current UTC time.
- Assignment config: a Statsig dynamic config (or experiment) per running experiment, cached in the host's config store; generated `schedule.json` / `page_assignment` values mirrored into it.

## Output

- The HTML for the assigned variant, with:
  - `<link rel="canonical" href="https://jordi.net<original path>">` on every page, so the `*.vercel.app` aliases and any variant URL point to jordi.net.
  - `Cache-Control` short enough that a window change or stop propagates within 10 minutes (`s-maxage=300` or less on experiment URLs).
  - `Vary` not set on `User-Agent` (content never depends on it).
- One `pageview` event to Statsig (fields in data-model.md, PageviewEvent).

## Rules

| Case | Behavior |
|------|----------|
| Path not in any running experiment | serve the site's normal page; still log a `pageview` if the client is a crawler or the referrer is an AI engine |
| `unit: page` | variant = `page_assignment[path]` |
| `unit: time_window` | variant = window whose `[start, end)` contains now |
| `unit: visitor` (engagement-only) | variant from Statsig hashing a per-request random unit; indexable HTML identical across arms |
| Statsig/config unreachable or times out (>50 ms) | serve `control`, log `assignment_fallback` |
| Experiment stopped | serve `control` (or the shipped variant once decided) |
| Variant URL requested after revert | `302` to the canonical path |
| Crawler request | same branch as any request; classification only sets `client_class`, `bot_name`, `bot_purpose` |

## Client classification (labels only)

1. Match `User-Agent` against the bot table in research.md §R6.
2. If matched, verify: IP in the engine's published JSON range, or reverse DNS + forward confirm where the engine documents it, or the host's verified-bot signal. Verified -> `crawler_verified`; else `crawler_unverified`.
3. Not matched -> `human` (generic bots like `curl` or headless browsers -> `crawler_unverified`, `bot_name: other`).

## Referrer classification

| Signal | `ref_class` |
|--------|-------------|
| host `chatgpt.com`, `chat.openai.com`, or `utm_source=chatgpt.com` | `ai:chatgpt` |
| host ends with `perplexity.ai` | `ai:perplexity` |
| host `claude.ai` | `ai:claude` |
| host `gemini.google.com` | `ai:gemini` |
| host `copilot.microsoft.com`, `edgeservices.bing.com` | `ai:copilot` |
| `google.*`, `bing.com`, `duckduckgo.com`, other engines | `search` |
| none | `direct` |
