# Research: Agent-Run SEO/GEO Experiments on jordi.net

Research date: 2026-11-06. Sources are vendor docs fetched that day unless marked **[3P]** (third-party only) or **[unverified]**. Facts read from jordi.net itself (DNS, headers) were checked the same day with DNS-over-HTTPS and `curl -I`.

## Current state of jordi.net (checked 2026-11-06)

- Served by GitHub Pages (`server: GitHub.com`), apex A records `185.199.108-111.153`, `www` CNAME `jordi-adame.github.io.`
- Nameservers: DNSimple (`ns1..ns4.dnsimple.com`). DNSimple supports ALIAS records.
- Mail: Google Workspace MX (`aspmx.l.google.com` and alternates) and SPF TXT `v=spf1 a include:_spf.google.com ~all`. **Note**: the SPF record includes `a`, so moving the apex A record changes which IPs SPF allows. Mail is sent by Google, so `include:_spf.google.com` still covers it, but the plan keeps SPF unchanged and tests mail after the move (SC-006).
- Repository `jordi-adame/jordi-adame.github.io` is **public**; default branch `master`. Anything committed (experiment records included) is world-readable on GitHub.

---

## R1. Hosting: GitHub Pages vs Vercel vs Railway

**Decision**: Recommend **Vercel Hobby** to start, with Routing Middleware assigning variants before HTML is sent and logging every pageview (crawlers included) to Statsig. Move to Vercel Pro only if the Statsig readout path (R3, V-1) fails and host logs are needed. Railway is the fallback if Jordi prefers a plain server. **Jordi decided OQ-2 A (Vercel Hobby) on 2026-11-06**; the move itself waits for his separate deploy yes.

**Rationale**:

| | GitHub Pages (today) | Vercel | Railway |
|---|---|---|---|
| Per-request assignment (crawlers see variants) | No. Static only; only time-window rotation via scheduled rebuilds | Yes: Routing Middleware runs before the response, all plans | Yes: you run the server |
| Cost | $0 | Hobby $0 (1M CDN requests, 1M function invocations, 4 h active CPU, 100 GB transfer); Pro $20/month | Hobby $5/month incl. $5 usage; a tiny Node service is about $2/month of usage [estimate] |
| Commercial-use rule | Not for running an online business; personal site fine | Hobby is "non-commercial personal use only" (no ads, affiliate-primary, paid work) | none for Hobby beyond ToS |
| Request logs with user agent | None documented | Runtime logs incl. "Request User Agent": Hobby 1 h, Pro 1 day, Pro + Observability Plus 30 days; Drains Pro only ($0.50/GB) | HTTP logs with `clientUa`, path, status: Hobby 7 days, Pro 30 days; `railway logs --http --json` |
| Statsig fit | Client-side only (crawlers would not see assignment) | Official: `@statsig/vercel-edge` (3.33.8) with Global Config sync via Marketplace; Flags SDK adapter `@flags-sdk/statsig` (Next.js/SvelteKit only) | Server Core `@statsig/statsig-node-core` (0.24.2) in a normal Node server |
| Apex DNS at DNSimple | current A records | change apex A to the value Vercel shows (76.76.21.21 or the newer pool, e.g. 216.198.79.1); `www` CNAME to the per-project target | ALIAS at apex to `<id>.up.railway.app` (DNSimple ALIAS explicitly supported) + TXT verification |
| Bot features | none | Firewall AI Bots managed ruleset (log/deny "AI Crawlers" and "AI Assistants"), verified-bot detection by IP ranges, reverse DNS and Web Bot Auth | none built in |
| Maintenance | none | none (static files + one middleware file) | a long-running server, restarts, Node updates; Serverless sleep can return 502 on wake |

Why Vercel over Railway: it keeps the site static (no server to keep alive), costs $0, has the official Statsig edge integration and verified-bot signals, and the 1-hour Hobby log limit does not matter because the middleware writes its own pageview and crawler events to Statsig (R3, R6). Railway wins only on host-log retention (7 days on $5), which matters only if Statsig cannot be read back (V-1).

Why not stay on GitHub Pages: it cannot assign per request, has no request logs, so no AI-crawler data at all, and sends no custom headers. Time-window experiments by scheduled rebuild (Actions `schedule`, minimum 5 minutes, may be delayed; schedules auto-disable after 60 days of repo inactivity) are possible, but the crawler half of GEO measurement is missing.

**Alternatives considered**: Cloudflare (Workers + `@statsig/serverless-client`, free logs via Workers Analytics) also fits technically, but the brief limits hosting to the three named; noted for Jordi only.

