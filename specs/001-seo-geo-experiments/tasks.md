---

description: "Task list for agent-run SEO/GEO experiments on jordi.net"
---

# Tasks: Agent-Run SEO/GEO Experiments on jordi.net

**Input**: Design documents from `specs/001-seo-geo-experiments/`

**Prerequisites**: plan.md, spec.md (approved by Jordi 2026-11-06 19:11), research.md, data-model.md, contracts/, quickstart.md

**Tests**: Included. The spec asks for them: Story 2's fixture tests for bad variants, the plan's middleware unit tests, and quickstart Part B validation.

**Organization**: Tasks are grouped by user story. Phase order follows dependencies, not only priority (see Dependencies).

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (US1 to US5 from spec.md)
- **Jordi does**: only Jordi performs the task. Agents prepare and verify, never perform.
- **Needs Jordi's deploy yes**: the task is part of the hosting move (Vercel project, production deploy, DNS). Nobody starts it before Jordi gives a separate deploy yes (OQ-2 decided A; the move itself is gated).
- **Live site**: merging the task's files to `master` changes what jordi.net serves. Today GitHub Pages serves it; after the move, Vercel does. Every live-site merge is done by Jordi under approval mode A (FR-012).

## Path Conventions

These are at the repository root, as in plan.md: `middleware.ts`, `lib/`, `scripts/`, `tests/`, `experiments/`, `x/`, `.github/`. `_config.yml` already keeps `specs/`, `experiments/`, `scripts/`, `middleware.ts`, `package*.json`, `tsconfig.json` and `vercel.json` out of the Jekyll build. `lib/` and `tests/` still need adding; that is T001.

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: tooling and repo guardrails. Nothing here changes the live site.

- [ ] T001 Add `lib/`, `tests/`, `Gemfile`, `Gemfile.lock`, `.gitleaks.toml` and `.github/` to the `exclude` list in `_config.yml`. Do not exclude `x/`, `robots.txt`, `llms.txt` or `about.md`. Confirm with `bundle exec jekyll build` that `_site/` holds only `index.html`, assets and `CNAME`.
- [ ] T002 [P] Create `package.json` and `tsconfig.json` at the repo root:
  - dependencies `@statsig/vercel-edge@3.33.8` and `@vercel/functions`;
  - devDependencies `vitest` and `html-validate`;
  - scripts `test` and `check`.
- [ ] T003 [P] Create `Gemfile` with the `github-pages` gem so Vercel builds the theme `jekyll-theme-midnight` the same way Pages does (V-3). Add `Gemfile.lock`.
- [ ] T004 [P] Create `vercel.json`:
  - framework `jekyll`;
  - `buildCommand` `bundle exec jekyll build`;
  - `outputDirectory` `_site`;
  - `Cache-Control: s-maxage=300` on HTML;
  - no `Vary: User-Agent`.
- [ ] T005 [P] Create `.claude/settings.json` deny rules:
  - `Read(~/.config/jordi-net/**)`
  - `Bash(cat ~/.config/jordi-net/*)`
  - `Bash(env)`
  - `Bash(printenv*)`
  - `Bash(vercel env pull*)`
- [ ] T006 [P] Create `.gitleaks.toml` and `scripts/hooks/pre-commit`, which runs `gitleaks protect --staged`. Add an install note in `experiments/README.md`.
- [ ] T007 [P] Create `.github/CODEOWNERS` naming Jordi as owner of these paths:
  - `experiments/allowed-domains.txt`
  - `experiments/approval-mode.yaml`
  - `middleware.ts`
  - `lib/`
  - `vercel.json`
  - `.github/`
  - `.claude/settings.json`
- [ ] T008 [P] Create `experiments/allowed-domains.txt`, seeded with OQ-6 default A: the seven employer domains linked from `index.html` today, plus Jordi's own profile domains once he lists them. Flag to Jordi that `index.html` links `https://pickwin.nmx`, which looks like a typo for `pickwin.mx`.
- [ ] T009 [P] Create `experiments/approval-mode.yaml` with `mode: A`, `decided_experiments: 0` and `mode_b_yes: null`. Its rules (data-model.md ApprovalMode):
  - OQ-1 is decided D.
  - The mode changes to `B` only in a PR Jordi merges that quotes his explicit yes.
  - `decided_experiments` never switches the mode.
