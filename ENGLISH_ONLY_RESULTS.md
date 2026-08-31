# English-Only Pipeline: Updated Results

This document is the complete, non-summarized reference for every result produced by the
English-only pipeline (`Final Project Progress/code_english_only/`). It is meant to be worked
from directly when writing the final report — every number below is pulled from actual notebook
outputs, saved model files, or saved CSVs, not from design-doc estimates or placeholders.

---

## Task 1: Data Cleaning

**Status: Complete.**

### `raw_combined`

**What it represents:** the concatenation of the 5 already trend-filtered CSVs
(`feta_pasta.csv`, `sourdough.csv`, `banana_bread.csv`, `baked_oats.csv`, `dalgona_coffee.csv`).
Each of these 5 files was already built, in an earlier and out-of-scope cleaning stage, by
matching each post's `text`/`hashtags` against that specific trend's keyword list — so the
**per-trend keyword search is already baked into `raw_combined`**; it has not yet had the
English-language filter or the null-row drop applied. Building `raw_combined` itself does two
things on top of that prior keyword-filtering: adds a `trend` label column (none of the 5 source
files had one), and recomputes `is_covid_framed` with the corrected 5-keyword list (`covid`,
`coronavirus`, `pandemic`, `quarantine`, `lockdown` — a superset of the original 3-keyword
hashtag-only version).

**Total rows: 190,704**

### Final cleaned dataset (`trends_combined_english.csv`)

Built from `raw_combined` by applying, in order: (1) the English-language filter
(`lang == "en"`), (2) dropping rows with a null `statistics.like_count` or
`statistics.comment_count`.

- After English filter: 126,730 rows (from 190,704)
- After dropping 320 rows with null likes/comments: **126,410 rows, final**

**Per-trend breakdown of the final cleaned dataset:**

| Trend | Rows | `is_covid_framed` = True |
|---|---|---|
| sourdough | 96,257 | 7,769 |
| banana_bread | 18,378 | 3,583 |
| dalgona_coffee | 11,352 | 2,840 |
| feta_pasta | 302 | 19 |
| baked_oats | 121 | 46 |
| **Total** | **126,410** | **14,257** |

Other confirmed properties of the final cleaned dataset: `hashtags` null rate 10.7% (expected,
not filtered out — a post can legitimately have no hashtags); zero nulls in `text`,
`statistics.like_count`, `statistics.comment_count`.

Output: `cleaned_data/trends_combined_english.csv`
Notebook: `code_english_only/00_data_cleaning.ipynb`

---

## Task 2: Engagement Analysis (English-Only)

**Status: Complete, reviewed and approved by project owner.**

### Overall Kruskal-Wallis results

| Metric | H-statistic | p-value |
|---|---|---|
| Likes | 103.88 | ≈0.0000 |
| Comments | 597.98 | ≈0.0000 |

Both metrics are unambiguously significant — a real difference exists somewhere among the 5
trends. Log-transformed one-way ANOVA (secondary robustness check) agrees: Likes F=20.76, p≈0;
Comments F=175.96, p≈0.

### Median engagement ranking, lowest to highest

**Likes:**

| Trend | N | Median likes |
|---|---|---|
| banana_bread | 18,378 | 197 |
| feta_pasta | 302 | 208 |
| sourdough | 96,257 | 223 |
| dalgona_coffee | 11,352 | 244 |
| baked_oats | 121 | 420 |

**Comments:**

| Trend | N | Median comments |
|---|---|---|
| sourdough | 96,257 | 8 |
| dalgona_coffee | 11,352 | 10 |
| feta_pasta | 302 | 11 |
| banana_bread | 18,378 | 11 |
| baked_oats | 121 | 37 |

### Full pairwise Mann-Whitney U results

**Likes** (sorted by p-value):

| Trend 1 | Median 1 | Trend 2 | Median 2 | p-value | Significant (p<0.05) | Higher median |
|---|---|---|---|---|---|---|
| sourdough | 223 | banana_bread | 197 | 1.32e-15 | Yes | sourdough |
| banana_bread | 197 | dalgona_coffee | 244 | 1.56e-10 | Yes | dalgona_coffee |
| banana_bread | 197 | baked_oats | 420 | 9.69e-09 | Yes | baked_oats |
| sourdough | 223 | baked_oats | 420 | 2.80e-08 | Yes | baked_oats |
| baked_oats | 420 | dalgona_coffee | 244 | 5.20e-06 | Yes | baked_oats |
| feta_pasta | 208 | baked_oats | 420 | 1.63e-05 | Yes | baked_oats |
| sourdough | 223 | dalgona_coffee | 244 | 8.40e-03 | Yes | dalgona_coffee |
| feta_pasta | 208 | banana_bread | 197 | 0.134 | No | feta_pasta (n.s.) |
| feta_pasta | 208 | sourdough | 223 | 0.713 | No | sourdough (n.s.) |
| feta_pasta | 208 | dalgona_coffee | 244 | 0.962 | No | dalgona_coffee (n.s.) |

**Comments** (sorted by p-value):

| Trend 1 | Median 1 | Trend 2 | Median 2 | p-value | Significant (p<0.05) | Higher median |
|---|---|---|---|---|---|---|
| sourdough | 8 | banana_bread | 11 | 1.74e-99 | Yes | banana_bread |
| sourdough | 8 | dalgona_coffee | 10 | 5.96e-30 | Yes | dalgona_coffee |
| sourdough | 8 | baked_oats | 37 | 5.29e-19 | Yes | baked_oats |
| baked_oats | 37 | dalgona_coffee | 10 | 6.34e-12 | Yes | baked_oats |
| banana_bread | 11 | baked_oats | 37 | 9.48e-11 | Yes | baked_oats |
| feta_pasta | 11 | baked_oats | 37 | 1.39e-08 | Yes | baked_oats |
| banana_bread | 11 | dalgona_coffee | 10 | 1.32e-05 | Yes | banana_bread |
| feta_pasta | 11 | sourdough | 8 | 4.95e-03 | Yes | feta_pasta |
| feta_pasta | 11 | dalgona_coffee | 10 | 0.451 | No | feta_pasta (n.s.) |
| feta_pasta | 11 | banana_bread | 11 | 0.870 | No | banana_bread (n.s.) |

