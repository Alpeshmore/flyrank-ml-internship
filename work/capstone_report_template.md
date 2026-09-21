

**Author:** Alpesh More  
**Lane:** Lane 2 — Refresh / Content Opportunity Scoring  
**Repo:** https://github.com/Alpeshmore/flyrank-ml-internship  
**Date:** 2026-09-21

---

## 0. Abstract

This project asks which content pages should be prioritized for manual review based on observed search visibility, traffic, engagement, freshness, and content signals. The analysis uses an anonymized starter dataset containing observable content and search-performance signals and evaluates a transparent baseline against supervised machine-learning models. The models were evaluated using client-holdout validation so that entire clients were kept out of training and used for evaluation. The random forest achieved the strongest measured starter-slice performance, with ROC AUC of 0.750, average precision of 0.618, and Precision@50 of 0.740, compared with 0.627, 0.468, and 0.240 for the baseline rules. The resulting ranking is intended to help editors prioritize pages for human review using observable evidence and reason codes, not to guarantee that refreshing a page will improve its performance.

---

## 1. Problem framing

### Decision supported

The decision is:

> Which content pages should enter the content review queue first?

The system prioritizes pages for manual review rather than automatically deciding what an editor should change.

### Unit of analysis

The unit of analysis is a content page identified by a pseudonymized content ID.

The pseudonymized client ID is used for client-holdout validation and grouping only. It is not used as a model feature.

### Output

The main output is a ranked review queue containing:

- content ID
- opportunity score
- rank
- reason code(s)
- supporting signals
- suggested editorial action
- confidence label

### Human action

A FlyRank editor can use the ranked queue to decide whether a page should be:

- manually reviewed for refresh
- reviewed for metadata/CTR improvement
- expanded or improved
- monitored
- left unchanged

The score is decision-support. It does not automatically prescribe an editorial action.

### Cost of a wrong call

A false positive can waste editorial review time on a page that does not need attention.

A false negative can cause a potentially useful page to remain untreated.

### Why ML helps

A transparent rule can identify obvious cases, but multiple search, traffic, engagement, freshness, and content signals can interact. A model can combine these observable signals into a ranked queue and can be evaluated against the transparent baseline.

The goal is not automated SEO. The goal is to test whether a learned ranking can produce a more useful review queue than a simple rule.

---

## 2. Data safety

### Data used

The project uses the FlyRank anonymized starter dataset:

`data/raw/content_refresh_anonymized.csv`

The starter dataset contains observable content and search-performance signals.

The broader FlyRank warehouse release is:

`flyrank_pseudonymized_warehouse_release_v20260703`

The warehouse contains pseudonymized IDs and observable performance signals.

### Candidate signals

The analysis uses signals such as:

- search volume
- competition
- CPC
- word count
- character count
- 90-day impressions
- 90-day clicks
- 90-day sessions
- AI sessions
- days with impressions
- days with sessions
- content age
- freshness
- CTR
- average position
- engagement rate
- scroll rate
- AI traffic percentage

Categorical signals include:

- competition level
- content type
- main intent
- age tier
- freshness tier
- word-count tier
- impression tier
- position tier

### Deliberately excluded

The following are excluded from model features or public output:

- client names
- domains
- URLs
- raw private queries
- identifying titles/text
- credentials
- product decision outputs
- future target-window metrics
- pseudonymous IDs as predictive features

Client IDs are used only for grouping/validation.

### Leakage risks

The main leakage risks considered were:

- using information calculated after the decision point
- using future-window metrics as features
- using target-derived fields as predictors
- allowing the same client to appear across train and validation
- allowing duplicate or related observations to make validation artificially easy
- using existing product decision outputs as ordinary model features

`trend_direction` is used to define the starter proxy label and is therefore not treated as an independent predictive feature.

### Public-safety check

The final public output uses pseudonymized IDs, aggregated metrics, safe charts, high-level examples, and generic content actions.

No client-identifying information should appear anywhere in `work/`.

---

## 3. Baseline

The first comparison is a transparent rule-based refresh score.

The baseline is:

```text
baseline_refresh_score =
    0.40 * visibility_score
  + 0.30 * freshness_risk_score
  + 0.25 * position_opportunity_score
  + 0.05 * depth_gap_score
The baseline uses transparent reason codes including:

stale_visible_page
declining_with_demand
thin_visible_page
page_one_decay_risk
low_ctr_visible_page
low_engagement_visible_page

This is a fair comparison because the baseline produces a ranking from observable signals and can be evaluated on the same validation population as the machine-learning models.

Results
[Method]	[ROC] [AUC]	[Average] [Precision]	[Precision@50]
[Baseline rules]	[0.627]	[0.468]	[0.240]
[Logistic Regression]	[0.700]	[0.522]	[0.400]
[Decision Tree]	[0.742]	[0.575]	[0.540]
[Random Forest]	[0.750]	[0.618]	[0.740]

These results come from a 30,000-row anonymized starter slice using client-holdout validation. They are not results from the full approximately 79-million-row daily warehouse.

4. Model / analysis

Method

Three supervised models were evaluated against the transparent baseline:

Logistic Regression
Decision Tree
Random Forest

The random forest produced the strongest measured performance on the starter validation experiment.

Target

The starter target is:

is_declining_label = trend_direction == "down"

This is treated as a proxy label because it is calculated from the observed current window rather than representing a confirmed future outcome.

A stronger future-looking capstone design would define features from a prior window and measure decline during a separate future window.

Features

The candidate model features include:

Numeric

search volume
competition
word count
character count
logged 90-day impressions
logged 90-day clicks
logged 90-day sessions
logged AI sessions
days with impressions
days with sessions
content age
freshness
CTR
average position
engagement rate
scroll rate
AI traffic percentage

Categorical

competition level
content type
main intent
age tier
freshness tier
word-count tier
impression tier
position tier

Deliberately excluded

The following were not used as normal predictive features:

trend_direction, because it defines the starter target
target-derived trend fields
future-window metrics
client IDs
content IDs
product decision scores or flags
identifying URLs, titles, domains, or queries

5. Evaluation

Validation design

The starter experiment uses client-holdout validation.

Entire clients are kept out of training and used for evaluation. This reduces the risk that the model simply learns patterns specific to clients it has already seen.

Metrics

The evaluation uses:

ROC AUC
Average Precision
Precision@50

Precision@50 is particularly relevant because the real decision is a ranked review queue.

Model comparison

Method                  ROC AUC    Average Precision    Precision@50
Baseline rules          0.627      0.468                0.240
Logistic Regression     0.700      0.522                0.400
Decision Tree           0.742      0.575                0.540
Random Forest           0.750      0.618                0.740

The random forest increased Precision@50 from 0.240 for the baseline to 0.740.

In practical terms, the baseline identified about 12 positive pages among its top 50 ranked pages, while the random forest identified about 37 positive pages among its top 50 under the selected starter label.

Error analysis

The ranking should still be reviewed for:

false positives
false negatives
low-volume pages
high-volume pages
stale pages
pages with weak CTR
pages with weak engagement
pages near the review cutoff

A high score does not guarantee that the page needs a refresh. Decline can also reflect consolidation, seasonality, SERP/AI click changes, or noise.

6. Interpretation

The measured results show that the supervised models produced stronger ranking performance than the transparent baseline on the 30,000-row starter experiment.

The random forest had:

ROC AUC: 0.750
Average Precision: 0.618
Precision@50: 0.740

The baseline had:

ROC AUC: 0.627
Average Precision: 0.468
Precision@50: 0.240

The largest practical difference appears in Precision@50, which is directly related to the use case of reviewing a limited number of pages first.

The results therefore support the narrower statement that a learned ranking performed better than the tested fixed-rule baseline on this starter validation slice. They do not establish that the model will perform identically on the full warehouse or in future production data.

Plain-language interpretation

The model combines multiple observable page signals instead of relying on one rule.

The ranking can therefore identify pages whose combination of visibility, freshness, content, and performance signals resembles the selected decline target.

Negative-result / limitation interpretation

The starter target is not a future causal outcome.

A page classified as declining does not mean that refreshing it will cause traffic or rankings to recover.

Similarly, the results do not prove that any individual feature is a Google ranking factor.

7. Recommendation

The model output should be used as a human review queue.

Priority 1 — Manual review

Review pages with:

high model score
sufficient impressions
sufficient sessions
clear reason codes

Priority 2 — CTR review

Inspect pages with:

meaningful impressions
reasonable average position
comparatively weak CTR

Potential actions include reviewing title/meta wording, intent match, and snippet structure.

Priority 3 — Engagement review

Inspect pages with:

sufficient sessions
weak engagement or scroll signals

Potential actions include reviewing content structure, relevance, readability, and user experience.

Priority 4 — Stale visible pages

Review older pages that continue to receive meaningful visibility.

Potential actions include checking whether the information, examples, structure, and search intent remain current.

Priority 5 — Monitor

Pages with insufficient evidence should be monitored rather than automatically changed.

Confidence

Higher-confidence review candidates should have enough traffic evidence and supporting signals. The confidence label should not be based on the model score alone.

Limits

The output does not prove:

that refreshing a page will increase traffic
that a page will recover after an edit
that a model feature is a Google ranking factor
that every declining page needs a refresh
that the model has identified a causal relationship

The output is a decision-support system for prioritizing human review.



8. Reproducibility
Repository

https://github.com/Alpeshmore/flyrank-ml-internship

Main notebook

work/notebooks/capstone.ipynb

Report

work/capstone_report.md

Paper URL

submission/paper_url.txt

Data

The starter experiment uses:

data/raw/content_refresh_anonymized.csv

Re-run

From a fresh clone:

git clone https://github.com/Alpeshmore/flyrank-ml-internship.git
cd flyrank-ml-internship

Then run the capstone notebook from top to bottom in the documented environment.

Random seed

Record the exact random seed used by the final notebook execution.

Random seed: [record from final notebook]
Environment

Record the actual versions used in the final run.

Python: [record actual version]
pandas: [record actual version]
numpy: [record actual version]
scikit-learn: [record actual version]
Evaluation artifact

The verified starter benchmark is represented by the model-results artifacts:

outputs/model_results.json
outputs/model_report.md

The final repository should contain the corresponding metrics artifact produced by the final capstone run.

9. Acknowledgments & data credit

Built on the FlyRank ML Internship dataset.

https://flyrank.ai


**Important:** before submitting, replace the three `[record ...]` placeholders in Section 8 with the actual value