- [ ] T010 [P] Create `scripts/lib/secrets.py`. It loads `~/.config/jordi-net/secrets.env` into the process environment, never prints values, and redacts any value that matches a secret pattern in script output.

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: assignment, classification and the manifest contract. This is code and tests only; nothing is deployed.

- [ ] T011 [P] Write fixture tests for manifest validation in `tests/checks/test_manifest.py`. Cover each rule 1 to 6 of `contracts/experiment-manifest.md`, including:
  - `unit` is one of `page`, `time_window`, `none`; "`visitor` only if every metric is `engagement`" (FR-004);
  - "exactly one `control`";
  - `planned_duration` is "at least 14 days for engagement, 28 days for search/AI metrics";
  - "at most one extension".
- [ ] T012 Implement `scripts/check/manifest.py` to pass T011. It validates fields, enforces one running experiment per URL (FR-006), and checks that the success rule, metric, minimum sample and duration are unchanged since approval.
- [ ] T013 [P] Write tests in `tests/middleware/bots.test.ts`, then implement `lib/bots.ts` with the bot table from research.md §R6.
  - Fields: name, operator, purpose `training` / `ai_search` / `user_fetch` / `search`.
  - Verification: IP range or reverse DNS.
  - Output: `client_class` `human` / `crawler_verified` / `crawler_unverified`.
- [ ] T014 [P] Write tests in `tests/middleware/referrer.test.ts`, then implement `lib/referrer.ts` with the referrer table of `contracts/assignment.md`. It reads both the referrer host and `utm_source=chatgpt.com`.
- [ ] T015 [P] Create `scripts/bots/refresh_ip_ranges.py`, which writes `lib/bot-ip-ranges.json` from the published JSON lists in research.md §R6. That data is public; the script only commits it.
- [ ] T016 Write tests in `tests/middleware/assign.test.ts`, then implement `lib/assign.ts`. It follows the rules table in `contracts/assignment.md`:
  - page unit and time-window unit;
  - kill flag serves `control`;
  - config unreachable, or slower than 50 ms, serves `control` and logs `assignment_fallback`;
  - a stopped experiment serves `control`.
  - The function never receives the user agent or IP.
- [ ] T017 [P] Implement `lib/statsig-log.ts`. It logs a `pageview` with the fields from data-model.md PageviewEvent:
  - unit ID is the page path or window id, never a visitor ID;
  - crawler events carry no `userAgent` (only `bot_name` metadata), so Statsig's bot filter keeps them;
  - no IP is sent;
  - events are flushed with `waitUntil`.
- [ ] T018 Implement `middleware.ts` (Vercel Routing Middleware). It must:
  - read `site_experiments` from Global Config;
  - call `lib/assign.ts` and rewrite to `/x/<experiment>/<variant>/`;
  - answer direct `/x/*` requests with a `302` to the canonical path;
  - call `lib/statsig-log.ts`;
  - support a `LOG_ONLY` mode that never rewrites.
  - Unit tests go in `tests/middleware/middleware.test.ts`.
- [ ] T019 [P] Add a canonical link `https://jordi.net{{ page.url }}` to every page through `_layouts/default.html`, copied from theme `midnight` with that one line added. Keep the rendered HTML otherwise identical; diff it against today's page. **Live site** once merged (Jordi merges).
- [ ] T020 [P] Write tests in `tests/schedule/test_generate.py`, then implement `scripts/schedule/generate.py`:
  - seeded paired switchback, so each pair of UTC days holds both arms in random order, written to `experiments/<id>/schedule.json`;
  - seeded page assignment written to `page_assignment`.
- [ ] T021 Implement `scripts/sync-config.py`. It builds the `site_experiments` JSON from the running manifests. `--dry-run` prints the JSON and a diff for Jordi. Without `--dry-run`, it writes through the Statsig Console API, which works only with a write-capable key (OQ-11 B). Under the OQ-11 default A, the script never writes.

**Checkpoint**: `npm test` and `pytest` pass. The middleware runs locally in log-only mode. Nothing is deployed.

---

## Phase 3: User Story 2 - Guardrails block anything harmful (Priority: P1)

**Goal**: no variant can reach Jordi or the site unless every check passes (FR-010 to FR-015).

**Independent Test**: run `scripts/check/run_all.py` on `tests/fixtures/bad-variants/*`. Every fixture fails with the expected check name (quickstart B7).

### Tests for User Story 2

