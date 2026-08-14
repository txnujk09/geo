# Phase 1: surface access diagnosis

Scope: can we programmatically sample citations from Perplexity, Google AI Overviews and
ChatGPT often enough, cheaply enough and reliably enough to make a defensible lift claim
over a 6 week pilot. No application code in this phase.

> **Superseded in part by `phase-1-measurement-design.md`.** That document revises three
> recommendations here after deeper research into base rates and sampling variance. The
> access findings below all still stand. The changed positions are: ChatGPT drops to an
> exploratory surface accessed only via the official OpenAI API (do not build the Bright
> Data scraper), the three weekly runs become three paraphrase variants rather than three
> repeats, and sampling should not be raised from 3 to 5. Read both, that one second.

Assumed volume: 80 prompts x 3 surfaces x 3 runs per week x 6 weeks = 1,440 runs per
surface, 4,320 runs total, roughly 1,040 runs per surface per month. If 80 prompts turns
out to mean 80 per client rather than 80 across both, double every number below. Nothing
in the cost analysis changes qualitatively, because we are nowhere near a cost ceiling.

---

## 1. Perplexity Sonar API

**Verdict: yes, this is the one genuinely solved surface. Use it directly.**

Sonar returns citations as structured top level fields on the chat completion response,
not as text we have to regex out of the answer body. Two fields matter:

- `citations`: a flat array of URL strings.
- `search_results`: the richer array, with `title`, `url` and `date` per entry. This is
  what we should persist and parse. Prefer it over `citations`, which is the older and
  thinner field.

Relevant request parameters: `search_domain_filter` (we should NOT use it, since filtering
to the client domain would destroy the measurement), `web_search_options.search_context_size`
(low / medium / high, this is the main cost and recall lever), and `response_format` for
JSON schema structured output. If we stream, note that search results arrive only in the
final chunk, so streaming buys us nothing here and we should not use it.

**The caveat that matters, and it is a real one.** The Sonar API is not the same surface as
consumer perplexity.ai. Perplexity's own community forum carries multiple threads reporting
citation set divergence and even different factual conclusions between the API and the web
UI for identical queries. One published comparison put overlap between API responses and the
rendered UI at roughly 20 percent. The API also drops the sources panel and related question
fan out that drive actual user visibility.

Implication for the schema: do not call this surface `perplexity`. Call it
`perplexity_sonar_api` and say so on the comparison view. We are measuring citation behaviour
of Perplexity's search stack via its API, which is a reasonable proxy and is defensible as
long as we never claim it is what a human user sees. If the clients push back on that, the
fallback is a scraped consumer Perplexity surface added later as a second, separately labelled
surface. I would not build that in Phase 2.

**Cost:** per request search fee by context size, roughly $5 per 1k requests (low), $8 (medium),
$12 (high) on base Sonar, plus token cost at about $1 per million in and out. At medium context:
1,440 x $0.008 = about $12, plus roughly $2 of tokens. **About $14 for the whole pilot.**

---

## 2. Google AI Overviews

No official API exists. All three candidates are scrapers wearing an API, and the failure mode
we care about is not downtime, it is silent under reporting: an AI Overview was present but the
parser missed it, and we record zero citations. That is indistinguishable from a true negative
unless we design against it.

| Provider | Rate | Pilot cost (1,440 queries) | Notes |
|---|---|---|---|
| DataForSEO | $0.0012 standard queue, $0.0024 priority, $0.004 live | **$2 to $6** | Cheapest by an order of magnitude, pay as you go, no subscription. Live mode returns in about 6 seconds. |
| Bright Data | $1.50 per 1k pay as you go, $1.30 at $499/mo scale plan | **about $2**, or $0 inside the 5k/month free tier | Dedicated AI Overview endpoint. Not billed for failed requests. Watch for enterprise minimums on some products. |
| SerpApi | $25/mo for 1k up to $275/mo for 30k, so $9 to $25 per 1k | **$150** (Developer plan, 2 monthly cycles) | Most mature parser and best documentation. Subscription only, and exhausting the allowance forces an early renewal at full plan price. |

