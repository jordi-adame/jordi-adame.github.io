# Quickstart: Setup for Jordi, then Validation

Part A is for Jordi to do by hand; no agent does any of it. Part B lists the checks that prove the setup works once the implementation exists.

Ground rules for every step:

- Paste keys only into the provider's own settings page or into your local secret file (§A5). Never into a chat, an issue, a PR or a commit.
- Do the steps in order. §A3 (DNS) is the only step visitors can notice, and it waits until the parity check in Part B passes.
- Choices that are still yours (OQ-3 to OQ-10 in [spec.md](spec.md)) are marked **(your call)**. OQ-1 (approval mode D) and OQ-2 (Vercel Hobby) are decided.
- §A2 and §A3 are the hosting move. Do them only when you give the deploy yes.

---

## Part A: Setup

### A1. Statsig (about 15 minutes)

1. Go to https://www.statsig.com/pricing and sign up for the free **Developer** tier. Use your personal account or a team account **(your call, OQ-3)**. No credit card is needed.
2. Create a project named `jordi-net`.
3. Open **Project Settings > Environments & Keys**. You will see a **Server Secret Key** and a **Client SDK Key**. Leave them there for now; §A2 step 5 copies them into Vercel.
4. Open **Project Settings** and find the ID types (custom unit IDs) section. Add two custom IDs: `pagePath` and `windowId`.
5. Leave **bot filtering** on (the default).
6. Create a **Dynamic Config** named `site_experiments` with this value:

   ```json
   {"kill": false, "experiments": {}}
   ```

7. Create the agent's read key. The safest option is a read-only key:
   1. Invite a second Statsig member for the agent and give it a read-only role (for example Viewer).
   2. Sign in as that member and create a **personal Console API key**. Personal keys are limited to their owner's role, so this key cannot change anything.
   3. If Statsig offers no read-only role on the free tier, stop here and tell the agent. Do not hand over a full-access key without deciding that on purpose.
8. Put the read key into your local secret file (§A5) as `STATSIG_CONSOLE_API_KEY`.
9. Check for switchback support (V-2): open **New Experiment > Advanced Settings > Experiment Type**. Note whether **Switchback Test** is offered. If it says to contact support, you can ask them to enable randomized switchback bucketing. This step is optional.

### A2. Vercel (about 20 minutes; hosting decided, OQ-2 A; do this when you give the deploy yes)

1. Sign up at https://vercel.com on the **Hobby** plan with "Continue with GitHub". Hobby is free and limited to non-commercial personal use.
2. Choose **Add New > Project** and import `jordi-adame/jordi-adame.github.io`.
   - Framework preset: **Jekyll**.
   - Production branch: `master`.
   - Vercel deploys at once to a `*.vercel.app` address. jordi.net is not affected yet.
3. Do not add a domain yet; that is §A3.
4. Open **Marketplace > Statsig** and choose **Connect** (link an existing account). Pick the `jordi-net` project.
   - This creates a Global Config store (formerly Edge Config) and adds its connection variable to the project.
   - Do not choose "Native": that would create a second Statsig account billed through Vercel.
5. Open **Project > Settings > Environment Variables**. Add each variable for both Production and Preview, and mark it **Sensitive**:
   - `STATSIG_SERVER_SECRET`: the Server Secret Key from §A1 step 3.
   - `STATSIG_CLIENT_KEY`: the Client SDK Key from §A1 step 3.
6. Open **Project > Firewall > Managed rulesets > AI Bots** and set it to **Log**, not Deny. Deny would block the crawlers the experiments measure.
7. Vercel API token: skip it on Hobby. Create one only if you later move to Pro logs. If you do, make it **project-scoped**, give it a 90-day expiry, and put it in §A5 as `VERCEL_TOKEN`.

### A3. DNS change at DNSimple (after Part B check B1 passes; about 10 minutes plus waiting)

The only records that change are the apex A records and the `www` CNAME. Do not touch the MX records (Google Workspace mail) or the SPF TXT record.

1. One day before the switch, set the TTL to 300 seconds on the four apex A records (`185.199.108.153`, `.109`, `.110`, `.111`) and on the `www` CNAME.
2. Send a test email to an address @jordi.net and confirm it arrives.
3. In Vercel, open **Project > Settings > Domains** and add `jordi.net` and `www.jordi.net`. Vercel shows the exact records it wants. This is an A value for the apex (for example `76.76.21.21` or an address from the newer pool) and a per-project CNAME for `www`. If it asks for a `_vercel` TXT record, add that too.
4. In DNSimple, delete the four `185.199.x.153` A records and add the A record Vercel showed. There are no AAAA records to remove today.
5. In DNSimple, change `www` from `jordi-adame.github.io.` to the CNAME Vercel showed.
6. Wait until Vercel shows **Valid Configuration** and has issued the certificate. Then open https://jordi.net and https://www.jordi.net.
7. Send a second test email and confirm it arrives.
8. To roll back: restore the four A records and the `www` CNAME from step 4 and step 5. Records and TTL from step 1 make this quick.
9. After one week on Vercel, open **GitHub > repo > Settings > Pages** and unpublish the Pages site **(your call)**. The `CNAME` file can stay.