- [ ] T022 [P] [US2] Create bad-variant fixtures in `tests/fixtures/bad-variants/`, one folder each:
  - impersonation;
  - unsourced claim about a named company;
  - link to a domain not on the allowed list;
  - `display:none` keyword block;
  - text positioned off-screen;
  - 20 new pages;
  - missing title;
  - a fake secret string;
  - a paragraph copied from a cited page;
  - an affiliate or ad link (Vercel Hobby forbids commercial use).
- [ ] T023 [P] [US2] Create good-variant fixtures in `tests/fixtures/good-variants/`: today's page, an ASCII header, and an enriched bio with Person schema.
- [ ] T024 [US2] Write `tests/checks/test_run_all.py`, asserting that every bad fixture fails with its named check and every good fixture passes.

### Implementation for User Story 2

- [ ] T025 [P] [US2] Implement `scripts/check/links.py`. Outbound domains must be in `experiments/allowed-domains.txt`. Links that are not editorial must have `rel` set to `nofollow` or `sponsored` (FR-013). It fails if `allowed-domains.txt` changed in the same PR as a variant.
- [ ] T026 [P] [US2] Implement `scripts/check/hidden.py`. It fails on CSS that hides text or links, off-screen positioning, or text the same colour as its background.
- [ ] T027 [P] [US2] Implement `scripts/check/render.py` with `html-validate`. It fails on a missing title or meta description, and on more than 5 new pages per experiment.
- [ ] T028 [P] [US2] Implement `scripts/check/parity.py`:
  - a static scan that fails if `middleware.ts` or `lib/` reads the user agent or IP anywhere except `lib/bots.ts` and `lib/statsig-log.ts`;
  - a request test that renders through the middleware locally with Googlebot and Chrome user agents and requires identical HTML.
- [ ] T029 [P] [US2] Implement `scripts/check/copied.py`, a shingle overlap check against every URL cited in the variant. Short quotations with attribution are allowed (FR-014).
- [ ] T030 [P] [US2] Implement `scripts/check/secrets.sh`, which runs `gitleaks detect` on the variant folder.
- [ ] T031 [P] [US2] Write the independent review agent definition in `.claude/agents/variant-reviewer.md`. It covers the policy list of research.md §R10 and nothing more. It returns JSON pass or fail with reasons, and it sees only the variant and the policy.
- [ ] T032 [US2] Implement `scripts/check/run_all.py`. It runs T012 and T025 to T031 and writes `experiments/<id>/checks/<timestamp>.json` (append-only). It exits non-zero on any fail and records the reason (FR-011).
- [ ] T033 [US2] Create `.github/workflows/checks.yml`, which runs `npm test`, `pytest` and `scripts/check/run_all.py` on every PR.
- [ ] T034 [US2] **Jordi does**: branch protection on `master` (PR plus Code Owner review required), secret scanning and push protection (quickstart §A6).

**Checkpoint**: US2 independent test passes. CI blocks a PR that holds a bad fixture as a real variant.

---

## Phase 4: User Story 4 - Jordi's one-time setup and the hosting move (Priority: P2)

**Goal**: accounts and keys exist, and the site runs on Vercel Hobby with the middleware in log-only mode. Mail is unaffected.

**Independent Test**: quickstart B1, B2, B4, B9 and B10 pass. `gitleaks detect` over the full history finds nothing (B8).

- [ ] T035 [US4] **Jordi does**: set up Statsig (quickstart §A1): project `jordi-net`, custom IDs `pagePath` and `windowId`, dynamic config `site_experiments`, a read-only Console API key in the local secret file. He also notes whether switchback is offered (V-2).
- [ ] T036 [P] [US4] **Jordi does**: set up Search Console and Bing (quickstart §A4): Domain property via a TXT record at DNSimple, a service account as a Restricted user, the Bing API key. The new TXT record is the only DNS change here; the SPF record stays as it is.
- [ ] T037 [P] [US4] **Jordi does**: create the local secret file `~/.config/jordi-net/secrets.env` (quickstart §A5).
- [ ] T038 [US4] **Jordi does, needs Jordi's deploy yes**:
  - create the Vercel Hobby project from the repo (§A2 steps 1 to 2);
  - connect the Statsig Marketplace integration in Connect mode (§A2 step 4);
  - add `STATSIG_SERVER_SECRET` and `STATSIG_CLIENT_KEY` as Sensitive (§A2 step 5);
  - set the AI Bots ruleset to Log (§A2 step 6).
  - This gives the `*.vercel.app` production URL; jordi.net is unchanged.
