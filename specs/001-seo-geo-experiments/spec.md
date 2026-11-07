# Feature Specification: Agent-Run SEO/GEO Experiments on jordi.net

**Feature Branch**: `feat/jordi-net-seo-spec` (spec directory `specs/001-seo-geo-experiments`; no numbered branch was created)

**Created**: 2026-11-06

**Status**: Approved by Jordi 2026-11-06 19:11 (specified, clarified, planned, tasks written; implementation not started)

**Input**: Jordi's goal: "create 'random' versions of my website and only optimize for seo/geo traffic or engagement". Variants can be anything: "a funny ascii animation, content about x topic, links to other sites, idk". "The idea is that you, an agent, will iterate on what to publish on jordi.net." Use Statsig experiments. The site deploys from GitHub today; Jordi is open to Vercel or Railway.

## Context

jordi.net today is one static page (473 bytes: a name and a list of seven past companies), served by GitHub Pages with the Jekyll theme `midnight`. The domain's DNS is at DNSimple and also carries Google Workspace mail records. Traffic is assumed to be very small (tens of visits a day at most), which shapes every measurement decision below.

The feature is a *setup*, not a single page: a way for an AI agent to propose a variant of the site, get it published, measure what search engines, AI answer engines and human visitors did with it, and decide what to keep, all inside guardrails that protect Jordi's name and domain.

Terms used throughout:

- **Experiment**: one pre-registered test with a hypothesis, variants, a primary metric, a duration and a success rule.
- **Variant**: one version of the content under test. "Control" is the current content.
- **Unit**: what gets randomized. Allowed units are a *page* (a URL path) or a *time window*. A single visitor is not an allowed unit for search or AI metrics (see FR-004).
- **Crawler**: any automated client: search engine bots (Googlebot, Bingbot) and AI bots (training crawlers, AI search crawlers and user-triggered AI fetchers).
- **AI referral**: a human visit that arrives from an AI answer engine (ChatGPT, Perplexity, Claude, Gemini, Copilot, Google AI features).
- **Experiment agent**: the AI agent (a Claude Code session) that proposes, ships, reads and decides.
- **Publishing approval**: Jordi's yes for a specific variant to go live.

## Clarifications

### Session 2026-11-06

- Q: What is the unit of randomization for experiments whose metric comes from search engines or AI engines? → A: A page (URL path) or a time window, never a single visitor. Every client, human or crawler, gets the same content for the same URL at the same time.
- Q: Do crawlers ever get different content from humans? → A: No. Content never depends on user agent, IP or bot status. Bot detection is used only to label requests in measurement, never to choose content.
- Q: How much may the agent publish without asking? → A: Nothing, under mode A. Every variant needs Jordi's publishing approval before it goes live, and every experiment start, extension and ship needs it too; an emergency stop does not. The agent may draft, lint and preview freely. Jordi decided OQ-1 (option D) on 2026-11-06: mode A for the first 3 experiments and afterwards until his explicit yes to mode B.
- Q: Where do secrets live? → A: Only in the hosting provider's encrypted environment variables and in Jordi's local secret store (outside the repo). Jordi places every secret himself. Agents never print, log, echo or commit secret values; they may reference secrets by name only.
- Q: Do human visitors get a tracking cookie or persistent ID? → A: No. Page-level and time-window units need no visitor ID, so measurement is per pageview and anonymous. This avoids a consent banner and keeps the site as light as it is today.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Agent ships one approved experiment end to end (Priority: P1)

The experiment agent writes an experiment proposal (hypothesis, variants, unit, primary metric, duration, success rule). It builds the variants, runs the guardrail checks and shows Jordi a preview link and the check results. Jordi approves. The experiment goes live for crawlers and humans alike, the agent reads results on a fixed cadence, and at the end it writes a decision (ship, revert, or extend once) with the numbers behind it. Jordi approves the decision before it takes effect.

**Why this priority**: This is the whole point of the feature. Without one full loop nothing else matters.

**Independent Test**: Run starter experiment E1 (see Starter Experiments) from proposal to decision and check that every step left a record, that Jordi approved the publish and the decision, and that the live site matched the approved variant during the run.

