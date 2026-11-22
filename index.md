# Ranking Content Pages for Review: A Future-Window Refresh Queue

**Author:** Vedhesh A | **Track:** FlyRank ML Internship, Applied Search Intelligence | **Lane:** Refresh / Content Opportunity Scoring

## Abstract

Content teams can review only a limited number of pages, so which pages should a human look at first? Using the pseudonymized FlyRank warehouse release, I built one row per page from a 90-day feature window and labeled whether its impressions fell by at least 20% over the following 30 days (about 100,000 pages from 44 clients for development, and about 111,000 pages from 50 clients in a sealed final month). I compared a transparent rule baseline with logistic regression and a random forest, validating by holding out whole clients and then scoring the final month once. On that sealed test the random forest ranked highest (ROC AUC 0.630, average precision 0.721 against a base rate of 0.643) and was consistently ahead of the rule across clients, but its lead over logistic regression could not be distinguished from zero, and the simple rule matched or beat both models among the top 50 to 1,000 pages. The signal is modest and observational: the results support prioritizing human review and do not show that refreshing a page causes recovery.

## 1. Introduction and problem statement

A content team can only review a limited number of pages per cycle. This work supports one decision: **which pages should a human reviewer look at first?** A wrong recommendation either wastes reviewer time on a healthy page or leaves a declining page unnoticed. The output is a ranked list with a reason code and a suggested action per page. It is built for decision support only, and a human makes the final call.

## 2. Data

- **Release:** FlyRank internship warehouse, build `flyrank_pseudonymized_warehouse_release_v20260703` (gated release on Hugging Face). Daily facts run through 2026-06-30.
- **Table used:** `fact_content_daily_performance` (search impressions, clicks and average position per client, page and day). No other tables were used.
- **Grain:** one row = one page of one client at one decision date.
- **Development window:** features from 2026-01-01 to 2026-03-31, label from 2026-04-01 to 2026-04-30. This gave 100,849 pages from 44 clients.
- **Sealed final test:** features from 2026-03-03 to 2026-05-31, label from 2026-06-01 to 2026-06-30. This gave 111,247 pages from 50 clients. It was built only after all modeling choices were frozen and scored once.
- **Minimum volume:** only pages with at least 100 impressions in the last 30 days of the feature window were kept.
- **Excluded and why:** any product score or flag (circular-result risk); anything dated after the decision date (leakage); raw client, URL, query and title fields (never in the release, and never reconstructed).
- **Client concentration:** the five largest clients held about two thirds of the development pages (the largest about 21%).

## 3. Methodology

