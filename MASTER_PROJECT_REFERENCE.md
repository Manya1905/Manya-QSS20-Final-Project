# COVID Food Trends Project: Master Reference Document

This document is a full snapshot of the project's structure and design as of now. It is meant to
be handed to a new conversation/assistant (or read by future-you) to pick up exactly where things
left off, with no missing context. **This document summarizes results and points to
`ENGLISH_ONLY_RESULTS.md` for full detail** (complete coefficient tables, full LDA topic lists,
full creator-network node/edge breakdowns, etc.) — that file is the one to work from when writing
the actual report; this file is the one to work from when you need to understand *why* the
project is shaped the way it is, what's actually been done, and what's left.

---

## ⚠️ SCOPE UPDATE (as of last update)

**The project's final scope is 4 core analyses.** Google Trends / "lasting impact" comparison and
the weekly lockdown-policy-timing analysis were both explored and produced real findings during
development, but have been **fully removed from the final write-up's scope**, not just
deprioritized, due to space constraints. Neither is documented in this file anymore; if either is
ever needed again, they were built in `code_english_only/02_weekly_lockdown_overlay.ipynb` and
`code_english_only/03_google_trends_comparison.ipynb` respectively (not part of the current
project structure below). The paper's actual framing now asks "how did this pandemic-era social
media phenomenon actually work": how virality varied, why, how far it extended, and who drove it.