**Acceptance Scenarios**:

1. **Given** a proposal with all required fields, **When** the agent submits it, **Then** Jordi sees the hypothesis, variants, unit, metric, duration, success rule, guardrail results and a preview, and nothing is live yet.
2. **Given** Jordi approved a variant, **When** the experiment starts, **Then** every request for the experiment's URLs during each time window gets the variant assigned to that URL and window, whether the client is a person or a crawler.
3. **Given** the experiment reached its planned end, **When** the agent evaluates it, **Then** it applies the pre-registered success rule unchanged and writes the decision with the data behind it.
4. **Given** a proposal is missing a success rule or duration, **When** the agent tries to submit it, **Then** it is rejected before Jordi is asked.

---

### User Story 2 - Guardrails block anything that could harm Jordi or the domain (Priority: P1)

Before any variant is shown to Jordi for approval, automated checks reject content that impersonates anyone, makes claims about real people or companies that are not sourced, links to sites outside the allowed list, hides text from humans, serves crawlers something different from humans, adds pages in bulk, or breaks the page. Jordi can stop any experiment at once and the site falls back to the control.

**Why this priority**: The site carries Jordi's name and his mail domain. One bad publish costs more than any traffic gain.

**Independent Test**: Feed the checks a set of deliberately bad variants (impersonation, unsourced claim about a named company, link to a non-allowed domain, `display:none` keyword block, 20 new pages at once) and confirm every one is blocked with a reason. Trigger the stop and confirm the control is live within the target time.

**Acceptance Scenarios**:

1. **Given** a variant with an outbound link to a domain not on the allowed list, **When** checks run, **Then** the variant is blocked and the reason names the domain.
2. **Given** a running experiment, **When** Jordi or the agent triggers stop, **Then** all experiment URLs serve the control within 10 minutes and the stop is recorded.
3. **Given** a variant whose rendered text differs between a crawler request and a human request, **When** checks run, **Then** the variant is blocked as cloaking.
4. **Given** a variant passed checks, **When** Jordi has not approved it, **Then** it is never served on jordi.net.

---

### User Story 3 - Agent reads results from every source into one record (Priority: P2)

On a fixed cadence the agent collects, per experiment and per variant: search impressions, clicks and position; AI referrals by engine; crawler fetches by bot and purpose (training, AI search, user-triggered); and human engagement events. It stores them as dated snapshots and reports whether the data is enough to decide.

**Why this priority**: Decisions need numbers; but a first experiment can be read by hand, so this follows Stories 1 and 2.

**Independent Test**: With one experiment live for 7 days, run the read step and check the snapshot holds a row per source per variant per day, with sources that had no data marked "no data" rather than zero.

**Acceptance Scenarios**:

1. **Given** search data arrives with a delay, **When** the agent reads, **Then** it records the latest complete day and does not treat missing recent days as zero.
2. **Given** traffic is below the experiment's minimum sample, **When** the agent evaluates, **Then** it reports "insufficient data" and does not declare a winner.
3. **Given** a request from a client claiming to be an AI crawler, **When** it is counted, **Then** it is labeled verified or unverified according to the engine's published verification method.

---

### User Story 4 - Jordi sets up accounts and keys once, by himself (Priority: P2)

Jordi follows a step-by-step guide to create the experiment platform project and keys, the hosting project (if moving), the search-console access and the DNS change. He places each secret where the guide says. No agent ever sees a secret value.

**Why this priority**: Nothing runs without it, but it is a one-time task Jordi does on his own schedule.

**Independent Test**: A reviewer follows the guide on a clean account and ends with a working preview deployment, keys stored only in the documented places, and no secret value in the repository history.

**Acceptance Scenarios**:

1. **Given** the guide, **When** Jordi completes it, **Then** every secret is in exactly one documented place and the repo contains only secret *names*.
2. **Given** the DNS change, **When** it is made, **Then** mail records (MX, SPF) are unchanged and mail still arrives.

---

### User Story 5 - Experiment history Jordi can audit (Priority: P3)

Every experiment, variant, approval, check result, data snapshot and decision is kept as a file in the repository so Jordi can see what was published on his domain, when, and why.