**Confirmed statistical ordering** (using only significant pairwise results, per the "tournament
bracket" method):

- Likes: **baked_oats > dalgona_coffee > sourdough > banana_bread**. feta_pasta unplaceable (not
  significantly different from banana_bread, sourdough, or dalgona_coffee — only confirmed
  significantly below baked_oats).
- Comments: **baked_oats > banana_bread > dalgona_coffee > sourdough**. feta_pasta unplaceable
  relative to banana_bread/dalgona_coffee (tied with both), but confirmed significantly above
  sourdough and below baked_oats.

**Bottom line:** baked_oats' dominance on both metrics is unambiguous — it wins every pairwise
comparison it's part of. **feta_pasta's unplaceability is not simply a function of its smaller
sample size (n=302) — it comes from exactly which specific pairwise comparisons do and don't
reach significance, and it's a different pair of trends on each metric:**

- On **likes**, feta_pasta has exactly one significant relationship in the entire pairwise table:
  significantly *below* baked_oats (p=1.63e-05). It is statistically indistinguishable from
  banana_bread (p=0.134), sourdough (p=0.713), and dalgona_coffee (p=0.962). So on likes, all
  that can be confidently said is that feta_pasta is less viral than baked_oats specifically —
  its position relative to the other three trends is genuinely unknown, not just imprecise.
- On **comments**, feta_pasta has two significant relationships: significantly *below* baked_oats
  (p=1.39e-08) and significantly *above* sourdough (p=4.95e-03). It is statistically
  indistinguishable from banana_bread (p=0.870) and dalgona_coffee (p=0.451). So on comments,
  feta_pasta's position is actually bounded on both sides — sourdough < feta_pasta < baked_oats —
  the real ambiguity is only about where it falls relative to banana_bread and dalgona_coffee
  specifically, which is a narrower and more precise statement than "unplaceable."

**Note:** sourdough is fixed as the regression baseline for both per-trend and — historically —
combined regression treatments regardless of this ranking (per-trend regressions no longer use a
baseline at all, see Task 5).

Output: no new data file (analysis-only rerun of existing `trends_combined_english.csv`)
Notebook: `code_english_only/01_engagement_analysis.ipynb`

---

## Task 3: Rule-Based Linguistic Features

**Status: Complete.**

Loaded `trends_combined_english.csv` (126,410, 23) and added 9 engineered columns:

- `word_count` / `log_word_count`: raw skewness 1.692 (confirmed right-skewed), log-transformed
  skewness -0.398 (confirmed close to symmetric). `log_word_count` used as the regression
  predictor.
- `sentiment_score` (VADER compound, -1 to +1): mean 0.62, median 0.82 — posts skew positive
  overall.
- `hashtag_count`: parsed from the stringified-list `hashtags` column via `ast.literal_eval`
  (confirmed 0 parse failures across all 112,922 non-null values). Mean 16.2, median 17.
  13,488 rows have 0 hashtags (includes the 10.7% with null `hashtags`, correctly treated as 0,
  not excluded).
- `exclamation_count` (mean 1.23), `question_count` (mean 0.31), `emoji_count`: 62.3% of posts
  have at least 1 emoji, mean 3.90 among those that do.
- `log_likes`, `log_comments`: log1p-transformed outcome variables for Task 5.
- `is_covid_framed` confirmed already present as boolean, no changes needed (14,257 True /
  112,153 False).

Output: `cleaned_data/trends_combined_english_features.csv`, shape (126,410, 32)
Notebook: `code_english_only/regression/00_rule_based_features.ipynb`

---

## Task 4: Zero-Shot Classification

**Status: Complete.**

### Scope decision

- **Stratified sample**, not the full ~126,400-row dataset: sourdough, banana_bread, and
  dalgona_coffee capped at 3,000 posts each (random, fixed seed=42); baked_oats (121) and
  feta_pasta (302) kept at full size. Total sampled: **9,423 posts**.
- **Model:** `claude-sonnet-5`. **Call mode:** Anthropic's async Message Batches API (50%
  cheaper, no manual rate-limit pacing).
- **Downstream consequence:** because this is a sample, `trends_combined_english_features.csv`
  now contains only these 9,423 rows (not all 126,410) — Task 5's per-trend regression models run
  on this same stratified sample.

### Label set

Final 5 labels: `recipe_instructional`, `personal_lifestyle`, `media_repost`, `meme_joke`,
`spam_low_content`. These are independent binary categories, not exhaustive: a post that doesn't
clearly fit any of them gets zero labels by design, not as a gap.

The system prompt went through revisions against real sampled post text before finalizing (see
`CLAUDE_CODE_TASKLIST.md` Task 4 for the full verbatim prompt). Two issues surfaced during that
process: the model treating a hashtag tail as a spam proxy rather than judging caption substance
— a post with real recipe content (flour percentages) plus heavy hashtags got only
`spam_low_content` — and two structurally similar personal-voice-plus-hashtags posts being
labeled inconsistently (one `spam_low_content` only, one `personal_lifestyle` only). The
finalized prompt added explicit clarifications: labels are evaluated independently, hashtags are
ignored when judging substance, and promotional/commercial intent alone is not a spam signal.
This fixed the promotional-content misclassification (business/ad posts with real substance now
correctly get zero labels instead of forced `spam_low_content`) and fixed the worst
hashtag-as-spam-proxy case, though one edge case (recipe content buried under a hashtag tail not
also getting `recipe_instructional`) remained only partially resolved — accepted as good enough
to proceed given the overall improvement.

### Run results

- **629/629 batch requests succeeded, 0 errored, 0 canceled, 0 expired.**
- All 9,423 posts received a classification result (0 missing).

**Label prevalence (of 9,423 sampled posts):**

| Label | Count | % |
|---|---|---|
| `is_personal_lifestyle` | 3,074 | 32.6% |
| `is_recipe_instructional` | 2,430 | 25.8% |
| `is_spam_low_content` | 1,967 | 20.9% |
| `is_media_repost` | 1,622 | 17.2% |
| `is_meme_joke` | 566 | 6.0% |

1,440 posts (15.3%) received zero labels — expected and by design.

**Multicollinearity check (co-occurrence correlation matrix):** all pairwise correlations are
weak, ranging from -0.336 (`personal_lifestyle` vs. `spam_low_content`) to -0.007
(`personal_lifestyle` vs. `meme_joke`) — no concerning label overlap.

Output: `cleaned_data/trends_combined_english_features.csv`, overwritten in place, shape
(9,423, 37) — now scoped to the stratified sample only (see scope decision above)
Notebook: `code_english_only/regression/01_zero_shot_classification.ipynb`

---

## Task 5: Regression Models (Method #6) — Per-Trend Models, Primary Analysis