**Risks**: Hobby's non-commercial clause: if Jordi later adds affiliate or paid content, Pro is required. Vercel renamed Edge Config to Global Config on 2026-07-29 (`@vercel/global-config`, `GLOBAL_CONFIG` env var; `EDGE_CONFIG` still works); docs and package names may lag.

Sources: https://docs.github.com/en/pages/getting-started-with-github-pages/github-pages-limits · https://github.com/orgs/community/discussions/54257 · https://docs.github.com/en/actions/reference/workflows-and-actions/events-that-trigger-workflows#schedule · https://vercel.com/pricing · https://vercel.com/docs/limits/fair-use-guidelines · https://vercel.com/docs/routing-middleware · https://vercel.com/changelog/edge-config-is-now-global-config · https://vercel.com/docs/logs/runtime · https://vercel.com/docs/drains · https://vercel.com/docs/cli/logs · https://vercel.com/docs/observability/observability-plus · https://vercel.com/docs/vercel-firewall/vercel-waf/managed-rulesets · https://vercel.com/docs/bot-management · https://vercel.com/kb/guide/a-record-and-caa-with-vercel · https://vercel.com/docs/domains/working-with-domains/add-a-domain · https://docs.statsig.com/integrations/vercel · https://flags-sdk.dev/providers/statsig · https://docs.railway.com/reference/pricing/plans · https://docs.railway.com/reference/app-sleeping · https://docs.railway.com/guides/logs · https://docs.railway.com/cli/logs · https://docs.railway.com/networking/domains/working-with-domains

---

## R2. SEO-safe experiment design

**Decision**:

1. Units are **pages** (URL paths) or **time windows**, never visitors, for any search or AI metric. Per-visitor splits are useless for ranking anyway: Google indexes one version of a URL, so a 50/50 visitor split just shows Google whichever version it happens to fetch.
2. **Same content for every client** at a given URL and time. Content never branches on user agent, IP or bot status; bot detection only labels events (FR-005).
3. Variants are served **in place** by an internal rewrite (the URL does not change). Variant source files live under an internal path; a direct request to that path gets a **302** to the canonical URL. Any variant reachable on its own URL carries `rel="canonical"` to the original.
4. Experiments are **time-boxed** with a planned end and at most one extension; when an experiment ends, the losing variant's elements are removed.
5. `noindex` is never put inside variant HTML (an internal rewrite would noindex the real URL).
6. Short cache lifetimes on experiment URLs so window changes and stops propagate within 10 minutes.