**Why this priority**: Useful for trust and for the agent's own memory across sessions, but the loop works without a polished history.

**Independent Test**: After two experiments, list the history and confirm each has its proposal, approvals, live dates, snapshots and decision.

**Acceptance Scenarios**:

1. **Given** a finished experiment, **When** Jordi opens its record, **Then** he sees what was live on which URL on which dates.

### Edge Cases

- Traffic is near zero for weeks: the agent reports "insufficient data", may extend an experiment once by its planned duration, then must close it as inconclusive.
- Search data is delayed by days: reads use the latest complete day; the experiment end is measured on complete days only.
- A variant gets indexed and the experiment is reverted: the URL keeps serving content (the control or the winning variant), never a 404, and variant-only URLs redirect temporarily to the canonical page.
- A crawler spoofs an AI or search user agent: the hit is counted as unverified and kept apart from verified hits.
- The experiment platform is unreachable at request time: every URL serves the control; the outage is logged.
- Two experiments want the same URL: not allowed; one URL belongs to at most one running experiment.
- A secret leaks (printed in a log, committed): treat as an incident: Jordi rotates the key; the agent stops all experiments until he confirms.
- A variant tops traffic but hurts reputation (for example an embarrassing joke): Jordi can veto any result; a veto wins over any metric.
- An AI engine cites the site with wrong facts: recorded as an observation; content corrections go through the normal approval path.

## Requirements *(mandatory)*

### Functional Requirements

**Experiments and assignment**

- **FR-001**: The system MUST let the agent define an experiment with: id, hypothesis, variants (including control), unit (page or time window), target URLs, primary metric, secondary and guardrail metrics, minimum sample, planned duration, and a success rule written before launch.
- **FR-002**: The system MUST reject an experiment definition missing any field in FR-001.
- **FR-003**: The system MUST assign variants on the server or edge, before the page is sent, so crawlers that do not run scripts see the assigned variant.
- **FR-004**: For any experiment whose primary metric comes from search or AI engines, the unit MUST be a page or a time window. Per-visitor assignment is allowed only for experiments whose metrics are human engagement alone, and then only for elements that do not change the page's indexable text or links.
- **FR-005**: Content MUST NOT depend on user agent, IP address or bot status. Every client gets the same content for the same URL in the same time window.
- **FR-006**: A URL MUST belong to at most one running experiment.
- **FR-007**: When a variant lives on its own URL, that URL MUST declare the original page as its canonical, and any redirect between variant and original MUST be temporary.
- **FR-008**: If the experiment platform cannot be reached, every URL MUST serve the control.
- **FR-009**: Jordi MUST be able to stop any experiment; all its URLs MUST serve the control within 10 minutes. The agent MUST be able to stop too, without asking, through `scripts/kill` and its dedicated Statsig write key (OQ-11 decided C); it records every stop and tells Jordi. The agent MUST NOT use that key for anything else: config changes and starts stay with Jordi.

**Guardrails**

- **FR-010**: Before Jordi is asked to approve, every variant MUST pass automated checks for: impersonation of any person or organization; claims about named real people or companies without a cited source; outbound links outside an allowed-domain list Jordi controls; hidden text or links; different content for crawlers and humans; more than a set number of new pages per experiment (default 5); broken page (invalid markup, missing title, failed render); and secret-like strings.
- **FR-011**: A variant that fails any check MUST NOT be offered for approval or published. The failure reason MUST be recorded.
- **FR-012**: Nothing MUST go live on jordi.net without Jordi's publishing approval for that exact variant. Experiment start, extension and ship decisions also need his approval; an emergency stop does not. Approval mode (OQ-1, decided: D): the first 3 experiments run on jordi.net (E0 counts) use mode A: every variant publish, experiment start, extension and ship needs Jordi's approval. Mode A stays in force after those 3 until Jordi gives a separate, explicit, recorded yes to move to mode B; no count, metric or elapsed time moves it on its own. Under mode B (only after that yes), variants in the pre-approved classes of OQ-1 B publish with a notice; new topics, outbound links and new pages still need approval.
- **FR-012a**: The system MUST record the approval mode in force and the count of decided experiments, and MUST refuse to publish without an approval while the mode is A.
- **FR-013**: The system MUST NOT add outbound links that are paid, reciprocal-for-exchange, or placed only to pass ranking signals; any outbound link that is not editorial MUST carry the appropriate rel qualifier (`nofollow` or `sponsored`).
- **FR-014**: Variants MUST NOT copy text from other sites beyond short quotations with attribution.
- **FR-015**: The site MUST keep identifying itself as Jordi Adame's personal site on the home page in every variant, unless Jordi decides otherwise (OQ-5).

