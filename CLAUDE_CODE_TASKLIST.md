# Claude Code Task List: Method #6/#7 (Virality Prediction + Topic Discovery)

This is the primary, concrete, step-by-step doc for Claude Code to execute against, and for the
project owner to track progress against. For background/design reasoning behind any given decision,
see `HANDOFF_method6_virality_predictors.md` and `MASTER_PROJECT_REFERENCE.md`, but this doc is
the one to actually follow task-by-task.

**Do not touch `04_statistical_tests.ipynb` or its results at any point in these tasks, that
remains explicitly out of scope.**

**All notebooks for Tasks 1-6 below go in a new folder, `code_english_only/`, a sibling to the
existing `code/` folder (same level as `data/` and `cleaned_data/`). Do not add these notebooks to
the original `code/` folder.** This keeps the original all-language pipeline (notebooks 01-05,
including `04_statistical_tests.ipynb`) fully separate from this English-only virality-focused
work. Relative paths to `data/` and `cleaned_data/` stay the same pattern, adjusted for depth, see
each task's exact steps below. **Tasks 3, 4, and 5 (rule-based features, zero-shot classification,
and the regression model itself) nest inside a `regression/` subfolder**, since they form one
connected sequence (feature-build → labeling → model). Tasks 1, 2, and 6 stay at the top level of
`code_english_only/`, since they're standalone analyses that don't feed into the regression.

**Full project path (Windows):**
`C:\Users\minni\Documents\GitHub\Manya QSS20 Final Project\Final Project Progress`
Note the spaces in `Manya QSS20 Final Project` and `Final Project Progress`, quote this path in
any shell commands, unquoted spaces will break path resolution.

```
Final Project Progress\    <- this is the project root
├── cleaned_data\
├── code\                          (existing, untouched: 01-05, including 04_statistical_tests.ipynb)
├── code_english_only\             (NEW)
│   ├── 00_data_cleaning.ipynb            (Task 1)
│   ├── 01_engagement_analysis.ipynb      (Task 2)
│   ├── regression\                       (NEW subfolder, Tasks 3-5)
│   │   ├── 00_rule_based_features.ipynb        (Task 3)
│   │   ├── 01_zero_shot_classification.ipynb   (Task 4)
│   │   └── 02_regression_models.ipynb          (Task 5)
│   └── 02_lda_topic_discovery.ipynb      (Task 6)
├── data\
└── Data Visualizations\
```

**Path note for notebooks inside `regression/`:** since that subfolder sits one level deeper than
`code_english_only/`, relative paths to `cleaned_data/` need an extra `../` (i.e.
`../../cleaned_data/...`, not `../cleaned_data/...`). Notebooks directly inside
`code_english_only/` (Tasks 1, 2, 6) keep the single `../cleaned_data/...` pattern.

## Task overview

