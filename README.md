# COVID-Era Viral Food Trends: English-Only Analysis Pipeline

Analyzes Instagram/Facebook discourse about five COVID-era viral food trends (sourdough, banana
bread, dalgona coffee, baked oats, feta pasta): whether engagement differs significantly across
trends (Method #5), what linguistic/content features predict virality within each trend
(Method #6), and what undiscovered food trends exist in the broader non-trend post population
(Method #7, LDA topic modeling).

## Repo structure

```
code/               all analysis notebooks, numbered in run order (see below)
output/              generated CSVs, figures, and model checkpoints
data/                NOT in this repo -- see Data section below
```

## Data

Raw data is not stored in this repo. It consists of four keyword-filtered Instagram/Facebook
post CSVs (`manya_first_12012024.csv`, `manya_0711-1104.csv`, `manya_1105_1201.csv`,
`manya_0410-0710.csv`) plus five trend-specific derived CSVs (`sourdough.csv`,
`banana_bread.csv`, `dalgona_coffee.csv`, `baked_oats.csv`, `feta_pasta.csv`).

Download the data from Google Drive and place it in a `data/` folder at the repo root (sibling
to `code/` and `output/`) before running any notebook below:

**[Raw and cleaned data (Google Drive)](https://drive.google.com/drive/folders/15I9zewgUfRI35mlo0zqg3tonn7rB4WCo?usp=drive_link)**

## Notebooks, in run order

### `code/` (top level)

1. **[`00_data_cleaning.ipynb`](code/00_data_cleaning.ipynb)**. Input: the 5 trend-specific
   CSVs in `data/`. Loads all 5, tags each row with its `trend`, recomputes the
   `is_covid_framed` flag from a corrected keyword list, combines them, filters to English-only
   (`lang == "en"`), and drops rows with null like/comment counts. Output:
   `output/cleaned_data/trends_combined_english.csv`.

2. **[`01_engagement_analysis.ipynb`](code/01_engagement_analysis.ipynb)**. Input:
   `output/cleaned_data/trends_combined_english.csv`. Runs Method #5: engagement distribution
   summaries, Kruskal-Wallis and one-way ANOVA (log-transformed) tests for whether engagement
   differs across the 5 trends, and pairwise Mann-Whitney U follow-up tests. Output:
   `output/engagement/viz_english_engagement_distributions.png` (the notebook's own console
   output holds the full statistical results).

### `code/regression/` (Method #6: what predicts virality)

3. **[`00_rule_based_features.ipynb`](code/regression/00_rule_based_features.ipynb)**. Input:
   `output/cleaned_data/trends_combined_english.csv`. Adds free, rule-based linguistic features:
   log word count, VADER sentiment score, hashtag count, exclamation/question/emoji counts, and
   log-transformed outcome variables (`log_likes`, `log_comments`). Output:
   `output/cleaned_data/trends_combined_english_features.csv`.

4. **[`01_zero_shot_classification.ipynb`](code/regression/01_zero_shot_classification.ipynb)**. Input: `output/cleaned_data/trends_combined_english_features.csv`; requires an Anthropic API
   key (`anthropic.Anthropic()` picks up `ANTHROPIC_API_KEY` from the environment). Draws a
   stratified sample (sourdough/banana_bread/dalgona_coffee capped at 3,000 posts each;
   baked_oats/feta_pasta kept at full size, ~9,423 posts total) and classifies each post via the
   Claude Message Batches API into 5 non-exclusive content-type labels (recipe/instructional,
   personal/lifestyle, media repost, meme/joke, spam/low-content). Makes real, paid API calls.
   Output: overwrites `output/cleaned_data/trends_combined_english_features.csv`, now scoped to
   the ~9,423-row stratified sample with 5 new `is_*` binary columns.

5. **[`02_regression_models.ipynb`](code/regression/02_regression_models.ipynb)**. Input:
   `output/cleaned_data/trends_combined_english_features.csv` (the stratified sample). Fits one
   pooled OLS model each for `log_likes` and `log_comments` across all 5 trends together, with
   trend dummy variables (sourdough as baseline) and a VIF multicollinearity check. Deprioritized
   in favor of the per-trend models below; kept for reference. Output: notebook console output
   only (model summaries), no file saved.

6. **[`03_per_trend_regressions.ipynb`](code/regression/03_per_trend_regressions.ipynb)**. Input: `output/cleaned_data/trends_combined_english_features.csv` (the stratified sample).
   **Primary Method #6 analysis.** Fits separate `log_likes`/`log_comments` OLS models per trend
   (no trend dummies needed, each model restricted to one trend already), compares coefficients
   across trends via forest plots, and checks R² and VIF per trend. `baked_oats` (n=121) is
   flagged as below the 10-observations-per-parameter reliability threshold. Output:
   `output/cleaned_data/per_trend_regression_comparison.csv`,
   `output/regression/forest_plot_likes.png`, `output/regression/forest_plot_comments.png`.

### `code/lda/` (Method #7: undiscovered trends via topic modeling)

7. **[`00_build_dataset_and_config.ipynb`](code/lda/00_build_dataset_and_config.ipynb)**. Input: the 4 raw Instagram/Facebook CSVs in `data/`. Builds the non-trend corpus: all
   English-language posts that do NOT match any of the 5 known trend keyword sets, after
   dropping rows with neither text nor hashtags. Also saves the shared preprocessing config
   (stopword lists, vectorizer settings, K values to test) used by both LDA notebooks below.
   Output: `output/cleaned_data/non_trend_english.csv`, `code/lda/lda_config.json`.

8. **[`01a_lda_text.ipynb`](code/lda/01a_lda_text.ipynb)**. Input:
   `output/cleaned_data/non_trend_english.csv`, `code/lda/lda_config.json`. Fits LDA topic models
   on post `text` at K=5,10,15,20,25 (lemmatized unigrams), selects K=20 by highest `c_v`
   coherence score, and saves top-words/topic-prevalence visualizations plus an interactive
   pyLDAvis panel. Output: `output/lda/lda_coherence_vs_k_text.png`,
   `output/lda/lda_top_words_text.png`, `output/lda/lda_topic_prevalence_text.png`,
   `output/lda/lda_visualization_text.html`, plus fitted models/vectorizer/DTM checkpoints under
   `output/lda/model_checkpoints/`.

9. **[`01b_lda_hashtags.ipynb`](code/lda/01b_lda_hashtags.ipynb)**. Same pipeline as `01a`, run
    on the `hashtags` field instead of `text`. Selects K=10 by highest `c_v` coherence score
    (highest-scoring K, not the largest K value tested). Output:
    `output/lda/lda_coherence_vs_k_hashtags.png`, `output/lda/lda_top_words_hashtags.png`,
    `output/lda/lda_topic_prevalence_hashtags.png`, `output/lda/lda_visualization_hashtags.html`,
    plus checkpoints under `output/lda/model_checkpoints/`.

## Requirements

pandas, numpy, scipy, statsmodels, matplotlib, scikit-learn, gensim, nltk, vaderSentiment, emoji,
networkx, pyLDAvis, anthropic (for `code/regression/01_zero_shot_classification.ipynb` only, plus
an `ANTHROPIC_API_KEY` environment variable).