**Measurement**

- **FR-016**: The system MUST record, per pageview: URL, variant, time window, referrer category (search engine, AI engine by name, social, direct, other), and whether the client is a human, a verified crawler or an unverified crawler, with the crawler's name and purpose.
- **FR-017**: The system MUST record human engagement events (link click, scroll depth reached, time on page over a threshold) without a persistent visitor ID.
- **FR-018**: The agent MUST collect search impressions, clicks, CTR and average position per URL and day from the search engines' own reporting.
- **FR-019**: The agent MUST collect crawler fetches per URL, bot and day from request logs.
- **FR-020**: The agent MUST store each read as a dated snapshot and keep earlier snapshots unchanged.
- **FR-021**: The agent MUST apply the pre-registered success rule unchanged; it MUST report "insufficient data" when the minimum sample is not reached.
- **FR-022**: Each experiment MAY be extended once, by at most its planned duration; after that it MUST close as ship, revert or inconclusive.

**Secrets**

- **FR-023**: Secret values MUST live only in the hosting provider's encrypted environment settings and Jordi's local secret store; the repository MUST hold secret names only.
- **FR-024**: Agents MUST NOT print, log, echo, paste or commit secret values, and MUST NOT ask Jordi to paste a secret into a chat.
- **FR-025**: The repository MUST block commits containing secret-like strings.
- **FR-026**: Keys MUST be scoped to the minimum needed: a browser-safe key in pages, a server key only on the server, and read-only keys for reporting where the provider offers them.

**Records**

- **FR-027**: Every experiment, check result, approval, snapshot and decision MUST be kept as a file in the repository, excluded from the published site.

### Key Entities

- **Experiment**: id, hypothesis, unit, target URLs, metrics, minimum sample, planned duration, success rule, status (draft, checked, approved, running, stopped, decided), dates.
- **Variant**: belongs to one experiment; id, description, content reference, check results, approval.
- **Assignment window**: for time-window units: start, end, variant live in that window.
- **Check result**: variant, check name, pass/fail, reason.
- **Approval**: who (Jordi), what (variant publish, start, ship, stop), when, his words.
- **Observation snapshot**: experiment, variant, source, day, metric values, data completeness.
- **Decision**: experiment, outcome (ship, revert, inconclusive, extend), numbers used, rule applied, Jordi's approval.
- **Allowed-domain list**: domains Jordi allows outbound links to.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: One experiment completes the full loop (proposal, checks, approval, live, reads, decision, approval) within 8 weeks of Jordi finishing setup.
- **SC-002**: 100% of variants served on jordi.net have a recorded approval and passing check results.
- **SC-003**: 0 secret values appear in the repository history, chat transcripts or logs written by agents.
- **SC-004**: Stopping an experiment restores the control on all its URLs within 10 minutes.
- **SC-005**: For every crawler request during an experiment, the content served equals the content a human got for the same URL and window (checked by sampling at least 10 paired requests per experiment).
- **SC-006**: Mail delivery for jordi.net is uninterrupted through any hosting move (no bounced test message before, during or after).
- **SC-007**: Each decision lists the metric values, the rule and the data completeness behind it; Jordi can verify any decision in under 5 minutes from its record.
- **SC-008**: The site receives no manual action or security issue in search console during the program.

## Starter Experiments

All four use page or time-window units, keep the bio on the home page, and need Jordi's approval for each variant. Thresholds are deliberately simple because traffic is tiny; see research.md §R8.