- [ ] T039 [US4] **Needs Jordi's deploy yes; Jordi merges; live site**: merge the Phase 1 and Phase 2 code (middleware with `LOG_ONLY=true`, `vercel.json`, `Gemfile`, layout with canonical) to `master`.
  - Vercel builds production on `*.vercel.app`.
  - GitHub Pages also rebuilds jordi.net from the same commit. Only the canonical line from T019 is visible there.
- [ ] T040 [US4] Run parity check B1: compare `https://jordi.net` (Pages) with the Vercel production URL and record the result in `experiments/setup-log.md` (V-3). This is a read-only check against public URLs.
- [ ] T041 [US4] Run checks B2, B4 and B10 against the Vercel production URL: crawler parity, fallback on a bad config (preview only) and the direct `/x/*` redirect. Record the results in `experiments/setup-log.md`.
- [ ] T042 [US4] **Jordi does, needs Jordi's deploy yes; live site**: switch DNS at DNSimple (quickstart §A3).
  - Before: lower the TTL and send a test email.
  - Change: replace the four apex A records with Vercel's A value, and point the `www` CNAME at Vercel's target. MX and SPF stay unchanged.
  - After: wait for "Valid Configuration" and send a second test email (B9).
  - Rollback is in §A3 step 8.
- [ ] T043 [US4] After T042, confirm that https://jordi.net and https://www.jordi.net come from Vercel, that the certificate is valid, and that `pageview` events with crawler labels arrive in Statsig. Record the results in `experiments/setup-log.md`.
- [ ] T044 [US4] **Jordi does; live site**: run kill-switch check B3. He sets `kill: true` in `site_experiments` in the Statsig console, the agent times propagation (10 minutes at most), then Jordi sets it back.
- [ ] T045 [US4] **Jordi does, after one week on Vercel**: unpublish GitHub Pages (quickstart §A3 step 9).

**Checkpoint**: jordi.net runs on Vercel in log-only mode. E0 observation data starts flowing.

---

## Phase 5: User Story 3 - Agent reads results into snapshots (Priority: P2)

**Goal**: dated snapshots from every source, following `contracts/metrics-snapshot.md` (FR-016 to FR-021).

**Independent Test**: after 7 days live, `scripts/read/run_all.py` writes one row per source per variant per day. Sources with nothing are marked `no_data: true`, not zero (quickstart B5 and B6).

### Tests for User Story 3

- [ ] T046 [P] [US3] Add recorded API fixtures (no secrets) in `tests/fixtures/api/`: Statsig Pulse or event counts, Search Console `searchanalytics.query` including `metadata.first_incomplete_date`, and Bing `GetPageStats`.
- [ ] T047 [P] [US3] Write `tests/read/test_snapshot.py`. It checks the completeness rules: drop days on or after `first_incomplete_date`, drop days that straddle a window change, `no_data` is not 0, and no rows hold raw IPs or full user agents.

### Implementation for User Story 3

- [ ] T048 [P] [US3] Implement `scripts/read/statsig.py`, which reads per-variant, per-day counts of human pageviews, engaged pageviews, crawler hits by bot and AI referrals. It uses the Console API (Pulse results, beta) with the read-only key. This is verification V-1.
- [ ] T049 [P] [US3] Implement `scripts/read/gsc.py`: `searchanalytics.query` with `dataState: final`, dimensions `date, page, query`, the service account from the secret file and scope `webmasters.readonly`.
- [ ] T050 [P] [US3] Implement `scripts/read/bing.py`: `GetPageStats` and `GetQueryPageStats` with the API key, in weekly buckets.
- [ ] T051 [P] [US3] Implement `scripts/read/gsc_ai_csv.py`. It imports any Generative AI performance CSV that Jordi drops into `~/Downloads/jordi-net-gsc-ai/` as source `gsc_ai_report`.
- [ ] T052 [US3] Implement `scripts/read/run_all.py`. It writes `experiments/<id>/snapshots/<YYYY-MM-DD>.json` and never edits earlier files (FR-020). It sets `status` and flags guardrail breaches.
- [ ] T053 [US3] Run the V-1 decision. If T048 cannot return per-variant counts on the Developer tier, write the finding to `experiments/setup-log.md` and raise the Vercel Pro fallback with Jordi under OQ-4. This is **Jordi's** budget decision. Do not upgrade.

**Checkpoint**: snapshots exist for the log-only period. V-1 is resolved or escalated.

---

## Phase 6: User Story 1 - One approved experiment end to end (Priority: P1) 🎯 MVP

