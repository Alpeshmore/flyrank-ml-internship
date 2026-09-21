

**Author:** Alpesh More  
**Lane:** Lane 2 — Refresh / Content Opportunity Scoring  
**Repo:** https://github.com/Alpeshmore/flyrank-ml-internship  
**Date:** 2026-09-21

---


## Abstract

Content teams need a practical way to prioritize which pages should receive manual review when the number of potentially improvable pages is larger than available editorial capacity. This study builds a content refresh opportunity scoring approach using observable search, traffic, engagement, freshness, and content signals, comparing a transparent rule-based baseline with supervised machine-learning models. On the available anonymized starter slice, the baseline achieved ROC AUC of 0.627 and Precision@50 of 0.240, while the random forest reached ROC AUC of 0.750 and Precision@50 of 0.740 on the same client-holdout validation design. These results indicate that a learned ranking approach can provide a more concentrated review queue than the fixed baseline in that tested slice, while reason codes can help connect model scores to editorial review actions. The results are directional and limited to the starter dataset and its proxy label, so they should not be interpreted as evidence that refreshing a page will cause traffic growth or as a demonstration of Google's ranking algorithm.

## Introduction / Problem

Content teams often have more pages that could potentially be reviewed than the time available for editors, SEO specialists, or content managers to inspect them individually. A useful system therefore does not need to automatically decide what should happen to every page; instead, it can help answer a narrower operational question:

**Which pages deserve manual review first?**

This project focuses on content refresh opportunity scoring. The goal is to combine observable performance and content signals into a ranked review queue that helps a human reviewer decide where to spend attention first.

The decision supported by the system is:

> **Which content pages should enter the manual review queue first?**

Possible editorial outcomes after review include refreshing existing content, improving metadata, expanding useful sections, monitoring the page, or leaving the page unchanged.

This framing is consistent with the FlyRank ML track's emphasis on opportunity scoring and connecting model output to specific actions rather than producing dashboards without a decision attached.

The system is therefore designed as **decision support**, not autonomous content optimization. A high score means that a page matches observed characteristics associated with the chosen review target; it does not mean that a refresh is guaranteed to improve performance.

---

## Data

### Dataset release

The analysis is based on the FlyRank pseudonymized warehouse release:

**`flyrank_pseudonymized_warehouse_release_v20260703`**

The release is described as a public-safe, pseudonymized warehouse containing observable search and engagement signals rather than FlyRank's internal product decisions. The release was exported on July 3, 2026, with the freshest three days intentionally excluded because very recent observations can be incomplete. 

The warehouse contains:

| Table                            | Approx. rows | Grain                                  | Main use                                     |
| -------------------------------- | -----------: | -------------------------------------- | -------------------------------------------- |
| `dim_clients`                    |          104 | One row per pseudonymized client       | Client grouping and history checks           |
| `dim_content`                    |      519,606 | One row per pseudonymized content item | Content metadata and joins                   |
| `fact_content_daily_performance` |   78,835,655 | Daily × client × content               | Time-series features, trends, and validation |
| `fact_content_query_90d`         |    2,414,248 | Client × content × query hash          | Query-mix features                           |

The daily performance table covers **January 27, 2025 through June 30, 2026**. The panel is unbalanced because different clients have different amounts of tracking history. 

### Data windows

For exploratory development, the internship guidance recommends using a middle month such as **March 2026** rather than repeatedly querying the full warehouse. The final month should be treated carefully because it represents the most recent period and can create future-information leakage when the target concerns subsequent performance. 

The March 2026 warehouse partition used during the data-contract work contained approximately **9.84 million daily rows covering March 1–31, 2026**.

### Exclusions

The analysis excludes:

* client names and domains;
* raw URLs;
* raw search queries and keywords;
* private identifiers;
* credentials;
* product decision flags and scores;
* future-window information when constructing leakage-safe features.

The warehouse intentionally contains pseudonymized identifiers rather than identifying client information. FlyRank's guidance also explicitly warns against using product decisions such as `health_score`, `priority_score`, or `action_type` as model features because doing so would allow a model to simply reproduce an existing product decision. 

### Public-safe treatment

Only pseudonymized identifiers, aggregated measurements, safe derived metrics, charts, and generic editorial recommendations should appear in the public research paper. No client names, domains, URLs, private queries, titles, or credentials should be published. 

---

## Methodology

### Research question

> **Which content pages should be prioritized for manual review based on observed search visibility, traffic, engagement, freshness, and content signals?**

### Features

Candidate features include observable measurements such as:

* search impressions;
* clicks;
* CTR;
* average search position;
* sessions;
* content age;
* freshness;
* word count;
* engagement rate;
* scroll rate;
* search-volume and competition signals where available.

The feature set should only contain information that would be available at the decision point. Future performance measurements must not be included.

### Label / proxy