**Status: Complete, all 5 trends.**

**Per-trend models, not a combined pooled model, are the primary Method #6 analysis.** The point
of Method #6 is to see whether — and how — the effect of linguistic features on virality differs
*across* trends. A single combined model with trend included only as a dummy/control variable
can show that trends differ in their baseline (intercept) engagement level, but it cannot show
whether, say, `hashtag_count` matters more for dalgona_coffee than for sourdough — every trend is
forced to share the same coefficient for every feature except the intercept. Only 5 separate
per-trend models, one per trend, can show that directly, which is why they're the primary result
here rather than a secondary add-on to a combined model.

Two parallel OLS regressions per trend (`log_likes`, `log_comments`), same 12-predictor set in
every trend's model: 6 rule-based features (Task 3) + `is_covid_framed` + 5 zero-shot labels
(Task 4). **No trend dummy variables** — each model already contains only one trend's posts, so
there's nothing to control for.

### Reliability by trend

Checked directly against the parameter count (12 predictors + intercept = 13 total), using the
project's 10-20 observations-per-parameter rule of thumb:

| Trend | N | Obs/parameter | Reliability |
|---|---|---|---|
| sourdough | 3,000 | 230.8 | Solid |
| banana_bread | 3,000 | 230.8 | Solid |
| dalgona_coffee | 3,000 | 230.8 | Solid |
| feta_pasta | 302 | 23.2 | Solid (right at the edge, but clears the threshold) |
| **baked_oats** | **121** | **9.3** | **Below the 10:1 threshold — unstable** |

**`baked_oats`' coefficients carry an explicit reliability caveat throughout this section** — with
fewer than 10 observations per parameter, they're likely to have wide confidence intervals and
could shift meaningfully with small changes to the data. Run at the project owner's explicit
request, not as an endorsement of equal confidence with the other 4 trends.

Note: sourdough, banana_bread, and dalgona_coffee are each capped at n=3,000 in this analysis
(matching Task 4's stratified sample), which is why their N differs from Task 2's full English
counts (96,257 / 18,378 / 11,352).

### R² by trend

| Trend | Likes R² | Comments R² |
|---|---|---|
| sourdough | 0.080 | 0.161 |
| banana_bread | 0.124 | 0.212 |
| dalgona_coffee | 0.080 | 0.170 |
| feta_pasta | 0.115 | 0.170 |
| baked_oats | 0.316* | 0.346* |

*baked_oats' R² is notably higher than the other trends, but with n=121 and 13 parameters this is
likely partly overfitting rather than genuinely stronger explanatory power — consistent with the
reliability caveat above, not a contradiction of it.

### Full coefficient tables (bold = significant at p<0.05)

**LIKES model**

| Predictor | sourdough | banana_bread | dalgona_coffee | feta_pasta | baked_oats* |
|---|---|---|---|---|---|
| const | **4.519** (p=0.000) | **4.724** (p=0.000) | **4.272** (p=0.000) | **4.653** (p=0.000) | **7.061** (p=0.000) |
| log_word_count | **0.104** (p=0.007) | -0.076 (p=0.110) | **0.194** (p=0.000) | 0.071 (p=0.670) | **-0.614** (p=0.015) |
| sentiment_score | 0.047 (p=0.528) | 0.099 (p=0.267) | -0.115 (p=0.210) | 0.444 (p=0.125) | -0.177 (p=0.650) |
| hashtag_count | **0.011** (p=0.000) | **0.021** (p=0.000) | **0.025** (p=0.000) | 0.003 (p=0.715) | **0.060** (p=0.000) |
| exclamation_count | 0.026 (p=0.072) | **0.076** (p=0.000) | -0.020 (p=0.237) | 0.067 (p=0.147) | **0.172** (p=0.001) |
| question_count | **0.083** (p=0.032) | 0.033 (p=0.307) | **0.111** (p=0.004) | 0.065 (p=0.693) | 0.201 (p=0.142) |
| emoji_count | **0.018** (p=0.008) | **0.024** (p=0.000) | **0.017** (p=0.006) | -0.002 (p=0.941) | 0.028 (p=0.231) |
| is_covid_framed | -0.062 (p=0.527) | **-0.154** (p=0.031) | **-0.190** (p=0.011) | -0.146 (p=0.707) | 0.360 (p=0.146) |
| is_recipe_instructional | **0.560** (p=0.000) | **0.790** (p=0.000) | 0.148 (p=0.094) | 0.090 (p=0.733) | 0.272 (p=0.335) |
| is_personal_lifestyle | **0.144** (p=0.019) | **0.166** (p=0.014) | 0.147 (p=0.075) | -0.217 (p=0.383) | -0.125 (p=0.624) |
| is_media_repost | **0.342** (p=0.000) | **0.384** (p=0.000) | **0.187** (p=0.039) | **0.700** (p=0.003) | -0.258 (p=0.473) |
| is_meme_joke | 0.242 (p=0.065) | **0.387** (p=0.003) | **0.598** (p=0.000) | 1.468 (p=0.079) | 0.297 (p=0.816) |
| is_spam_low_content | -0.032 (p=0.690) | **-0.304** (p=0.001) | **-0.206** (p=0.035) | -0.622 (p=0.086) | -0.742 (p=0.300) |

**COMMENTS model**

| Predictor | sourdough | banana_bread | dalgona_coffee | feta_pasta | baked_oats* |
|---|---|---|---|---|---|
| const | **0.907** (p=0.000) | **1.453** (p=0.000) | **1.018** (p=0.000) | **2.210** (p=0.000) | **2.851** (p=0.004) |
| log_word_count | **0.171** (p=0.000) | -0.042 (p=0.337) | **0.176** (p=0.000) | -0.185 (p=0.220) | -0.183 (p=0.435) |
| sentiment_score | **0.148** (p=0.033) | 0.049 (p=0.551) | 0.071 (p=0.374) | **0.664** (p=0.012) | -0.093 (p=0.798) |
| hashtag_count | **0.013** (p=0.000) | **0.023** (p=0.000) | **0.032** (p=0.000) | **0.020** (p=0.017) | **0.050** (p=0.001) |
| exclamation_count | **0.056** (p=0.000) | **0.110** (p=0.000) | 0.008 (p=0.588) | **0.115** (p=0.006) | **0.113** (p=0.021) |
| question_count | **0.111** (p=0.002) | **0.096** (p=0.002) | **0.112** (p=0.001) | 0.064 (p=0.669) | 0.178 (p=0.164) |
| emoji_count | **0.023** (p=0.001) | **0.022** (p=0.001) | **0.027** (p=0.000) | -0.004 (p=0.880) | 0.025 (p=0.253) |
| is_covid_framed | -0.026 (p=0.775) | 0.013 (p=0.848) | **-0.134** (p=0.040) | -0.191 (p=0.589) | 0.279 (p=0.229) |
| is_recipe_instructional | **0.492** (p=0.000) | **0.998** (p=0.000) | **0.216** (p=0.005) | 0.428 (p=0.076) | 0.022 (p=0.933) |
| is_personal_lifestyle | **0.466** (p=0.000) | **0.556** (p=0.000) | **0.371** (p=0.000) | 0.102 (p=0.652) | -0.050 (p=0.832) |
| is_media_repost | **-0.231** (p=0.001) | -0.014 (p=0.861) | **-0.229** (p=0.004) | 0.287 (p=0.180) | **-1.021** (p=0.003) |
| is_meme_joke | **0.305** (p=0.013) | **0.465** (p=0.000) | **0.551** (p=0.000) | 0.950 (p=0.210) | 0.195 (p=0.871) |
| is_spam_low_content | -0.052 (p=0.492) | -0.089 (p=0.293) | **-0.191** (p=0.025) | **-0.946** (p=0.004) | **-1.452** (p=0.032) |

*baked_oats column carries the reliability caveat above — interpret with less confidence than the
other 4 columns.

Forest plots visualizing all coefficients and 95% CIs side-by-side across the 5 trends:
`Regression Visualizations/forest_plot_likes.png`, `forest_plot_comments.png`. The plots make the
reliability difference visible directly — baked_oats' error bars are noticeably wider than the
other 4 trends' across nearly every predictor, the visual counterpart to its 9.3 obs/parameter
ratio.

### Multicollinearity (VIF), per trend

Same predictor set, checked separately within each trend's subset (rather than once for a pooled
combined model), since correlation between features could plausibly differ by trend. `const`'s
own VIF is omitted from each table below (its VIF is always large in this kind of design and not
a meaningful multicollinearity signal); all reported feature VIFs are comfortably low.

