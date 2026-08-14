# Phase 1b: measurement design, and a revised opinion

Companion to `phase-1-diagnosis.md`. That document answered "can we get the data".
This one answers "will the data be able to prove anything", which turns out to be the
harder question and changes three of the recommendations.

Short version of what changed after deeper research:

1. Google AI Overviews should be the primary endpoint and ChatGPT should be demoted to
   exploratory. This is driven by base rates, not by engineering convenience, and it
   happens to delete the most fragile component in the whole system.
2. The three weekly runs should be three fixed paraphrase variants of each prompt, not
   three repeats of identical text. Same cost, materially more statistical power, and it
   measures the right thing.
3. Prompt intent mix is a hard gate that has to be cleared before week 1, not a reporting
   dimension. A commercial heavy prompt set makes the Google surface structurally unable
   to produce data.

Plus one addition that costs nothing and roughly doubles the credibility of the result:
run weeks 1 and 2 as a pre treatment baseline and analyse as difference in differences.

---

## 1. Base rates decide which surface can prove anything

YouTube's share of citations differs by an order of magnitude across the three surfaces.

| Surface | YouTube share of citations | Source |
|---|---|---|
| Google AI Overviews | 29.5 percent, the single most cited domain, ahead of Wikipedia | Otterly YouTube AI Citation Study, Mar 2026 |
| Perplexity | 22.7 percent | same |
| ChatGPT | 9.5 percent share, roughly 3 percent of top 10 citations | same |

Otterly's cut of where YouTube citations originate is starker still: Perplexity 38.7
percent, Google AI Overviews 36.6 percent, ChatGPT 4.4 percent.

**Read carefully: these are upper bounds on the opportunity, not our expected appearance
rate.** "29.5 percent of AI Overview citations point at some YouTube video" is a very
different claim from "a specific client video appears in 29.5 percent of runs". Our real
per run appearance rate for one client's one video will plausibly sit somewhere in the 1
to 10 percent range. The base rates matter because they set the ceiling and the ordering,
and the ordering is unambiguous: this is a video pilot, and ChatGPT barely cites video.

**Consequence.** On ChatGPT we would be trying to detect a lift on a surface where the
entire addressable citation pool for our content type is about 3 percent. Even a doubling
of a 3 percent rate needs roughly 750 independent observations per arm at 80 percent
power, and our observations are clustered within prompts, which inflates that requirement
by a factor of roughly 2 to 3. We will have 1,440 runs per surface. On ChatGPT that is not
close to enough. On Google, where the pool is ten times larger, it is workable.

**So: do not build the fragile thing.** My Phase 1 recommendation was Bright Data's ChatGPT
scraper as a high fidelity primary plus the OpenAI API as a fallback. I now think that is
backwards effort allocation. Build only the cheap official `chatgpt_api` surface via the
Responses API `web_search` tool, label it exploratory, and spend the saved engineering time
on Google and Perplexity where the pilot can actually win. Revisit the scraper only if the
clients specifically demand consumer ChatGPT numbers, and if they do, tell them up front
that the sample cannot support a significance claim on that surface.

This removes the single most likely thing to break from the critical path.

---

## 2. Repeated identical prompts are the wrong way to spend the sampling budget

The published evidence on this is unusually clear and unusually consistent.

**Run to run instability is severe and is not caused by index churn.** Schulte, Bleeker and
Kaufmann (arXiv 2604.07585, University of St. Gallen) ran 8 prompts per campaign across
four Swiss verticals against ChatGPT, Perplexity, Gemini and Google AI Mode from January to
March 2026, up to 10 runs per engine and prompt pair, 3,409 pairwise comparisons. Overlap
between repeated runs on cited sources was **32 to 43 percent**. Brand mention overlap day
to day averaged 45 to 59 percent. Identical prompts executed minutes apart under identical
conditions diverge substantially. That is model stochasticity, not drift.