The starter workflow uses:

`is_declining_label = trend_direction == "down"`

This is a **proxy label**, not a future causal outcome. It represents whether the page is categorized as declining within the available observation window. The internship guidance explicitly identifies this as a beginner proxy and recommends a stronger future-looking design for a more rigorous capstone, such as:

> prior 90-day features → decline during the following 30 days.



Therefore, the current starter result should be described as a benchmark for the scoring workflow rather than as proof of future refresh success.

### Baseline

The transparent baseline combines four components:

```text
baseline_refresh_score =
    0.40 × visibility_score
  + 0.30 × freshness_risk_score
  + 0.25 × position_opportunity_score
  + 0.05 × depth_gap_score
```

The baseline also produces human-readable reason codes such as:

* `stale_visible_page`
* `declining_with_demand`
* `thin_visible_page`
* `page_one_decay_risk`
* `low_ctr_visible_page`
* `low_engagement_visible_page`

These reason codes are important because a ranking system should provide an explanation for why a page appears in the review queue. 

### Machine-learning models

The starter workflow evaluates several supervised approaches:

* Logistic Regression
* Decision Tree
* Random Forest

The purpose is not to choose the most complex model automatically. The model should earn its additional complexity by improving the ranking decision enough to justify it.

### Leakage controls

The analysis treats leakage as a major risk.

The following information should not be used as ordinary model features:

* future target-window measurements;
* target-derived fields;
* existing FlyRank product decision scores;
* action flags that already encode the desired answer.

The internship guidance specifically warns that using an existing decision such as a `priority_score` or `action_type` as a feature can produce a circular result where the model merely learns to copy an existing rule. 

### Validation

The existing starter benchmark uses **client-holdout validation**, keeping complete clients outside the training data so that the model is evaluated on clients it did not see during training.

For the stronger warehouse capstone, a **time-aware or grouped/time-aware validation design** should be used when constructing a future-looking target. This is especially important because pages from the same client can otherwise appear in both training and validation data, and because future performance information can leak into features.

For a ranking decision, **Precision@K** is particularly useful. For example, Precision@50 asks:

> Of the 50 pages placed at the top of the review queue, how many actually match the selected positive label?

The internship guidance recommends ranking metrics such as Precision@20 or Precision@50 when they match the real review capacity. 

---

## Results

### Starter benchmark

The previously verified starter-model results are:

| Method              |   ROC AUC | Average Precision | Precision@50 |
| ------------------- | --------: | ----------------: | -----------: |
| Baseline rules      |     0.627 |             0.468 |        0.240 |
| Logistic Regression |     0.700 |             0.522 |        0.400 |
| Decision Tree       |     0.742 |             0.575 |        0.540 |
| Random Forest       | **0.750** |         **0.618** |    **0.740** |



The baseline Precision@50 of 0.240 corresponds to approximately **12 positive pages among the top 50**, while the random forest Precision@50 of 0.740 corresponds to approximately **37 positive pages among the top 50** under the selected proxy label.

This means that, in the tested starter slice, the learned model produced a more concentrated ranking than the transparent baseline.

However, these results come from a **30,000-row anonymized starter slice**, not the full approximately 79-million-row daily warehouse. The validation design was client-holdout, and the result therefore should be presented as a starter benchmark rather than as a final claim about the complete warehouse. 

### Recommended charts

The final capstone should include at least:

**Chart 1 — Baseline vs Model Precision@50**

A bar chart comparing:

* baseline: 0.240
* logistic regression: 0.400
* decision tree: 0.540
* random forest: 0.740

**Chart 2 — Baseline vs Model ROC AUC**

Compare the four methods using the values above.

**Chart 3 — Precision@K**

Plot precision at different queue sizes, for example:

* Precision@10
* Precision@20
* Precision@50
* Precision@100

This directly connects model performance to editorial capacity.

**Chart 4 — Top-20 Reason-Code Distribution**

Show which reason codes occur most frequently among the highest-ranked review candidates.

The final warehouse capstone should replace illustrative values with the actual metrics produced by the executed capstone notebook.

### Honest interpretation

The available benchmark suggests that machine learning can improve ranking concentration relative to the tested fixed rule. The random forest's Precision@50 was 0.740 compared with 0.240 for the baseline in this benchmark.

The result does **not** establish that the random forest will perform equally well on the full warehouse or on future data. It also does not establish that pages identified by the model will improve after being refreshed.

The model answers a narrower question:

> **Can observed page characteristics help produce a more useful review queue than the tested baseline?**

---

## Limitations & Honest Framing

This analysis should use the following language throughout:

* **Observed** — the data records what happened in the available measurement period.
* **Measured** — metrics such as impressions, clicks, CTR, sessions, and position are measurements available in the dataset.
* **Directional** — model findings indicate patterns worth investigating rather than universal rules.
* **Decision-support** — the output helps humans prioritize review; it does not automatically determine the correct editorial action.

