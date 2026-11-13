# Capstone Report — Refresh / Content Opportunity Scoring

- **Author:** Franklin ML Work
- **Lane:** Refresh / Content Opportunity Scoring
- **Repo:** https://github.com/StickRift/Rift
- **Date:** November 13, 2026

## 0. Abstract

This study investigates whether historical content freshness and search-performance signals can help identify and prioritise web pages that may need content review. Using the FlyRank ML Internship dataset, the analysis combines monthly content and Google Search Console performance data to construct a dataset for predicting declines in average daily impressions. A Random Forest classifier was evaluated against a majority-class baseline using a forward-in-time, client-grouped holdout split. The Random Forest achieved a ROC-AUC of 0.6772 and average precision of 0.8536, outperforming the baseline on several discrimination metrics but not on overall accuracy. The resulting probability scores and ranked review list are intended to help content editors prioritise pages for human investigation rather than automatically determine which pages should be updated.

## 1. Problem framing

Websites may contain thousands of content pages, making it difficult for editors to decide which pages deserve attention first. This project investigates whether historical content freshness and search-performance data can help prioritise pages for human review.

The decision supported by this analysis is: **Which content pages should a FlyRank editor investigate first because their historical signals suggest a possible decline in search visibility?**

The unit of analysis is an individual content item or page observed over a monthly reporting period.

The output is a ranked list of content items with estimated probabilities of experiencing a decline in average daily Google Search Console impressions in the following month.

A FlyRank content or SEO editor can use the ranking to investigate a page's search history, relevance, accuracy, content quality, and technical condition. The editor then decides whether to update the content, investigate a technical issue, or continue monitoring the page.

A false positive may cause an editor to spend time reviewing a page that does not decline. A false negative may cause a page experiencing a decline to receive less attention than it needs.

Machine learning is useful here because it can examine several historical signals together and produce a consistent prioritisation score. However, the score is decision support, not an automatic content-update decision.

## 2. Data safety

### Data source

The analysis used the anonymised FlyRank ML Internship dataset:

- Hugging Face dataset: `FlyRank/internship-warehouse`
- Dataset release: `v20260703`
- Main performance table: `fact_content_daily_performance`
- Content information table: `dim_content`

Daily performance records were aggregated into monthly observations. Only records with usable Google Search Console data were included in the modelling frame. Feature and outcome months also had to meet the minimum requirement of 15 usable reporting days.

### Features used

The model used five features:

1. `days_since_update`
2. `avg_daily_impressions`
3. `avg_daily_clicks`
4. `ctr`
5. `avg_position`

These features describe content freshness and historical search performance.

### Fields deliberately excluded

Client and content identifiers were used only where necessary for joining, grouping, and separating clients between training and testing. They were not used as predictive model features.

The following fields were deliberately excluded:

- `trend_direction`
- `trend_pct`

These fields were excluded because they are label-derived or could reveal information closely related to the outcome.

Other available fields, including `word_count`, `search_volume`, and `backlinks`, were not included in the final five-feature model.

Future outcome information was not used as a model input.

Google Search Console unavailability was not treated as zero performance. Missing feature values were handled through median imputation.

### Privacy considerations

The analysis used the anonymised internship dataset. Client names, identifiable website URLs, and other client-identifying details are not included in this report or its intended portfolio outputs.

The analysis does not attempt to identify clients or redistribute the raw dataset.

## 3. Baseline

Before evaluating the Random Forest model, a simple majority-class baseline was created using:

`DummyClassifier(strategy='most_frequent')`

This classifier predicts the most common outcome for every test observation. It provides a transparent comparison because it does not learn relationships between the input features and the target.

The baseline and Random Forest were evaluated on the same test observations.

### Baseline results

- Test observations: 6,796
- Decline observations: 5,088
- No-decline observations: 1,708
- Test decline rate: 74.87%