- **Label (future observed outcome):** a page is positive if its next-30-day impressions are below 80% of its last-30-day impressions. The 20% threshold is a policy choice, not a universal truth. The positive rate was 0.498 in development and 0.643 in June, so results are compared against each month's base rate.
- **Features (nine, all computed from dates up to the decision date):** impressions in the last 30 days, impressions in the 60 days before that, total impressions and clicks over 90 days, click-through rate, impression-weighted average position, number of active days, standard deviation of daily impressions, and a trend ratio (last-30-day impressions divided by the earlier daily rate).
- **Baseline (a transparent rule, no labels used):** score = 0.4 x percentile rank of recent demand + 0.6 x size of the recent drop in impressions. It was fixed before modeling and not changed afterwards.
- **Models:** logistic regression (standardized inputs) and a random forest (300 trees, minimum leaf size 5).
- **Validation:** 5-fold client-grouped cross-validation on the development window (whole clients held out of training), then a single score on the sealed final month.
- **Primary metric (committed before the final test):** average precision. Precision@K and lift over the base rate (K = 50, 200, 1,000) are secondary, because top-K numbers at K = 50 are noisy.
- **Leakage audit:** (1) the feature window ends at the decision date and the label uses only later dates, so the windows do not overlap; (2) no product scores or flags are used; (3) a deliberate-leak test, adding a column built from the future window, raised AUC from 0.588 to 1.0, which shows this kind of leak would be caught, and the leak column is not in any reported model; (4) whole clients are held out, so no page appears in both train and test; (5) the final month was scored once with choices frozen beforehand.
- **Tested and dropped:** client-relative features (each page compared with its own client's median) lowered development average precision (random forest 0.557 to 0.527), so they were not used.

## 4. Results

### Development window (client-grouped cross-validation, base rate 0.498)

| Method | ROC AUC | Average precision | Precision@50 |
|---|---|---|---|
| Rule baseline | 0.475 | 0.483 | 0.62 |
| Logistic regression | 0.548 | 0.511 | 0.86 |
| Random forest | 0.588 | 0.557 | 0.68 |

The rule scored below chance on development data. Precision@50 varied a lot between folds (logistic regression ranged from 0.28 to 0.94), so it is not a reliable basis for choosing a model.

### Sealed final month (June, base rate 0.643)

| Method | ROC AUC | Average precision | P@50 | P@200 | P@1000 |
|---|---|---|---|---|---|
| Rule baseline | 0.520 | 0.671 | 0.82 | 0.795 | 0.766 |
| Logistic regression | 0.596 | 0.702 | 0.72 | 0.645 | 0.700 |
| Random forest | 0.630 | 0.721 | 0.66 | 0.705 | 0.757 |

![Average precision against the chance line on the sealed June test](charts/precision_at_k.png)

**Comparing methods across clients.** Resampling whole clients, the random forest's average precision was higher than the rule's by 0.051 (95% range 0.015 to 0.092), so it was consistently ahead of the rule. Its lead over logistic regression was 0.023 (range -0.017 to 0.062), which cannot be distinguished from zero.

**What the numbers mean.** Both learned models and the random forest in particular carried a modest signal into a new month and a larger set of clients: the random forest's average precision was about 12% above the base rate. In the top of the ranking, however, the rule was as good or better than the models (precision@50 of 0.82 against 0.66), and the random forest's precision@50 was close to the June base rate. The signal is in the broad ordering of pages, not in the highest few dozen.

![Random forest feature importance](charts/feature_importance.png)



**Shape of the signal.** In development, the share of pages that declined was close to the base rate for most values of the trend ratio, and clearly lower for pages whose recent impressions were far above their earlier level (about 0.29 for the highest group). The trend-bin analysis shows that decline rates were not consistently higher for pages with lower trend ratios across most bins. However, pages in the highest trend-ratio group had a substantially lower observed decline rate of 0.292, compared with 0.542 in the lowest group. This suggests that pages experiencing a strong recent increase in impressions were less likely to meet the decline label in the following window, although trend ratio alone does not consistently separate declining pages across all groups. The visible highest-ranked examples were dominated by the high_model_risk reason code and the monitor action, rather than an immediate refresh recommendation. The displayed top-ranked pages had relatively low recent impression volumes (121–828) and very high trend ratios (approximately 5.3–84.6), indicating that the highest model scores may be influenced by noisy or rapidly changing page activity. This reinforces the need to manually verify recent volume and trends before acting on the ranking.

## 5. Limitations and honest framing

- These results are **observed, directional and decision-support**. They do not show that editing a page causes recovery, and they say nothing about how any search engine ranks pages.
- A "decline" in this label mixes real decline with natural fall-back after a peak, seasonality and noise. The base rate rose from 0.498 to 0.643 between the development window and June, so metrics are not directly comparable across months, and the data cannot separate these explanations.
- The signal is modest: average precision of 4% to 12% above the base rate in June. The top of the model's list was no more reliable than the rule's, and precision at K = 50 is a small, noisy sample.
- The random forest and logistic regression could not be separated statistically at the client level.
- A few large clients hold most pages, so results may not carry to small clients or to clients with short tracking history.
- There was one development period and one test month. The 20% threshold and the 100-impression minimum are policy choices.
- A checklist audit only catches the leaks it tests for: no leakage was found by this checklist, which is not proof that none exists.

## 6. Ranked recommendations (action playbook)

The model score is a way to order pages for human review. It is not an instruction to edit them.

| Action | When to use | What would make it wrong |
|---|---|---|
| Review for refresh (`declining_with_demand`) | The page's recent impressions are already below its earlier level and it has real volume. | The drop is seasonal, a sibling page absorbed the demand, or the page is evergreen and still correct. |
| Monitor (`high_model_risk`) | The model scores the page as high risk but there is no visible drop yet. | The page has low recent volume or a sudden ramp-up, so a small swing looks like a decline. Many of the highest-scored pages in June looked like this, so check volume before acting. |
| Compare against the simple rule | Use the transparent rule alongside the model score when only a few dozen pages can be reviewed. | In this data the rule matched or beat the models at the top of the list, so the extra complexity of the model should earn its place. |

Practical reading: use the model score to rank the whole inventory and decide where review effort is spent in bulk. For the very top handful of pages, check the numbers by hand before acting.

## 7. Reproducibility

- Repo: [github.com/vedheshatthi104ff-hub/Internship](https://github.com/vedheshatthi104ff-hub/Internship)
- Notebooks: `work/notebooks/` (weekly notebooks and `capstone.ipynb`).Project colab Notebook: [github.com/vedheshatthi104ff-hub/Project/blob/main/Project.ipynb](https://github.com/vedheshatthi104ff-hub/Project/blob/main/Project.ipynb) .
- To rerun: open `capstone.ipynb` in Colab, add a read-only `HF_TOKEN` secret (request access to the gated release first), and run all cells. The development data is cached after the first run, and random seeds are fixed.

## 8. Acknowledgments and data credit

Built on the [FlyRank ML Internship dataset](https://flyrank.ai). Thanks to the FlyRank internship team for the data and guidance.