**Goal**: the full loop of propose, check, ask, ship, read, decide and clean, first on E0 and then on E1. Both run under approval mode A (OQ-1 decided D: the first 3 experiments, E0 counted, ask every time).

**Independent Test**: E1 runs from proposal to decision. Every step has a record. Jordi's publish, start and ship approvals are recorded verbatim. The live site matched the approved variant on every sampled day (spec US1).

### Tests for User Story 1

- [ ] T054 [P] [US1] Write `tests/analyze/test_rules.py` covering:
  - the pre/post rule (E1);
  - the Beta-Binomial switchback rule, P ≥ 0.90 with at least 100 human pageviews per arm (E2);
  - the large-effect rule, at least 2x with at least 20 events (E3);
  - "insufficient data" below the minimum sample.

### Implementation for User Story 1

- [ ] T055 [US1] Implement `scripts/analyze/rules.py`. It applies the frozen `success_rule` from the manifest and writes `experiments/<id>/decision.md` with the numbers, the rule and data completeness (FR-021, FR-022).
- [ ] T056 [US1] Implement `scripts/approvals.py`. It appends to `experiments/<id>/approvals.md` with Jordi's verbatim words, the place and the time. While `approval-mode.yaml` has `mode: A`, it refuses to mark a variant publishable without an approval (FR-012a).
- [ ] T057 [US1] Write the loop runbook `experiments/README.md`, using the agent loop from plan.md. Mode A rules:
  - Every publish, start, extension and ship asks Jordi.
  - An emergency stop does not need his approval. Under OQ-11 default A, the agent asks Jordi to set `kill`.
  - Mode A stays after 3 experiments until Jordi's explicit yes to B.
- [ ] T058 [US1] Draft the E0 proposal `experiments/E0-llms-txt/experiment.yaml` (unit `none`) and its content files `llms.txt` and `about.md`, using only facts already on `index.html`. Run `scripts/check/run_all.py` and open the PR.
- [ ] T059 [US1] **Jordi does; live site**: approve and merge the E0 PR, then say "start E0". This is the publish and start approval; record it with T056.
- [ ] T060 [US1] Read E0 for 4 weeks, then write `experiments/E0-llms-txt/decision.md`. **Jordi approves** keeping or dropping the files. Dropping them is a **live site** merge, done by Jordi.
- [ ] T061 [US1] Draft the E1 proposal `experiments/E1-enriched-bio/`, including the time-window schedule, a 4-week pre-period from Search Console, and the variant `x/E1-enriched-bio/v1-bio/index.html` with Person schema. **Jordi supplies** the facts: roles, years, profile links, and whether to fix the `pickwin.nmx` link. Run the checks and open the PR with a power estimate.
- [ ] T062 [US1] **Jordi does; live site**: approve and merge the E1 PR, then say "start E1". Run `scripts/sync-config.py --dry-run`; **Jordi applies** the JSON in the Statsig console (OQ-11 default A). Record both approvals.
- [ ] T063 [US1] Run E1 for 6 weeks with daily reads (T052). If a guardrail is breached, ask Jordi to set `kill` and record the request.
- [ ] T064 [US1] At the E1 planned end, run `scripts/analyze/rules.py`. **Jordi approves** ship, revert or the one extension.
  - On ship: the enriched bio moves into `index.html`. **Live site**; Jordi merges.
  - Clean up the losing variant's files.
  - Increment `decided_experiments` in `experiments/approval-mode.yaml`, which stays `mode: A`.

**Checkpoint**: MVP done (SC-001). Two of the first three experiments under mode A are decided.

---

## Phase 7: User Story 5 - Auditable history (Priority: P3)

**Goal**: Jordi can see what was live, where, when and why (FR-027, SC-007).

**Independent Test**: after E0 and E1, `experiments/HISTORY.md` lists each experiment with its proposal, approvals, live dates, snapshots and decision. Each decision can be checked in under 5 minutes.

- [ ] T065 [P] [US5] Write `tests/history/test_history.py`. It checks that the history lists every experiment folder and the live date ranges from the approvals and decision files.
- [ ] T066 [US5] Implement `scripts/history.py`, which generates `experiments/HISTORY.md`. It runs in CI and fails if the file is out of date.

---

## Phase 8: Polish & Cross-Cutting Concerns