| Predictor | sourdough | banana_bread | dalgona_coffee | feta_pasta | baked_oats |
|---|---|---|---|---|---|
| log_word_count | 2.22 | 2.48 | 2.68 | 2.73 | 2.15 |
| sentiment_score | 1.49 | 1.46 | 1.43 | 1.51 | 1.37 |
| hashtag_count | 1.12 | 1.15 | 1.32 | 1.25 | 1.16 |
| exclamation_count | 1.23 | 1.23 | 1.21 | 1.48 | 1.37 |
| question_count | 1.09 | 1.09 | 1.09 | 1.09 | 1.22 |
| emoji_count | 1.19 | 1.18 | 1.11 | 1.50 | 1.42 |
| is_covid_framed | 1.04 | 1.05 | 1.07 | 1.04 | 1.13 |
| is_recipe_instructional | 1.21 | 1.48 | 1.50 | 1.89 | 1.45 |
| is_personal_lifestyle | 1.34 | 1.30 | 1.47 | 1.48 | 1.24 |
| is_media_repost | 1.16 | 1.13 | 1.33 | 1.24 | 1.23 |
| is_meme_joke | 1.05 | 1.06 | 1.10 | 1.06 | 1.06 |
| is_spam_low_content | 1.59 | 1.66 | 1.85 | 1.36 | 1.29 |

No multicollinearity concerns anywhere — every feature VIF is well under the conventional 5-10
concern range, in every trend. (This is a meaningfully cleaner picture than the pooled combined
model produced, where `log_word_count` reached VIF=12.35 once trend dummies were added to the
same design matrix — another point in favor of the per-trend approach for this project.)

### Key findings

- **`recipe_instructional`**: significant positive for the three large trends; banana_bread's
  effect is by far the largest (0.79 likes / 1.00 comments) vs. sourdough (0.56 / 0.49) vs. dalgona
  (weaker, only significant in comments, 0.22). Not significant for feta_pasta or baked_oats.
- **`meme_joke`**: significant positive for the three large trends; **dalgona has the largest
  effect** (0.60 likes / 0.55 comments), larger than banana_bread (0.39/0.47) and sourdough (not
  significant on likes; 0.30 on comments). Not significant for feta_pasta or baked_oats (small
  samples), though feta_pasta's point estimate is large (1.47 likes) with a wide interval.
- **`media_repost`**: positive/significant for likes in sourdough, banana_bread, dalgona_coffee,
  and feta_pasta; flips negative/significant for sourdough, dalgona_coffee, and baked_oats in
  comments. Reposted content gets liked but doesn't spark discussion — a consistent pattern
  across most trends, not just an artifact of pooling them together.
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
- **`sentiment_score`** is only significant for sourdough (comments) and feta_pasta (comments) —
  it's not a reliable predictor for banana_bread or dalgona_coffee on either outcome, or for
  either outcome in baked_oats.
- Model fit (R²) modest throughout except baked_oats (0.32 likes / 0.35 comments), which is likely
  inflated by overfitting given its small sample (see reliability caveat), not genuinely stronger
  explanatory power.

This is a descriptive, exploratory comparison across 5 trends, not a formally tested relationship
— n=5 trends is nowhere near enough for a statistical test of "linguistic features predict trend
longevity" or similar claims. Frame any cross-trend pattern in the write-up with hedged language
("consistent with," "suggestive of"), not causal or confirmatory language.

**Note:** `baked_oats`' regression sample also isn't fully independent of `banana_bread`'s (63.6%
of its 121 rows also carry a `banana_bread` label, largely via a real "banana bread baked oats"
recipe genre, not a data error) — this compounds its existing small-sample caveat above; treat
`baked_oats`' regression coefficients with the same reduced confidence for this reason as well.

Output: `cleaned_data/per_trend_regression_comparison.csv`
Notebook: `code_english_only/regression/03_per_trend_regressions.ipynb`
Visualizations: `Regression Visualizations/forest_plot_likes.png`, `forest_plot_comments.png`

---

## Task 6: LDA Topic Discovery (Method #7)

