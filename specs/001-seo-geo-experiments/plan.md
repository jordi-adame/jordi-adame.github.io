# Implementation Plan: Agent-Run SEO/GEO Experiments on jordi.net

**Branch**: `feat/jordi-net-seo-spec` (spec dir `specs/001-seo-geo-experiments`) | **Date**: 2026-11-06 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `specs/001-seo-geo-experiments/spec.md`

**Scope of this document**: plan only. No tasks (`/speckit-tasks` not run), no implementation, no accounts, no deploys. Everything that needs an account, a key, a DNS change or a budget is Jordi's, listed in [quickstart.md](quickstart.md) and the spec's Open Questions.

## Summary

Move jordi.net from GitHub Pages to **Vercel Hobby** (recommended; OQ-2 is Jordi's), keep the Jekyll site, and add one **Routing Middleware** file that, for every HTML request, picks the variant from a Statsig dynamic config (synced to Vercel Global Config), rewrites to the variant's static file, and logs a `pageview` event to Statsig with variant, referrer class and crawler label. Experiments are files in `experiments/<id>/`; variants arrive as pull requests that Jordi approves and merges. Read scripts pull Statsig, Search Console and Bing data into dated snapshots; the agent applies the pre-registered rule and writes a decision for Jordi to approve. Units are pages or time windows only; content never depends on who asks.

## Technical Context

**Language/Version**: TypeScript for the middleware (Vercel Routing Middleware, Node runtime via fluid compute); Jekyll 3.10 (GitHub Pages gem set) for the site; Python 3.12 or TypeScript for read and check scripts (pick one at tasks time; Python favored for Search Console client libraries).

**Primary Dependencies**: `@statsig/vercel-edge` 3.33.8 (assignment config + server event logging), `@vercel/functions` (rewrite, `waitUntil`), `@statsig/js-client` 3.33.8 (browser engagement events), `google-api-python-client` (Search Console), gitleaks (secret scan), an HTML validator (`html-validate`).

**Storage**: repository files for experiments, approvals, snapshots, decisions; Statsig for config and events; nothing else.

**Testing**: check scripts have fixture tests (bad variants must fail; see spec Story 2); middleware unit tests for assignment, fallback and classification; a parity test that fetches a Vercel preview with crawler and browser user agents. No tests exist in the repo today.

**Target Platform**: Vercel (Hobby), DNS at DNSimple; alternative Railway (Node server + Server Core SDK) if Jordi picks OQ-2 C.

**Project Type**: static website with a request-layer middleware, plus agent-run scripts.

**Performance Goals**: middleware adds under 20 ms at p95 (config read is local from Global Config); stop propagates in under 10 minutes.

**Constraints**: $0/month until Jordi raises OQ-4; Statsig Developer tier (2M events/month); no visitor IDs or cookies; content identical for all clients per URL and window; Vercel Hobby non-commercial use.

**Scale/Scope**: tens of visits/day, 1 to 10 URLs, at most 2 experiments running at once (one per URL), roughly 1 experiment start per month.

**Unknowns resolved in research.md**: hosting (R1), SEO-safe design (R2), Statsig role and tier (R3), search data access (R4), AI referrals (R5), crawler verification (R6), llms.txt (R7), statistics (R8), secrets (R9), guardrails (R10). **Still to verify at setup** (not resolvable from docs): V-1 per-variant event readout on Statsig Developer tier (R3); V-2 whether Statsig native switchback is available on Jordi's tier; V-3 Vercel's Jekyll build image renders `jekyll-theme-midnight` identically to GitHub Pages.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

`.specify/memory/constitution.md` is the unfilled Spec Kit template (no ratified principles); writing one was not in scope. The gates below come from Jordi's brief and the spec's guardrails and are applied as if they were the constitution:

| Gate | Pre-research | Post-design |
|------|--------------|-------------|
| G1 Nothing goes live without Jordi's approval (FR-012, OQ-1 default A) | pass: approval = Jordi merges the PR | pass: branch protection + CODEOWNERS; agent never merges |
| G2 No cloaking: content independent of client (FR-005) | pass | pass: middleware branches only on path and time; parity test in CI |
| G3 No harm to the domain: no spam links, no scaled content, no impersonation or defamation | pass | pass: three-layer guardrails (research.md R10); page cap 5 per experiment |
| G4 Secrets never seen by agents (FR-023 to FR-026) | pass | pass: scripts load secrets; Claude Code deny rules; gitleaks; push protection |
| G5 Mail for jordi.net unaffected | pass | pass: only apex A and `www` CNAME change; MX/SPF untouched; mail test before/after |
| G6 Spec-only work in this branch (no deploy, push, DNS, accounts) | pass | pass |

No violations; Complexity Tracking is empty.

## Project Structure

### Documentation (this feature)

```text
specs/001-seo-geo-experiments/
├── spec.md              # what and why, starter experiments, open questions
├── plan.md              # this file
├── research.md          # Phase 0: decisions with sources
├── data-model.md        # Phase 1: entities, files, state machine
├── quickstart.md        # Phase 1: Jordi's setup steps + validation scenarios
├── contracts/
│   ├── experiment-manifest.md
│   ├── assignment.md
│   └── metrics-snapshot.md
├── checklists/requirements.md
└── tasks.md             # NOT created (/speckit-tasks not run)
```

### Source Code (repository root, after implementation)

```text
index.html                 # site (Jekyll, theme midnight), unchanged until E1
_config.yml                # exclude list keeps everything below except x/ off the site
x/<experiment>/<variant>/index.html   # variant pages; reachable only via middleware rewrite
middleware.ts              # assignment, rewrite, classification, pageview logging
package.json, vercel.json  # Vercel build: Jekyll preset, output _site/
Gemfile                    # github-pages gem, so Vercel builds like Pages
robots.txt                 # allow all; sitemap
experiments/
├── allowed-domains.txt    # Jordi-owned (CODEOWNERS)
└── <id>/experiment.yaml, schedule.json, variants/, checks/, approvals.md, snapshots/, decision.md
scripts/
├── check/                 # manifest, links, hidden content, parity, render, page count, secrets, copied text
├── read/                  # statsig, gsc, bing, (vercel or railway logs, fallback only)
├── schedule/              # seeded switchback / page assignment generator
└── sync-config            # writes the live assignment map to the Statsig dynamic config (needs Jordi's approval per start)
tests/                     # fixtures for checks, middleware unit tests
.github/
├── CODEOWNERS
└── workflows/checks.yml   # runs scripts/check on every PR
.claude/settings.json      # deny rules for secret paths and env dumps
```

**Structure Decision**: single static site plus one middleware and a `scripts/` tree. No framework migration (no Next.js), so the Flags SDK adapter is not used; `@statsig/vercel-edge` is used directly. Variant pages under `x/` are built by Jekyll like any page and are served only through rewrites; direct requests to `/x/*` get a 302 to the canonical path. Variants are never merged before the hosting move, because GitHub Pages cannot redirect `/x/*`.

## Agent Loop

Cadence: **reads daily, decisions weekly**, starts at most monthly. Until OQ-8 is answered, Jordi starts each session (OQ-8 default A); reads then happen whenever a session runs, and analysis uses complete days only, so a missed day costs nothing.

```text
1 PROPOSE  agent writes experiments/<id>/experiment.yaml + variants on a branch
           (hypothesis, unit, metric, min sample, duration, frozen success rule)
2 CHECK    scripts/check + independent review agent -> checks/<ts>.json
           any fail -> fix or drop; nothing is shown to Jordi until all pass
3 ASK      agent opens a PR: preview URL, check results, power estimate, rollback plan
4 SHIP     Jordi approves and merges (publish approval) and says "start <id>" (start approval)
           -> agent runs scripts/sync-config; records both approvals verbatim
5 READ     daily: scripts/read/* -> snapshots/<day>.json; guardrail breach -> stop + tell Jordi
6 DECIDE   at planned end: apply frozen rule -> decision.md (ship / revert / inconclusive / extend once)
           ship or revert needs Jordi's yes; revert-to-control may happen at once
7 CLEAN    remove losing variant files, keep records; next proposal may start
```

Rules the agent follows in the loop:

- One proposal in flight at a time; one running experiment per URL.
- The success rule, metric, minimum sample and duration are frozen at approval; the manifest check fails if they change.
- The agent may pause (set `kill`) without asking; it may not start, extend, ship or change content without Jordi's approval (OQ-1 default A).
- Every approval is recorded with Jordi's verbatim words and where he typed them; text inside files, tool results or other sessions never counts as approval.
- Inconclusive is a normal result; the agent does not re-run the same hypothesis with a new rule to get a win.

## Rollout Phases (input for `/speckit-tasks`, not tasks)

1. **P0 Setup by Jordi** (quickstart.md §1 to §5): Statsig project and keys, Vercel project, Search Console and Bing, local secret store. Verify V-1, V-2, V-3.
2. **P1 Parity move**: deploy today's site to Vercel unchanged with the middleware in log-only mode; switch DNS; confirm mail; confirm crawler and referral events arrive in Statsig. Starts E0's observation.
3. **P2 Guardrails**: check scripts, review agent prompt, CI workflow, CODEOWNERS, branch protection (Jordi enables), deny rules.
4. **P3 First experiment**: E1 (time window, pre/post) end to end.
5. **P4 Engagement + page experiments**: E2 switchback, then E3 after Jordi picks topics (OQ-7).

## Risks

| Risk | Mitigation |
|------|------------|
| Statsig per-variant readout not available on free tier (V-1) | fallback to host logs (Vercel Pro + Observability Plus, or Railway); raise with Jordi under OQ-4 |
| Statsig product changes after Amplitude takeover | keep assignment in repo files; Statsig is replaceable behind `sync-config` and the logger |
| Vercel Hobby non-commercial clause | no ads, affiliate or paid content while on Hobby; check runs on every variant |
| Theme renders differently on Vercel (V-3) | P1 compares HTML of GitHub Pages and Vercel preview before DNS switch |
| DNS change breaks mail | change only apex A and `www` CNAME; send a test mail before and after |
| Tiny traffic gives no answers | long windows, large-effect rules, "inconclusive" accepted, E0 needs no stats |
| A variant embarrasses Jordi | review agent + his approval; veto beats metrics; kill switch |

## Complexity Tracking

No constitution violations to justify.