Note that this is a different and larger problem than the 40 to 60 percent monthly domain
drift figure in the brief. The brief's premise (one query proves nothing) is correct, and
the real number is worse than assumed: it is not just that the cited set drifts month over
month, it is that it barely reproduces minute to minute.

**But more repeats stop helping quickly.** Evertune's repetition analysis puts the margin of
error on a single prompt's measured rate at roughly **plus or minus 27 points at 5 runs,
plus or minus 12 points at 12 runs, and plus or minus 6 points at 100 runs**. The
variance components paper (arXiv 2607.13304) reaches the same conclusion from theory: a
repeat past the fifth reduces relative error variance by about 0.0003, while adding
paraphrases, models or languages reduces it far more per unit of query budget.

**Paraphrases buy far more effective sample than repeats do.** Paraphrases of the same
buying intent produce only 14 to 29 percent recommendation overlap (arXiv 2605.27440), which
is *lower* overlap than repeated identical prompts. Three phrasings of one question behave
much more like three separate prompts than like three runs of one prompt, because they draw
from different retrieval clusters.

**Recommendation: three fixed paraphrase variants per prompt intent, one run each per week.**

Identical request volume, identical cost, no change to the cron. What changes:

- Effective sample size rises substantially, because variants are near independent draws
  while repeats are correlated draws.
- External validity rises. A single phrasing measures the rate at which a brand is cited in
  response to that literal string, which is not a proxy for visibility across the way real
  buyers actually phrase the question.
- The variant set is fixed across weeks and across both cohorts, so it is a controlled
  factor rather than a new source of noise. Variant becomes a random effect in the model.

Schema implication: `prompts` becomes prompt intents, and a new `prompt_variants` table
holds the three phrasings. `runs` references `prompt_variant_id`, not `prompt_id`.

I would still keep the total at 3 per week rather than pushing to 5. The Schulte paper's
own recommendation is a minimum of three measurements per query per platform over a 7 day
window, which we meet exactly, and the marginal repeat is demonstrably low value. If there
is appetite to spend more, spend it on more prompt intents, not more runs.

---

## 3. Prompt intent mix is a gate that must be cleared before week 1

This is the finding most likely to quietly kill the pilot, and it is entirely preventable.

AI Overviews do not trigger on every query, and the trigger rate varies by an order of
magnitude with intent and phrasing:

| Query type | AI Overview trigger rate |
|---|---|
| Comparison, "X vs Y" | 95.4 percent |
| Review queries | 86.3 percent |
| Question format | 85.9 percent |
| B2B technology queries (aggregate) | 82 percent |
| Informational (general) | 36 percent, range 35 to 65 |
| Commercial | 8 percent |
| Transactional | 5 percent |

Sources: Seer Interactive analysis of 49,353 queries, Semrush commercial intent study,
BrightEdge tracking putting overall AIO presence at roughly 48 percent of queries.

**Do the arithmetic on the bad case.** If the 80 prompts are written the way marketing teams
usually write them, heavy on commercial intent ("best X software", "X pricing"), the Google
surface triggers an overview on roughly 8 percent of runs. Of 1,440 Google runs, about 115
would return an overview at all. The other 1,325 are correctly recorded as `no_answer`, and
the primary endpoint of the pilot has about a hundred usable observations spread across 80
prompts and two cohorts. Nothing is provable from that.

In the good case, a prompt set skewed to question and comparison formats triggers on 85 to
95 percent of runs, and we get 1,200 or more usable Google observations.

**That is a 10x swing in usable sample driven entirely by how the prompts are worded**, and
it is decided before a single line of ingestion code runs.

**Recommendation: a one off prompt qualification run before week 1.** Fire all 80 prompt
intents (all 240 variants) at Google once, record the AI Overview trigger rate per prompt,
and drop or rewrite anything below a threshold of roughly 40 percent. At DataForSEO rates
this costs about **$1**. It is the highest return dollar in the entire project.

