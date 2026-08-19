# COVID-Era Viral Food Trends

## Project summary

This project asks whether COVID-era viral food trends (sourdough, banana bread, dalgona coffee, baked oats, and feta pasta) produced lasting shifts in real-world consumer behavior, or whether they faded once online discourse moved on. It compares the volume and timing of Instagram/Facebook discourse about each trend against real-world U.S. retail sales data (USDA Weekly Retail Food Sales) to see whether spikes in online conversation coincided with, preceded, or lagged actual changes in consumer purchasing, particularly around the March 2020 U.S. national emergency declaration.

## Data

- **Instagram/Facebook posts** (`data/manya_*.csv`) — keyword-filtered posts covering food-related content from December 2019 through December 2020, obtained via professor-provided keyword search.
- **USDA Weekly Retail Food Sales** (`data/NationalTotalAndSubcategory.csv`, `data/StateAndCategory.csv`) — national weekly retail scanner data on grocery purchases by product subcategory.

Raw data files live in `data/`. Intermediate files generated while running the notebooks (combined/cleaned data) are saved to `cleaned_data/`.

## Notebooks

The analysis is split into three notebooks in `code/`, meant to be run in order:

1. **[`01_data_pull.ipynb`](code/01_data_pull.ipynb)** — Loads the four raw Instagram/Facebook CSVs and the two raw USDA CSVs, does basic exploration (shape, columns, head), and saves the combined Instagram data to `cleaned_data/instagram_combined_raw.csv`.

2. **[`02_data_cleaning.ipynb`](code/02_data_cleaning.ipynb)** — Loads the combined Instagram data, cleans it (drops unmatchable rows, parses dates), and filters it into five trend-specific DataFrames by keyword (feta pasta, sourdough, banana bread, baked oats, dalgona coffee), flagging whether each post is COVID/quarantine-framed. Also loads and cleans the USDA data, filtering it down to the "Flour and mixes" and "Sweet mixes" subcategories used as retail-sales proxies. Saves all seven resulting DataFrames to `cleaned_data/`.

3. **[`03_analysis.ipynb`](code/03_analysis.ipynb)** — Loads the cleaned trend DataFrames and USDA subsets, and generates all charts: a peak-month comparison across the five trends, monthly post-frequency charts per trend (split by COVID-framing), and the two USDA retail sales charts. Exports all generated charts into a single zip file.

## Running the notebooks

Run in order from inside the `code/` folder — each notebook depends on files saved by the one before it:

```
cd code
jupyter notebook
```

Then run `01_data_pull.ipynb`, then `02_data_cleaning.ipynb`, then `03_analysis.ipynb`.

**Note:** the intermediate files saved to `cleaned_data/` are large (the combined raw Instagram file is several hundred MB), since they carry all original columns per row. If this project is version-controlled with git, `cleaned_data/` should be added to `.gitignore` rather than committed.

## Requirements

- pandas
- numpy
- matplotlib