### A4. Search Console and Bing Webmaster Tools (about 20 minutes)

1. Open https://search.google.com/search-console and add a **Domain** property for `jordi.net`.
2. Add the TXT record Google gives you at DNSimple. This is a new TXT record; leave the existing SPF TXT record alone.
3. Open https://console.cloud.google.com and create a project (for example `jordi-net-reports`). Enable the **Google Search Console API** in it.
4. In that project, create a **service account** (no project roles needed). Create a **JSON key** for it and save the file to `~/.config/jordi-net/gsc-sa.json`. Then run `chmod 600 ~/.config/jordi-net/gsc-sa.json`.
5. In Search Console, open **Settings > Users and permissions > Add user**. Enter the service account's email and choose **Restricted**.
6. Open https://www.bing.com/webmasters and import the site from Search Console.
7. In Bing, open **Settings > API access** and generate an API key. Put it in §A5 as `BING_WEBMASTER_API_KEY`.
8. Optional: export the Search Console **Generative AI performance** report as CSV now and then. Save it to `~/Downloads/jordi-net-gsc-ai/`. No API is known for it yet.

### A5. Local secret file (about 5 minutes)

Run these yourself in a terminal (not through an agent):

```sh
mkdir -p ~/.config/jordi-net && chmod 700 ~/.config/jordi-net
touch ~/.config/jordi-net/secrets.env && chmod 600 ~/.config/jordi-net/secrets.env
```

Open the file in your editor and fill in the values. The variable names are:

```sh
STATSIG_CONSOLE_API_KEY=
GSC_SERVICE_ACCOUNT_JSON=~/.config/jordi-net/gsc-sa.json
BING_WEBMASTER_API_KEY=
# only if used:
# VERCEL_TOKEN=
# RAILWAY_TOKEN=
```

A password manager CLI works too, as long as it exports the same variable names to the read scripts.

### A6. GitHub repository settings (about 5 minutes)

1. Open **Settings > Code security** and turn on **Secret scanning** and **Push protection**.
2. Open **Settings > Branches** and add a protection rule for `master`. Require a pull request and a review from Code Owners. The implementation adds a `CODEOWNERS` file naming you.

### A7. Railway instead of Vercel (not used: OQ-2 decided A; kept for reference only)

1. Sign up at https://railway.com on **Hobby** ($5/month, includes $5 of usage).
2. Create a project from the GitHub repo and enable autodeploy from `master`.
3. Add the variables `STATSIG_SERVER_SECRET` and `STATSIG_CLIENT_KEY` to the service.
4. Add the custom domain `jordi.net`. Railway shows a target `<id>.up.railway.app` and a TXT verification record.
5. In DNSimple, replace the apex A records with an **ALIAS** record to that target. Add the TXT record. Point `www` at the same target with a CNAME.
6. Follow the mail tests and rollback from §A3.
7. For log reads, create a **project token** and put it in §A5 as `RAILWAY_TOKEN`.

---

## Part B: Validation scenarios (after implementation)

| Id | Scenario | Expected |
|----|----------|----------|
| B1 | Parity: compare `https://jordi.net` (GitHub Pages) with the Vercel production URL, same path | identical visible HTML apart from host-specific headers (V-3) |
| B2 | Crawler parity: request an experiment URL with a Googlebot user agent and a Chrome user agent in the same window | byte-identical HTML; both produce `pageview` events with different `client_class` |
| B3 | Kill switch: set `kill: true` in `site_experiments` | every experiment URL serves control within 10 minutes; `assignment_fallback` not logged (kill is deliberate) |
| B4 | Statsig outage: point the middleware at an invalid config in a preview | control is served; `assignment_fallback` is logged |
| B5 | Readout (V-1): after 3 days in P1, run `scripts/read/statsig` | per-day, per-path counts of human pageviews, crawler hits by bot and AI referrals; or a clear error that triggers the fallback decision |
| B6 | Search read: run `scripts/read/gsc` | rows for `/` with `complete: true` only for days before `first_incomplete_date` |
| B7 | Guardrail fixtures: run `scripts/check` on `tests/fixtures/bad-variants/*` | every fixture fails with the expected check name |
| B8 | Secret hygiene: run gitleaks on the full history; grep agent transcripts for key prefixes | no findings |
| B9 | Mail: test messages from §A3 steps 2 and 7 | both delivered |
| B10 | Direct variant URL: request `/x/<exp>/<variant>/` | `302` to the canonical path |