**The lazy load trap, specific to SerpApi and worth understanding for all three.** For less
common queries Google lazy loads the AI Overview. SerpApi's initial search response then
returns only a short lived `page_token`, and the actual overview requires a second call to the
`google_ai_overview` engine. That second call bills as a second search credit, so real cost is
up to 2x the table above. The token expires in about 1 to 4 minutes, so a queued or retried
fetch will simply fail. Any retry policy that waits before retrying is wrong for this specific
call and must refetch from the beginning instead.

**Recommendation: DataForSEO as the primary for all 1,440 runs, SerpApi on a 10 percent audit
sample.** Cost is so low here that the interesting question is not price, it is whether we can
trust a single parser. Running a second provider over roughly 144 duplicate queries per pilot
gives us a measured provider disagreement rate, which is the only way to know whether a flat
week in the data is a real flat week or a parser regression. Audit sample cost: SerpApi Starter
at $25/mo x 2 cycles = $50.

**AI Overview cost total: about $55 for the pilot.**

---

## 3. ChatGPT

This is the weakest surface and there is no clean answer. Four options, in descending order of
fidelity to what a human user actually sees.

**Option A: third party scraper of the consumer ChatGPT surface** (Bright Data ChatGPT Scraper,
DataForSEO `ai_optimization/chat_gpt/llm_scraper`). Returns structured JSON with the answer plus
sources and citations from ChatGPT web search, which is the actual surface we care about.
Bright Data's free tier is 5,000 records per month, which covers our entire 1,440 run volume;
paid is roughly $1.50 per 1k. DataForSEO's LLM Scraper starts around $0.0012 per results page
plus token and feature fees, with live execution up to 120 seconds. Money cost: **effectively
$0 to $30 for the pilot.** Fragility cost: high. These are unofficial, there is no SLA, OpenAI
can change the surface at any time, and response variance from session and personalization
state is not something the vendor controls.

**Option B: OpenAI Responses API with the `web_search` tool.** Officially supported, returns
`url_citation` annotations with URL, title and character offsets. Costs roughly $25 to $50 per
1k tool calls depending on search context size, so about $43 for the pilot. Rock solid
availability. But it is explicitly not consumer ChatGPT: different retrieval stack, different
ranking, no memory or personalization layer. Using it alone would mean the deliverable claims
lift on a surface no customer of our clients uses.

**Option C: drive a logged in browser ourselves.** Zero API cost, maximum fidelity in principle,
but it violates OpenAI's terms of use, risks the account, and is the single highest maintenance
component we could possibly choose. Not recommended.

**Option D: drop ChatGPT.** Honest, and worth keeping on the table if Option A proves flaky in
week 1, but it guts the deliverable since ChatGPT is the surface both clients will ask about first.

**Original recommendation: run A and B as two separately labelled surfaces, never merged.**
`chatgpt_web` (scraped, high fidelity, expected to fail sometimes) and `chatgpt_api` (official,
boring, always works).

> **Revised: build only Option B.** ChatGPT cites YouTube in roughly 3 to 9.5 percent of
> citations, against 29.5 percent for AI Overviews. For a video pilot at this sample size,
> ChatGPT cannot support a significance claim no matter how faithfully we scrape it, so the
> fidelity gained from Option A buys nothing statistically while carrying all of the
> fragility. Ship `chatgpt_api` only, label it exploratory, and revisit the scraper only if
> the clients specifically ask for consumer ChatGPT numbers. See
> `phase-1-measurement-design.md` section 1 for the base rates and the power argument.

If both surfaces are ever built, the comparison view must label them distinctly and must never
sum them into one "ChatGPT" number.

---

## 4. YouTube Data API

**Metadata: yes, straightforward.** `videos.list` with `part=snippet,contentDetails,statistics`
costs 1 quota unit per call and returns title, full description, publish date and duration for
any public video. 100 videos is 100 units. Trivial.

**Chapters: not an API field.** YouTube has no chapters endpoint. Chapters are derived by
YouTube from timestamp lines in the description. We parse them ourselves from the description
text (first timestamp must be `00:00`, minimum 3 timestamps, ascending, each segment at least
10 seconds). This is actually good news for the pilot, because chapter presence is one of our
treatment variables and parsing the description is exactly how we verify the treatment landed.