| Metric | Majority-class baseline |
|---|---:|
| Accuracy | 0.7487 |
| Balanced accuracy | 0.5000 |
| Precision | 0.7487 |
| Recall | 1.0000 |
| F1-score | 0.8563 |
| ROC-AUC | 0.5000 |
| Average precision | 0.7487 |

The baseline achieved 74.87% accuracy because approximately 74.87% of the test observations belonged to the decline class.

However, it could not distinguish between individual pages. It predicted decline for every observation, resulting in a balanced accuracy of 0.5000 and ROC-AUC of 0.5000.

This demonstrates why accuracy alone is not sufficient for evaluating this task.

## 4. Model / analysis

### Model selection

A Random Forest classifier was selected because it can examine relationships between multiple numerical features and capture non-linear patterns without requiring a simple linear relationship between each feature and the outcome.

A median imputation step was included in the modelling pipeline to handle missing feature values.

### Model configuration

The final model used:

- Algorithm: Random Forest Classifier
- Number of trees: 300
- Maximum depth: 12
- Minimum samples per leaf: 20
- Class weighting: `balanced_subsample`
- Random state: 42
- Number of jobs: -1

### Feature list

The model used exactly five features:

- `days_since_update`
- `avg_daily_impressions`
- `avg_daily_clicks`
- `ctr`
- `avg_position`

Identifiers and label-derived fields were excluded to reduce leakage and prevent the model from learning client identity rather than useful performance patterns.

### Target definition

The target is a binary indicator equal to 1 when the next month's average daily Google Search Console impressions are lower than the current feature month's average, and 0 otherwise.

The model produces estimated probabilities of decline. These probabilities are used to rank content items for human review.

A predicted probability is not a guarantee that a page will decline.

## 5. Evaluation

### Evaluation strategy

The evaluation used a forward-in-time split combined with client grouping.

This approach was chosen to reduce the risk of evaluating the model on observations from clients already represented in the training data.

The split produced:

| Measurement | Training set | Testing set |
|---|---:|---:|
| Observations | 23,876 | 6,796 |
| Clients | 23 | 5 |
| Feature periods | February and March 2026 | April 2026 |
| Outcome periods | March and April 2026 | May 2026 |
| Decline rate | 55.24% | 74.87% |

There was no client overlap between the training and testing sets.

The testing set contained five clients that were not represented in training.

### Model versus baseline

Both approaches were evaluated on the same 6,796 test observations.

| Metric | Majority-class baseline | Random Forest |
|---|---:|---:|
| Accuracy | 0.7487 | 0.6953 |
| Balanced accuracy | 0.5000 | 0.6096 |
| Precision | 0.7487 | 0.8054 |
| Recall | 1.0000 | 0.7818 |
| F1-score | 0.8563 | 0.7935 |
| ROC-AUC | 0.5000 | 0.6772 |
| Average precision | 0.7487 | 0.8536 |

### Error analysis

The Random Forest confusion matrix was:

| | Predicted no decline | Predicted decline |
|---|---:|---:|
| Actual no decline | 747 | 961 |
| Actual decline | 1,110 | 3,978 |

The model correctly identified 747 observations that did not experience a decline and 3,978 observations that did.

It also produced 961 false positives and 1,110 false negatives.

The model achieved lower overall accuracy than the majority-class baseline. However, it performed better in balanced accuracy, precision, ROC-AUC, and average precision.

Its ROC-AUC of 0.6772 indicates modest ability to distinguish between the two classes. Its average precision of 0.8536 was higher than the test-set decline rate of 0.7487.

These results suggest that the model provides some ranking value beyond simply predicting the majority class, but its performance is not strong enough to justify fully automated decisions.

### Evaluation limitations

The test set contained five clients, while the training set contained 23. The decline rate also differed between training and testing.

These factors limit how confidently the results can be generalised to other clients or future periods.

The evaluation used May 2026 as the outcome period. It was not a sealed June 2026 evaluation.

## 6. Interpretation

The Random Forest generated estimated probabilities that were used to rank content items for review.

The top-20 review list contained probabilities ranging from approximately 0.6858 to 0.8223.

