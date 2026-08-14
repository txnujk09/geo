# Phase 1: provider diagnosis and measurement plan

Status: proposal. No application code written. Awaiting confirmation before Phase 2.

Prices verified by research on 2026-08-14. Every figure marked (verify) should be
re-checked against the live pricing page before we commit spend, because vendor
pricing pages were not directly reachable from this environment.

## 0. Headline

Provider cost is not the constraint. The whole 6 week pilot costs roughly 40 to 80
dollars in API spend. The constraints are:

1. Two of the three surfaces cannot be measured as the consumer product. We measure
   a proxy and we have to be honest in the UI about which proxy.
2. The pilot as specified is underpowered for per prompt claims. It is adequately
   powered only when pooled across prompts within a cohort.
3. YouTube will not give us transcripts for videos through the official API unless
   the client authorises us as the video owner.

Each of these is a design decision, not a bug, and each needs to be settled before
we write ingestion code.

## 1. Perplexity Sonar API

Verdict: usable, cheapest and most reliable of the three surfaces. Adopt.

Citations come back structured, as top level fields on the chat completion response
alongside the message content:

- `search_results`: array of objects with `url`, `title`, `date` and snippet text.
  This is the field to parse.
- `citations`: legacy flat array of URL strings. Kept for backwards compatibility.
  Store it but do not build on it.

Request side controls that matter for us:

- `search_domain_filter` (allowlist or denylist domains). Do not use it for
  measurement runs. Filtering the search space invalidates the appearance rate.
- `search_recency_filter`.
- `web_search_options.search_context_size` set to `low`, `medium` or `high`. This
  changes both cost and the number of sources returned, so it must be pinned per
  surface config and recorded on every run. Changing it mid pilot breaks the
  time series.

Cost (verify): billed as tokens plus a per request search fee. Sonar base is about
1.00 dollar per 1M tokens in and out. The per request search fee runs about 5 to 14
dollars per 1000 requests depending on context size. At roughly 1.5k tokens per
response, budget 0.012 dollars per run.

The caveat that matters: the Sonar API is not the consumer Perplexity product. It is
a separate retrieval stack with its own index and ranking. A citation lift measured
on Sonar is evidence about Sonar, and it is decent circumstantial evidence about
consumer Perplexity, but it is not the same measurement. The comparison view has to
label this surface honestly. I would rather say "Perplexity Sonar API" in the UI than
"Perplexity".

## 2. Google AI Overviews

Recommendation: DataForSEO primary, Bright Data as the fallback provider.

| Provider | Unit cost (verify) | AI Overview handling | Notes |
| --- | --- | --- | --- |
| DataForSEO | 0.0006 standard queue, plus 0.0006 for `load_async_ai_overview`, refunded if no AIO present | `ai_overview` appears in `item_types` when present. Async variant fetched in the same task when the flag is set | Pay as you go, 50 dollar minimum deposit. Cheapest by an order of magnitude |
| SerpApi | 25 dollars per 1000 at entry tier, down to about 9 dollars per 1000 at 275/mo | AI Overview often inline. When Google defers it, returns `ai_overview.page_token` that must be spent on the separate `google_ai_overview` engine as a second billable search, and the token expires in about 60 seconds | Subscription only, no pay as you go, unused searches do not roll over. Best documentation of the three |
| Bright Data | about 1.50 per 1000 pay as you go, 1.30 at the 499/mo tier, 5000 records/mo free | Dedicated Google AI Overview endpoint returning summary, sources and follow up questions | Free tier alone nearly covers the pilot. Good fallback |

DataForSEO wins on cost and on the async handling, which is the part most likely to
silently lose data. The SerpApi 60 second `page_token` expiry is a real operational
hazard for a queue based worker: if the worker is backed up, the token dies and the
first search is billed for nothing. DataForSEO resolving the async overview inside a
single task removes that failure mode.

