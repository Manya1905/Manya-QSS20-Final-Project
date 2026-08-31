# Methods Reference

**Format note:** this is a bulleted reference, not final prose — written to be turned into paper
text by the project owner, not copy-pasted directly. Each bullet is meant to stand on its own
without requiring a trip back to the notebooks or design docs.

**Scope note:** covers the 4 pieces requested — (A) the shared data cleaning pipeline, (B) Method
#5 (virality across trends), (C) Method #6 (per-trend linguistic regression), (D) Method #7 (LDA
undiscovered-trend topic discovery). The creator-trend network analysis is out of scope for this
document. Method #6 here refers exclusively to the **per-trend regression models**, which are the
project's primary Method #6 analysis; the earlier pooled/combined model is mentioned only where
needed to explain *why* the per-trend design was chosen, per the design-choice-rationale
requirement below.

**Sourcing note:** every claim below is grounded in `MASTER_PROJECT_REFERENCE.md`,
`ENGLISH_ONLY_RESULTS.md`, `CLAUDE_CODE_TASKLIST.md`, `HANDOFF_method6_virality_predictors.md`,
`TASK6_LDA_design_doc.md`, and `PER_TREND_REGRESSIONS_STEP_BY_STEP.md`. Anything ambiguous,
undocumented, or inconsistent across these docs is flagged as a question in the final section
rather than resolved by guessing. Notebook citations use `[LINE_TBD]` placeholders per your
instruction, since exact line numbers aren't available yet.

---

## A. Shared Data Cleaning Pipeline

Used, unmodified, as the input to both Method #5 (B) and Method #6 (C). Method #7 (D) uses a
**separate** cleaning pipeline, described in its own section — it does not reuse this dataset.

### A.1 Data sources

- **4 raw pull files** (Instagram/Facebook post data, professor-provided, Dec 2019–Dec 2020):
  `data/manya_first_12012024.csv`, `data/manya_0711-1104.csv`, `data/manya_1105_1201.csv`,
  `data/manya_0410-0710.csv`.
- These 4 files are **not** read directly by the shared pipeline below. They were first used
  (upstream, out of scope for this project's `code_english_only/` folder) to build **5
  trend-specific CSVs** — one file per trend — via keyword matching. That original construction
  script is `code/02_data_cleaning.ipynb` **[LINE_TBD] — not reviewed for this document; flagged
  in Open Questions below.**
- **5 trend-specific CSVs** (the actual input to the shared pipeline): `cleaned_data/feta_pasta.csv`,
  `cleaned_data/sourdough.csv`, `cleaned_data/banana_bread.csv`, `cleaned_data/baked_oats.csv`,
  `cleaned_data/dalgona_coffee.csv`. None of these files has a `trend` column — that's added in
  Step 1 below.

### A.2 How a post is assigned to a trend (defines the `trend` variable)

- Each of the 5 CSVs above already contains only posts matched against that trend's specific
  keyword list, via case-insensitive regex search of the post's `text` OR `hashtags` field
  (`.str.contains(pattern, case=False, na=False, regex=True)`, OR logic).
- Keyword lists used (source: `MASTER_PROJECT_REFERENCE.md` Section 3; the same lists are
  independently re-implemented, and confirmed identical, in two other places in this project's
  codebase — the LDA dataset build and the creator-network build):

```python
trend_keywords = {
    "feta_pasta": ["feta pasta", "baked feta pasta", "baked feta"],
    "sourdough": ["sourdough", "sourdough starter", "sourdough bread"],
    "banana_bread": ["banana bread", "banana bread recipe", "bananabread"],
    "baked_oats": ["baked oats", "baked oatmeal", "bakedoats"],
    "dalgona_coffee": ["dalgona coffee", "whipped coffee", "dalgona"],
}
```
- Keyword phrases were kept as targeted multi-word phrases rather than broadened to single-word
  wildcards, specifically to avoid pulling in unrelated year-round cooking content (e.g. plain
  "coffee" or "oats" posts with no connection to the trend).
- **This is a within-post text-matching/labeling operation, not a merge or join between separate
  data sources.** No record linkage, key-based join, or fuzzy/blocking match occurs anywhere in
  this step — each post is independently evaluated against each trend's keyword list.
- **Note: `trend` is not a strictly mutually exclusive partition** — a small share of posts (1.72%
  of the dataset) match more than one trend's keywords and appear under both labels, most
  notably `baked_oats` (63.6% of its 121 rows also carry another trend's label, mainly
  `banana_bread`, largely via a real "banana bread baked oats" recipe genre rather than a
  keyword-matching error). Not treated as a material issue for this project's scope; see C.5 for
  the one place it's relevant (compounds `baked_oats`' existing small-sample caveat).

### A.3 Shared cleaning steps

Notebook: `code_english_only/00_data_cleaning.ipynb` **[LINE_TBD]**