**The project's actual, current scope is:**
1. Engagement/virality analysis (Method #5): did virality differ across trends?
2. Linguistic features predicting engagement (Method #6): why did it differ?
3. Undiscovered COVID-era food trend topic discovery (Method #7, LDA): how far did the shift
   extend beyond the 5 headline trends?
4. Creator-trend network analysis: who was actually driving each trend, and did the same
   creators cross between trends?

**All 4 analyses are now complete.** Trade data and USDA grocery retail data remain fully out of
scope. All actual project work lives in `code_english_only/`. All analysis is English-only (see
Section 4 for the reasoning already established for this).

---

## 1. The Project, In One Paragraph

During COVID lockdowns, cooking became a widespread coping mechanism, so widespread that it
produced real, documented economic ripple effects (flour, yeast, and sourdough starter shortages
that forced supply chains to adjust). Social media was many people's primary source of both
entertainment and information during this period, making it the arena where these food trends
actually played out, spread, and competed for attention. This project investigates that social
media phenomenon directly, using a professor-provided dataset of Instagram/Facebook posts (Dec
2019 to Dec 2020) covering five viral pandemic-era food trends (sourdough, banana bread, dalgona
coffee, baked oats, feta pasta): how virality varied across these trends, what about the content
itself drove that variation, how far the underlying shift extended beyond just these five trends,
and who, in terms of creator communities, was actually driving each one.

---

## 2. Research Question and Sub-Questions

**Main RQ:** How did the pandemic-era viral food trend phenomenon on social media actually work,
did virality vary across trends, what drove that variation, how far did the shift extend beyond
the most visible trends, and was it driven by one unified wave of creators or several distinct
communities?

**Sub-questions:**
1. **Virality across trends.** Does engagement (likes, comments) differ meaningfully across the
   five trends? (Method #5)
2. **What predicts virality, and does it differ by trend.** Do linguistic features from post text
   (sentiment, length, content type, tone) predict engagement, and does the *effect* of those
   features vary across the 5 trends? (Method #6, primarily via 5 separate per-trend models — see
   Section 9 for why a pooled combined model can't answer the "does it differ by trend" half of
   this question)
3. **Undiscovered trends.** Are there other COVID-era food trends in the broader dataset that the
   current five keyword lists missed? (Method #7, LDA)
4. **Creator structure.** Do the same creators post across multiple trends, or does each trend have
   its own distinct creator community? Does creator type (individual creator / business / personal
   account) relate to this? (creator-trend network analysis)

**Opening hook for the write-up:**
> "During COVID lockdowns, cooking became a widespread coping mechanism, so widespread that it
> produced real economic ripple effects: documented shortages of flour, yeast, and sourdough
> starter forced supply chains to adjust to a surge in home baking. And since social media was
> many people's primary source of both entertainment and information during this period, it
> became the arena where these trends actually played out, spread, and competed for attention.
> This project investigates that social media phenomenon directly: how virality varied across
> pandemic-era food trends, what made certain content resonate, how far the shift extended beyond
> the most visible trends, and who was actually driving it."

---

## 3. Trend Keyword Sets

```python
trend_keywords = {
    "feta_pasta": ["feta pasta", "baked feta pasta", "baked feta"],
    "sourdough": ["sourdough", "sourdough starter", "sourdough bread"],
    "banana_bread": ["banana bread", "banana bread recipe", "bananabread"],
    "baked_oats": ["baked oats", "baked oatmeal", "bakedoats"],
    "dalgona_coffee": ["dalgona coffee", "whipped coffee", "dalgona"],
}
covid_keywords = ["covid", "coronavirus", "pandemic", "quarantine", "lockdown"]
```
Matched against both `text` and `hashtags` columns (OR logic). Trend keywords kept as targeted
phrases, not broadened to single-word wildcards, to avoid pulling in unrelated year-round cooking
content.

---

## 4. Data Sources: Full Status

### Layer 1: Instagram/Facebook discourse
- Status: **Done.** Loaded, cleaned, filtered into 5 trend DataFrames, COVID-framing flag applied.
- No geographic/location field; national-level analysis only.
- **Restricted to English-only for all analysis** (`lang == "en"`).

### Layer 2: Creator-trend network
- Status: **Done.** Built and run in `code_english_only/04_creator_trend_network.ipynb`, per
  `CREATOR_TREND_NETWORK_STEP_BY_STEP.md`. Full Jaccard table and network/heatmap findings are in
  `ENGLISH_ONLY_RESULTS.md` (Supplementary: Creator-Trend Network Analysis) — this section keeps
  the headline design decisions and top-line findings.
- Purpose: does the same pool of creators post across multiple trends, or does each trend have its
  own distinct community?
- **Design decisions (tested against real data):**
  - Creator identifier: `post_owner.username`, not `post_owner.id`. Checked directly: zero owner
    IDs map to more than one username and zero usernames map to more than one owner ID (perfect
    1-to-1 across 45,163 unique creators), no missing values in either column, so there is no
    accuracy tradeoff, `username` was chosen for its readability advantage in visualizations.
  - Not a full bipartite graph of all 31,540 unique creators, only 13.42% (4,232) post across more
    than one trend, and a raw bipartite graph at that scale would be unreadable. Built as a
    **trend-to-trend projection** instead (5 nodes, one per trend), with edges weighted by
    normalized creator overlap.
  - Edge weights use **Jaccard similarity** (shared creators ÷ total unique creators across both
    trends), not raw shared-creator counts, since raw counts would be dominated by sourdough's much
    larger creator base (19,368 vs. baked_oats' 77).
  - Scope: English-only, consistent with the rest of the project.
- **Key finding:** sourdough and banana_bread have by far the strongest creator overlap of any
  pair (Jaccard 0.1073), more than double the next-highest pair (banana_bread-dalgona_coffee,
  0.0664). **Feta_pasta and baked_oats share zero creators**, the only pair with no overlap at
  all — and both are, more broadly, the two most creator-isolated trends in the network (every
  edge either is involved in is among the weakest in the table). Full 10-pair Jaccard table in
  `ENGLISH_ONLY_RESULTS.md`.
- **Creators active in 3+ trends:** 494 total (484 in exactly 3 trends, 10 in exactly 4 trends), a
  much smaller and more tightly-connected subset than the 4,232 creators active in 2+ trends
  overall.

---

## 5. Project File Structure

```
Final Project Progress/                (project root)
├── code/                              (OUT OF SCOPE, do not build on or reference)
├── code_english_only/                 (all current, in-scope project work)
│   ├── 00_data_cleaning.ipynb         (builds trends_combined_english.csv)
│   ├── 01_engagement_analysis.ipynb   (Method #5, English-only Kruskal-Wallis/Mann-Whitney)
│   ├── 04_creator_trend_network.ipynb      (Layer 2, done)
│   ├── regression/
│   │   ├── 00_rule_based_features.ipynb
│   │   ├── 01_zero_shot_classification.ipynb
│   │   ├── 02_regression_models.ipynb       (Method #6, combined model, all 5 trends pooled —
│   │   │                                      built but deprioritized, not primary, see Section 9)
│   │   └── 03_per_trend_regressions.ipynb   (Method #6, 5 individual per-trend models — PRIMARY)
│   └── lda/
│       ├── 00_build_dataset_and_config.ipynb  (builds non_trend_english.csv + lda_config.json)
│       ├── 01a_lda_text.ipynb                 (done through K-selection + final K=20 model)
│       ├── 01b_lda_hashtags.ipynb             (done through K-selection + final K=10 model)
│       ├── lda_model_checkpoints/             (per-K models, coherence logs, final models,
│       │                                        vectorizers, saved DTMs/feature names)
│       └── 02_interpret_topics.ipynb          (NOT built — see note below)
├── data/                              (raw CSVs)
├── cleaned_data/                      (intermediate outputs)
├── Lda visualizations/                (Method #7 chart outputs: coherence-vs-K, top words,
│                                        topic prevalence, interactive pyLDAvis panels, both models)
├── Regression Visualizations/         (per-trend regression forest plots)
└── Data Visualizations/               (pre-scope-change charts, largely superseded)
```

**Note on `02_interpret_topics.ipynb`:** the design doc (`TASK6_LDA_STEP_BY_STEP.md`) called for a
4th LDA notebook that loads both final models and manually labels/classifies each topic. That
notebook was never built — instead, the 30 topics' top words and top-scoring example posts were
pulled directly from the saved models and hand-classified straight into
`ENGLISH_ONLY_RESULTS.md` (genuine trend candidate / known-trend sub-cluster / noise), using the
same real-text sanity-check method used to validate the zero-shot label set in Method #6. If a
formal, reproducible interpretation notebook matching the original design is wanted later, this is
the one piece of Method #7 still not built to spec.

**Key `cleaned_data/` files (current scope):**
- `trends_combined_english.csv` — all 5 trends combined, English-only, `trend` column added,
  126,410 rows
- `trends_combined_english_features.csv` — same, with rule-based + zero-shot label columns added
  (Method #6 input)
- `non_trend_english.csv` — English posts NOT matching any of the 5 trend keyword sets (73,933
  rows), used for LDA topic discovery
- `per_trend_regression_comparison.csv` — coefficient/p-value comparison table across the 5
  per-trend models
- `trend_creator_overlap.csv` — Jaccard similarity table across the 5 trends (Layer 2)

---

## 6. Statistical Methods: Full Status

| # | Method | Status | Notes |
|---|---|---|---|
| 5 | Kruskal-Wallis/Mann-Whitney on engagement | **Done** | English-only, via `01_engagement_analysis.ipynb`; see Section 8 |
| 6 | Linguistic features + regression, per-trend models | **Done, reviewed, all 5 trends — the primary Method #6 analysis** | Forest plots reviewed; `baked_oats` flagged as statistically unreliable (see caveat below); see Section 9. A pooled combined model (with `trend` as a control dummy) was also built (`regression/02_regression_models.ipynb`) but has been **deprioritized, not part of the primary write-up** — the project's actual question is how the *effect* of each feature differs across trends, which a shared-coefficient combined model can't show, only 5 separate per-trend models can |
| 7 | LDA topic discovery | **Done** | K selected per-model by highest coherence: K=20 for text, **K=10 for hashtags** (corrected from an earlier K=20 pick that had lower coherence than K=10 — see Section 10); 30 topics total (20 text + 10 hashtags) manually labeled and classified; see Section 9 and `ENGLISH_ONLY_RESULTS.md` |
| Layer 2 | Creator-trend network | **Done** | See Section 4, Layer 2 |

---

## 7. Methodology Notes Already Written (for the eventual write-up)

**Why Kruskal-Wallis/Mann-Whitney were used instead of the standard t-test/ANOVA the professor's
reference paper (boba tea paper) used:**
That paper used standard t-tests/ANOVA directly on mean likes/views/comments. Their comparison
groups were consistently large (thousands of posts per group), so the Central Limit Theorem
protected them even with skewed underlying data. This project's trend group sizes are far more
uneven, `baked_oats` (n=121 English) and `feta_pasta` (n=303 English) are two to three orders of
magnitude smaller than `sourdough` (n=96,479 English). Kruskal-Wallis and Mann-Whitney U
(rank-based, no normality assumption) were used as primary tests, with log-transformed ANOVA as a
secondary robustness check; both agreed (p ≈ 0 for both likes and comments).

**Why comments and likes are analyzed as separate outcome variables in the regression:**
Comments require more user effort than likes; research suggests platforms treat comments as a
*stronger* algorithmic signal, not a weaker one needing discounting, so there is no defensible
single weighting scheme to combine them.

**Why baked_oats' individual regression results need an explicit reliability caveat:**
With 12 predictors + intercept = 13 parameters and only 121 observations (~9.3 observations per
parameter), baked_oats falls below the 10:1 rule of thumb used elsewhere in this project. All 5
trends were run at the project owner's explicit request, but baked_oats' coefficients should not
be reported with the same confidence as the other 4 trends, visible directly in the forest plots
via wider confidence intervals. `feta_pasta` (n=303, ~23 observations per parameter) clears the
threshold comfortably and does not need the same caveat.

**Why weekly discourse volume for baked_oats/feta_pasta can't be trusted as a real pattern:**
Checked directly: actual weekly-count standard deviation for baked_oats and feta_pasta is close to
what pure Poisson (random chance) noise would produce (ratio of actual-to-expected-noise std dev:
1.01 for baked_oats, 1.20 for feta_pasta). Dalgona coffee and sourdough show ratios of 25.45 and
13.64 respectively, dramatically higher than noise, confirming real, structured patterns. (This
analysis, `02_weekly_lockdown_overlay.ipynb`, is out of the final write-up's scope per the SCOPE
UPDATE above, but the underlying noise-floor-checking method is worth reusing wherever another
low-count time series shows up.)

**Why the creator-trend network uses a trend-level projection, not a full bipartite graph, and
Jaccard similarity, not raw counts:** see Section 4, Layer 2, both were verified as necessary
(scale and size-bias reasons respectively) against real data before building.

**Why LDA runs on `non_trend_english.csv`, not the full English-filtered dataset or the 5 known
trends' own data:** running on `trends_combined_english.csv` would only re-derive the 5 already-
known categories, since every row in it already matched a trend keyword by construction. Running
on the full English-filtered dataset (298,635 → 207,038 rows) was considered but rejected: even
with trend-specific vocabulary excluded, sourdough alone (~136,000 posts) could still let its
sub-topics dominate purely by volume, diluting the discovery signal. Restricting to
`non_trend_english.csv` (73,933 posts explicitly outside all 5 trends) targets discovery directly.

**Why LDA coherence uses `gensim`'s `c_v` `CoherenceModel`, not sklearn's `.score()`:** sklearn's
`LatentDirichletAllocation.score()` returns log-likelihood, a measure of statistical fit to the
data, not of whether a topic's top words are semantically related to each other. Using it as a
stand-in for topic quality would have been a real methodological error; real coherence required
converting the sklearn model's output into gensim's expected format (a `Dictionary` plus tokenized
documents), not a one-line substitution.

**Why LDA uses `ngram_range=(1,1)` (unigrams only), not `(1,2)`:** the original plan used bigrams
specifically to exclude 8 known trend phrases (e.g. "banana bread") while keeping their component
words available. Checked directly against `non_trend_english.csv`: all 8 phrases have **zero**
occurrences in it, by construction (the dataset already excludes every post matching any trend
keyword). The bigram-exclusion mechanism was solving a problem that didn't exist in this dataset,
while adding real cost — it was the root cause of a feature-name/matrix mismatch bug that had to
be guarded against three separate times, and it complicates coherence scoring. Switching to
unigrams-only removed the bug class and the complication with no loss of the original goal.

**A real bug caught during LDA: stopwords must be lemmatized too, or they silently leak through.**
`CountVectorizer` applies its `stop_words` filter *after* the custom tokenizer (which includes
lemmatization) runs. An inflected stopword like `redacted`, `minutes`, or `ingredients` — or even
sklearn's own built-in `further` — never matches its lemmatized form (`redact`, `minute`,
`ingredient`, `far`), so it leaks straight into topic top-words uncaught. Caught by noticing these
exact tokens showing up as real top words in early K=5/K=10 results, and confirmed by a warning
sklearn itself raises during `vectorizer.fit()`. Fix: lemmatize the stopword list itself (keeping
all 4 POS variants of each word, since a bare stopword has no sentence context to POS-tag against)
before handing it to `CountVectorizer`.

---

## 8. Results So Far: Engagement/Virality Analysis (Method #5)

`statistics.views` is missing for 100% of posts and could not be used; likes and comments were
analyzed instead.

**Overall test:** both Kruskal-Wallis and log-transformed ANOVA came back p ≈ 0.0000 for both likes
and comments, a strong, clearly real effect across the 5 trends.

**Median engagement, all-language data (original run, pre-English-restriction, kept here only as
historical context for how the English filter changed the ranking):**

| Trend | N | Median likes | Median comments |
|---|---|---|---|
| baked_oats | 125 | 430 | 37 |
| feta_pasta | 328 | 240 | 11 |
| dalgona_coffee | 26,312 | 217 | 7 |
| banana_bread | 27,037 | 229 | 12 |
| sourdough | 136,366 | 212 | 8 |

**Confirmed ranking, English-only (final, used in the write-up):**
- Likes: baked_oats > dalgona_coffee > sourdough > banana_bread
- Comments: baked_oats > banana_bread > dalgona_coffee > sourdough
- **feta_pasta's unplaceability is not simply "sample size too small" — it comes from exactly
  which pairwise comparisons reach significance, and it's a different pair on each metric.** On
  likes, feta_pasta has exactly one significant result in the whole table (below baked_oats,
  p=1.63e-05) and is statistically tied with the other three trends. On comments, it has two
  (below baked_oats, p=1.39e-08; **above** sourdough, p=4.95e-03), so its comments position is
  actually bounded on both sides (sourdough < feta_pasta < baked_oats), tied only with
  banana_bread and dalgona_coffee. Full pairwise table in `ENGLISH_ONLY_RESULTS.md`, Task 2.

**Statistical power caveat:** comparisons involving sourdough (largest sample) have very different
statistical power than comparisons among smaller trends; a significant result against sourdough
doesn't necessarily indicate as large a practical difference as one between two mid-sized trends.

Full Kruskal-Wallis H-statistics/p-values and the complete 10-pair Mann-Whitney U table (both
likes and comments) are in `ENGLISH_ONLY_RESULTS.md`, Task 2.

---

## 9. Results So Far: Method #6 (Linguistic Features Regression), Method #7 (LDA)

### Combined model (5 trends pooled) — built, deprioritized, not part of the primary write-up
Built via `regression/02_regression_models.ipynb`, same predictor set as the per-trend models plus
`trend` dummy variables (sourdough baseline), N=9,423 (Task 4's stratified sample). R² = 0.087
(likes), 0.177 (comments). **Per project-owner feedback, this is no longer the primary Method #6
analysis** — the project's actual question is whether and how the *effect* of each linguistic
feature on virality differs across the 5 trends, and a combined model with `trend` only as an
intercept-shifting dummy can't show that (every trend is forced to share the same coefficient for
every feature except the intercept). Only the 5 separate per-trend models below can show that
directly, so they're now the primary Method #6 result. The combined model's results are no longer
carried in `ENGLISH_ONLY_RESULTS.md`; its old headline findings (media_repost sign flip, recipe/
meme as strongest zero-shot predictors, covid_framed negative for likes only) all replicate at the
per-trend level too, per the summary below, so nothing analytically unique was lost by
deprioritizing it. The notebook itself is untouched and still runnable if the pooled view is ever
wanted again.

### Per-trend models — done, reviewed, all 5 trends, the primary Method #6 analysis
Real coefficients reviewed from actual notebook output (both likes and comments models, all 5
trends: sourdough, banana_bread, dalgona_coffee, feta_pasta, baked_oats). Selected findings:

- **`recipe_instructional`**: significant positive for the three large trends; banana_bread's
  effect is by far the largest (0.79 likes / 1.00 comments) vs. sourdough (0.56 / 0.49) vs. dalgona
  (weaker, only significant in comments, 0.22). Not significant for feta_pasta or baked_oats.
- **`meme_joke`**: significant positive for the three large trends; **dalgona has the largest
  effect** (0.60 likes / 0.55 comments), larger than banana_bread (0.39/0.47) and sourdough (not
  significant on likes; 0.30 on comments). Not significant for feta_pasta or baked_oats (small
  samples), though feta_pasta's point estimate is large (1.47 likes) with a wide interval.
- **`media_repost`**: positive/significant for likes in sourdough, banana_bread, dalgona_coffee,
  and feta_pasta; flips negative/significant for sourdough, dalgona_coffee, and baked_oats in
  comments. Reposted content gets liked but doesn't spark discussion, a pattern that replicates
  down to the per-trend level, not just the combined model.
- **`spam_low_content`**: significant negative for banana_bread and dalgona_coffee in likes; for
  comments, significant negative for dalgona_coffee, feta_pasta, and baked_oats (the latter two
  with notably larger magnitudes, -0.95 and -1.45, though both carry a small-sample caveat). Never
  significant for sourdough on either outcome.
- **`is_covid_framed`**: significant negative for banana_bread and dalgona_coffee on likes
  (explicit pandemic framing hurt engagement); dalgona_coffee only, on comments. Not significant
  for sourdough, feta_pasta, or baked_oats.
- **`hashtag_count`**: the single most consistently significant predictor across all 5 trends —
  significant and positive in every comments model, and in 4 of 5 likes models (all but
  feta_pasta). Dalgona's effect is consistently among the largest.
- **`log_word_count`**: significant positive for sourdough and dalgona_coffee (both outcomes);
  not significant for banana_bread or feta_pasta at all; significant **negative** for baked_oats on
  likes (-0.61) — a sharp reversal, though read alongside baked_oats' reliability caveat.
- Model fit (R²) modest throughout except baked_oats (0.32 likes / 0.35 comments), which is likely
  inflated by overfitting given its small sample (see caveat, Section 7), not genuinely stronger
  explanatory power.

Full coefficient/p-value tables for all predictors × all 5 trends × both outcomes, per-trend VIF
(no multicollinearity concerns anywhere — every feature VIF is well under 3 in every trend,
notably cleaner than the pooled combined model's VIF=12.35 on `log_word_count`), plus both forest
plots, are in `ENGLISH_ONLY_RESULTS.md`, Task 5.

**Note:** `trend` is not strictly mutually exclusive (a small share of posts match more than one
trend's keywords, most notably 63.6% of `baked_oats`' rows also carrying a `banana_bread` label,
largely via a real "banana bread baked oats" recipe genre) — compounds `baked_oats`' existing
small-sample caveat; see `ENGLISH_ONLY_RESULTS.md` Task 5 and `methods.md` Section C.5.

### Zero-shot label set (used in both regression models)
Finalized after real-batch testing and prompt revisions (fixing caption-vs-hashtag confusion,
label independence, and promotional-intent-as-spam issues). Final 5 labels:
`recipe_instructional`, `personal_lifestyle`, `media_repost`, `meme_joke`, `spam_low_content`. A
6th candidate label, `trend_commentary` (posts that comment on the trend phenomenon itself without
being personal, a repost, a joke, or low-content), was considered during design and dropped with
no replacement — not because of a numeric prevalence threshold, but because posts that fit that
description are expected and intended to simply receive zero labels under the final 5-label set,
which is correct by design, not a gap needing a 6th label to fill. (An earlier draft of this note
cited "~1.8% prevalence, below a 15% threshold" as the reason — that framing doesn't match the
documented final decision in `CLAUDE_CODE_TASKLIST.md` Task 4 and has been corrected here.)

### LDA topic discovery — done
Both models (`text`, `hashtags`) run on `non_trend_english.csv` (73,933 English posts outside all
5 known trends), K tested at 5/10/15/20/25 using real `gensim` `c_v` coherence. **Final K is
selected by highest coherence in each model's own tested range, applied consistently: K=20 for
text (coherence 0.598), K=10 for hashtags (coherence 0.700).** An earlier pass had used K=20 for
the hashtags model too, despite K=10 scoring higher (0.700 vs. 0.680) — that override had no
recorded rationale and has been corrected; the saved `final_lda_hashtags.pkl` artifact and its
visualizations have been regenerated at K=10 to match.

All 30 resulting topics (20 text + 10 hashtags) were manually labeled and classified as (a) a
likely genuine new trend, (b) a sub-cluster of the known COVID-baking wave, or (c) noise/generic
content, using each topic's top 15 words plus its top-scoring real example posts. **Headline
result: no topic reconstructs a wholly new single-dish COVID-era trend on the scale of the 5
already studied** — this is a real, reportable negative finding, not a shortfall of the method.
The clearest positive discoveries are **diet/lifestyle patterns rather than dish trends**:
keto/low-carb content forms its own coherent topic independently in *both* models (2.29%
text/4.39% hashtags — a strong, convergent signal that survived the K correction), and
vegan/plant-based content forms its own coherent hashtag topic (6.90%, hashtags only, no matching
text-model topic). **A second cross-model match, verified directly against prevalence and top
words:** an Indian food-blogger community topic (Ahmedabad/Mumbai/Delhi) also appears
independently in both models — text Topic 11 (2.05%) and, far more prominently, hashtags Topic 5
(10.52%) — currently the only ethnic/regional food community distinct enough to form its own
topic in either model. (An earlier draft of this section named a "whipped feta dip" cluster as
the single most trend-like candidate discovery — that specific label has been corrected: the
phrase "whipped feta" never appeared in the topic's actual top-15 words, only in 2 of 4 thinly
sampled example posts. The topic's real signature supports a broader "Mediterranean/Middle
Eastern feta-based mezze" label; whether it still counts as a genuine discovery is an open
classification call, not a settled finding — see `ENGLISH_ONLY_RESULTS.md` Task 6.) A large share
of the remaining topics are food-media aggregator/repost accounts (Feedfeed, Buzzfeed, HuffPost
Taste, The Kitchn, Food & Wine) recurring as their own topic(s) in both models — a real,
structural content-supply pattern distinct from organic trend content — plus regional
food-blogger communities, generic health/diet-coaching content, and preprocessing artifacts
(non-Latin-script hashtags truncated by the ASCII-only regex; stylized-Unicode-font hashtag
spam). **Real tradeoff of the K correction:** the coarser K=10 hashtags model blends some
patterns that were cleanly separated at K=20 (e.g. 3 aggregator topics collapse into ~1; the
stylized-font-spam topic merges into a generic foodie-hashtags topic) — a direct, visible
instance of "fewer topics risk blending distinct patterns together."

Full per-topic tables (all 30 topics' top words, prevalence, label, and classification — labels
for 8 specific topics project-owner-verified against the actual model output) and the complete
findings discussion are in `ENGLISH_ONLY_RESULTS.md`, Task 6. Note: the narrative synthesis of
what these findings mean is being written directly by the project owner from that data, not
finalized here or there.

**Real-world corroboration for the LDA findings (keto, vegan/plant-based, and the Indian
food-blogger community as independent, genuine patterns beyond the 5 headline trends) has not yet
been checked against
industry/news reporting** — see Section 11, Next Steps.

---

## 10. Known Gotchas / Lessons Learned (worth remembering)

- **Jupyter does not auto-refresh stale cell output.** Downstream notebooks must be fully restarted
  (Kernel → Restart & Run All) after an upstream file changes, not just have individual cells
  re-run.
- **Windows file paths** in this project use relative paths, depth-dependent
  (`../cleaned_data/...` for notebooks directly in `code_english_only/`,
  `../../cleaned_data/...` for notebooks one level deeper in `regression/` or `lda/`). Has caused
  real, caught-and-fixed path bugs, double check path depth whenever adding a new notebook folder.
- **Low-count weekly/monthly time series data (under ~10-15 observations per bucket) can look
  falsely "sporadic" purely from random noise.** Always check actual variability against a
  Poisson-noise baseline before describing a low-count trend as showing a real pattern.
- **Vocabulary/feature-name mismatches are a recurring bug class whenever a document-term matrix
  gets manually trimmed after fitting.** If a matrix's columns are ever altered after a vectorizer
  fits, any downstream code calling `vectorizer.get_feature_names_out()` directly will silently
  return the wrong, out-of-sync vocabulary. Keep trimmed feature-name arrays explicitly saved. (LDA
  sidestepped this entirely by switching to unigrams-only, see Section 7 — worth remembering that
  the safest fix to this bug class is sometimes removing the thing that causes it, not guarding
  against it more carefully.)
- **A custom tokenizer/lemmatizer must be applied to the stopword list too, not just the corpus,**
  or inflected stopwords silently leak through into results uncaught (see Section 7's LDA
  stopword-lemmatization bug). Watch for sklearn's own "stop_words may be inconsistent with your
  preprocessing" warning as a direct signal this is happening.
- **An ASCII-only cleaning regex silently destroys non-Latin-script content instead of erroring.**
  LDA's `text = re.sub(r"[^a-z\s]", " ", text)` preprocessing step turned non-English-script
  hashtags (confirmed CJK-origin from context) into meaningless truncated fragments (`uc`, `ub`,
  `ud`) that then surfaced as their own topic rather than being cleanly dropped. Worth checking
  whether a "how much content got mangled, not just filtered" audit is needed anywhere else a
  similar regex is used on user-generated text.
- **Before assuming a unique identifier needs a "safer" but less readable version (e.g. an internal
  ID over a display name), check the actual data first.** `post_owner.username` was assumed less
  reliable than `post_owner.id` by default, but a direct check showed a perfect 1-to-1 mapping with
  no missingness in this dataset, no accuracy was actually being traded away for readability.
- **When projecting a bipartite structure (creators × trends) down to a smaller graph, use a
  normalized similarity measure (Jaccard), not raw counts,** if group sizes vary by orders of
  magnitude (sourdough's 19,368 creators vs. baked_oats' 77), raw counts would just reflect group
  size, not genuine relative overlap.
- **A model-selection choice (like LDA's K) needs its rationale written down at the moment it's
  made, even when it seems obvious at the time — and absent that rationale, don't assume an
  override was intentional.** The hashtags model's final K was originally set to 20 despite K=10
  scoring higher on the actual coherence metric being optimized (0.680 vs. 0.700), with no note
  recording *why* surviving past a single code comment. When this was written up, it was initially
  guessed to be a deliberate choice (e.g. for consistency with the text model). On review, it
  turned out to just be a mistake — corrected to K=10, the actual highest-coherence value,
  consistent with how K was chosen everywhere else in this project. Lesson: an unexplained
  deviation from your own stated selection criterion is more likely an error than a hidden reason,
  and should be flagged and re-checked, not charitably reverse-engineered into a rationale.

---

## 11. Immediate Next Steps, In Order

1. **Check the LDA findings against real-world corroboration** — industry/news reporting on
   ingredient or cuisine shifts (keto, vegan/plant-based, the Indian food-blogger community
   cross-model match) during the same window, the way this was done informally for other clusters
   during lightning-talk
   planning. Not yet done; called for in the original Method #7 design and still open.
2. **(Optional) Build `02_interpret_topics.ipynb` to spec**, if a formally reproducible topic
   interpretation notebook (matching the original 4-file LDA design) is wanted for the appendix/
   methods write-up, rather than the direct hand-labeling that currently backs the results in
   `ENGLISH_ONLY_RESULTS.md`. Not required for the results themselves, which are already complete.
3. **Write the discussion/synthesis section tying all 4 analyses together.** Suggested thesis,
   already drafted in conversation: the pandemic's food-and-social-media moment was **"several
   related phenomena happening in parallel, not one uniform trend"**, supported directly by (a) the
   creator network finding that feta_pasta and baked_oats share zero creators, and (b) the LDA
   finding that the "discovery" beyond the 5 trends turned out to be parallel diet/lifestyle
   movements (keto, vegan) rather than more single-dish trends of the same kind. Structure: (1)
   virality varied, not uniformly (Method #5), (2) content type explains some of why (Method #6,
   combined + per-trend), (3) the shift beyond the 5 headline trends looks more like adjacent
   lifestyle movements than undiscovered dish trends (Method #7), (4) distinct, only
   partially-overlapping creator communities drove different trends, not one unified wave (Layer
   2/creator network).
4. **Use the finalized opening hook (Section 2) in the actual write-up draft.**
5. **Work from `ENGLISH_ONLY_RESULTS.md` for every number cited in the write-up** — it now holds
   the complete, non-summarized version of every result referenced above.