**Transcripts: this is the one real obstacle, and it has a clean solution specific to our
situation.** `captions.download` requires OAuth 2.0 where the authenticated user owns the
video. For a third party public video it returns 403. There is no official way to pull
auto generated transcripts for videos you do not own.

But these are our clients' own videos. **The correct path is to have each client grant OAuth
access as the channel owner**, after which `captions.list` (50 units) and `captions.download`
(200 units) are fully legitimate for both the treated and control cohorts. That is 250 units
per video. At 100 videos that is 25,000 units against a 10,000 unit per day default quota, so
we spread the initial backfill over three days or file for a quota increase. Ongoing cost after
backfill is near zero, since transcripts only need refetching when we change a video.

Avoid `search.list` entirely: it costs 100 units per call and we have no need for it.

**Fallback for anything we cannot get owner OAuth for** (competitor videos, a client who will
not grant access): the `youtube-transcript-api` library, which reverse engineers an internal
endpoint. Be aware that YouTube now blocks most known cloud provider IP ranges (AWS, GCP,
Azure, DigitalOcean) by ASN, so this returns `RequestBlocked` or `IpBlocked` from Vercel by
default and needs rotating residential proxies to work in production. Treat this as a manual,
run it locally, best effort path. Do not put it on the cron.

---

## Recommended providers and cost summary

| Surface | Provider | Pilot cost |
|---|---|---|
| `perplexity_sonar_api` | Perplexity Sonar, medium search context | $14 |
| `google_ai_overview` | DataForSEO live, primary | $6 |
| `google_ai_overview` audit | SerpApi, 10 percent duplicate sample | $50 |
| `chatgpt_web` | Bright Data ChatGPT Scraper | $0 to $30 |
| `chatgpt_api` | OpenAI Responses API `web_search` | $43 |
| Response parsing | Gemini 2.5 Flash / Flash Lite over 4,320 responses | about $8 |
| **Total pilot (6 weeks)** | | **about $125 to $155** |
| **Monthly run rate** | | **about $85 to $105** |

Provider API cost is not a meaningful constraint on this project. What the numbers are actually
telling us is that we should spend freely on redundancy: duplicate providers, more runs per
prompt, re-running anything that smells wrong. The binding constraints are statistical power
and scraper fragility, not budget.

One caveat on the ChatGPT and Bright Data figures: those rest on published list rates and free
tier allowances that I could not verify against the vendors' live pricing calculators from this
environment, and Bright Data has historically applied plan minimums to some products. Confirm
before committing, but the exposure is small enough that even a 5x miss does not change the plan.

---

## Recommended schema

Changes from the shape in the brief are marked. The organizing principle: raw responses are
immutable and stored forever, parsing is versioned and re-runnable, and a provider failure is
never confusable with a true negative.

```
clients
  id, name, primary_domain, additional_domains text[], created_at

prompts
  id, client_id, text, intent_category, is_active, created_at

assets
  id, client_id, youtube_url, video_id, cohort ('treated' | 'control'),
  treatment_applied_at, landing_page_url,
  -- CHANGED: cached YouTube metadata so we can verify the treatment landed
  title, published_at, description, transcript_text, chapters jsonb,
  metadata_fetched_at

run_batches                                        -- NEW
  id, scheduled_for, week_index, status, started_at, finished_at
  -- groups the runs from one cron firing so a partial failure is visible as a unit

runs
  id, batch_id, prompt_id, surface, provider, provider_request_id,
  -- CHANGED: status is the single most important column in this schema
  status ('ok' | 'no_answer' | 'provider_error' | 'timeout' | 'blocked'),
  raw_response jsonb, raw_response_hash, http_status, error_text,
  latency_ms, cost_usd, run_at
  -- raw_response is never deleted and never overwritten

citation_extractions                               -- NEW
  id, run_id, parser_version, extracted_at, model_used
  -- one row per (run, parser version); re-parsing adds rows, never destroys them

citations
  id, extraction_id, run_id, url, normalized_url, domain, position,
  is_client_domain, matched_asset_id, match_method ('video_id' | 'landing_page' |
  'domain_only' | 'none'), created_at
```