- **Step 1 — load, label, recompute `is_covid_framed`, concatenate.** Each of the 5 CSVs is loaded,
  a `trend` column is added (its value = that file's trend name), and `is_covid_framed` is
  recomputed (see A.4 below) before the 5 DataFrames are concatenated:
```python
dfs = []
for trend_name, fname in trend_files.items():
    d = pd.read_csv(f"../cleaned_data/{fname}")
    d["trend"] = trend_name
    d["is_covid_framed"] = (
        d["text"].str.contains(covid_pattern, case=False, na=False, regex=True) |
        d["hashtags"].str.contains(covid_pattern, case=False, na=False, regex=True)
    )
    dfs.append(d)
combined = pd.concat(dfs, ignore_index=True)
```
  **This is a concatenation (`pd.concat`, i.e. stacking rows), not a join** — the 5 inputs share
  the same column schema and no key-based matching occurs between them. No columns are dropped
  from the original files at this step.
- **Step 2 — filter to English-only:** `combined = combined[combined["lang"] == "en"].copy()`.
  `lang` is a pre-existing column in the source data, not derived.
- **Step 3 — drop rows with a null outcome variable:**
  `combined.dropna(subset=["statistics.like_count", "statistics.comment_count"])`. No imputation
  is performed — rows with either outcome missing are dropped entirely, not filled.
- **Explicit non-step:** null `hashtags` values are **not** dropped and **not** treated as missing
  data — a post can legitimately have zero hashtags. `hashtags` nulls are carried through and only
  resolved downstream, in Method #6's feature engineering (Section C), where a null becomes
  `hashtag_count = 0`.
- **Sanity check, not a filter:** after Step 2, `text` is confirmed to have zero nulls (asserted in
  the notebook) — this is a data-quality check, not a cleaning operation that removes rows.
- Output saved as `cleaned_data/trends_combined_english.csv`. This is the dataset both Method #5
  (Section B) and Method #6's feature-engineering stage (Section C) load and build from.

### A.4 `is_covid_framed` variable definition

- Boolean flag. `True` if the post's `text` OR `hashtags` field contains any of 5 keywords
  (case-insensitive regex): `covid`, `coronavirus`, `pandemic`, `quarantine`, `lockdown`.
- This is a **corrected** version of an earlier definition that checked only 3 hashtag-specific
  terms (`quarantinebaking`, `pandemicbaking`, `quarantinecooking`). The corrected 5-keyword list
  was confirmed to be a strict superset of the original 3-keyword version before being adopted —
  i.e., the correction only adds true positives the original missed, it does not remove or
  reclassify anything the original caught.

### A.5 No merge/join anywhere in this pipeline

- There is no relational join (exact-match or fuzzy-match) at any point in Sections A.2–A.3. The
  entire pipeline is: (1) independent keyword-based labeling of each of the 5 already-separate
  trend files, (2) concatenation (row-stacking) of those 5 labeled files, (3) row-level filtering
  (language, null-outcome). No blocking variables or fuzzy-match logic are used or needed, since no
  join occurs.

### A.6 Variable glossary (shared pipeline)

| Variable | Definition |
|---|---|
| `trend` | Categorical, 5 levels (`sourdough`, `banana_bread`, `dalgona_coffee`, `feta_pasta`, `baked_oats`). Assigned per A.2. |
| `is_covid_framed` | Boolean. Regex match of the 5-keyword COVID list against `text` OR `hashtags`. Defined in A.4. |
| `lang` | Pre-existing column in source data; `"en"` after the Step 2 filter (constant, but retained in the file). |
| `text` | Pre-existing column; the post's caption text. |
| `hashtags` | Pre-existing column; stringified list of the post's hashtags. Can be legitimately null. |
| `statistics.like_count` / `statistics.comment_count` | Pre-existing numeric columns; the two outcome variables used throughout Methods #5 and #6. |

### A.7 Not an analysis

- This section describes data preparation only. No predictive, inferential, or causal claim is
  made here — the train/test-split and causal-identification questions below don't apply to this
  section; they apply to Sections B–D.

---

## B. Method #5: Virality Across Trends (Kruskal-Wallis / Mann-Whitney)

Notebook: `code_english_only/01_engagement_analysis.ipynb` **[LINE_TBD]**

### B.1 Data used

- The shared cleaned dataset from Section A (`trends_combined_english.csv`), used as-is. No
  additional cleaning, feature engineering, or merging is performed for this analysis.
- Grouping variable: `trend` (5 levels, defined in A.2/A.6).
- Outcome variables: `statistics.like_count`, `statistics.comment_count` (raw values for the
  primary test; log1p-transformed versions used only for the secondary ANOVA check, B.3).

### B.2 Procedure

- **Primary test:** Kruskal-Wallis H-test, run twice — once on `statistics.like_count`, once on
  `statistics.comment_count` — each time comparing all 5 `trend` groups simultaneously.
- **Secondary robustness check:** one-way ANOVA on `log1p`-transformed likes and comments
  separately, to confirm the Kruskal-Wallis result isn't an artifact of the rank-based method.
- **Post-hoc pairwise comparisons:** Mann-Whitney U test, run separately for likes and comments,
  for all 10 possible pairs among the 5 trends. Significance threshold: p < 0.05.
- **No multiple-comparisons correction (e.g. Bonferroni) is applied across the 10 pairwise
  tests.** [See Open Questions — no documented rationale specifically covers this decision for
  this test family; the "no correction" rationale found in the docs applies only to Method #6's
  regression coefficients, not to these pairwise tests.]

### B.3 Rationale for Kruskal-Wallis/Mann-Whitney over standard t-test/ANOVA

- The project's reference paper (a "boba tea" trend-comparison study) used standard t-tests/ANOVA
  directly on mean engagement metrics. That paper's comparison groups were consistently large
  (thousands of posts per group), so the Central Limit Theorem protected the validity of those
  tests even with skewed underlying engagement data.
- This project's trend group sizes are far more uneven: `baked_oats` (n=121 English) and
  `feta_pasta` (n=302 English) are two to three orders of magnitude smaller than `sourdough`
  (n=96,257 English). This size disparity is the stated reason parametric tests were not used as
  the primary method.
- Kruskal-Wallis and Mann-Whitney U (rank-based, no normality assumption) were adopted as the
  primary tests specifically because they don't require the large-sample/normality conditions a
  t-test/ANOVA relies on. The log-transformed ANOVA is kept only as a secondary check that both
  methods agree, not as a primary result.

### B.4 Reliability/validity caveats (as documented)

- The uneven trend sample sizes (noted in B.3) are the documented reason for the parametric-vs-
  nonparametric test choice, not merely a footnote — this is the central methodological
  justification, not incidental.
- **Statistical power caveat:** comparisons involving `sourdough` (the largest sample) have
  meaningfully different statistical power than comparisons among mid-sized or small trends. A
  significant result against sourdough does not necessarily indicate as large a practical
  difference as a significant result between two mid-sized trends — this is stated explicitly as
  a caveat on interpreting the pairwise results, not something to smooth over in the write-up.

### B.5 Predictive/causal framing

- **Not predictive.** This is a distributional hypothesis-testing procedure, not a model fit to
  make predictions. A train/test split is not applicable here.
- **Not causal.** `trend` is an observed, keyword-matched grouping variable, not an experimentally
  or quasi-experimentally assigned treatment. No causal identification strategy (instrument,
  discontinuity, matching design, etc.) is used or claimed. This is a purely descriptive/inferential
  comparison of engagement distributions across naturally occurring content categories.

---

## C. Method #6: Per-Trend Linguistic Regression

This is the project's **primary** Method #6 analysis (see C.3 for why, replacing an earlier pooled
model design).