Cost for the pilot is negligible either way. The 50 dollar DataForSEO minimum deposit
is larger than the entire AI Overview spend.

## 3. ChatGPT

This is the genuinely hard one. The user is correct that the OpenAI API does not
reproduce consumer ChatGPT browsing and citation behaviour. Three real options:

**Option A: OpenAI Responses API with the `web_search` tool.**
Structured `url_citation` annotations carrying url, title and location. Stable,
first party, no anti bot risk. Cost (verify) around 25 to 30 dollars per 1000
queries for the search tool call, plus search result content billed as input tokens
at model rates, which is the part people underestimate. Fragility: very low.
Fidelity to the consumer product: low. Different retrieval, different ranking,
different citation surface. This measures "the OpenAI web search tool", not ChatGPT.

**Option B: a hosted consumer surface scraper.** DataForSEO AI Optimization API
(`ai_optimization/chat_gpt/llm_responses`) and Bright Data ChatGPT Scraper API both
drive the actual consumer interface and return the citations and source links the
official API does not expose. Cost (verify) roughly 0.01 to 0.02 per grounded
response. Fragility: moderate, but it is outsourced fragility. When OpenAI changes
the DOM or tightens anti bot, the provider absorbs it and we see an outage or a
degraded response rather than a maintenance burden.

**Option C: self hosted browser automation** against a logged in account, with
residential proxies. Highest fidelity, and I recommend against it. It violates
OpenAI terms of service, risks account bans mid pilot, and puts us in a permanent
cat and mouse game. For a client facing measurement product where the whole value is
trustworthy numbers, betting the pilot on an automation stack that can be banned in
week 3 is a bad trade.

**Recommendation: Option B as the primary ChatGPT surface, Option A recorded in
parallel as a separate surface.** Running both is cheap, roughly 30 dollars extra
across the pilot, and it buys two things. It gives a fallback series if the scraper
breaks, and if the two series move together it is meaningful corroboration, while if
they diverge that divergence is itself a finding worth showing the client.

This means `surface` is not a three value enum. It is at minimum:
`perplexity_sonar`, `google_ai_overview`, `chatgpt_consumer`, `openai_websearch`,
and optionally `gemini`.

## 4. YouTube Data API

Short answer on transcripts: no, not for arbitrary videos.

- `captions.download` requires OAuth and the authenticated user must own the video.
  Third party public videos return 403. There is no official endpoint for auto
  generated transcripts of videos you do not own.
- This is fine for the pilot, because we are optimising the clients' own videos.
  The correct path is a one time OAuth grant from each client's YouTube channel,
  after which `captions.list` and `captions.download` work legitimately for both the
  treated and the control cohort. This needs to go on the client onboarding checklist
  now, because it is a human dependency with a lead time.
- The unofficial `timedtext` route (the `youtube-transcript-api` family) works
  without OAuth but is heavily blocked from datacenter IP ranges, which includes
  Vercel. Using it in production requires rotating residential proxies at roughly
  30 dollars a month. Reserve this for competitor videos if we ever need them, not
  for the pilot's own assets.

Other fields:

- Description and publish date: `videos.list` with `part=snippet`, 1 unit, up to 50
  video ids per call. Trivial.
- Chapters: not exposed by the API in any form. They must be parsed out of the
  description as timestamp lines. Worth encoding YouTube's own rendering rules as
  validation, since a chapter list that YouTube will not render is not a treatment
  that was actually applied: the first timestamp must be 00:00, there must be at
  least three chapters, and each must be at least 10 seconds long.

Quota: 10,000 units per day per Google Cloud project, resetting at midnight Pacific.
`videos.list` is 1 unit, `captions.list` is 50, `captions.download` is 200,
`search.list` is 100. For roughly 200 assets refreshed daily this is nowhere near the
ceiling, as long as we never call `search.list` in a loop. Quota increases are not
self service and take weeks to months, so the design rule is simply: resolve video
ids from stored URLs, never search for them.

