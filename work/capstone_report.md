# Capstone Report  Refresh / Content Opportunity Scoring

- **Author:** Muhammad Usman Rizwan
- **Lane:** Refresh / Content Opportunity Scoring
- **Repo:** https://github.com/UsmanRizwan20/week1
- **Date:** September 7, 2026

## 0. Abstract

This project asks which content items in a client's site are declining, recovering, or worth a
review action within a given month, to support prioritization of an editor's content-refresh
queue. Using FlyRank's search-performance warehouse (month=2026-03), features were built from
the first half of the month (impressions, clicks, CTR, position, active days) and a decline
label from the second half. A hand-written CTR-vs-position rule was compared against a logistic
regression model using a client-grouped 5-fold split to avoid memorization. The rule scored
AUC 0.510 (indistinguishable from chance); the model reached AUC 0.559, a modest but real and
validated improvement. The main finding is that decline in this window is only weakly
predictable from these signals — a directional, decision-support result rather than a strong
predictive one, useful for ranking review priority rather than making confident individual calls.

## 1. Problem framing

The unit of analysis is one content item over one month. The output is a risk score per item,
ranked into an action queue (review_priority / monitor / no_action) that an editor can work
through. The decision it supports: where a human spends their next hour of content-refresh
time. A wrong call here costs an editor's time reviewing a page that wasn't actually declining,
or missing one that was — low stakes per item, which is why a directional ranking (not a
confident yes/no) is the appropriate output.

## 2. Data safety

Source: `fact_content_daily_performance` for `month=2026-03` (a mid-panel month, not the sealed
final month), joined to `dim_clients`/`dim_content` only for grouping keys (`client_hash_id`,
`content_hash_id`) — these are pseudonymous and never used as model features. Excluded:
`gsc_avg_position = 0` rows (this means "no data," not literal rank zero, so it's dropped from
position averages); any second-half-window column (`imp_second_half`) once the label was built,
since those define the label and would leak it back in as a feature; the `_sample` table, since
that is the sealed final month (2026-06) reserved for a one-time holdout check, never for
iteration. No client names, URLs, or raw queries appear anywhere in this repo.

## 3. Baseline

The baseline is a transparent, hand-written rule built in week 4: a content item is flagged
`low_ctr_visible_page` if it has meaningful search volume (`imp_first_half >= 100`), ranks well
enough that clicks should follow (`avg_position_first_half` between 1 and 20), and its CTR sits
below the average CTR for other pages in the same position tier. This is a fair comparison
because it uses the exact same first-half-only features as the model, with no fitted weights —
it's the "would a human's simple rule already catch this?" bar the model has to clear.

## 4. Model / analysis

Method: Logistic Regression, chosen because the target (`is_declining`, a yes/no label) is
observed and a readable linear model is the appropriate starting point before adding
complexity, per the project's model-selection guidance. Features (all computed from days 1–15
only): `imp_first_half`, `clicks_first_half`, `ctr_first_half`, `avg_position_first_half`,
`active_days_first_half`. Label: `is_declining` = 1 if impressions in days 16–end fall below
80% of impressions in days 1–15, else 0. Deliberately excluded from features: any raw
second-half column, since those define the label itself.

## 5. Evaluation

Split: 5-fold cross-validation grouped by `client_hash_id` (GroupKFold), not a random split —
a random split would let the model partially memorize a client's typical pages rather than
generalize to pages it hasn't seen. Base rate: 0.296 (about 3 in 10 items declined).

| | AUC |
|---|---|
| Base rate (majority-class reference) | 0.296 |
| Baseline rule (`low_ctr_visible_page`) | 0.510 |
| Logistic Regression | 0.559 |

The baseline rule performs at essentially chance level for this label — CTR-vs-position signal
and mid-month decline are largely unrelated. The model finds a modest, real signal beyond the
rule and beyond chance, but it is weak: an AUC of 0.559 means it ranks a random declining item
above a random non-declining item only slightly more often than a coin flip would.

**Leakage check:** adding `imp_second_half` (a second-half, label-window column) as a feature
pushed AUC to 1.000 — confirming both that this would be leakage and that the evaluation
harness correctly detects it. That feature was then removed; the honest AUC of 0.559 stands.

## 6. Interpretation

The model's coefficients showed no single feature dominating in a "too good to be true" way,
consistent with the leak test result — the 0.559 score reflects genuine, if weak, structure
rather than a hidden shortcut. In plain words: knowing a page's first-half volume, clicks, CTR,
position, and active days gives only a slight edge in guessing whether it will decline in the
back half of the month. This is itself a useful negative-leaning result: it suggests that
whatever drives mid-month decline in this dataset is not well captured by within-month search
performance alone, and that better features would likely need a longer history or signals
outside this table (e.g. seasonality, external demand).

## 7. Recommendation

Use the model's risk score to rank content items into an editor queue (`review_priority` above
0.6, `monitor` between 0.3–0.6, `no_action` below 0.3), attaching the `low_ctr_visible_page`
reason code where the baseline rule also fired, since that gives editors a concrete, explainable
reason alongside the score. Given the modest AUC, this ranking should be used as one input
among several for prioritization, not as a confident automated decision — an editor should
still sanity-check top-ranked items rather than act on the score alone.

## 8. Reproducibility

Environment: Python 3, `duckdb`, `huggingface_hub`, `scikit-learn`, `pandas`, `matplotlib`
(installed via the notebook's first cell). Random seed: `random_state=42` used in all
train/test operations. To reproduce: clone the repo, open `work/notebooks/capstone.ipynb` in
Colab, set an `HF_TOKEN` secret (Hugging Face read token with gated-repo access to
`FlyRank/internship-warehouse`), and Run All. The ranked queue is written to
`work/outputs/capstone_ranked_queue.csv` (not committed — regenerated each run) and the
comparison chart to `work/outputs/figures/model_vs_baseline.png` (committed).

## 9. Acknowledgments & data credit

Built on the FlyRank ML Internship dataset — [flyrank.ai](https://flyrank.ai)