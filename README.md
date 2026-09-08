# Lifecycle and Fragility Analysis of Association Rules in Online Retail Transactions

An extended Association Rule Mining (ARM) pipeline that goes beyond a single-snapshot FP-Growth report. Built on the **Online Retail II** dataset (13 months of UK online retail transactions), it classifies rules into lifecycle stages, scores them for cancellation "fragility," and validates them on a genuine held-out time window â€” testing whether standard ARM outputs (support, confidence, lift) can actually be trusted for production recommendations.

> Coursework project - MSc Computing, Dublin City University.

## Why this exists

Most retail basket-analysis projects mine rules once on the full dataset and rank them by lift. That approach hides three problems:

1. **Time**  retail catalogues change; a rule that looks strong overall may already be dead.
2. **Cancellations** pipelines usually discard cancelled orders and never revisit them, even though a frequently-cancelled bundle is behaviourally unstable in a way support/lift can't see.
3. **Generalisation** rules are typically evaluated on the same data they were mined from, so there's no honest measure of whether they'd hold on fresh data.

This project builds three extra stages on top of a standard FP-Growth workflow to address each problem directly.

## Research questions

- **RQ1.** How are association rules distributed across lifecycle types (persistent, declining, emerging, seasonal) over a 13-month period?
- **RQ2.** Which rules are the most fragile (highest cancellation co-occurrence), and does fragility differ between domestic (UK) and export markets?
- **RQ3.** Do mined rules generalise to a held-out time window, and is lift a reliable proxy for out-of-sample relevance?

## Dataset