## 5. Estimated cost

Volume as specified: 80 prompts x 3 runs x 6 weeks = 1,440 runs per surface.

| Line item | Pilot total (verify) |
| --- | --- |
| Perplexity Sonar, 1,440 runs at ~0.012 | ~17 |
| Google AI Overview via DataForSEO, 1,440 runs at 0.0012 | ~2 |
| ChatGPT consumer via DataForSEO, 1,440 runs at ~0.015 | ~22 |
| OpenAI web search corroboration, 1,440 runs at ~0.03 plus tokens | ~45 |
| YouTube Data API | 0 |
| **Burn across 6 weeks** | **~86** |
| Budget with 2x headroom for retries and historical re-runs | ~175 |

Monthly run rate is therefore around 60 dollars, or about 120 with headroom. Double
all of it if 80 prompts is per client rather than across both. Separately, DataForSEO
requires a 50 dollar minimum deposit, which is credit rather than burn.

Drop Option A corroboration and the pilot costs about 40 dollars total. I think the
45 dollars is worth spending.

## 6. Recommended schema

Departures from the shape in the brief are marked and justified.

```sql
create table clients (
  id uuid primary key default gen_random_uuid(),
  name text not null,
  created_at timestamptz not null default now()
);

-- NEW. A client is not one domain. Citations may land on the marketing site, a docs
-- subdomain, the paired landing pages or the YouTube channel. is_client_domain has
-- to be a lookup, not a string compare against a single field.
create table client_domains (
  id uuid primary key default gen_random_uuid(),
  client_id uuid not null references clients(id),
  domain text not null,          -- stored normalised, no scheme, no www
  kind text not null,            -- primary | docs | landing | youtube_channel | other
  unique (client_id, domain)
);

create table prompts (
  id uuid primary key default gen_random_uuid(),
  client_id uuid not null references clients(id),
  text text not null,
  intent_category text not null,
  is_active boolean not null default true,
  created_at timestamptz not null default now()
);

create table assets (
  id uuid primary key default gen_random_uuid(),
  client_id uuid not null references clients(id),
  youtube_url text not null,
  video_id text not null,
  cohort text not null check (cohort in ('treated','control')),
  treatment_applied_at timestamptz,       -- null for control
  landing_page_url text,                  -- NEW, the paired page is a citable target
  unique (client_id, video_id)
);

-- NEW, and the most important addition. Cohort is a property of the ASSET, but
-- appearance rate is measured per PROMPT. Without an explicit eligibility mapping
-- there is no fair denominator: we would be comparing a treated video against
-- prompts it was never a plausible answer to. This table is what makes
-- treated vs control a like for like comparison rather than an artefact.
create table prompt_assets (
  prompt_id uuid not null references prompts(id),
  asset_id uuid not null references assets(id),
  primary key (prompt_id, asset_id)
);

-- NEW. Groups the runs a single scheduled sweep intended to make, so partial
-- failure is visible as "we planned 240 and completed 231" rather than silently
-- shrinking the denominator.
create table run_batches (
  id uuid primary key default gen_random_uuid(),
  scheduled_for timestamptz not null,
  week_index int not null,
  planned_count int not null,
  created_at timestamptz not null default now()
);

create table runs (
  id uuid primary key default gen_random_uuid(),
  batch_id uuid references run_batches(id),
  prompt_id uuid not null references prompts(id),
  surface text not null,          -- perplexity_sonar | google_ai_overview |
                                  -- chatgpt_consumer | openai_websearch | gemini
  provider text not null,         -- who we actually bought it from
  status text not null,           -- ok | surface_absent | provider_error | timeout
  attempt int not null default 1,
  surface_present boolean,        -- NEW and critical, see section 7
  raw_response jsonb not null,    -- always stored, even on error
  request_params jsonb not null,  -- context size, locale, model version, etc
  error text,
  run_at timestamptz not null default now()
);

create index on runs (prompt_id, surface, run_at);

-- Citations are DERIVED, not source data. Keying them by parser_version means a
-- change to extraction logic re-parses history into a new version alongside the old
-- one, rather than destroying the previous extraction. This is what makes
-- "re-parse historically" actually safe.
create table citations (
  id uuid primary key default gen_random_uuid(),
  run_id uuid not null references runs(id),
  parser_version int not null,
  url text not null,
  domain text not null,
  is_client_domain boolean not null,
  matched_asset_id uuid references assets(id),
  match_method text,              -- video_id | landing_page | domain_only
  position int,                   -- rank within the citation list where available
  unique (run_id, parser_version, url)
);

create index on citations (matched_asset_id, parser_version);
```