Two secondary consequences:

- `intent_category` is not just a reporting dimension, it is a confounder. If treated assets
  happen to map to more question format prompts than control assets do, we will measure a
  large fake lift. Intent mix must be balanced across cohorts and the balance must be
  checked and reported, not assumed.
- Prompts that never trigger an overview still cost money every week and contribute nothing.
  Qualification pays for itself immediately.

---

## 4. Run weeks 1 and 2 as a pre treatment baseline

Currently the design compares a treated cohort against a matched control cohort. That rests
entirely on the cohorts being genuinely matched, and given the citation variance documented
above, "matched" on subscriber count and topic is not the same as matched on baseline
citation propensity. A hostile reader will point at any observed gap and say the treated
videos were simply better to begin with.

**Fix: do not apply the treatment until week 3.** Measure both cohorts unchanged for two
weeks, apply treatment, measure four more weeks, and analyse as difference in differences.
The claim becomes "the treated cohort's appearance rate rose relative to its own baseline by
more than the control cohort's did", which survives baseline imbalance between cohorts and is
the standard defence against exactly this objection.

Cost: zero. It is the same 6 weeks of measurement. It only requires that treatment rollout
is scheduled rather than already done. `assets.treatment_applied_at` is already in the
schema, so the model can express this, but it needs the operational decision now, because
once the videos are optimised the baseline is unrecoverable.

If treatment has already been applied to some assets, say so and I will restructure around a
staggered rollout instead, which is weaker but still usable.

---

## 5. What the analysis actually has to be

Three points, in order of how much they change the deliverable.

**The design is paired, not two independent samples.** A single run of one prompt on one
surface is simultaneously an opportunity for a treated asset to be cited and for a control
asset to be cited. Treating treated and control appearance rates as two independent samples
throws away that pairing and understates power considerably. The correct model is a mixed
effects logistic regression: cohort and period as fixed effects with their interaction as the
estimand, prompt intent and asset as random effects. Discordant runs, where one cohort is
cited and the other is not, carry the signal.

**Clustering is not optional.** Runs within a prompt are strongly correlated. Naive binomial
confidence intervals will be far too narrow and will manufacture significance that is not
there. Cluster by prompt intent. Expect the design effect to inflate required sample by
roughly 2 to 3x relative to the independent case.

**Per prompt weekly claims are impossible and the UI must say so by default.** At 3 runs the
margin of error on a single prompt's weekly rate is around plus or minus 27 to 35 points.
There is no effect this pilot could plausibly produce that is detectable at that resolution.
The brief asks for "a clear callout when sample size is too small to claim a difference".
Based on this evidence, that callout is not an edge case, it is the **default state of every
per prompt view**, and only the pooled cohort level view over the full window should ever
render a confidence interval and a claim.

Concretely, for the comparison view:

- Headline: cohort level pooled appearance rate, treated vs control, pre vs post, with
  clustered 95 percent CIs and the difference in differences estimate.
- Per prompt rows: sparklines and raw counts only. No percentages with implied precision, no
  significance markers, an explicit "n too small for a claim" badge.
- Always visible: `n_eff`, the number of `ok` runs, and the number of runs excluded as
  `provider_error`. Following MaxAEO's reporting convention, every figure should carry point
  estimate, interval, effective sample, design, engine and window.

---

## 6. Two smaller findings that affect the schema

**AI Overviews cite YouTube at the chapter level, not the video level.** The retrieval path
is: pull the full transcript of candidate videos, segment by chapter marker or sentence
boundary, match each segment against the query, and cite the best matching segment with its
timestamp attached. Roughly 78 percent of timestamped videos are cited across 2 to 5
distinct chapters, meaning each chapter is a separate citation surface.