**Status: Complete** (model fitting and K-selection complete; topic labeling/classification below
performed directly from the fitted models' top words and top-scoring example posts, following the
same real-text-sanity-check method used for the zero-shot label set in Task 4 — not run through a
separate `02_interpret_topics.ipynb` review pass, so treat labels/classifications as a first-pass
read to sanity-check against the report's needs, not an independently re-verified ground truth).

### Why this doesn't run on the 5 known trends' data

LDA does **not** run on `trends_combined_english.csv` — every row there already matched one of
the 5 trend keyword lists, so it could only re-derive the same 5 categories. Instead it runs on a
dedicated dataset of everything **outside** the 5 known trends, so it can actually discover
something new.

**`non_trend_english.csv` construction** (confirmed against real data):
1. Combine the 4 raw Instagram files: 298,635 rows
2. Filter to English only: 207,038 rows
3. Exclude any post matching any of the 5 trend keyword sets, by `id`: 81,409 rows remain
4. Drop rows with neither `text` nor `hashtags` present: **73,933 rows** final (7,476 dropped)

This is substantial headroom for genuine discovery — more English posts than 3 of the 5 named
trends individually have.

### Methodology

- **Two fully separate models**, run on `text` and on `hashtags` independently (replicating the
  project's boba tea reference paper's approach, which found text skews personal/narrative while
  hashtags skew promotional/discovery-oriented).
- **Full dataset (73,933 rows), no sampling** — LDA runs locally with no API cost pressure, and
  sampling would directly undermine the discovery goal (rare emerging patterns are exactly what a
  sample would lose).
- **Preprocessing:** lowercase, strip URLs/non-letter characters, POS-aware lemmatization (NLTK
  `WordNetLemmatizer`), `ngram_range=(1,1)` (unigrams only — an earlier bigram-based plan to
  exclude 8 trend phrases like "banana bread" was dropped once real-data checks confirmed those
  phrases have zero occurrences in `non_trend_english.csv` by construction, so the exclusion
  mechanism did no real work while adding complexity), `min_df=5`, `max_df=0.5`.
- **Custom stopwords** (on top of sklearn's standard English list): tokenization fragments (`s`,
  `t`, `m`, `ve`, `g`), platform artifacts (`mention`, `redacted`, `link`, `bio`), generic recipe
  boilerplate (`recipe`, `make`, `made`, `food`, `like`, `just`, `delicious`, `love`, `good`,
  `day`, `today`, `time`), generic units/measurements (`cup`, `tsp`, `tbsp`, `add`, `salt`, `oil`,
  `minutes`, `ingredients`), and the two trend-specific single words with no meaning outside their
  trend (`sourdough`, `dalgona`). **Explicitly kept in vocabulary:** `cheese`, `bread`, `coffee`,
  `banana`, `oats`, `oatmeal`, `chocolate`, `baked`, `baking`, `feta` — all generic enough that
  excluding them would block discovery of a genuinely different trend built on the same
  ingredient. A real bug was caught and fixed here: `CountVectorizer` filters stopwords *after*
  lemmatization runs, so an un-lemmatized stopword (e.g. `redacted`, `minutes`, `further`) would
  silently leak through into topic top-words unless the stopword list is lemmatized the same way
  (all 4 POS forms kept) before being passed in.
- **Vocabulary sizes:** 31,247 terms (text model), 23,412 terms (hashtags model).

### K selection

Tested K = 5, 10, 15, 20, 25 for each model, scored with real semantic topic coherence
(`gensim`'s `c_v` `CoherenceModel`, not sklearn's log-likelihood `.score()`, which measures
statistical fit, not whether a topic's top words are semantically related). **The final K for each
model is the highest-coherence K in its own tested range** — applied consistently across both
models.

**Text model coherence by K:**

| K | Coherence |
|---|---|
| 5 | 0.5839 |
| 10 | 0.5834 |
| 15 | 0.5925 |
| 20 | **0.5980** |
| 25 | 0.5872 |

**K=20 selected for the text model** — the highest coherence of the tested range.

**Hashtags model coherence by K:**

| K | Coherence |
|---|---|
| 5 | 0.5850 |
| 10 | **0.7005** |
| 15 | 0.6002 |
| 20 | 0.6803 |
| 25 | 0.6518 |

**K=10 selected for the hashtags model** — the highest coherence of the tested range, and clearly
so (0.7005 vs. the next-best 0.6803 at K=20). An earlier pass had used K=20 for the hashtags model
despite K=10 scoring higher, with no recorded rationale for the override — that was a mistake,
now corrected. Both the saved model artifact (`final_lda_hashtags.pkl`) and the notebook's
`CONFIRMED_K` value have been updated to K=10, and every hashtags visualization
(`lda_top_words_hashtags.png`, `lda_topic_prevalence_hashtags.png`,
`lda_visualization_hashtags.html`) has been regenerated to match.

Both final models (`final_lda_text.pkl` at K=20, `final_lda_hashtags.pkl` at K=10) are fit and
saved in `code_english_only/lda/lda_model_checkpoints/`. Coherence-vs-K charts, top-words bar
charts, topic prevalence charts, and interactive pyLDAvis panels for both models are saved in
`Final Project Progress/Lda visualizations/`.

**One real tradeoff of correcting to the highest-coherence K for hashtags:** at K=10, several
distinct patterns that were cleanly separated at K=20 now blend together into single, coarser
topics — this is a direct, observable instance of the design doc's own warning that "too few
topics risks blending distinct patterns together into one blurry cluster." Two concrete examples
below: what were 3 separate food-media-aggregator topics at K=20 collapse into essentially 1-2
broader ones at K=10, and a cluster of stylized-Unicode-font hashtag spam that stood alone as its
own topic at K=20 gets folded into a generic "foodie hashtags" topic at K=10. Coherence is a
measure of within-topic word relatedness, not of how finely real underlying patterns get resolved
— worth keeping in mind when reading the coarser K=10 hashtags results below.

### Full topic results: Text model (K=20)

**Labels for Topics 0, 3, 11, 13, 15 below are project-owner-verified** (checked directly against
prevalence and top words); Topic 4's label has been corrected (see note after the table). All
other labels/classifications are the original first-pass read and have not been independently
re-verified. Prevalence = number/% of the 73,933-post dataset for which this is the single most
likely topic.

| Topic | Prevalence | Top 15 words | Label | Classification |
|---|---|---|---|---|
| 0 | 7,479 (10.12%) | salad, feta, olive, tomato, pepper, cucumber, lemon, fresh, red, cheese, dress, greek, onion, bowl, chop | Salads, savory/Mediterranean (tomato, lemon, onion, olive, greek) | (c) Noise — generic Mediterranean/Greek salad content, ingredient-adjacent to feta_pasta but not the dish itself |
| 1 | 1,346 (1.82%) | milk, pancake, butter, seed, banana, fruit, breakfast, snack, use, egg, slice, peanut, oat, water, coconut | Breakfast/snack prep (pancakes, energy balls, overnight oats) | (c) Noise/generic — broad breakfast-prep content, no single coherent dish |
| 2 | 2,381 (3.22%) | quarantinecooking, quarantine, noodle, bread, quarantinelife, quarantinebaking, rice, homemade, dumpling, stayhome, pork, quarantinekitchen, quarantinefood, ramen, bake | Quarantine-era Asian home cooking (noodles, dumplings, ramen) | (c) Noise — generic pandemic-cooking umbrella, not a specific dish trend |
| 3 | 3,742 (5.06%) | cheese, chicken, pizza, feta, sauce, burger, grill, onion, fry, tomato, mozzarella, steak, garlic, bbq, beef | Meat-based recipes / American grilling (chicken, burgers, steak, bbq) — note pizza/cheese/mozzarella also present, so "comfort food" more broadly, meat as the anchor | (c) Noise — standard comfort food, no COVID-era emergence signal |
| 4 | 4,876 (6.60%) | feta, roast, tomato, cheese, spinach, lamb, pastry, hummus, greek, dip, dish, serve, pumpkin, pie, bread | Mediterranean/Middle Eastern feta-based mezze (roasted vegetables, dips, pastry, Greek-style dishes) — **corrected**, see note below | (b)/(c) — needs your call; see note below on why this was downgraded from the earlier "whipped feta dip" framing |
| 5 | 1,270 (1.72%) | healthyfood, healthy, lunch, meal, healthyeating, nutrition, healthylifestyle, chicken, healthyrecipes, mealprep, quarantinecooking, healthyliving, fitness, cleaneating, eat | Health/nutrition coaching content | (c) Noise — lifestyle/promotional content, not a food trend |
| 6 | 2,823 (3.82%) | order, pm, delivery, menu, restaurant, available, cook, feta, wine, chef, quarantinecooking, open, fish, seafood, quarantine | Restaurant delivery/takeout promo posts | (c) Noise — business promotional content |
| 7 | 7,251 (9.81%) | cook, pepper, garlic, onion, chop, heat, pan, sauce, olive, water, use, mix, tomato, serve, oven | Generic recipe-instruction vocabulary | (c) Noise — this is the residual "how-to-cook" verb/step vocabulary, a modeling artifact rather than content |
| 8 | 3,776 (5.11%) | foodie, quarantinecooking, foodporn, foodphotography, instafood, foodstagram, follow, foodblogger, yummy, foodgasm, homemade, indianfood, foodlover, cook, foodiesofinstagram | Indian street food & foodie-hashtag content | (c) Noise — regional foodie-community content, not a dish-level trend |
| 9 | 652 (0.88%) | sp, waffle, ww, point, myww, feta, video, wwcommunity, weightwatchers, la, try, wwfamily, wwsisterhood, wwambassador, en | Weight Watchers (WW) branded content | (c) Noise — branded diet-program community jargon |
| 10 | 2,861 (3.87%) | quarantinecooking, pasta, cook, foodie, homecooking, dinner, greek, cookingathome, yum, foodporn, easyrecipes, feedfeed, greece, home, fresh | Generic home-cooking/easy dinner content | (c) Noise — broad "cooking at home" content, no distinct dish |
| 11 | 1,515 (2.05%) | quarantinecooking, ahmedabad, follow, mumbaifoodie, foodtalkindia, instagram, quarantine, indianfoodbloggers, cheese, delhifoodie, thehungryclones, indianstreetfood, nomnom | Indian food-blogger community (Ahmedabad/Mumbai/Delhi) — **the only ethnic/regional food community distinct enough to form its own topic in this model; it also surfaces independently, and far more prominently, in the hashtags model as Topic 5 (10.52%)** — see that row below | (c) Noise — geographically narrow foodie-community cluster, but a real cross-model-convergent one; classification worth reconsidering given the cross-model match |
| 12 | 10,956 (14.82%) | know, cook, share, use, thing, home, want, week, don, try, new, eat, ll, need, work | Generic personal/narrative filler language | (c) Noise — functional/stopword-like language, not food-related content at all |
| 13 | 4,084 (5.52%) | egg, feta, vegan, avocado, breakfast, toast, brunch, potato, taco, cheese, spinach, tomato, pepper, sweet, tortilla | Savory breakfast/brunch (toast, eggs, avocado, tomato) — also carries taco/tortilla, a Tex-Mex-brunch edge | (a) Genuine trend candidate — flagged for your review: this is a broader, more generic label than the original "avocado toast" framing, worth reconsidering whether (a) still fits or whether this reads more like (c) |
| 14 | 2,318 (3.14%) | quarantinecooking, lockdown, cook, home, quarantine, gluttonyguilts, step, cooking, chilli, curry, try, stayhome, quarantinelife, covid, coronavirus | Quarantine-era Indian home cooking (dal/curry) | (c) Noise — same pandemic-cooking umbrella as Topic 2, regional-staples flavor |
| 15 | 4,419 (5.98%) | salad, feta, watermelon, summer, fresh, sweet, flavor, fruit, perfect, veggie, favorite, cheese, strawberry, meal, mint | Salads, fruit-based (watermelon, strawberry, mint, fig+feta) — a distinct, fruitier profile from Topic 0's savory salads | (c) Noise — seasonal produce content |
| 16 | 892 (1.21%) | youtube, quarantinecooking, channel, sandwich, homemade, subscribe, ramadan, chef, follow, stayhome, video, share, staysafe, foodphotography, facebook | Cross-platform food video promo + Indian street snacks | (c) Noise — promotional cross-platform content mixed with regional snacks |
| 17 | 5,409 (7.32%) | bake, chocolate, flour, sugar, butter, dough, cream, cake, cooky, mix, use, egg, vanilla, powder, quarantinebaking | Quarantine baking — cakes & cookies | (b) Sub-cluster of the broader "COVID baking boom" that sourdough/banana_bread are also part of, but generic cake/cookie content, not either named trend specifically |
| 18 | 1,691 (2.29%) | keto, cheese, protein, fat, calorie, meal, low, feta, carbs, diet, eat, egg, high, health, healthy | Keto/low-carb diet content | (a) Genuine trend candidate — confirmed independently by the hashtags model's Topic 3 (K=10); a real, coherent, non-overlapping diet trend |
| 19 | 4,192 (5.67%) | cake, feedfeed, quarantinebaking, gram, tomato, huffposttaste, foodblogfeed, eeeeeats, foodie, buzzfeast, profile, quarantinecooking, instafood, thefeedfeed, bake | Food-media aggregator reposts (Feedfeed/Buzzfeed/HuffPost Taste) | (c) Noise — this is literally the `media_repost` phenomenon (aggregator accounts resharing content), not organic trend content |

**Correction to Topic 4's label (was "Mediterranean mezze / whipped feta dips"):** the phrase
"whipped feta" does not appear anywhere in the topic's actual top-15 words — it was pulled from 2
of only 4 example posts sampled in the original pass (an aubergine-shawarma post and a
dipping-plates post that both happened to use that exact phrase), not from the topic's real
defining vocabulary. That over-specified the topic from a thin, non-representative sample. The
topic's actual signature (`feta, roast, tomato, cheese, spinach, lamb, pastry, hummus, greek, dip,
dish, serve, pumpkin, pie, bread`) supports a broader label — Mediterranean/Middle Eastern
feta-based mezze — not a specific "whipped feta" sub-trend. The classification call (whether this
still counts as a genuine adjacent discovery or is better read as noise) is left for you to make.

### Full topic results: Hashtags model (K=10)

**Labels for Topics 1, 3, 7 below are project-owner-verified.** Topic 5 has a cross-model note
added (see Topic 11 above). All other labels/classifications are the original first-pass read.

| Topic | Prevalence | Top 15 words | Label | Classification |
|---|---|---|---|---|
| 0 | 11,616 (15.71%) | feedfeed, gram, thekitchn, huffposttaste, community, quarantinebaking, imsomartha, thebakefeed, tastingtable, foodandwine, thenewhealthy, bhgfood, beautifulcuisines, makesmewhole, foodblogfeed | Food-media aggregator/publication hashtags (Feedfeed/Kitchn/HuffPost/Food&Wine) | (c) Noise — media-repost/aggregator hashtag cluster (what were 3 separate aggregator topics at K=20 collapse into this single topic at K=10) |
| 1 | 9,609 (13.00%) | healthyfood, salad, healthyeating, healthylifestyle, healthy, healthyrecipes, nutrition, mealprep, cleaneating, healthyliving, weightloss, eatclean, glutenfree, fitness, vegetarian | Healthy/fitness recipes | (c) Noise/generic |
| 2 | 7,430 (10.05%) | quarantinebaking, bake, dessert, cake, quarantinecooking, chocolate, quarantine, foodie, homemade, cooky, homebaker, cakesofinstagram, sweet, yummy, bakersofinstagram | Dessert/cake baking hashtags (quarantine baking wave) | (b) Sub-cluster of the broader COVID baking wave, parallel to sourdough/banana_bread's own hashtag communities |
| 3 | 3,245 (4.39%) | ud, keto, udc, lowcarb, quarantinecooking, ketorecipes, ketodiet, ketogenic, lchf, ketogenicdiet, ketomeals, ketoweightloss, greece, lowcarbrecipes, salad | Keto/low-carb | (a) Genuine trend candidate — confirms text model's Topic 18; one of the clearest coherent non-5-trend patterns found by this task. (Two truncated tokens, `ud`/`udc`, are the same non-Latin-script preprocessing artifact noted elsewhere — see below — riding along with the real keto signal, not evidence against it.) |
| 4 | 4,445 (6.01%) | quarantinecooking, quarantine, quarantinelife, ahmedabad, cookingathome, quarantinecuisine, quarantinekitchen, cook, quarantineandchill, stayhome, instagram, chef, cheflife, chefsofinstagram, quarantinemeals | Core pandemic-framing + chef/professional cooking hashtags | (c) Noise/context — blends the pure COVID-framing signal with chef-branding content |
| 5 | 7,775 (10.52%) | quarantinecooking, indianfood, lockdown, quarantine, stayhome, quarantinelife, indianfoodbloggers, mumbaifoodie, foodtalkindia, delhifoodie, easyrecipes, quarantineandchill, staysafe, homemade, foodphotography | Indian food-blogger community hashtags — **same community as text Topic 11, found independently here at more than 5x the prevalence (10.52% vs. 2.05%)** | (c) Noise — regional foodie-community content, but cross-model-convergent; classification worth reconsidering alongside text Topic 11 |
| 6 | 11,584 (15.67%) | ud, foodie, foodporn, quarantinecooking, feta, foodphotography, instafood, foodstagram, yummy, ude, foodblogger, breakfast, homecooking, cheese, udf | Generic Instagram foodie hashtags (with stylized-font spam contamination) | (c) Noise — top-scoring example posts for this topic are dominated by stylized Unicode-font hashtag spam, which stood alone as its own topic at K=20; at K=10 it gets folded into this broader generic-foodie topic instead, a direct example of the coarser-K blending tradeoff noted above |
| 7 | 5,101 (6.90%) | vegan, quarantinecooking, plantbased, glutenfree, veganrecipes, veganfood, vegetarian, whatveganseat, vegetarianrecipes, plantbaseddiet, dairyfree, veganfoodshare, veganfoodie, foodblog, plantbasedrecipes | Vegan/plant-based | (a) Genuine trend candidate — a distinct, coherent lifestyle-diet trend independent of the 5 known food-specific trends |
| 8 | 3,524 (4.77%) | quarantinecooking, uc, ub, ud, homecooking, foodie, foodstagram, foodporn, instafood, foodphotography, quarantinefood, asianfood, quarantinelife, homemade, follow | Asian food hashtags (Korean/Filipino) mixed with garbled non-ASCII tokens | (c) Noise — same preprocessing artifact as above; top-scoring example posts here directly confirm the cause — real Korean-script hashtags getting truncated to meaningless fragments (`uc`/`ub`/`ud`) by the ASCII-only cleaning regex |
| 9 | 9,604 (12.99%) | quarantinecooking, foodie, foodstagram, feedfeed, instafood, gram, eeeeeats, huffposttaste, buzzfeast, easyrecipes, yum, foodporn, cook, comfortfood, foodlover | Generic foodie + media-aggregator hashtags (catch-all) | (c) Noise — catch-all blending generic foodie tags with aggregator accounts |

### Summary data (synthesis/interpretation left to the project owner)

**Classification counts, as currently marked (30 topics total: 20 text + 10 hashtags) —
unchanged from the first pass except Topic 4, now marked as needing review rather than asserted:**
4 topics marked (a) genuine trend candidate (text Topic 13, text Topic 18, hashtags Topic 3,
hashtags Topic 7); 2 marked (b) sub-cluster of the COVID-baking wave (text Topic 17, hashtags
Topic 2); 1 (text Topic 4) now marked (b)/(c) pending review, down from its earlier (a)/(b)
framing; the remaining 23 marked (c) noise/generic.

**Verified cross-model matches (topics that appear independently in both the text and hashtags
models, i.e. the same content pattern was found twice by two separately-fit models):**

| Pattern | Text model | Hashtags model |
|---|---|---|
| Keto/low-carb | Topic 18 (2.29%) | Topic 3 (4.39%) |
| Indian food-blogger community | Topic 11 (2.05%) | Topic 5 (10.52%) |

No cross-model match exists for vegan/plant-based (hashtags Topic 7 only — no equivalently
distinct vegan topic in the text model's 20 topics) or for the food-media aggregator pattern,
which is split across multiple topics in both models rather than mapping cleanly one-to-one (text
Topic 19 alone; hashtags Topics 0, 6, and 9 partially overlapping — see the K=10 blending note
above).

**Known preprocessing artifact, factual, not interpretive:** non-Latin-script hashtags are
truncated into meaningless short tokens (`uc`, `ub`, `ud`, etc.) by the ASCII-only cleaning regex
in the LDA preprocessing pipeline. These tokens appear in hashtags Topics 3, 6, and 8's top-15
words. Hashtags Topic 8's example posts contain literal Korean-script hashtags, confirming the
truncation is affecting real non-English content, not filtering noise as intended.

Output: no new combined data file; per-topic top words, coherence logs, and final fitted models
saved in `code_english_only/lda/lda_model_checkpoints/`
Notebooks: `code_english_only/lda/00_build_dataset_and_config.ipynb`,
`code_english_only/lda/01a_lda_text.ipynb`, `code_english_only/lda/01b_lda_hashtags.ipynb`
Visualizations: `Lda visualizations/lda_coherence_vs_k_{text,hashtags}.png`,
`lda_top_words_{text,hashtags}.png`, `lda_topic_prevalence_{text,hashtags}.png`,
`lda_visualization_{text,hashtags}.html` (interactive pyLDAvis)

---

## Supplementary: Creator-Trend Network Analysis

**Status: Complete.** Examines whether the same creators post across multiple trends, or whether
each trend has its own separate creator community, via a trend-to-trend network built from creator
overlap.

### Design decisions

- **Creator identifier: `post_owner.username`**, not `post_owner.id`. Confirmed a perfect 1-to-1
  relationship between the two across all 45,163 unique creators (zero collisions either
  direction), so username was used for label readability with no accuracy tradeoff.
- **Trend-to-trend projection, not a full bipartite graph.** 31,540 unique creators post about at
  least one of the 5 trends; only 13.42% (4,232) post in more than one. A raw bipartite graph of
  tens of thousands of creator nodes would be unreadable, so the analysis projects down to 5 trend
  nodes with edges weighted by creator overlap.
- **Edge weights: Jaccard similarity** (shared creators ÷ total unique creators across both
  trends), not raw shared-creator counts — raw counts would be dominated by sourdough's much
  larger creator base (19,368 creators vs. baked_oats' 77).
- **Scope: English-only**, consistent with the rest of the pipeline.

### Creator counts per trend

| Trend | Unique creators |
|---|---|
| sourdough | 19,368 |
| banana_bread | 8,692 |
| dalgona_coffee | 7,898 |
| feta_pasta | 241 |
| baked_oats | 77 |

Total unique creators across all 5 trends: 31,540 (out of 45,163 total unique creators in the
raw English-language pull, before restricting to creators who matched a trend keyword).

### Jaccard similarity table (all 10 trend pairs)

| Trend 1 | Trend 2 | Shared creators | Union creators | Jaccard similarity |
|---|---|---|---|---|
| sourdough | banana_bread | 2,718 | 25,342 | **0.1073** |
| banana_bread | dalgona_coffee | 1,033 | 15,557 | 0.0664 |
| sourdough | dalgona_coffee | 1,162 | 26,104 | 0.0445 |
| banana_bread | baked_oats | 66 | 8,703 | 0.0076 |
| feta_pasta | sourdough | 131 | 19,478 | 0.0067 |
| feta_pasta | banana_bread | 55 | 8,878 | 0.0062 |
| feta_pasta | dalgona_coffee | 25 | 8,114 | 0.0031 |
| sourdough | baked_oats | 44 | 19,401 | 0.0023 |
| baked_oats | dalgona_coffee | 16 | 7,959 | 0.0020 |
| feta_pasta | baked_oats | 0 | 318 | 0.0000 |

### Network graph and heatmap findings

Both a spring-layout network graph and an exact-value heatmap were built from the table above
(`creator_trend_network.png`, `creator_trend_heatmap.png`).

- **Sourdough and banana_bread have by far the strongest creator overlap** — more than double the
  next-highest pair (0.1073 vs. 0.0664) — consistent with both being "baking" trends likely to
  draw the same home-baker audience.
- **Feta_pasta and baked_oats share zero creators**, the only pair with no overlap at all, despite
  both being comparatively small trends (241 and 77 creators respectively) where some overlap by
  chance might be expected.
- **Baked_oats and feta_pasta are the two most creator-isolated trends overall** — every one of
  their pairwise edges is among the weakest in the table (all their Jaccard scores are ≤0.0076
  except feta_pasta-sourdough at 0.0067, itself still small), meaning both trends draw largely
  self-contained creator communities rather than overlapping meaningfully with any other trend's
  audience.
- **Sourdough, as the largest trend, is not uniformly the best-connected node** — its overlap with
  banana_bread is strong, but its overlap with baked_oats (0.0023) and feta_pasta (0.0067) is
  weak, showing raw size doesn't guarantee cross-trend creator reach.

**Creators active in 3+ trends:** 494 total (484 in exactly 3 trends, 10 in exactly 4 trends) —
a much smaller, more tightly-connected subset than the 4,232 creators active in 2+ trends overall,
worth a closer look if a creator-level (rather than trend-level) view is wanted for the write-up.

Output: `cleaned_data/trend_creator_overlap.csv`
Notebook: `code_english_only/04_creator_trend_network.ipynb`
Visualizations: `code_english_only/creator_trend_network.png`,
`code_english_only/creator_trend_heatmap.png`