The dataset has several limitations.

First, the daily panel is unbalanced. Clients have different amounts of historical data, and only some clients have sufficiently long histories for meaningful seasonal analysis. 

Second, not every metric is available for every daily row. The warehouse contains substantially more rows with search impressions than rows with clicks, sessions, scroll events, or AI sessions. Minimum-volume filters are therefore important when evaluating CTR and engagement signals. 

Third, the starter label is based on `trend_direction == "down"` and is therefore a proxy rather than a clean future outcome. A future-window target would provide a stronger test of whether the model can prioritize pages that subsequently decline.

Fourth, this analysis is observational. **It does not prove that a content refresh causes traffic growth, ranking improvement, or recovery.** Demonstrating that would require an appropriate experiment or causal design.

Finally, the analysis **does not prove how Google's ranking algorithm works**. The signals are measurements associated with observed search performance in this dataset, not confirmed Google ranking factors.

---

## Ranked Recommendations

The final system should produce a ranked queue rather than a generic list of SEO suggestions.

Each row should contain:

| Rank | Content ID       |       Score | Reason code | Recommended action |
| ---- | ---------------- | ----------: | ----------- | ------------------ |
| 1    | Pseudonymized ID | Model score | Reason code | Manual review      |
| 2    | Pseudonymized ID | Model score | Reason code | Refresh / expand   |
| 3    | Pseudonymized ID | Model score | Reason code | Metadata review    |
| ...  | ...              |         ... | ...         | ...                |

### Priority queue logic

Pages should be prioritized using:

1. model probability or opportunity score;
2. supporting search-volume evidence;
3. freshness or age context;
4. CTR/position context;
5. engagement evidence;
6. confidence and minimum-volume requirements.

The ranking should not automatically translate into a mandatory refresh.

### Reason codes

Examples include:

* **Model decline risk** — model probability exceeds the selected threshold.
* **Visible model opportunity** — sufficient impressions combined with elevated model probability.
* **CTR review candidate** — meaningful impressions, reasonable position, but comparatively low CTR.
* **Engagement review candidate** — sufficient sessions with weaker engagement signals.
* **Stale visible page** — older content with meaningful search visibility.
* **Declining with demand** — observed decline combined with meaningful impressions.

These reason codes allow a reviewer to understand **why a page entered the queue** instead of receiving only an unexplained score. 

### Editorial actions

The score should support, rather than replace, human review:

**High-priority review**
→ inspect content quality, intent alignment, freshness, metadata, and competing coverage.

**CTR opportunity**
→ review title/snippet alignment and search-result presentation.

**Engagement opportunity**
→ review content structure, clarity, usefulness, and page experience.

**Stale visible page**
→ check whether the information remains current and whether an update is justified.

**Low-confidence candidate**
→ monitor rather than immediately allocate editorial effort.

The important distinction is that the model identifies **pages worth reviewing**, not pages guaranteed to benefit from a particular action.

---

## Reproducibility

The project should be reproducible from the GitHub repository and executed notebooks.

### Repository

**GitHub:**
`https://github.com/Alpeshmore/flyrank-ml-internship`

### Key notebooks

* `work/notebooks/w03_data_contract.ipynb`
* `work/notebooks/w04_signal_audit.ipynb`
* `work/notebooks/w04_baseline_score.ipynb`
* `work/notebooks/w05_model.ipynb`
* `work/notebooks/w06_validation_audit.ipynb`
* `work/notebooks/w07_action_playbook.ipynb`
* `work/notebooks/capstone.ipynb`

The capstone notebook should contain the final executed pipeline from data preparation through validation, ranking, reason codes, charts, and final recommendations.

The repository should also contain:

`submission/paper_url.txt`

with exactly one line containing the deployed research-paper URL.

FlyRank's capstone guidance defines the final artifact as the deployed research paper plus the repository containing the reproducible notebooks and paper URL. 

## Conclusion

This project frames content refresh as a prioritization problem rather than an automatic optimization problem. Observable search, traffic, engagement, freshness, and content signals can be combined into a ranked review queue, while a transparent baseline provides a clear reference point for evaluating whether machine learning adds value. The available starter benchmark shows stronger ranking metrics for the tested random forest than for the fixed baseline, particularly on Precision@50. The next step for the full capstone is to reproduce this comparison on the warehouse using a clearly defined future-looking target and leakage-safe grouped/time-aware validation. The final output should remain a public-safe, evidence-based decision-support system that helps content teams decide where to investigate first without claiming that the model proves causality or Google's ranking mechanisms.



## 9. Acknowledgments & data credit

Built on the FlyRank ML Internship dataset.

https://flyrank.ai


**Important:** before submitting, replace the three `[record ...]` placeholders in Section 8 with the actual value