### C.1 Additional data preparation (beyond the shared pipeline)

Two sequential notebooks build on top of Section A's `trends_combined_english.csv`, each producing
an updated version of the same output file.

**C.1.a — Rule-based feature engineering.**
Notebook: `code_english_only/regression/00_rule_based_features.ipynb` **[LINE_TBD]**

| Variable | Definition / code |
|---|---|
| `word_count` | `text.str.split().str.len()` |
| `log_word_count` | `np.log1p(word_count)`. Used as the actual regression predictor (not raw `word_count`) because raw word count is confirmed right-skewed (skewness 1.692) and the log-transformed version is close to symmetric (skewness -0.398). |
| `sentiment_score` | VADER `compound` score via `vaderSentiment`, range -1 (most negative) to +1 (most positive): `analyzer.polarity_scores(text)["compound"]`. |
| `hashtag_count` | Parsed from the stringified-list `hashtags` column via `ast.literal_eval`; null `hashtags` → `0` (not excluded, not imputed with any other value). |
| `exclamation_count` / `question_count` | `text.str.count("!")` / `text.str.count(r"\?")`. |
| `emoji_count` | Via the `emoji` package's `emoji.emoji_count()` function applied to `text`. |
| `log_likes` / `log_comments` | `np.log1p()` of `statistics.like_count` / `statistics.comment_count`. These are the two outcome variables for the regression models. |
| `is_covid_framed` | Carried through unchanged from Section A.4. |

- **Features considered and explicitly excluded from the model** (documented rationale, not an
  oversight): `has_mention` — dropped because it bundles several distinct behaviors (tagging a
  recipe creator, a brand, or a friend) into one ambiguous feature; `has_allcaps_word` — dropped
  because it's frequently just a recipe-title formatting convention (e.g. "STRAWBERRY BANANA BAKED
  OATMEAL!"), not a clean emphasis signal, and likely redundant with `exclamation_count` and the
  `recipe_instructional` zero-shot label. "Link in bio" mentions and ellipsis usage were also
  checked and excluded — both under 15% real prevalence, too rare to reliably detect an effect.