### Top-20 review findings

The rule-based reason codes associated with the top 20 items were:

| Reason code | Number of items |
|---|---:|
| VISIBLE_ONLY | 11 |
| NO_PRIORITY_SIGNAL | 9 |
| STALE_ONLY | 0 |
| STALE_AND_VISIBLE | 0 |

The visibility rule used a median impressions threshold of 23.25.

The staleness rule used an update age greater than 90 days.

Eleven of the top 20 items were classified as `VISIBLE_ONLY`. Nine were classified as `NO_PRIORITY_SIGNAL`.

None of the top 20 items crossed the staleness threshold.

This does not demonstrate that content freshness is irrelevant. It only shows that none of the selected top-20 items met the chosen staleness rule.

### What the results mean

The model was able to produce a ranked list that contained useful prioritisation information, but the results also show that the model is imperfect.

The evaluation did not establish that updating the selected pages would improve their search performance.

Feature importance values are not reported in this analysis. The interpretation therefore focuses on the measured evaluation results and the reason-code distribution in the top-20 review list rather than making unsupported claims about individual feature contributions.

## 7. Recommendation

The results support using the model as an additional prioritisation tool for human content review.

The recommended workflow is:

1. Start with the highest-ranked content items.
2. Investigate the 11 top-20 items classified as `VISIBLE_ONLY`, paying attention to their historical visibility signals.
3. Review the nine items classified as `NO_PRIORITY_SIGNAL` rather than assuming that they do not need attention.
4. Check content quality, relevance, accuracy, search intent, and possible technical issues before recommending an update.
5. Monitor subsequent search performance to see whether the observed patterns continue.

The model should not automatically trigger content updates or replace editorial judgement.

### Confidence and limitations

Confidence in the model's ability to provide useful ranking information is cautious.

The ROC-AUC of 0.6772 shows modest discrimination, and the Random Forest did not outperform the majority-class baseline in overall accuracy.

The evaluation also involved only five test clients, and the decline rate was higher in testing than in training.

The model should therefore be treated as an experimental decision-support tool. Additional evaluation across more clients and future reporting periods would be needed before relying on it in a production workflow.

## 8. Reproducibility

### Repository

The project repository is:

https://github.com/StickRift/Rift

### Main project files

- Notebook: `work/notebooks/capstone.ipynb`
- Model comparison results: `work/outputs/capstone_model_comparison.csv`
- Top-20 review results: `work/outputs/capstone_top20_review.csv`
- Model comparison chart: `work/outputs/capstone_model_comparison.png`

### Environment

The project dependencies are listed in `requirements.txt`:

- pandas >= 2.2
- numpy >= 1.26
- scikit-learn >= 1.4
- matplotlib >= 3.8
- reportlab >= 4.0
- duckdb >= 1.0
- huggingface_hub >= 0.24

These are minimum dependency versions, not a record of the exact installed package versions.

### Reproduction steps

From a fresh environment:

1. Clone the repository:

   `git clone https://github.com/StickRift/Rift.git`

2. Enter the project folder:

   `cd Rift`

3. Install the dependencies:

   `pip install -r requirements.txt`

4. Open `work/notebooks/capstone.ipynb` in Google Colab or a compatible Jupyter environment.

5. Configure Hugging Face authentication and access to the gated `FlyRank/internship-warehouse` dataset.

6. Run the notebook from beginning to end.

7. Compare the generated evaluation results with the committed CSV files and chart.

### Reproducibility settings

The Random Forest used `random_state=42`.

The evaluation used a forward-in-time, client-grouped holdout split with no client overlap.

The final evaluation used February and March 2026 feature periods for training and April 2026 features for testing, with May 2026 as the test outcome period.

This was not a sealed June evaluation.

Reproduction requires access to the gated dataset and a compatible Python environment. Exact package versions may differ from the original execution environment.

## 9. Acknowledgments & data credit

Built on the [FlyRank ML Internship dataset](https://flyrank.ai) for research and educational purposes.