- [ ] T067 [P] Draft the E2 proposal `experiments/E2-ascii-header/`: paired daily switchback for 28 days. Use Statsig native switchback if V-2 found it available, else `scripts/schedule/generate.py`. Add the browser engagement events via `@statsig/js-client@3.33.8` with the client key and no cookie. E2 is the third experiment under mode A. **Jordi approves** publish and start; **live site**, Jordi merges.
- [ ] T068 [P] Prepare E3 only after **Jordi picks** the topics (OQ-7 default: no new topic pages). Until then, write nothing under `experiments/E3-*`.
- [ ] T069 Run every quickstart Part B scenario (B1 to B10) once more and record the results in `experiments/setup-log.md`.
- [ ] T070 After E0, E1 and E2 are decided, report to Jordi the results and the fact that mode A is still in force. Ask whether he wants mode B (OQ-1 D). Do not change `approval-mode.yaml` without his explicit, recorded yes in a PR he merges.

---

## Dependencies & Execution Order

### Phase dependencies

- Setup (Phase 1) has no dependencies.
- Foundational (Phase 2) depends on Setup.
- US2 guardrails (Phase 3) depend on Foundational. They need no hosting and can run before the move.
- US4 setup and move (Phase 4):
  - T035 to T037 can start at any time; they are Jordi's.
  - T038, T039 and T042 wait for Jordi's deploy yes.
  - T039 needs Phase 2 and Phase 3 complete.
- US3 reads (Phase 5) depend on T043 (live events) and T035 to T037 (keys).
- US1 MVP loop (Phase 6) depends on US2, US4 and US3.
- US5 history (Phase 7) depends on the first US1 records (after T060).
- Polish (Phase 8) depends on US1.

### Why the order differs from priority

US1 (P1) is the MVP goal. Its live loop cannot run without guardrails (US2, P1), the hosting move (US4, P2) and readouts (US3, P2), so those phases come first. Priorities are unchanged.

### Within each story

Tests come first and must fail before implementation. Manifest and classifier libraries come before the middleware. The middleware comes before the move. Reads come before decisions.

## Parallel Examples

- Phase 1: T002 to T010 all run in parallel (different files).
- Phase 2: T011, T013, T014, T015, T017, T019 and T020 run in parallel. T016 needs T013 and T014's interfaces only for types. T018 needs T016 and T017.
- US2: T022 and T023 run in parallel, then T025 to T031 in parallel, then T032, then T033.
- US3: T046 to T051 run in parallel, then T052.

## Implementation Strategy

1. **MVP first**: Phases 1 to 3 (code and guardrails) need no accounts and no deploy yes. Then Jordi does T035 to T037.
2. **Gate**: Jordi's deploy yes unlocks T038, T039 and T042. Without it, everything stops at Phase 3 plus his Statsig and Search Console setup.
3. **Incremental**:
   - log-only on Vercel (E0 data);
   - reads (US3);
   - E0 decision;
   - E1 end to end (MVP done);
   - history;
   - E2.
4. **Approval mode**: A throughout this task list. E0, E1 and E2 are the first 3 experiments. B needs Jordi's later explicit yes (T070).

## Tasks needing Jordi or touching the live site

| Task | Who / gate | Live site |
|------|------------|-----------|
| T019 | Jordi merges | yes (canonical line on Pages, later on Vercel) |
| T034 | Jordi does (GitHub settings) | no |
| T035 | Jordi does (Statsig) | no |
| T036 | Jordi does (Search Console, Bing, a TXT record at DNSimple) | no (DNS TXT only) |
| T037 | Jordi does (local secret file) | no |
| T038 | Jordi does, needs Jordi's deploy yes (Vercel project) | no (`*.vercel.app` only) |
| T039 | needs Jordi's deploy yes, Jordi merges | yes |
| T042 | Jordi does, needs Jordi's deploy yes (DNS switch) | yes |
| T044 | Jordi does (kill-switch check) | yes, briefly |
| T045 | Jordi does (unpublish Pages) | no (after the move) |
| T053 | Jordi decides budget (OQ-4) if V-1 fails | no |
| T059 | Jordi approves and merges E0, says start | yes |
| T060 | Jordi approves the E0 decision | yes if files are dropped |
| T061 | Jordi supplies bio facts | no |
| T062 | Jordi approves and merges E1, says start, applies config | yes |
| T064 | Jordi approves the E1 decision, merges on ship | yes |
| T067 | Jordi approves and merges E2, says start | yes |
| T068 | Jordi picks topics (OQ-7) | no |
| T070 | Jordi decides mode B (OQ-1 D) | no |