- `word_count` is kept continuous and log-transformed, **not** bucketed into short/medium/long
  categories — bucketing was explicitly rejected because arbitrary cutoffs throw away information
  a continuous predictor retains.
- Output: `cleaned_data/trends_combined_english_features.csv`, 126,410 rows at this stage (same
  row count as Section A's output — no filtering happens here, only column additions).

**C.1.b — Zero-shot classification + stratified sampling.**
Notebook: `code_english_only/regression/01_zero_shot_classification.ipynb` **[LINE_TBD]**

- **Sampling procedure (cost-driven, not a train/test split):** `sourdough`, `banana_bread`, and
  `dalgona_coffee` are each randomly capped at 3,000 posts (fixed seed=42); `baked_oats` (121) and
  `feta_pasta` (302) are kept at full size since they're already small. Total sampled: 9,423
  posts. **This subsampling exists to control the cost/time of the classification step below (API
  calls), not to create a held-out evaluation set.** The same 9,423-post sample is used both to
  produce the classification labels and, later, to fit the regression — there is no separate
  train/test partition at any point in this pipeline.
- **`post_id` is a synthetic identifier, not the source data's `id` column.** After sampling, the
  DataFrame is reset-indexed and `post_id` is set to that new row index
  (`combined["post_id"] = combined.index`). A code comment in `01_zero_shot_classification.ipynb`
  **[LINE_TBD]** flags this as deliberate, since `id` has "~1,098 duplicates dataset-wide." This is
  the same phenomenon documented in A.2, viewed from this notebook's dataset: `id` itself is unique
  in the raw source data, but by the time a post reaches `trends_combined_english_features.csv` it
  may have been pulled into more than one trend's file, so its `id` appears more than once in this
  specific dataset. Using a fresh row-index `post_id` instead of `id` correctly avoids the
  ambiguity this would otherwise cause when writing classification results back onto rows.
- **Classification model:** `claude-sonnet-5`, called via Anthropic's asynchronous Message Batches
  API in batches of 15 posts per request (chosen for cost — roughly 50% cheaper than real-time
  calls — and to avoid manual rate-limit pacing), using structured JSON-schema output
  (`output_config={"format": {"type": "json_schema", "schema": LABEL_SCHEMA}}`) rather than
  free-text parsing.
- **Labels produced (5 independent binary variables, not mutually exclusive):**
  `is_recipe_instructional`, `is_personal_lifestyle`, `is_media_repost`, `is_meme_joke`,
  `is_spam_low_content`. A post can receive zero, one, or multiple labels.
- **Finalized classification prompt** (sent verbatim to the model; defines exactly what each label
  means):
```
You are classifying Instagram/Facebook posts about COVID-era food trends (sourdough, banana bread,
dalgona coffee, baked oats, feta pasta) into content-type categories.

For EACH post, assign zero, one, or multiple of the following labels. These are independent binary
categories, not mutually exclusive, and not exhaustive: a post that doesn't clearly fit any of them
should simply get zero labels, do not force a fit.

- recipe_instructional: post provides or centers on a recipe/cooking instructions (ingredients,
  steps, "how to make X")
- personal_lifestyle: personal/narrative content about the poster's day, life, family, or mood,
  not primarily about the recipe/food itself
- media_repost: clearly reposted/shared content from another source (press release, news article,
  another account's or business's content, ad copy) rather than original personal content
- meme_joke: humor/meme/joke-framed content
- spam_low_content: post has minimal substantive content, e.g. just a hashtag string with no real
  caption, or generic engagement-bait text with little real connection to the food/recipe itself

Important clarifications:
- Labels are independent. Evaluate each one on its own merits. A post can be both
  recipe_instructional (if the caption has real recipe content) AND spam_low_content (if it is
  also mostly hashtag padding) at the same time. Do not let one label's presence suppress another
  that also applies.
- A real, substantive caption followed by hashtags is NOT spam_low_content just because hashtags
  are present. Hashtags are standard Instagram convention and should be ignored when judging
  substance. Only the caption content itself (ignoring the hashtag block) determines this label.
- Promotional or commercial intent alone does NOT make a post spam_low_content. A business account
  taking orders, describing a product, or advertising a delivery service with a real, on-topic
  caption is not spam just because it is commercial, judge only whether the caption itself is
  substantive. If it is substantive but doesn't fit any other category, it should receive no
  label, not spam_low_content.
- A post can legitimately receive zero labels. This is expected and correct, do not stretch a
  definition to force a label onto every post.

Respond only with structured JSON, no other text.
```
- **A 6th candidate label, `trend_commentary`**, was considered during prompt design and dropped
  with no replacement — posts fitting that description are expected to simply receive zero labels
  under the final 5-label set. This is a labeling-scheme design decision, not a result of running
  the classifier.
- Output: `cleaned_data/trends_combined_english_features.csv` overwritten in place — now scoped to
  **only the 9,423-row stratified sample** (not all 126,410 rows). This smaller file is the actual
  input to the regression models in C.2.

### C.2 Model specification

- **Design:** one OLS regression per trend, per outcome — 5 trends × 2 outcomes (`log_likes`,
  `log_comments`) = **10 separate models total**. No trend dummy variables are included — each
  model's data is already restricted to a single trend, so there is nothing to control for.
- **Predictor set** (identical 12 predictors + intercept in every one of the 10 models):
  `log_word_count`, `sentiment_score`, `hashtag_count`, `exclamation_count`, `question_count`,
  `emoji_count`, `is_covid_framed` (cast to int), `is_recipe_instructional`,
  `is_personal_lifestyle`, `is_media_repost`, `is_meme_joke`, `is_spam_low_content`.
- **Fitting code** (per trend, per outcome):
```python
X = sm.add_constant(subsets[trend]["X"])
model_likes = sm.OLS(subsets[trend]["data"]["log_likes"], X).fit()
model_comments = sm.OLS(subsets[trend]["data"]["log_comments"], X).fit()
```
- Notebook: `code_english_only/regression/03_per_trend_regressions.ipynb` **[LINE_TBD]**

### C.3 Rationale: per-trend models over a single pooled model

- Method #6's actual question is whether — and how — the *effect* of each linguistic feature on
  virality differs **across** the 5 trends. A single pooled model with `trend` included only as a
  dummy/control variable can show that trends differ in baseline (intercept) engagement level, but
  it cannot show whether, e.g., `hashtag_count` matters more for one trend than another — every
  trend is forced to share the same coefficient for every feature except the intercept. Only 5
  separate per-trend models can show that directly.
- **Design history, for traceability:** a pooled model (with `trend` as a control dummy, sourdough
  fixed as the baseline category) was the originally planned primary design. The deciding factor at
  the time was sample size — `baked_oats` (n=121) and `feta_pasta` (n=302) were judged too small to
  support their own stable regression once 10+ parameters are in play. After reviewing results,
  this was revisited: the 5 per-trend models were built anyway, with an explicit reliability
  caveat attached to `baked_oats` specifically (C.5) rather than excluding it, and were promoted to
  the primary analysis. The pooled model still exists as a built, runnable artifact
  (`code_english_only/regression/02_regression_models.ipynb` **[LINE_TBD]**) but is not part of the
  primary analysis and is not otherwise described in this document.

### C.4 Rationale for other design choices

- **Two separate outcome models (`log_likes`, `log_comments`), not one combined "virality score."**
  Comments require more user effort than likes; the stated rationale is that research suggests
  platforms treat comments as a *stronger* algorithmic engagement signal, not a weaker one needing
  discounting — so there is no defensible single weighting scheme to combine the two into one
  score. Keeping them separate also preserves information: if a feature predicts one outcome but
  not the other, that is itself treated as a meaningful pattern rather than noise to average away.
- **Both outcomes log1p-transformed before modeling.** Likes and comments are both confirmed
  right-skewed (mean well above median; histograms only become approximately bell-shaped after the
  log transform). Comments additionally show zero-inflation (a large spike of exactly-zero-comment
  posts, since commenting requires more effort than liking) — a structurally different pattern from
  plain skew. This zero-inflation is the stated reason the comments model should be trusted
  somewhat less than the likes model, not a reason to change the modeling approach.
- **Multi-label zero-shot classification, not forced single-label** (see C.1.b). Rationale: real
  posts often genuinely span multiple categories (e.g. instructional content delivered in a casual
  personal voice); forcing a single choice on a borderline post injects label noise, which is
  expected to bias regression coefficients *toward* zero (i.e., attenuate real effects), not
  fabricate false ones. Documented tradeoff: multi-label design increases multicollinearity risk if
  labels co-occur frequently — addressed via the VIF check in C.5, not treated as a blocker.
- **Additive-only model, no interaction terms.** Confirmed acceptable for this project's scope;
  the additive assumption (a feature's effect on engagement does not vary with the level of another
  feature, within a given trend's model) should be stated as an explicit limitation in the
  write-up, not silently assumed.
- **No multiple-comparisons correction** across the 12 predictors, both outcomes, and 5 trend
  models. Confirmed acceptable for this project's scope in the original Method #6 design
  discussion; this decision predates the per-trend redesign but was not revisited or overridden
  when per-trend models were adopted.
- **No baseline/reference-category concept applies here.** The "sourdough as fixed baseline" design
  decision documented elsewhere in this project applies only to the deprioritized pooled model
  (C.3) — the per-trend models have no dummy-coded categorical variable and therefore no baseline
  category at all.

### C.5 Reliability/validity caveats (as documented)

- **General rule of thumb:** 10–20 observations per model parameter. **Specific threshold applied
  to flag a trend's results as unreliable:** falling below a 10:1 observations-per-parameter ratio.
  Every model has 13 total parameters (12 predictors + intercept).
- **`baked_oats`:** n=121 → ≈9.3 observations per parameter → **below the 10:1 threshold**. Its
  coefficients are documented as likely to have wide confidence intervals and to shift meaningfully
  with small data changes; they are run and reported (per explicit project-owner request — not
  excluded from the analysis) but are documented to require an explicit reliability caveat
  wherever they appear, and should not be presented with the same confidence as the other 4
  trends' results.
- **`feta_pasta`:** n=302 → ≈23.2 observations per parameter → clears the 10:1 threshold
  comfortably; no reliability caveat is documented as necessary for this trend. [Note: one project
  doc states feta_pasta's n as 303 rather than 302 — see Open Questions.]
- **Multicollinearity (VIF):** computed separately within each trend's own predictor matrix (not
  once on a pooled matrix), since feature correlations could plausibly differ by trend. The
  documented concern threshold is a VIF in the conventional 5–10 range; no predictor in any
  trend's model reaches that range.
- **`baked_oats`' regression sample also isn't fully independent of `banana_bread`'s**: 63.6% of
  its 121 posts also match `banana_bread`'s keywords (see A.2) — largely a real "banana bread
  baked oats" recipe genre, not a data error. This compounds its existing small-sample caveat
  above; don't present `baked_oats`' regression coefficients with the same confidence as the other
  4 trends for this reason as well.

### C.6 Predictive/causal framing (stated plainly, per rubric requirement)

- **This is a descriptive/associational analysis. It is not causal and not predictive.**
- **Not causal:** no causal identification strategy is used or claimed anywhere in this analysis.
  Both the outcome (engagement) and every predictor (linguistic/content features) are observed
  characteristics of the same post, not the result of an experimental or quasi-experimental
  assignment. There is no instrument, discontinuity, matched comparison, or other identification
  device.
- **Not predictive:** there is no train/test split, no held-out evaluation set, and no predictive
  accuracy metric reported anywhere in this analysis. The reported R² values describe in-sample
  fit only. Source docs explicitly instruct against causal or confirmatory language when
  describing cross-trend patterns from this analysis (e.g. "consistent with," "suggestive of," not
  "predicts" or "causes").

---

## D. Method #7: Undiscovered Trends (LDA Topic Discovery)

This analysis uses a **separate data cleaning pipeline** from Sections A–C; it does not reuse
`trends_combined_english.csv`.

### D.1 Why a separate dataset is required

- LDA must not run on `trends_combined_english.csv`, because every row in that file already
  matched one of the 5 trend keyword lists by construction (Section A.2) — running LDA on it could
  only re-derive the same 5 known categories, not discover anything new.
- Running on the full English-filtered raw dataset (in-trend and out-of-trend posts together) was
  considered and rejected: even with trend-specific vocabulary excluded from the model, the sheer
  volume of the largest known trend (sourdough, ~136,000 posts) could still let its own sub-topics
  dominate the topic model purely by post count, diluting the actual discovery signal.
- Decision: build a dedicated dataset containing **only** posts outside all 5 known trends.

### D.2 Dataset construction (`non_trend_english.csv`)

Notebook: `code_english_only/lda/00_build_dataset_and_config.ipynb` **[LINE_TBD]**

- **Step 1:** concatenate the same 4 raw pull files listed in A.1 directly (not the 5 per-trend
  CSVs used in Section A — this pipeline starts one level further upstream).
- **Step 2:** filter to English only (`lang == "en"`).
- **Step 3:** exclude, by post `id`, any post matching any of the 5 trend keyword sets (same
  `trend_keywords` dict as A.2, applied here as an **exclusion** filter rather than a labeling/
  inclusion filter): a set of matching `id` values is collected via keyword search, then
  `df_full[~df_full["id"].isin(trend_ids)]` drops every row whose `id` is in that set. **Verified
  while fact-checking this document: `id` is perfectly unique in the raw 4-file source this step
  reads (298,635 rows, 298,635 unique `id` values, 0 duplicates)** — so this exclusion step is not
  affected by the multi-trend-match phenomenon described in A.2 (that phenomenon only arises once
  posts are split across the 5 separate per-trend CSVs, which this pipeline does not use).
- **Step 4:** drop rows with **neither** `text` nor `hashtags` present (i.e., keep any row with at
  least one of the two — this is a looser criterion than Section A's pipeline, which is unaffected
  since it doesn't apply this specific null-check).
- Output: `cleaned_data/non_trend_english.csv`.
- **No merge/join occurs in this pipeline either** — it is concatenation followed by sequential
  row-level filtering, structurally identical in kind to Section A's pipeline, just with an
  exclusion filter (Step 3) instead of an inclusion/labeling filter.

### D.3 Preprocessing (applied identically to both the `text` model and the `hashtags` model)

Notebooks: `code_english_only/lda/01a_lda_text.ipynb`, `code_english_only/lda/01b_lda_hashtags.ipynb`
**[LINE_TBD each]**

- **Two fully separate models**, one fit on `text`, one fit on `hashtags`, not combined. Rationale:
  replicates the project's reference paper's finding that text and hashtags surface meaningfully
  different content types (text skewing personal/narrative, hashtags skewing promotional/
  discovery-oriented) — worth checking for the same pattern here rather than picking one field
  arbitrarily.
- **No sampling — the full 73,933-row dataset is used for both models.** Rationale: LDA runs
  locally with no API cost pressure (unlike Method #6's zero-shot classification step), and
  sampling would work directly against the discovery goal, since rare/emerging patterns are
  exactly what a sample is most likely to lose.
- **Text cleaning:** lowercase; strip URLs (`re.sub(r"http\S+", "", text)`); strip all non-letter
  characters (`re.sub(r"[^a-z\s]", " ", text)`). **This regex is ASCII-only** — documented
  elsewhere in this project as truncating non-Latin-script hashtags (e.g. Korean-script tags) into
  meaningless short fragments rather than cleanly removing them. This is a known preprocessing
  limitation of the current pipeline, not something that was subsequently fixed.
- **Lemmatization:** NLTK `WordNetLemmatizer`, POS-aware — tokens are POS-tagged first so that verb
  forms lemmatize correctly (e.g. "baking"/"baked"/"bakes" → "bake"; a plain default-noun
  lemmatizer would leave these unchanged).
- **Vectorization:** `sklearn.feature_extraction.text.CountVectorizer`, `ngram_range=(1,1)`
  (unigrams only — see D.4 for why), `min_df=5`, `max_df=0.5`.
- **Stopwords:** sklearn's standard English stopword list, plus a custom list in 5 categories:
```python
tokenization_fragments = ["s", "t", "m", "ve", "g"]
platform_artifacts = ["mention", "redacted", "link", "bio"]
recipe_boilerplate = ["recipe", "make", "made", "food", "like", "just", "delicious",
                       "love", "good", "day", "today", "time"]
units_measurements = ["cup", "tsp", "tbsp", "add", "salt", "oil", "minutes", "ingredients"]
trend_single_words = ["sourdough", "dalgona"]
```
- **Explicitly retained in the vocabulary** (i.e., deliberately *not* stopworded, despite being
  common ingredient words): `cheese`, `bread`, `coffee`, `banana`, `oats`, `oatmeal`, `chocolate`,
  `baked`, `baking`, `feta`. Rationale: excluding generic ingredient words would block the model
  from ever discovering a genuinely different trend built on the same common ingredient (e.g. a
  different feta-based dish, a different coffee drink), which is the entire point of this task.
- **Implementation detail worth flagging as a methods note:** the custom stopword list itself must
  be lemmatized (keeping all 4 part-of-speech variants of each word) before being passed to
  `CountVectorizer`, because the vectorizer applies its stopword filter *after* tokenization/
  lemmatization runs. Without this step, an inflected stopword (e.g. "minutes," "redacted") would
  not match its lemmatized form and would silently leak into the topic vocabulary uncaught.

### D.4 Rationale: `ngram_range=(1,1)` (unigrams only), not `(1,2)`

- The original plan used bigrams (`(1,2)`) specifically so that 8 known trend phrases (e.g.
  "banana bread," "dalgona coffee") could be excluded as two-word units while their individual
  component words stayed available in the vocabulary. This was revised after confirming, directly
  against `non_trend_english.csv`, that all 8 of those exact phrases have **zero** occurrences in
  the dataset by construction (D.2's Step 3 already excludes every post matching a trend keyword,
  including these phrases). The bigram-exclusion mechanism was solving a problem that did not exist
  in this dataset, while adding real complexity (documented as the root cause of an implementation
  bug class and added complication to coherence scoring). Switching to unigrams-only removed that
  cost with no loss to the original exclusion goal.

### D.5 Model and K-selection procedure

- **Model:** `sklearn.decomposition.LatentDirichletAllocation`, `random_state=42`.
- **K values tested:** 5, 10, 15, 20, 25 — tested independently for the text model and the
  hashtags model (10 fitted models total across both).
- **Coherence metric:** `gensim`'s `c_v` `CoherenceModel`, **not** sklearn's built-in
  `LatentDirichletAllocation.score()`. Rationale: sklearn's `.score()` returns log-likelihood, a
  measure of statistical fit to the data — it does not measure whether a topic's top words are
  semantically related to each other, which is what "coherence" is actually assessing here. Using
  log-likelihood as a stand-in for topic quality would have been a methodological error.
- **Final K selection rule:** the highest `c_v` coherence score within each model's own tested
  range, applied consistently across both models. Final K used: **20 for the text model, 10 for
  the hashtags model.**
- **[Flag — see Open Questions]:** no held-out/train-test split is used anywhere in this
  coherence-based K selection — coherence is computed on the same corpus each model is fit on, for
  every tested K. This is standard practice for this kind of unsupervised hyperparameter tuning,
  but it is not explicitly discussed or justified as a methodological choice in any project doc
  reviewed for this reference.

### D.6 Topic interpretation and classification procedure

- For each fitted topic (in both the text and hashtags models): the top 15 highest-weighted words
  are pulled, along with the top-scoring real example posts (by that topic's document-topic
  weight).
- Each topic is manually assigned a human-readable label, and classified into exactly one of 3
  categories: **(a)** a likely genuine new food trend, **(b)** a sub-cluster within one of the 5
  known trends, or **(c)** generic noise/unrelated content. All three categories are reported, not
  only the topics classified as interesting, so that "we checked and mostly found noise" remains a
  valid, reportable outcome if that is what the data shows.
- **Documented validity limitation:** the project's original design specified a 4th, formally
  separate notebook (`02_interpret_topics.ipynb`) to perform this labeling step in a reproducible,
  re-runnable way. That notebook was never built. Labeling was instead performed directly by
  reading each topic's top words and example posts (the same manual real-text-review method used
  to validate the Method #6 zero-shot label set, C.1.b), without a separate, independently
  re-verified review pass. This should be stated plainly as a limitation in the paper — the topic
  labels/classifications are a documented first-pass read, not independently re-verified against a
  second reviewer or a formal rubric.

### D.7 Predictive/causal framing

- **Not predictive.** LDA is an unsupervised topic-discovery method. There is no held-out test set
  and no predictive-accuracy metric — coherence (D.5) evaluates internal topic quality, not
  predictive performance on unseen data.
- **Not causal.** Topic assignment is a descriptive, exploratory output of an unsupervised model,
  not a tested hypothesis or a causal claim about why certain content clusters together.

---

## Open Questions / Flags

Per instruction, these are documented gaps, ambiguities, or inconsistencies found while writing
this reference — not resolved by guessing:

1. **Original per-trend CSV construction not directly reviewed.** The 5 trend-specific CSVs used
   as Section A's input were built via keyword matching against the 4 raw pull files in
   `code/02_data_cleaning.ipynb`, which is explicitly out of scope for the `code_english_only/`
   project folder and was not reviewed for this document. The keyword-matching *logic* (regex
   `.str.contains` against `text` OR `hashtags`) is documented in `MASTER_PROJECT_REFERENCE.md`
   and independently re-implemented identically in two other places in this codebase (the LDA
   dataset build, D.2; the creator-network build), which is strong indirect evidence for how the
   original 5 CSVs were built — but the original script itself was not directly confirmed.
2. **No documented rationale for the lack of a multiple-comparisons correction in Method #5's 10
   pairwise Mann-Whitney tests.** A "no multiple-comparisons correction" decision is documented for
   Method #6's regression coefficients specifically, but no equivalent documented rationale covers
   Method #5's pairwise comparisons. Confirm intent before stating a justification in the paper.
3. **feta_pasta's final English-cleaned sample size is inconsistent across docs: 302 vs. 303.**
   `ENGLISH_ONLY_RESULTS.md` (Tasks 1, 2, and 5) consistently states 302, sourced from actual
   notebook output. `MASTER_PROJECT_REFERENCE.md` (Section 7, twice) and
   `PER_TREND_REGRESSIONS_STEP_BY_STEP.md`'s reliability table state 303. This document uses 302
   throughout, per `ENGLISH_ONLY_RESULTS.md`'s framing as the actual-output source of record — the
   discrepancy should be reconciled before finalizing the paper's methods numbers.
4. **LDA's K-selection coherence scoring has no held-out/train-test split**, and this isn't
   discussed as a limitation anywhere in the docs (see D.5). Standard practice for this kind of
   tuning, but worth a deliberate one-line acknowledgment in the paper's methods/limitations
   section rather than silence.
5. **The formal LDA topic-interpretation notebook was never built** (D.6) — flagged there as a
   stated limitation, repeated here since it's the most significant validity gap this document
   surfaced and the rubric specifically asks about reliability/validity caveats.
6. **RESOLVED: `id` uniqueness checked directly against the raw data — not a concern for the LDA
   pipeline.** A code comment in `01_zero_shot_classification.ipynb` noting "~1,098 duplicates" in
   `id` initially looked like it could undermine the LDA dataset build's `id`-based exclusion step
   (D.2, Step 3). Checked directly: `id` is perfectly unique (0 duplicates) in the raw 4-file
   source that both the LDA pipeline and the original per-trend CSVs are built from. The
   "duplicates" the zero-shot notebook's comment refers to are the same multi-trend-matched-post
   phenomenon noted in A.2, which only exists in `trends_combined_english.csv`, a dataset the LDA
   pipeline never reads. No action needed on this one.