Two schema notes worth arguing about in review:

- `runs.raw_response` as `jsonb` rather than a separate blob table. At 4,320 runs the
  volume is small enough that inline jsonb is simpler and queryable. If we scale past
  a pilot this wants to move to object storage with a pointer.
- `surface_present` is nullable on purpose. Null means we do not know, which is
  different from false.

## 7. Top 3 things most likely to break

**1. Statistical power. The pilot as specified cannot support per prompt claims.**

Three runs per prompt per week for six weeks is 18 observations per prompt per
surface. Against a background of 40 to 60 percent monthly drift in cited domains, 18
observations cannot distinguish a real lift from noise at the prompt level. Going
from a 10 percent to a 25 percent appearance rate needs roughly 100 observations per
arm to detect reliably. Pooled across a 40 prompt cohort we have around 720
observations per surface per client, which is fine. Per prompt we do not.

This is not an argument against the pilot. It is an argument that the comparison view
must be built cohort first, with per prompt rows shown as exploratory detail carrying
an explicit insufficient sample state, and that we should report the difference in
differences (treated change minus control change) rather than a treated versus
control endpoint gap. I would encode a hard rule now: never render a lift number
without a Wilson confidence interval next to it, and grey out any per prompt cell
below a configured minimum n.

**2. Denominator corruption on AI Overviews.**

AI Overviews do not fire for every query, and whether they fire varies by location,
by personalisation and by Google's own triggering thresholds, which move. A prompt
that returned an AI Overview in week 1 and returns none in week 4 has not lost a
citation. The surface disappeared. If those two cases collapse into the same "not
cited" bucket, the appearance rate is measuring Google's triggering rate rather than
our client's visibility, and the treated versus control comparison inherits that
noise.

This is why `surface_present` is in the schema and why it is nullable. Appearance
rate must be computed as cited divided by runs where the surface was present, with
the surface presence rate reported alongside it as its own series. If AIO presence
itself trends over the six weeks, we need to see that.

**3. Silent degradation of the ChatGPT scraper.**

An outage is easy: the request fails, we retry, we log a partial failure. The
dangerous failure is the provider returning HTTP 200 with a plausible answer and an
empty citations array because a DOM selector changed upstream. That reads as "nobody
was cited this week" and it is indistinguishable from a real result unless we test
for it.

Mitigation is a canary: one or two prompts per sweep whose answer reliably cites a
known stable domain. If the canary returns zero citations, the sweep is marked
suspect and excluded from aggregates until a human looks. Cheap to build, and it is
the difference between finding out in week 1 and finding out at the readout.

## 8. Open questions for you

1. Is 80 prompts the total across both clients, or 80 each? It doubles the cost,
   which still barely matters, but it changes the power calculation materially.
2. Will the clients grant YouTube OAuth on their channels? If not, the transcript
   ingestion path changes to residential proxies and I would want to know now.
3. Do you want the OpenAI web search corroboration surface, at roughly 45 dollars
   for the pilot, or should we run ChatGPT on the consumer scraper alone?
4. Locale. AI Overviews are location sensitive. We need to pin one locale per client
   and hold it fixed, and I need to know which.