**Rationale** (Google, https://developers.google.com/search/docs/crawling-indexing/website-testing):

- "Don't show one set of URLs to Googlebot, and a different set to humans", whether "by server logic or by robots.txt, or any other method."
- "use the rel="canonical" link attribute on all of your alternate URLs to indicate that the original URL is the preferred version."
- "use a 302 (temporary) redirect, not a 301 (permanent) redirect."
- "remove all elements of the test as soon as possible"; running unnecessarily long may be seen as deceptive. No fixed day count is given.
- "Googlebot generally doesn't support cookies", so Googlebot sees what cookieless visitors see; our design sets no cookie at all.

Spam policies an agent must never trip (https://developers.google.com/search/docs/essentials/spam-policies): cloaking, doorway abuse, sneaky redirects, hidden text and links, keyword stuffing, **scaled content abuse** ("whatever tool creates them"), link spam, scraping, thin affiliation, expired domain abuse, site reputation abuse (now "Site reputation policy", updated 2026-08). On AI content: "not all use of automation, including AI generation, is spam", but automation "with the primary purpose of manipulating ranking" is (https://developers.google.com/search/blog/2023/02/google-search-and-ai-content, 2023, still the canonical statement).

For AI features: "no additional requirements to appear in AI Overviews or AI Mode"; Google's May 2026 optimization guide says "You don't need to create new machine readable files, AI text files, markup, or Markdown" (https://developers.google.com/search/docs/appearance/ai-features, https://developers.google.com/search/docs/fundamentals/ai-optimization-guide). A Search Console opt-out from AI features exists since 2026-08-31 (https://support.google.com/webmasters/answer/16908024); the agent must never change it.

**Alternatives considered**: separate variant URLs with canonicals (classic SEO split test): rejected for the home page because the canonical makes Google consolidate signals to the original, so the variant URL never ranks on its own; kept only for new topic pages, where each page is its own unit. Showing bots the control while humans get variants: rejected as cloaking.

---

## R3. Statsig: ownership, pricing, SDKs, how it is used here

**Facts**:

- **Ownership changed twice.** OpenAI signed to acquire Statsig on 2025-09-02. An editor's note on the same post now says "Statsig was acquired by Amplitude on May 5, 2026"; the original team stayed at OpenAI and Amplitude "continues to develop Statsig", calling it a "strategic partnership" with an "integrated roadmap". No sunset announced; a competitor predicts consolidation **[3P]**. Statsig is still sold to new customers. (https://www.statsig.com/blog/openai-acquisition, https://www.amplitude.com/blog/amplitude-and-statsig-partnership, https://martech.org/amplitude-and-statsig-deal-raises-questions-for-customers/)
- **Pricing** (https://www.statsig.com/pricing): Developer free, 2M metered events/month (above that, Pulse stops computing until upgrade), unlimited flag checks, 1-year retention, A/B tests. Pro $150/month for 5M events then $0.05/1K, adds "advanced experimentation", approvals, "API controls". Enterprise custom (warehouse native, data export, SSO). Billable: exposures (deduped) and logged events; each extra event dimension counts as another event. Per-tier placement of switchback, CUPED and sequential testing did not render **[unverified]**; they are listed as advanced, which suggests Pro.
- **Switchback tests** exist (Advanced Settings > Experiment Type = Switchback; window size in minutes, burn-in/out; bootstrapped CIs; no CUPED, sequential or daily time series). Randomized bucketing must be enabled by Statsig support; tier not stated (https://docs.statsig.com/experiments/types/switchback-tests).
- **Page-level units**: a Custom Unit ID (e.g. `pagePath`) defined in Project Settings and passed as `customIDs` (https://docs.statsig.com/guides/experiment-on-custom-id-types). A time-bucket ID can be a custom ID the same way (inference).
- **SDKs** (npm, 2026-11-06): `@statsig/vercel-edge` 3.33.8 (`handleWithStatsig`, `StatsigVercelClient`, flush with `waitUntil(client.flush())`); `@statsig/statsig-node-core` 0.24.2 (native binaries, not for edge); `@statsig/js-client` 3.33.8 (browser); `@flags-sdk/statsig` 0.3.1 (Next.js/SvelteKit). Legacy: `statsig-node` (docs call it legacy), `statsig-node-vercel` 0.8.1; `statsig-js` deprecated ("Support for this package ends Jan 31 '25").
- **Keys** (Project Settings > Environments & Keys, https://docs.statsig.com/sdk-keys/api-keys): server secret key (downloads config, logs events; can be limited by Target Apps); client SDK key (browser-safe, precomputed hashed values, logs events); Console API key (full CRUD on the project; personal Console API keys are limited to the holder's Statsig role).
- **Reading results**: Console API at `https://statsigapi.net`, header `STATSIG-API-KEY`, version `20240601`; experiment endpoints include create/start/`make_decision`/`abandon`, and `GET /console/v1/experiments/{id}/pulse_results` (**Beta**) returning `percentChange`, `confidenceInterval`, `pValue`. Writes limited to about 1,800 per project per 15 min. A Statsig MCP server (`https://api.statsig.com/v1/mcp`, 18 tools, read tools for read-only users) exists. (https://docs.statsig.com/console-api/introduction, https://docs.statsig.com/api-reference/experiments/retrieve-pulse-results-beta, https://docs.statsig.com/integrations/mcp)
- **Bots**: Statsig removes known bots from **exposures** by parsed browser name (per-project, admins can opt out); filtered data cannot be viewed later. Whether custom events are filtered is **[unverified]** (https://docs.statsig.com/experiments/monitoring/bots).

**Decision**: Statsig plays four roles, none of which needs a paid tier:

1. **Config and kill switch**: one dynamic config `site_experiments` holds the live assignment map (mirrored from the repo manifests) and a `kill` flag. It syncs to Vercel Global Config through the Marketplace "Connect" integration, so the middleware reads it with no network call and Jordi can stop everything from the Statsig console.
2. **Event pipeline**: the middleware logs `pageview` (humans and crawlers); the browser logs engagement events with the client key. Unit ID is the page path or window id, never a visitor ID. For crawler events the SDK user object gets **no `userAgent`** (only `bot_name` metadata), so Statsig's bot filter does not drop them.
3. **Experiments of record**: each experiment is also created in Statsig with ID type `pagePath` or `windowId`, so the console shows it and Pulse can compute what it can.
4. **Native switchback** for time-window experiments if Jordi's tier and Statsig support allow it; otherwise the repo-generated schedule (R8) is used and Statsig only records.

Assignment for page and time units comes from the repo manifest (seeded, auditable), not from Statsig hashing, because with 2 to 6 pages a hash can easily put every page in one arm.

**Verification item V-1 (setup, before E1)**: confirm on the Developer tier that per-variant event counts can be read by API (Pulse results for an experiment with custom ID type, or the MCP `api_read` path). **Fallback**: Vercel Pro with Observability Plus (30-day logs queryable with `vercel logs --json`) or Railway HTTP logs; either costs about $20/month or $5/month and would be raised with Jordi under OQ-4.

**Reputation note for Jordi**: Statsig is now an Amplitude product; event data (paths, referrer class, bot name; no IPs, no visitor IDs) goes to Amplitude-owned infrastructure. Jordi asked for Statsig, so this is recorded, not reopened.

---

## R4. Search data: Google Search Console and Bing

**Decision**: Domain property for `jordi.net` verified by DNS TXT at DNSimple. The agent reads with a **Google Cloud service account** added in Search Console as a **Restricted** user, scope `webmasters.readonly`. Bing Webmaster Tools via API key (read). Reads daily, analysis on complete days only.

**Facts**:

- `searchanalytics.query`: dimensions `country, device, page, query, searchAppearance, date, hour`; `type` web/image/video/news/discover/googleNews; regex filters; `rowLimit` 1 to 25,000; `dataState` `final` (default), `all` (fresh), `hourly_all`; `metadata.first_incomplete_date` marks days that may change (https://developers.google.com/webmaster-tools/v1/searchanalytics/query).
- Data "typically available after 2-3 days"; anonymized queries are dropped when grouping by query (https://developers.google.com/webmaster-tools/v1/how-tos/all-your-data).
- Quotas: Search Analytics 1,200 QPM per site; URL Inspection 2,000 QPD per site (https://developers.google.com/webmaster-tools/limits).
- AI Overviews and AI Mode traffic is counted inside the "Web" search type, not separable in the API (https://developers.google.com/search/docs/appearance/ai-features).
- New **Generative AI performance report** (2026-06-03, all sites since 2026-08-31): AI Overviews and AI Mode impressions by page, country, device, date; no clicks or queries; API access **[unverified]**, export only (https://developers.google.com/search/blog/2026/06/gen-ai-performance-reports, https://support.google.com/webmasters/answer/16984139). The agent can only use it from a CSV export Jordi drops in, unless an API appears.
- Indexing API is only for JobPosting/BroadcastEvent; not usable here (https://developers.google.com/search/apis/indexing-api/v3/quickstart).
- Bing: Webmaster API with API key (`GetQueryStats`, `GetPageStats`, `GetQueryPageStats`, weekly buckets) (https://learn.microsoft.com/en-us/dotnet/api/microsoft.bing.webmaster.api.interfaces.iwebmasterapi.getquerypagestats). **AI Performance** report (public preview 2026-02-10) shows citations in Copilot, Bing AI summaries and partners, by grounding query and cited page; no API at launch **[3P]** (https://blogs.bing.com/webmaster/February-2026/Introducing-AI-Performance-in-Bing-Webmaster-Tools-Public-Preview). IndexNow lets the site notify Bing of changed URLs (https://www.bing.com/indexnow/getstarted).

Service-account-as-user pattern is **[3P]** documented (https://glama.ai/mcp/servers/gk2ywxsde1); OAuth with Jordi's account is the alternative but puts a refresh token for his Google account on disk, which is a wider secret.

---

## R5. AI referrals

**Decision**: classify on the server from `Referer` host **and** `utm_source`, per the table in [contracts/assignment.md](contracts/assignment.md). Count as `ai:<engine>`.

**Facts** (all **[3P]**; no vendor documents its referrer behavior): ChatGPT appends `utm_source=chatgpt.com` to many citation links (since about June 2025); referrer hosts `chatgpt.com`, `chat.openai.com`. Others: `perplexity.ai`, `claude.ai`, `gemini.google.com`, `copilot.microsoft.com`, `edgeservices.bing.com`. Many AI clicks arrive with no referrer and look like direct traffic, so AI referrals are a floor, not a total (https://ahrefs.com/blog/ai-chatbot-traffic/, https://authoritytech.io/blog/ai-traffic-attribution-how-to-track-chatgpt-perplexity-gemini). Vercel Web Analytics has a referrers panel but UTM parameters need Web Analytics Plus (+$10/month), so the server-side classification is the primary source (https://vercel.com/docs/analytics/limits-and-pricing).

---

## R6. AI crawlers: who, why, how to verify

**Decision**: the middleware labels each request with bot name, purpose and verification state; counts are logged to Statsig. Verification uses the engine's published IP JSON (fetched daily and cached), reverse-DNS where documented, and Vercel's verified-bot signal when available. robots.txt keeps allowing all of them (blocking is Jordi's call and would end GEO measurement).

| Bot | Operator | Purpose | Verify |
|-----|----------|---------|--------|
| GPTBot | OpenAI | training | https://openai.com/gptbot.json |
| OAI-SearchBot | OpenAI | ChatGPT search index | https://openai.com/searchbot.json |
| ChatGPT-User | OpenAI | user-triggered fetch ("robots.txt rules may not apply") | https://openai.com/chatgpt-user.json |
| ClaudeBot | Anthropic | training | claude.com/crawling/bots.json |
| Claude-SearchBot | Anthropic | search index | same |
| Claude-User | Anthropic | user-triggered | same |
| PerplexityBot | Perplexity | search index ("not used to crawl content for AI foundation models") | https://www.perplexity.ai/perplexitybot.json |
| Perplexity-User | Perplexity | user-triggered ("generally ignores robots.txt") | https://www.perplexity.ai/perplexity-user.json |
| Googlebot | Google | Search incl. AI Overviews/AI Mode | reverse DNS `*.googlebot.com`/`*.geo.googlebot.com`; https://developers.google.com/static/crawling/ipranges/common-crawlers.json |
| Google-Extended | Google | robots.txt token only (Gemini training and grounding); no user agent, cannot be counted | n/a |
| Bingbot | Microsoft | Bing and Copilot grounding | reverse DNS `*.search.msn.com`; https://www.bing.com/toolbox/bingbot.json (docs page failed to load; from prior knowledge) |
| Applebot / Applebot-Extended | Apple | search (may feed training) / opt-out token only | reverse DNS `*.applebot.apple.com` |
| CCBot | Common Crawl | dataset used for training | reverse DNS `*.crawl.commoncrawl.org`; https://index.commoncrawl.org/ccbot.json |
| meta-externalagent / meta-webindexer / meta-externalfetcher | Meta | training and indexing / Meta AI search / user-triggered | https://developers.facebook.com/docs/sharing/webmasters/web-crawlers/ |

Sources: https://developers.openai.com/api/docs/bots · https://support.claude.com/en/articles/8896518 · https://docs.perplexity.ai/guides/bots · https://developers.google.com/search/docs/crawling-indexing/google-common-crawlers · https://support.apple.com/en-us/119829 · https://commoncrawl.org/ccbot · https://bots.fyi

Purpose matters for interpretation: AI-search and user-triggered fetches are leading indicators of citations; training-crawler fetches are not.

---

## R7. llms.txt and GEO tactics

- **llms.txt**: no major engine officially consumes it. Google's guide says such files are unnecessary for Search; OpenAI and Anthropic crawler docs mention only robots.txt. Adoption about 10% of sites **[3P]** (https://presenc.ai/research/state-of-llms-txt-2026). Hence E0 is framed as a near-free probe, not a bet.
- **GEO paper** (Aggarwal et al., KDD 2024, arXiv 2311.09735, https://arxiv.org/html/2311.09735): on research-era engines, adding quotations (about +41% visibility), statistics (about +33%), fluency (+29%) and cited sources (+28%) helped; keyword stuffing hurt; "efficacy of these strategies varies across domains". Older than 12 months; used only to motivate E3's answer-first format.

---

## R8. Statistics at tiny traffic

**Decision**: pre-registered, simple rules; long windows; directional results labeled as such.

- Classic SEO split testing needs page groups of 20 to 50+ pages and about 100 days of pre-period **[3P]**; SearchPilot's old rule of thumb was about 30k organic sessions/month per group (https://searchpilot.com/resources/blog/weve-doubled-the-sensitivity-of-our-seo-split-testing-platform). jordi.net has one page, so true page-group tests are not feasible.
- Search and AI metrics: **pre/post (interrupted time series)** against the site's own pre-period, with a rule fixed in advance (E1), and optionally CausalImpact (Bayesian structural time series) once there are 8+ weeks of pre-period (https://www.jcchouinard.com/causalimpact-for-seo). Search Console keeps 16 months of history, so the pre-period exists as soon as the property is verified.
- Engagement metrics: **paired daily switchback** (each pair of days holds both arms in random order, seeded) with a Beta-Binomial comparison of engaged-pageview share; decide at P(better) ≥ 0.90 with a minimum of 100 human pageviews per arm. Days that straddle a window change and the first hour of each window (cache burn-in) are dropped.
- New topic pages (E3): too few pages for inference; a large-effect rule (≥2x with ≥20 events) and the word "directional" in the decision.
- Statsig's power calculator (https://www.statsig.com/calculator) is used to state the minimum detectable effect before launch so Jordi sees how large an effect the test can find.
- Extend once, then close; "inconclusive" is a normal outcome.

---

## R9. Secrets

**Decision**: Jordi creates and places every secret; agents see names only.

| Secret | Where Jordi puts it | Used by | Notes |
|--------|---------------------|---------|-------|
| `STATSIG_SERVER_SECRET` (server secret key) | Vercel project env vars, Production + Preview, marked Sensitive | middleware | never in the browser |
| `STATSIG_CLIENT_KEY` (client SDK key) | Vercel env var, exposed to the page at build time | browser engagement events | browser-safe by design, still not committed |
| Global Config connection (`GLOBAL_CONFIG` / `EDGE_CONFIG`) | created by the Vercel Statsig integration | middleware | read-only token embedded in the connection string |
| `STATSIG_CONSOLE_API_KEY` (agent reads) | Jordi's local secret store (below) | read scripts | personal key of a Statsig user with a read-only role, so the key cannot write |
| `GSC_SERVICE_ACCOUNT_JSON` path | local secret store; the JSON file itself at `~/.config/jordi-net/gsc-sa.json`, mode 600 | read scripts | Restricted user in Search Console |
| `BING_WEBMASTER_API_KEY` | local secret store | read scripts | |
| `VERCEL_TOKEN` | local secret store; **project-scoped**, expiry 90 days | read scripts (`vercel logs`), optional | only if Pro logs are adopted; Hobby needs none |
| `RAILWAY_TOKEN` (project token) | local secret store, only if Railway is chosen | read scripts | project token, one environment |

**Local secret store**: `~/.config/jordi-net/secrets.env` (mode 600, outside the repo), loaded by the read scripts themselves, or Jordi's password-manager CLI if he prefers (the scripts take secrets from environment variables either way).

**Enforcement**:

- Read scripts load secrets themselves; the agent runs the scripts and never needs the values.
- Project `.claude/settings.json` deny rules: `Read(~/.config/jordi-net/**)`, `Bash(cat ~/.config/jordi-net/*)`, `Bash(env)`, `Bash(printenv*)`, `Bash(vercel env pull*)`; scripts redact any value matching a secret pattern in their output.
- Pre-commit secret scan (gitleaks) and GitHub secret scanning with push protection (free for public repos; Jordi enables it).
- Agents never ask Jordi to paste a secret into chat; setup steps tell him to paste into the provider UI or the local file.
- A leaked key: Jordi rotates it in the provider; `kill` is set (by Jordi, or by the agent if OQ-11 gave it a key that can) and experiments wait for his confirmation (spec edge case).

Sources: https://docs.statsig.com/sdk-keys/api-keys · https://vercel.com/kb/guide/how-do-i-use-a-vercel-api-access-token · https://vercel.com/docs/api · https://docs.railway.com/integrations/api

---

## R10. Guardrail enforcement

**Decision**: three layers; any one can block.

1. **Automated checks** (scripts, run on every variant and in CI on every PR): manifest validation; outbound-link domains against `experiments/allowed-domains.txt`; hidden-content scan (CSS that hides text or links, off-screen positioning, same-color text); page-count limit; HTML validity, title and description present; crawler parity (render the preview with a Googlebot user agent and a browser user agent and require identical HTML); secret scan; copied-text check against the pages cited in the variant.
2. **Independent review agent**: a separate agent session, given only the variant and the policy list (no impersonation of any person or organization; no claim about a named real person or company without a cited source; nothing defamatory, sexual, hateful or political; no spam links; nothing that implies Jordi endorses a product), returns pass/fail with reasons, recorded as a check result. This covers what scripts cannot judge.
3. **Jordi's approval** of each variant on its pull request (Vercel gives a preview URL per PR). Branch protection on `master` requires his review; CODEOWNERS makes him owner of `experiments/allowed-domains.txt` and of the middleware. The agent never merges.

Emergency stop needs no approval: setting `kill: true` in the Statsig dynamic config serves the control everywhere within the cache lifetime (≤5 minutes).