Two implications. First, citation URLs will carry `&t=` timestamp parameters, so URL
normalisation must strip the timestamp for asset matching but **retain it in a separate
column**, because which chapter got cited is a direct measure of whether the chaptering
treatment worked. Second, "number of distinct chapters cited" is a better and more sensitive
outcome measure than the binary "video was cited", and it is free to compute if we store the
timestamp. I would add it as a secondary endpoint.

**Videos under about 4 minutes are structurally uncitable.** Reported figures put 94 percent
of YouTube AI citations on long form video, with Shorts and sub 4 minute clips functionally
invisible. Any Short in either cohort is a guaranteed zero that adds noise and dilutes the
estimate. Add a minimum duration criterion to cohort selection and exclude Shorts from both
arms. `contentDetails.duration` comes back from the same 1 unit `videos.list` call we are
already making.

---

## 7. Revised recommendation summary

| Decision | Phase 1 position | Revised position |
|---|---|---|
| Primary surface | all three equal | **Google AI Overviews primary, Perplexity secondary, ChatGPT exploratory** |
| ChatGPT access | Bright Data scraper primary, OpenAI API fallback | **OpenAI Responses API only. Do not build the scraper.** |
| Sampling | 3 identical runs per week, consider raising to 5 | **3 fixed paraphrase variants per week, do not raise to 5** |
| Prompt set | assumed given | **qualify against AIO trigger rate first, about $1, drop anything below 40 percent** |
| Timeline | 6 weeks treated vs control | **2 weeks baseline, then 4 weeks treated, difference in differences** |
| Analysis | appearance rate with CIs | **mixed effects logistic, clustered by prompt, paired within run** |
| Per prompt claims | flagged when small | **never claim at prompt level, this is the default state** |
| Cohort selection | matched control set | **plus minimum duration, exclude Shorts, balance intent mix across cohorts** |

Cost is essentially unchanged and still trivial, and dropping the ChatGPT scraper reduces it
slightly. Removing the scraper from the build also removes the highest maintenance component,
so the revised plan is cheaper in engineering time as well as in money.

---

## 8. Revised top 3 risks

1. **The prompt set never triggers AI Overviews.** 10x swing in usable sample, decided by
   wording, invisible until the data is already collected. Mitigated entirely by the
   qualification run. This is now risk number one because it is both the most damaging and
   the most preventable.
2. **The pilot returns a true null and nobody can tell it apart from a measurement failure.**
   Given the variance documented above, a genuinely effective treatment can easily produce a
   flat looking chart at this sample size. Mitigations: the paired analysis, the pre period
   baseline, chapter level citation counts as a more sensitive secondary endpoint, and the
   provider cross check so a parser regression is never mistaken for a flat week.
3. **Silent AI Overview under reporting.** Unchanged from Phase 1. Lazy loading, short lived
   `page_token`, JavaScript rendering and parser drift all land in the data as zero citations.
   The `no_answer` versus `provider_error` split, the 10 percent SerpApi cross check, and a
   weekly "fraction of Google runs that returned any overview" line remain the mitigations.

The ChatGPT scraper has dropped off this list because we are no longer building it.

---

## 9. Open questions, revised

Unchanged and still blocking for Phase 2:

1. Is 80 prompts total, or 80 per client?
2. Can we get channel owner OAuth from both clients for transcripts?

New and now more important than either of the above:

3. **Has treatment already been applied to any assets?** If not, hold it until week 3 and we
   get difference in differences for free. If it has, tell me which and I will restructure.
4. **Can I see the prompt list?** If it is commercial intent heavy, that is a bigger problem
   than anything else in either document, and rewriting prompts is cheaper before the pilot
   starts than after.
5. Do the clients need consumer ChatGPT numbers specifically? If yes, we build the scraper
   knowing it cannot support a significance claim. If no, we skip it.

Superseded: the question about raising sampling from 3 to 5 runs. The evidence answers it.
Keep 3, make them paraphrase variants.

Still stopping here for confirmation before any application code.