Three deliberate decisions:

1. **`runs.status` drives the denominator.** Appearance rate counts only `status = 'ok'` runs in
   the denominator. A run where the provider 500'd is not a run where we were not cited, and
   collapsing those two is the single easiest way to fabricate a fake lift or hide a real one.
   For Google specifically we need to separate "no AI Overview was shown for this query" from
   "the provider failed", which is why `no_answer` is its own status rather than an `ok` with
   zero citations.
2. **`citation_extractions` makes re-parsing non destructive.** When we improve extraction logic
   we bump `parser_version`, re-run over stored `raw_response` values, and get a new set of rows.
   Old and new results stay side by side and comparable. Views read the current parser version
   by default.
3. **`match_method` on citations** records how confidently we tied a citation to an asset. A
   YouTube URL containing the video ID is a hard match. A hit on the client's root domain is not
   the same thing and should not be presented as if it were.

---

## Top 3 things most likely to break

**1. Statistical power, which is a design problem and not a bug.** 3 runs per week for 6 weeks
gives 18 observations per prompt per surface. That is nowhere near enough to claim a difference
at the per prompt level: detecting a move from 10 percent to 30 percent appearance at 80 percent
power needs roughly 60 observations per arm, and a 10 to 25 percent move needs well over 100.
Pooled at the cohort level (40 treated assets x 18 runs = 720 observations) the numbers work
fine. Two consequences for Phase 2. First, the comparison view's headline claim must be cohort
level, with per prompt rows shown as exploratory detail carrying explicit "insufficient sample"
badges. Second, runs within a prompt are correlated, so naive binomial confidence intervals will
be too narrow; we need to cluster by prompt or the intervals will overstate our certainty.

> **Revised on the sampling rate.** I initially wanted to raise this to 5 runs per week. The
> evidence says do not: the marginal repeat past the fifth is worth almost nothing, and
> paraphrase variants buy far more effective sample for the same money. Keep 3 per week and
> make them 3 fixed paraphrase variants. See `phase-1-measurement-design.md` section 2.

**2. Silent AI Overview under reporting.** Google lazy loads AI Overviews, serves them on only
about half of queries, and renders them via JavaScript. Every failure mode here (expired
`page_token`, parser drift after a Google layout change, overview genuinely absent) lands in
the data as "zero citations". If a parser regression hits mid pilot it looks exactly like the
treatment stopped working. Mitigations: the `no_answer` versus `provider_error` split above,
the 10 percent SerpApi cross check to detect parser drift, and a per week dashboard line
showing what fraction of Google runs returned any AI Overview at all. If that fraction moves
sharply, we investigate before we interpret.

**3. The ChatGPT scraper surface.** Unofficial, no SLA, subject to OpenAI changing its surface
or blocking the vendor, and carrying uncontrolled response variance from session state. Expect
it to break at least once in 6 weeks. Mitigations: the parallel `chatgpt_api` surface as a
continuity floor, per surface isolation in the scheduler so a ChatGPT failure never aborts the
Perplexity or Google runs in the same batch, and prominent gap marking on the comparison view
rather than interpolation across missing weeks.

Honourable mention: the Perplexity API versus UI divergence covered in section 1. It will not
break, it is just quietly not measuring what a stakeholder will assume it measures, which is
arguably worse. Label it explicitly in the UI.

---

## What I need before Phase 2

1. Confirmation of the provider picks: DataForSEO primary plus SerpApi audit for Google,
   Bright Data plus OpenAI for the two ChatGPT surfaces, Sonar for Perplexity.
2. Whether 80 prompts is total or per client.
3. Whether we can get channel owner OAuth from both clients. If not, the transcript path
   changes materially and I would want to solve it before building ingestion.
4. A decision on raising sampling from 3 to 5 runs per prompt per week.
5. Whether Gemini is in or out as a fourth surface. It is cheap to add via DataForSEO's
   LLM Scraper, which supports ChatGPT and Gemini today.

Stopping here for confirmation before writing any application code.