| # | Task | Notebook | Status | Depends on |
|---|---|---|---|---|
| 1 | Data cleaning: combine, correct `is_covid_framed`, filter English | `code_english_only/00_data_cleaning.ipynb` | **Tested against real data, ready to build** | Nothing |
| 2 | Rerun Method #5 (engagement) English-only | `code_english_only/01_engagement_analysis.ipynb` | Ready to execute, **stop for results review before continuing** | Task 1 |
| 3 | Build rule-based linguistic features | `code_english_only/regression/00_rule_based_features.ipynb` | Ready to execute | Task 1 |
| 4 | Zero-shot classification (multi-label) | `code_english_only/regression/01_zero_shot_classification.ipynb` | Needs one scope decision first | Task 1, 3 |
| 5 | Build regression models (Method #6) | `code_english_only/regression/02_regression_models.ipynb` | Ready once Tasks 3-4 done | Tasks 3, 4 |
| 6 | LDA topic discovery (Method #7) | `code_english_only/02_lda_topic_discovery.ipynb` | Not yet fully designed, see note | Task 1 |

Work through tasks in order where a dependency exists. Note: Task 5 no longer strictly depends on
Task 2's output (the regression baseline is now fixed as sourdough regardless, see Task 5), but
Task 2 should still be completed before or alongside Task 5 for accurate write-up reporting.

---

# Task 1: Data Cleaning (Build `trends_combined_english.csv`)

## Goal

One self-contained cleaning notebook, replacing what was previously split into a separate
"patch existing CSVs" step and a separate "build combined dataset" step. Loads the 5 already
trend-filtered CSVs, recomputes `is_covid_framed` with a corrected keyword list, combines,
filters to English-only, drops rows unusable for the outcome variables, and saves one clean
dataset that everything else in this folder builds on.

**Explicitly out of scope for this notebook: do not reprocess USDA/grocery store or trade data.**
That work stays entirely in the original `code/02_data_cleaning.ipynb` and is not touched here.

## Why `is_covid_framed` needed correcting

Originally computed with only 3 hashtag-style keywords (`quarantinebaking`, `pandemicbaking`,
`quarantinecooking`), matching the professor's original data-pull hashtag category. Real data
checks showed this substantially undercounts actual pandemic-related language in post text (posts
can mention "pandemic," "lockdown," "coronavirus," etc. without using the specific hashtag format).
Confirmed the broader list below is a strict superset of the original 3, nothing lost, only gained.

## Input files

Located in `cleaned_data/`: `feta_pasta.csv`, `sourdough.csv`, `banana_bread.csv`,
`baked_oats.csv`, `dalgona_coffee.csv`. **Note: none of these files have a `trend` column**, it
must be added per file before combining.

## Exact steps

**Step 1: Load each of the 5 CSVs, add a `trend` column, and recompute `is_covid_framed` with the
corrected keyword list, before combining.**
```python
trend_files = {
    "feta_pasta": "feta_pasta.csv",
    "sourdough": "sourdough.csv",
    "banana_bread": "banana_bread.csv",
    "baked_oats": "baked_oats.csv",
    "dalgona_coffee": "dalgona_coffee.csv",
}

covid_keywords = ["covid", "coronavirus", "pandemic", "quarantine", "lockdown"]
covid_pattern = "|".join(covid_keywords)

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
**Keep the full original column set** (no column dropping). Print `combined.shape` and
`combined["trend"].value_counts()`, confirm against known per-trend totals (feta_pasta=332,
sourdough=136,692, banana_bread=27,153, baked_oats=125, dalgona_coffee=26,402; total=190,704).

**Step 2: Filter to English-only.**
```python
combined = combined[combined["lang"] == "en"].copy()
```
Confirm `combined.shape` matches the known English total: 126,730 rows.

**Step 3: Drop rows with null `statistics.like_count` or `statistics.comment_count`.**
```python
before = len(combined)
combined = combined.dropna(subset=["statistics.like_count", "statistics.comment_count"]).copy()
print(f"Dropped {before - len(combined)} rows with null likes/comments")
```
Expect to drop ~320 rows (confirmed against real data).

**Step 4: Do NOT filter on null `hashtags`.** A null/missing `hashtags` value is legitimate (a
post can have no hashtags and still be usable), not something to exclude. When `hashtag_count` is
built in Task 3, null should become `0`, not an excluded row.

**Step 5: Confirm `text` has no nulls (sanity check, not a filtering step).**
```python
null_text_count = combined["text"].isna().sum()
print(f"Null text count (should be 0): {null_text_count}")
assert null_text_count == 0, "Unexpected: null text found after English filter, investigate before proceeding"
```

**Step 6: Save the result.**
```python
combined.to_csv("../cleaned_data/trends_combined_english.csv", index=False)
print(f"Saved trends_combined_english.csv, shape: {combined.shape}")
```
Save the notebook itself as `code_english_only/00_data_cleaning.ipynb`.

## Expected final result (confirmed by running this exact pipeline against real data)

- Final shape: **(126,410, 23)** — 126,410 rows, 23 columns (22 original + `trend`)
- Per-trend `is_covid_framed` counts after correction and English filtering:
  feta_pasta=19, sourdough=7,769, banana_bread=3,583, baked_oats=46, dalgona_coffee=2,840
- `lang` column entirely `"en"` at this point (kept in the file, even though now constant)
- 0 nulls in `text`, `statistics.like_count`, `statistics.comment_count`
- `hashtags` will still have nulls (~10.7% of rows), expected and correct, not a bug

## What NOT to do in this task

- Do not touch `04_statistical_tests.ipynb`, USDA data, or trade data, out of scope
- Do not filter out null `hashtags` rows
- Do not drop any columns from the original files
- Do not build any engineered features yet (sentiment, word count, zero-shot labels, etc.), that's
  Task 3, this task only produces the clean combined base dataset

---

# Task 2: Rerun Method #5 (Engagement Analysis), English-Only

## Goal

Redo the Kruskal-Wallis/Mann-Whitney engagement comparison from `05_engagement_analysis.ipynb`, but
on `trends_combined_english.csv` (built in Task 1) instead of the original all-language data. This
produces the accurate English-only engagement ranking for the project write-up. **Note: this task
no longer determines the regression baseline for Task 5, sourdough is fixed as the baseline for
both models regardless of this task's outcome (see Task 5 for the corrected reasoning), but this
rerun is still needed for accurate reporting.**

## Why this matters (short version)

The original all-language run found sourdough as the lowest-engagement trend. Real data checks
already showed this flips once restricted to English-only, banana_bread becomes the lowest instead.
This is worth knowing and reporting accurately, even though it no longer drives the regression
baseline decision.

## Exact steps

**Step 1: Load `../cleaned_data/trends_combined_english.csv`, confirm shape matches Task 1's
output.** (Single `../`, this notebook lives directly inside `code_english_only/`, not inside the
deeper `regression/` subfolder.)
```python
combined = pd.read_csv("../cleaned_data/trends_combined_english.csv")
print(combined.shape)
```

**Step 2: Repeat the exact same analysis as `05_engagement_analysis.ipynb`**, just pointed at this
new file instead of the 5 separate per-trend CSVs:
- Kruskal-Wallis test on `statistics.like_count` and `statistics.comment_count`, across the 5
  `trend` groups
- Log-transformed one-way ANOVA as the secondary robustness check
- Pairwise Mann-Whitney U for all 10 trend pairs, for both likes and comments
- Distribution histograms (log1p) for both metrics, per trend
- Styled pairwise comparison tables (the `.style.apply(highlight_significant, ...)` version built
  last, not the plain-text print version)

**Step 3: Report the trend with the lowest median `statistics.like_count`, for accurate write-up
reporting** (this no longer sets the regression baseline, see the note above, but is still useful
context and should be reported):
```python
median_by_trend = combined.groupby("trend")["statistics.like_count"].median().sort_values()
print("Trend ranking, lowest to highest median likes (English-only):")
print(median_by_trend)
```

**Step 4: Compare against the original all-language results** (already in
`MASTER_PROJECT_REFERENCE.md` Section 10). Note explicitly which findings hold up (expect
baked_oats to remain the clear highest either way) and which change (expect the lowest-ranked trend
to differ).

**Step 5: Save this rerun as a new notebook in `code_english_only/`, do not overwrite
`05_engagement_analysis.ipynb` (which stays in the original `code/` folder).**
Suggested name: `code_english_only/01_engagement_analysis.ipynb` (this can be built and run any
time after Task 1, it doesn't depend on Tasks 3/4's features, the number just reflects a sensible
reading order within the folder), keeping the original as a documented
"all-language" reference point, not deleting it.

**Step 6: Report back the results directly in the conversation, and STOP.** Present, in this order:
1. The overall Kruskal-Wallis results (H-statistic, p-value) for likes and comments
2. The median engagement ranking, lowest to highest
3. The full pairwise Mann-Whitney U table for both likes and comments (which pairs are significant
   at p<0.05, which aren't, and which direction/which trend has the higher median in each pair)
4. A direct comparison against the original all-language results, calling out specifically which
   findings held up and which changed

**Do not update `MASTER_PROJECT_REFERENCE.md` Section 10 yet, and do not proceed to Task 3 or any
other task, until the project owner has reviewed these results and explicitly says to continue.**
This is a deliberate review checkpoint, the project owner wants to go over what's significant and
what changed before anything downstream is built on top of these results.

## What NOT to do in this task

- Do not overwrite or delete `05_engagement_analysis.ipynb`
- Do not update `MASTER_PROJECT_REFERENCE.md` or proceed to any other task until the project owner
  has reviewed Step 6's results and given explicit approval to continue
- Do not proceed to Task 5 until this task's actual lowest-median trend is known, do not guess or
  reuse sourdough by default

---

# Task 3: Build Rule-Based Linguistic Features

## Goal

Add the fast, free, non-API-dependent features to `trends_combined_english.csv`, so they're ready
to use in Task 5's regression regardless of how Task 4 (zero-shot) goes.

## Exact steps

**Step 1: Load `../../cleaned_data/trends_combined_english.csv`** (note the extra `../`, this
notebook lives one level deeper, inside `code_english_only/regression/`).
```python
combined = pd.read_csv("../../cleaned_data/trends_combined_english.csv")
print(combined.shape)
```

**Step 2: Word count and log-transformed word count.**
```python
combined["word_count"] = combined["text"].str.split().str.len()
combined["log_word_count"] = np.log1p(combined["word_count"])
```
Confirmed already: raw word count skewness = 1.692 (substantially right-skewed), log-transformed
skewness = -0.398 (close to symmetric). Use `log_word_count` as the actual regression predictor.

**Step 3: Sentiment score via VADER.**
```python
# pip install vaderSentiment if not already installed
from vaderSentiment.vaderSentiment import SentimentIntensityAnalyzer
analyzer = SentimentIntensityAnalyzer()
combined["sentiment_score"] = combined["text"].apply(lambda t: analyzer.polarity_scores(t)["compound"])
```
`compound` score ranges -1 (most negative) to +1 (most positive), the standard single-number VADER
summary score.

**Step 4: Hashtag count, treating null as 0 (not as missing/excluded).**
```python
combined["hashtag_count"] = combined["hashtags"].fillna("").apply(
    lambda h: len(h.split(",")) if h else 0
)
```
Adjust the split logic to match the actual `hashtags` column format (check a few real values first,
this project's `hashtags` column has historically been a stringified list like
`["quarantinebaking","baking",...]`, may need `ast.literal_eval` or regex extraction instead of a
plain comma split, verify against real data before finalizing this exact line).

**Step 5: Punctuation and emoji features.** Original plan only covered exclamation/question marks;
emoji count was added after checking real prevalence (56.6% of a 2,000-post sourdough English
sample had at least 1 emoji, mean 2.01 per post). Mention count and all-caps-word were considered
and dropped: mentions bundle several distinct behaviors (tagging a recipe creator, a brand, or a
friend) into one ambiguous feature; all-caps words are frequently just a recipe-title formatting
convention (e.g. "STRAWBERRY BANANA BAKED OATMEAL!") rather than a clean emphasis/shouting signal,
and likely redundant with `exclamation_count` and the `recipe_instructional` zero-shot label anyway.
```python
# pip install emoji if not already installed
import emoji

combined["exclamation_count"] = combined["text"].str.count("!")
combined["question_count"] = combined["text"].str.count(r"\?")
combined["emoji_count"] = combined["text"].apply(lambda t: emoji.emoji_count(str(t)))
```

**Step 6: Log-transformed outcome variables.**
```python
combined["log_likes"] = np.log1p(combined["statistics.like_count"])
combined["log_comments"] = np.log1p(combined["statistics.comment_count"])
```

**Step 7: Confirm `is_covid_framed` already exists and needs no changes** (carried through from
earlier cleaning, just confirm it's present as a boolean/0-1 column in the loaded file).

**Step 8: Save the updated file as `../../cleaned_data/trends_combined_english_features.csv`**
(do not overwrite `trends_combined_english.csv` from Task 1, keep that as the clean base dataset;
this new file adds the engineered columns on top). Save the notebook itself as
`code_english_only/regression/00_rule_based_features.ipynb`.

## What NOT to do in this task

- Do not attempt zero-shot classification here, that's Task 4, kept separate since it needs a
  scope decision first and involves API calls, this task should be fast and free
- Do not bucket `word_count` into short/medium/long categories, keep continuous, already decided

---

# Task 4: Zero-Shot Classification (Multi-Label)

## Status: scope decided, prompt finalized, ready to run at full scale

## Goal

Classify each post into one or more of 5 categories using Claude API (via Claude Code), as
multi-label binary indicator columns, not a single forced category.

## Label set (revised — `trend_commentary` dropped entirely, `spam_low_content` redefined)

```
recipe_instructional, personal_lifestyle, media_repost, meme_joke, spam_low_content
```

`trend_commentary` was removed with no replacement: posts that only comment on the trend
phenomenon itself (not personal, not a repost, not a joke, not low-content) are expected and
intended to end up with **zero labels** — this is correct, not a gap to fix.

## Finalized system prompt (sent verbatim to the API)

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

This went through two revision rounds against real sampled post text: v1 (6 labels, including
`trend_commentary`) looked reasonable but conflated "promotional" with "spam." v2 (5 labels,
`trend_commentary` dropped) exposed two issues — a post with real recipe content plus a heavy
hashtag tail got only `spam_low_content` (missing `recipe_instructional`), and two structurally
similar personal-voice-plus-hashtags posts got inconsistent labels (one `spam_low_content` only,
one `personal_lifestyle` only) — both traced to the model treating "hashtag tail present" as a
spam proxy instead of judging the caption's substance on its own. The clarifications above (v3,
finalized) were added specifically to fix this: labels evaluated independently, hashtags ignored
when judging substance, and promotional intent alone explicitly ruled out as a spam signal.

## Scope decision (confirmed with project owner)

`trends_combined_english.csv` has ~126,400 rows after Task 1. Sourdough alone is ~96,000+ of
those. **Decided: a stratified sample, not the full dataset** — sourdough, banana_bread, and
dalgona_coffee capped at 3,000 posts each (random sample, fixed seed), baked_oats (121) and
feta_pasta (302) kept at full size since they're already small (~9,400 posts total). **Model:
`claude-sonnet-5`. Call mode: Anthropic's async Message Batches API** (50% cheaper than
real-time calls, no manual rate-limit pacing needed).

**Downstream consequence, explicitly accepted:** because this is a sample rather than the full
dataset, the features file this task saves contains only the ~9,400 sampled/classified rows, not
all 126,410. Task 5's regression therefore runs on this same stratified sample, not the full
English-only dataset.

## Exact steps (once scope is confirmed)

**Step 0: Load `../../cleaned_data/trends_combined_english_features.csv`** (Task 3's output, into a
DataFrame named `combined`, matching the variable name used in the code below; note the extra
`../`, this notebook also lives inside `code_english_only/regression/`). Do not load Task 1's
`trends_combined_english.csv` directly, that file lacks the rule-based features Task 3 already
added; building on top of Task 3's output avoids redoing that work.
```python
combined = pd.read_csv("../../cleaned_data/trends_combined_english_features.csv")
print(combined.shape)
```

**Step 1: Decide and implement batching.** Classify multiple posts per API call (e.g. batches of
10-20 posts with structured output requesting a label list per post), not one call per post, for
efficiency. Design the actual prompt/schema before running at scale, test on a small batch (~20
posts) first and manually check the outputs look reasonable against the actual post text before
scaling up.

**Step 2: Run classification, saving results incrementally** (checkpoint progress to disk
periodically, e.g. every N batches), so a crash partway through does not lose all completed work.

**Step 3: Add the 5 binary indicator columns to the DataFrame.**
```python
for label in ["recipe_instructional", "personal_lifestyle", "media_repost",
              "meme_joke", "spam_low_content"]:
    combined[f"is_{label}"] = 0  # then set to 1 where the post was assigned that label
```

**Step 4: Sanity-check the classification output.** Print the count/percentage of posts assigned
each label, and pull a small random sample of posts per label to manually confirm the assignment
looks reasonable, similar to how the label set itself was originally sanity-checked against real
text.

**Step 5: Check for multicollinearity between labels** (VIF or a simple co-occurrence correlation
matrix), since some labels may frequently co-occur (e.g. `recipe_instructional` and
`personal_lifestyle`). Report this, do not need to fix it, just flag it before Task 5 interprets
coefficients.

**Step 6: Save the updated file** with the new label columns included (save as
`../../cleaned_data/trends_combined_english_features.csv`, adding the zero-shot columns on top of
Task 3's output, same filename, overwriting in place). Save the notebook itself as
`code_english_only/regression/01_zero_shot_classification.ipynb`.

## What NOT to do in this task

- Do not force single-label classification, multi-label (independent binary columns) was the
  decided approach
- Do not run at full 126,400-row scale without explicit confirmation of scope first

---

# Task 5: Build Regression Models (Method #6 Core Analysis)

## Status update: superseded as the primary Method #6 analysis

**The combined pooled model this task describes was built, but is no longer the primary Method #6
analysis.** Per project-owner feedback after reviewing results, the project's actual question is
whether and how the *effect* of each linguistic feature on virality differs across the 5 trends —
a pooled model with `trend` only as an intercept-shifting dummy can't answer that (every trend is
forced to share the same coefficient for every feature except the intercept). **5 separate
per-trend models, built in `regression/03_per_trend_regressions.ipynb` per
`PER_TREND_REGRESSIONS_STEP_BY_STEP.md`, are now the primary Method #6 result.** The steps below
are kept as accurate documentation of how the combined model (still a real, runnable artifact,
just deprioritized) was built — see `MASTER_PROJECT_REFERENCE.md` Section 9 and
`ENGLISH_ONLY_RESULTS.md` Task 5 for the current primary analysis and its results.

## Goal

Two parallel OLS regressions, one predicting `log_likes`, one predicting `log_comments`, using the
features from Tasks 3 and 4, with `trend` included as a control.

## Prerequisites

Tasks 3 and 4 must be complete for their feature outputs. Task 2 is good practice to complete first
for accurate write-up reporting, but its output no longer determines anything in this task, see
below.

## Baseline decision (settled, not dependent on Task 2)

**Sourdough is the fixed baseline trend for BOTH the likes model and the comments model.** This was
revised from an earlier plan to use "whichever trend has the lowest engagement," for two reasons:
(1) baseline choice is a narrative convenience, not a statistical requirement, what actually matters
for model stability is using a large, low-noise reference group, and sourdough has by far the
largest sample (96,479 English posts); (2) using "lowest" risked different baselines for the likes
vs. comments models (they were not the same trend even in the original all-language analysis),
which would make the two models hard to compare directly. Using one fixed, large-sample baseline for
both models avoids this and keeps them directly comparable. Do not use Task 2's ranking to pick a
different baseline.

## Exact steps

**Step 1: Load `../../cleaned_data/trends_combined_english_features.csv`** (the output of Task 3,
further updated by Task 4 with the zero-shot label columns added on top, same filename throughout,
Task 4 does not create a separate file; note the extra `../`, this notebook also lives inside
`code_english_only/regression/`).
```python
combined = pd.read_csv("../../cleaned_data/trends_combined_english_features.csv")
print(combined.shape)
```

**Step 2: Dummy-encode `trend`, with sourdough as the dropped/baseline category.**
```python
trend_dummies = pd.get_dummies(combined["trend"], prefix="trend", drop_first=False)
trend_dummies = trend_dummies.drop(columns=["trend_sourdough"])
```

**Step 3: Assemble the predictor matrix.**
```python
rule_based_cols = ["log_word_count", "sentiment_score", "hashtag_count",
                    "exclamation_count", "question_count", "emoji_count"]
zero_shot_cols = ["is_recipe_instructional", "is_personal_lifestyle", "is_media_repost",
                   "is_meme_joke", "is_spam_low_content"]

predictor_matrix = pd.concat([
    combined[rule_based_cols],
    combined["is_covid_framed"].astype(int),  # cast bool to 0/1, regression needs numeric
    combined[zero_shot_cols],
    trend_dummies,
], axis=1)
```
Combine: `log_word_count`, `sentiment_score`, `hashtag_count`, `exclamation_count`,
`question_count`, `emoji_count`, `is_covid_framed` (cast to int, not left as boolean), the 6
zero-shot label binary columns from Task 4, and the trend dummy columns from Step 2.

**Step 4: Fit the likes model.**
```python
import statsmodels.api as sm
X = sm.add_constant(predictor_matrix)
model_likes = sm.OLS(combined["log_likes"], X).fit()
print(model_likes.summary())
```

**Step 5: Fit the comments model**, same predictor matrix, `log_comments` as the outcome. Note in
the interpretation: comments have confirmed zero-inflation (unlike likes' clean log-normal shape),
so this model's fit should be treated as less reliable than the likes model, state this explicitly
when reporting results, do not present both models with equal confidence.

**Step 6: Report coefficients, p-values, and R² for both models.** Expect R² to be fairly low
(individual post virality is inherently hard to predict from a handful of text features), state
this plainly if it happens, not as a failure.

**Step 7: Multicollinearity check (VIF)** across the full predictor set (not just the zero-shot
labels from Task 4's Step 5), since word count/hashtag count/etc. may also correlate with each
other.

**Step 8: Save this as `code_english_only/regression/02_regression_models.ipynb`.**

## What NOT to do in this task

- ~~Do not build 5 separate per-trend models, combined model with trend as control was the decided
  approach (Option A)~~ — **reversed, see the Status update at the top of this task.** 5 separate
  per-trend models were built and are now the primary Method #6 analysis.
- Do not apply a multiple-comparisons correction (e.g. Bonferroni), already decided against for
  this project's scope
- Do not add interaction terms (trend × feature), additive-only model already decided as acceptable
  for this project's scope

---

# Task 6: LDA Topic Discovery (Method #7)

## Status: not yet fully designed, do not build without further design discussion first

This task's purpose (running LDA on the full English-filtered dataset to check for COVID-era food
trends the original 5 keyword lists may have missed) was established early in the project, but
implementation details (number of topics K, preprocessing specifics, whether to run separately on
`text` vs. `hashtags`, how to interpret/label resulting topic clusters) have not been worked
through in as much depth as Tasks 1-5 above. **Do not build this task from assumptions.** Flag this
back to the project owner for a short design discussion (similar to how Task 4's label set was
sanity-checked against real sampled data before finalizing) before writing code, rather than
guessing at parameters like K. Once designed, this would be saved as
`code_english_only/02_lda_topic_discovery.ipynb`, for consistency with the folder's numbering.