[Online Retail II](https://archive.ics.uci.edu/dataset/502/online+retail+ii) (UCI Machine Learning Repository)  transaction-level records from a UK-based online retailer, 1 December 2009 to 9 December 2010.

After cleaning: **509,467** purchase rows, **10,206** cancellation rows, **3,437** products, **21,810** unique invoices, pivoted into a binary invoice Ã— product basket matrix with **0.66%** density.

The raw file (`online_retail_II.xlsx`) is not included in this repo â€” download it from the link above and place it in the working directory (or upload it when prompted, if running in Google Colab).

## Pipeline / Methodology

1. **Cleaning** drop rows with missing invoice/stock code/quantity; separate cancellation invoices (IDs starting with `C`) *before* any other filtering, since they're the denominator for fragility later; resolve stock codes with multiple descriptions by keeping the most frequent one (481 collisions resolved); drop products appearing in fewer than 10 invoices.
2. **Basket encoding**  pivot to a binary invoice Ã— product matrix.
3. **Train / hold-out split**  first **10 months** (Dec 2009â€“Sep 2010) used for mining; last **3 months** (Octâ€“Dec 2010) held out entirely for evaluation.
4. **Rule mining**  FP-Growth (`mlxtend`) with `min_support=0.01`, `min_confidence=0.50`, `min_lift=1.5`.
5. **Rule enrichment**  every rule is additionally scored with **conviction**, **Zhang's metric**, and **Kulczynski**, so ranking doesn't rely on lift alone.
6. **Lifecycle classification (survivorship-bias corrected)**  rules are labelled *persistent*, *emerging*, *declining*, *seasonal*, or *sporadic* from their monthly support trajectory. Because rules that died early would never appear in the globally-mined set, the first 6 months are re-mined separately and any early-only rules are added back in as "declining" â€” otherwise the declining category would look artificially small.
7. **Statistical validation**  chi-square becomes meaningless at this sample size (nearly everything is p < 0.05), so **CramÃ©r's V** (effect size, threshold 0.10) is used as the primary filter instead.
8. **Cancellation fragility**  for a rule `A â†’ C`, `fragility = (cancellation baskets containing AâˆªC) / (purchase baskets containing AâˆªC)`, computed globally, per month (to flag unstable/low-denominator estimates), and separately for domestic (UK) vs. export baskets, with Wilcoxon and Mann-Whitney U tests comparing the two markets.
9. **Held-out evaluation**  recompute support for every training-mined rule on the untouched 3-month hold-out basket; a rule "generalises" if it still clears the support threshold. Spearman correlation checks whether training-period lift predicts hold-out support.
10. **Stability checks**  month-to-month Jaccard overlap of active rule sets (with bootstrap confidence interval) and k-means clustering (k â‰¥ 3, silhouette-selected) on per-rule monthly support trajectories.
11. **Pipeline scorecard & visualisations**  a 5-dimension quantitative scorecard plus 7 summary figures.
12. **Business recommendations**  rules are grouped into a core bundle engine (persistent, low-fragility), a remove list (declining + fragile), a watch list (emerging), and export-specific opportunities.

## Key findings

| # | Result |
|---|---|
| RQ1 | **64.5%** of the 479 rules (after survivorship correction) are persistent, **26.7%** declining, 4.8% emerging, 3.9% seasonal. The declining share is only visible because of the early-mining correction. |
| RQ1 (robustness) | Across a 125-combination threshold sweep, the persistent share varies 47.4â€“69.9% and emerging 2.3â€“12.7%, but declining barely moves (Â±1.3 pp) â€” the declining label is the most robust. |
| Statistical strength | All 479 rules pass CramÃ©r's V â‰¥ 0.10 (mean 0.452); mean lift 15.7, mean Zhang's metric 0.985. |
| RQ2 | Fragility is highly skewed (median 0, mean 0.0026, max 0.136); 48 rules flagged fragile (90th percentile), 28 of those also flagged as unreliable due to high month-to-month variance. Support is statistically indistinguishable between UK and export markets (Wilcoxon, p = 0.148), but **fragility differs significantly between them** (Mann-Whitney U, **p < 0.0001**) â€” identical-looking bundles can carry different hidden cancellation risk by market. |
| RQ3 | Only **34.0%** of the 479 training-mined rules (163) still meet the support threshold on the 3-month hold-out â€” persistent rules generalise at 44.3%, more than 3Ã— the rate of declining rules (13.3%). |
| RQ3 (headline) | Training-period lift is **negatively** correlated with hold-out support (**Spearman Ï = âˆ’0.538**, p < 0.0001) â€” the highest-lift rules on this dataset are the *least* likely to generalise, often narrow, family-specific colour-variant pairings inflated by small denominators. |
| Stability | Mean consecutive-month Jaccard overlap of active rule sets is 0.888 (95% bootstrap CI [0.850, 0.924]) â€” the same rule identifiers keep reappearing even though many drift just below the strict hold-out support cut-off. K-means (k=3) reaches 75.6% cluster purity against the lifecycle labels; the cluster dominated by declining rules also has by far the highest mean lift (63), echoing the negative lift-generalisation finding. |

**Bottom line:** ranking rules by lift alone is misleading for production use on this dataset  **support combined with CramÃ©r's V** is the safer default, and cross-border recommendations should be re-scored per market rather than relying on a single global ranking.

## Repository contents

| File | Description |
|---|---|
| `ARM_Final.py` | End-to-end pipeline script  data cleaning, FP-Growth mining, lifecycle classification, fragility scoring, hold-out evaluation, stability checks, and figure generation. |
| `ARM_Final_1.ipynb` | Jupyter notebook version of the same pipeline, with inline output, tables, and figures. |
| `arm_report.pdf` | Full written report ("Lifecycle and Fragility Analysis of Association Rules in Online Retail Transactions") with related work, methodology, results, and discussion. |

## Requirements

- Python 3.8+
- `pandas`, `numpy`, `matplotlib`, `seaborn`
- `mlxtend` (FP-Growth, association rule generation)
- `scikit-learn` (KMeans, scaling, silhouette score)
- `scipy` (chi-square, Wilcoxon, Mann-Whitney U, Spearman correlation, entropy)
- `openpyxl` (reading the `.xlsx` dataset)

Install everything with:

```bash
pip install pandas numpy matplotlib seaborn mlxtend scikit-learn scipy openpyxl
```

## Usage

1. Download the [Online Retail II dataset](https://archive.ics.uci.edu/dataset/502/online+retail+ii) and save it as `online_retail_II.xlsx` in the project's working directory.
2. Run the pipeline:

   ```bash
   python ARM_Final.py
   ```

   (or open `ARM_Final_1.ipynb` in Jupyter/Colab and run all cells  if the dataset file isn't found and the notebook is running in Google Colab, it will prompt for an upload).

3. Outputs are written to the working directory:
   - `fig1_lifecycle_silhouette.png`  lifecycle distribution & silhouette sweep
   - `fig2_trajectories.png`  monthly support trajectories by lifecycle
   - `fig3_fragility.png`  fragility distribution & boxplot by lifecycle
   - `fig4_support_fragility.png`  support vs. fragility scatter
   - `fig5_metrics.png`  lift vs. CramÃ©r's V vs. conviction vs. Kulczynski
   - `fig6_domestic_export.png`  domestic vs. export support & fragility comparison
   - `fig7_jaccard.png`  temporal Jaccard stability with bootstrap CI

Key parameters (minimum support/confidence/lift, lifecycle thresholds, train/hold-out month split, statistical cut-offs) are all configurable at the top of the script/notebook.

## Limitations

- The dataset spans only 13 months, so true annual seasonality can't be tested.
- Fragility estimates for low-support rules are inherently unstable (small denominators); these are flagged but not fully resolved.
- The hold-out test uses a single rolling split  an expanding-window cross-validation would give a tighter estimate.
- Export markets are treated as one bucket, which masks within-export differences by country.

## Future work

- Extend the pipeline to multi-year retail data to test genuine seasonality.
- Replace the single train/hold-out split with rolling cross-validation.
- Test whether the negative lift-vs-generalisation relationship found here replicates on other retail datasets.

## Authors

Aadesh Sunil Kshetre, Haritha Ramadass, Kanika Pathak, Priyadharshini Dhanaraj Muthamil Selvi  MSc Computing, Dublin City University.

## References

Key references followed in this work: Agrawal & Srikant (Apriori); Han, Pei & Yin (FP-Growth); Tan, Kumar & Srivastava (interestingness measures survey); Zhang (Zhang's metric); Wu, Chen & Han (Kulczynski + imbalance ratio); Webb (effect-size vs. p-value inflation); Hahsler et al. (rule reproducibility). Full citations are in `arm_report.pdf`.