| Id | Hypothesis | Variants | Unit | Primary metric | Duration | Success rule |
|----|------------|----------|------|----------------|----------|--------------|
| E0 | AI crawlers already fetch the site, and a plain-text profile file is fetched at all (no major engine says it reads `llms.txt`; Google says such files are unnecessary, so this is a near-free probe). | Add `/llms.txt` and `/about.md` (facts already on the page) vs. nothing. Observational, no split. | none (probe) | Verified AI-crawler fetches of the new files and of `/` | 4 weeks | Keep the files if any verified AI crawler fetches them; otherwise keep or drop at Jordi's choice (cost is near zero). |
| E1 | A fuller, factual home page (short bio, one line per company with role and years, Person structured data, links to Jordi's own profiles) raises search impressions for name queries and AI-engine citations. | Control: today's list. Treatment: enriched bio. | time window (4 weeks control baseline already in data, then 6 weeks treatment) | Search impressions on `/` for queries containing "jordi" or "adame" | 6 weeks after launch, plus 4-week pre-period | Ship if the weekly impressions median in treatment exceeds the highest pre-period week and no guardrail metric drops; inconclusive if fewer than 50 total impressions. |
| E2 | A small ASCII animation header makes human visitors stay and click more, without hurting search. | Control: static header. Treatment: animated ASCII header; indexable text unchanged. | time window, daily switchback (alternate days, order randomized in pairs) | Share of human pageviews with an engagement event (outbound click or 15 s on page) | 28 days (14 per arm), extend once | Ship if the treatment share is higher with at least 90% probability (Bayesian) on at least 100 human pageviews per arm and search impressions do not fall; otherwise inconclusive. |
| E3 | Answer-first pages (a one-paragraph direct answer, a fact list, cited sources) get more AI-crawler fetches and AI referrals than narrative essays on the same kind of topic. | 4 new topic pages Jordi picks, randomly split 2 and 2 between "answer-first" and "narrative" formats. | page | AI referrals + verified AI-search and user-triggered crawler fetches per page | 8 weeks | Directional only at this traffic: adopt answer-first for future pages if its pages have at least 2x the fetches and referrals of narrative pages with at least 20 events total; else inconclusive. |

## Open Questions for Jordi

These are Jordi's decisions. Each has options and a recommendation. OQ-1 and OQ-2 were decided by Jordi on 2026-11-06 (main chat menu, 19:12), and OQ-11 at 20:04; OQ-3 to OQ-10 stay on the default in the last column, which applies until he answers.

| Id | Question | Options | Recommendation | Decision / default until decided |
|----|----------|---------|----------------|-----------------------|
| OQ-1 | How much may an agent publish later without asking each time? | A: ask for every variant, start and ship. B: pre-approved classes publish with a notice (layout and visual tweaks, rewording of existing bio facts, ASCII art); new topics, outbound links and new pages still ask. C: anything that passes the guardrails publishes, with a weekly digest and veto. D: ramp: A for the first 3 experiments, then B after Jordi's explicit yes. | D | **Decided 2026-11-06: D.** Mode A for the first 3 experiments; moving to B needs Jordi's explicit yes later (FR-012). |
| OQ-2 | Which host? | A: Vercel Hobby (free; personal non-commercial use). B: Vercel Pro (about $20/month; longer logs, log drains). C: Railway (about $5/month plus usage; full logs, a server to maintain). D: stay on GitHub Pages (time-window experiments by scheduled rebuilds only; no request logs, so no AI-crawler data). | A, move to B only if log retention proves too short (see research.md §R1) | **Decided 2026-11-06: A, Vercel Hobby.** The move itself (Vercel project, DNS) waits for Jordi's separate deploy yes. Until then the site stays on GitHub Pages. |
| OQ-3 | Which accounts own the services? | A: new Statsig, Vercel/Railway projects under Jordi's personal accounts. B: under an existing company or team account. | A | none created |
| OQ-4 | Monthly budget ceiling? | A: $0 (free tiers only). B: up to $25. C: up to $50. | A to start; B if logs or events need a paid tier | $0 |
| OQ-5 | How far may variants move away from "Jordi's personal site"? | A: bio always on `/`; variants add around it and on new paths. B: `/` may change freely; bio kept on `/about`. C: anything anywhere. | A | A |
| OQ-6 | Which outbound links are allowed? | A: only Jordi's own profiles and past employers already listed. B: a list Jordi curates (A plus sites he names). C: any reputable site the agent cites, `nofollow` by default. | B | A |
| OQ-7 | Which topics may new pages cover (E3 and later)? | A: Jordi names the topics. B: agent proposes, Jordi picks each. C: agent picks within a topic list Jordi approves once. | B | no new topic pages |
| OQ-8 | Where and how often does the agent run? | A: Jordi starts a session when he wants (manual). B: a scheduled cloud routine weekly. C: a scheduled GitHub Action daily for reads only, weekly agent session for decisions. | C once OQ-1 allows; A until then | A |
| OQ-9 | Statsig was acquired by Amplitude on 2026-05-05 (research.md §R3). Event data (paths, referrer class, bot names; no IPs or visitor IDs) would go to Amplitude-owned infrastructure. Use it as planned? | A: yes, as asked. B: yes, but review again if Amplitude announces a Statsig sunset or pricing change. C: use another experiment tool. | B (assignment lives in repo files, so Statsig stays replaceable) | A, as Jordi asked |
| OQ-10 | Which AI crawlers may fetch the site? | A: all, as today (no robots.txt). B: AI-search and user-triggered bots yes, training crawlers (GPTBot, ClaudeBot, CCBot, meta-externalagent) no. C: none. | A (GEO needs the fetches; the content is public bio facts) | A (today's state) |
| OQ-11 | Found while writing tasks: the agent's Statsig key is read-only (research.md §R9), so it cannot flip the kill flag or write the assignment config itself. Give it write access? | A: no; Jordi applies every config change and every stop in the Statsig console (the agent prepares the JSON). B: a second Statsig member with write role for the agent, its personal key in the local secret file, used only by `scripts/sync-config` and `scripts/kill`. C: as B, but only for `kill`; config changes stay with Jordi. | C (fast stops without letting the agent start things) | **Decided 2026-11-06 20:04: C.** A Statsig write key is used only by `scripts/kill`. Config changes and starts stay with Jordi. Jordi creates the key and places it in `~/.config/jordi-net/secrets.env`. |

## Assumptions

- Traffic is tens of visits a day or fewer; most experiments will give directional results, not statistical certainty, and the spec prefers long durations and simple rules over fine-grained tests.
- Jordi wants to keep `jordi.net`, its mail and its current DNS provider.
- The Statsig free tier is enough for this volume (see research.md §R3).
- Jordi will verify the domain in Google Search Console and Bing Webmaster Tools himself.
- Writing and approving variants happens in this repository through branches and pull requests; the agent never pushes to `master` itself.
- The starter thresholds are first guesses; the agent may propose changes to them, but only before an experiment starts.

## Where the brief's questions are answered

| Brief question | Answer in short | Detail |
|----------------|-----------------|--------|
| 1. Hosting | Decided: Vercel Hobby (OQ-2 A); the move waits for Jordi's deploy yes: edge assignment before HTML is sent, Statsig adapter, request logs; DNS change at DNSimple keeps MX. Railway works but means running a server; GitHub Pages cannot assign per request or show logs. | research.md §R1, plan.md |
| 2. SEO-safe design | Page and time-window units only, same content for every client, canonicals and temporary redirects for variant URLs, no UA-based content, bots labeled not targeted. | research.md §R2, FR-003 to FR-009 |
| 3. Metrics and sources | Statsig events for engagement, Search Console and Bing for search, referrer parsing for AI referrals, host request logs for AI crawlers; daily reads, weekly decisions, simple pre-registered rules. | research.md §R4 to §R8 |
| 4. Agent loop | Propose, check, ask, ship, read, decide; weekly; guardrails enforced by automated checks plus Jordi's approval (mode A for the first 3 experiments and until his yes to B). | plan.md "Agent loop", FR-010 to FR-015 |
| 5. Secrets | Host env vars and Jordi's local secret store; names only in the repo; secret scanning; agents never see values. | research.md §R9, FR-023 to FR-026 |
