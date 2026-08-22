# Capstone Report — Refresh / Content Opportunity Scoring
- **Author:** Muhammad Ahmad Ishtiaq
- **Lane:** Refresh / Content Opportunity Scoring
- **Repo:** https://github.com/MuhammadAhmadIshtiaq/ml-internship-muhammadahmadishtiaq
- **Date:** 2026-08-22

> Copy this file to `work/capstone_report.md` and fill it in as you build. Sections 1–8
> mirror the Pass / Needs-Work rubric axes, so nothing here is optional. Sections 0 and 9
> are **paper sections**: your deployed research paper must carry both, and they're here so
> you never rebuild them from memory at ship time.

## 0. Abstract

This project asks which content pages are most likely to be declining, growing, or stable in search performance, and whether a ranked scoring model can prioritize them for review more effectively than a simple threshold rule. Using the FlyRank internship warehouse (`fact_content_daily_performance`, ~78.8M rows), January 2026 and March 2026 click, impression, and position data were aggregated per (client, content) pair to define a three-class label based on percentage change in clicks. A Random Forest classifier trained on January-only signals (clicks, impressions, average position, CTR) was compared against a naive majority-class baseline and a position-based heuristic, using a client-grouped train/test split to prevent leakage. The Random Forest achieved macro F1 = 0.305, beating both the naive baseline (0.266) and the heuristic (0.108), with the advantage confirmed stable across five different train/test splits (range: 0.284–0.354). The output is a ranked, reason-coded review queue intended to help a content strategist prioritize a limited number of pages for refresh, not replace human judgment.

## 1. Problem framing

**Decision supported:** Which content pages a strategist or SEO editor should prioritize for review and possible refresh, given limited time to review every page individually.

**Unit of analysis:** A (client, content) pair — one row per unique content item per client, aggregated over a calendar month window.

**Output:** A ranked priority list of pages, each assigned a predicted class (Declining / Growing / Stable), a decline-confidence score, a recommended action (Refresh / Monitor / Protect), and a plain-language reason code.

**Action a human takes:** A content strategist works down the ranked list, starting with the highest decline-confidence pages, and decides per page whether to refresh the content, leave it alone, or continue monitoring it next window.

**Cost of a wrong call:** Missing a real decline means a page keeps losing visibility unnoticed until the next review cycle. Flagging a stable or growing page as declining wastes editor review time on a page that didn't need it. Missed declines are treated as the costlier error, so scoring is designed to favor not missing real decliners over avoiding false alarms.

**Why data/ML helps here:** A simple threshold rule (e.g., "flag if clicks dropped more than 20%") is a reasonable starting point and is used here as the baseline. However, it can't weigh multiple signals — clicks, impressions, position, and CTR — together, and page behavior varies across content types and volume levels. A model earns its place only if it measurably beats this baseline on the same data and metric; that comparison is reported honestly in Section 5, including cases where the improvement is modest.

## 2. Data safety

**Data used:** FlyRank internship warehouse (Hugging Face, gated, build v20260703), specifically `fact_content_daily_performance` (grain: report_date × client_hash_id × content_hash_id, ~78.8M rows, partitioned by month) and `dim_clients` (104 rows, client-level access and history metadata).

**Date windows:** Development used a mid-panel window — January 2026 as the "before" period, March 2026 as the "after" period — rather than the final panel month (June 2026, matching the `_sample` table), which was treated as sealed/held-out and not touched during model development or threshold tuning.

**Columns deliberately excluded:**
- `trend_direction` and `trend_pct`-style precomputed fields were never used as features, since they are themselves derived from the same outcome being predicted (label-derived leakage risk).
- Rows before each client's GA4 data-start date were excluded from any GA4-dependent calculation, since GA4 columns are zero-filled (not genuinely zero) before that date.
- Pages with fewer than 10 January clicks were excluded entirely — with such small denominators, percentage change is dominated by noise (e.g., 1→3 clicks reads as "+200%" despite reflecting no meaningful signal). This dropped the eligible set from 41,101 to 8,557 (client, content) pairs.

**Leakage risks considered:**
- The label (`pct_change_clicks`, and the resulting Declining/Growing/Stable bucket) is computed using March data. The predictive model was deliberately restricted to January-only features (`jan_clicks`, `jan_impressions`, `jan_avg_position`, `jan_ctr`) so it never sees the outcome window it is predicting.
- `client_hash_id` and `content_hash_id` are pseudonymous identifiers used only for grouping (aggregation keys and train/test split grouping) — never used as model features.
- The train/test split is grouped by `client_hash_id` (GroupShuffleSplit), so no single client's content appears in both train and test, preventing client-specific leakage.

**Client-identifying data confirmation:** No client names, domains, URLs, private queries, credentials, or raw exports appear anywhere in `work/`. All identifiers in the notebook, charts, and outputs are pseudonymous hash IDs (e.g., `client_08a6a72ff48e62c0`, `content_1516ec061fe1f8a5`) as provided by the warehouse release.

## 3. Baseline

Two baselines were built, both using only January-window information (no March data), since the ±20% threshold rule that defines the label itself requires March data and cannot serve as a fair predictive baseline.

**Naive baseline:** Always predicts the majority class (Declining, 38.1% of the labeled set). This is the floor any real method must beat.
- Accuracy: 0.663 (misleading — driven entirely by a class-imbalanced test set)
- Macro F1: 0.266
- Per-class: 0% precision/recall on Growing and Stable — it never correctly identifies these classes.

**Heuristic baseline:** A simple rule using only `jan_avg_position` — position worse than 20 → predict Declining; position better than 8 → predict Growing; otherwise → predict Stable. No model fitting.
- Accuracy: 0.120
- Macro F1: 0.108
- This heuristic performed worse than random guessing across three classes (~33%), a genuine and reportable finding about how weakly position alone predicts future click direction in this data.

Both baselines are evaluated on the exact same client-grouped test split used for the model (Section 5), making the comparison fair.

## 4. Model / analysis

**Method:** Random Forest classifier (`n_estimators=200`, `max_depth=6`, `class_weight='balanced'`, `random_state=42`), chosen because it can combine multiple weakly-informative signals (clicks, impressions, position, CTR) non-linearly without requiring manual feature engineering of interactions, and handles the moderate class imbalance reasonably well when combined with `class_weight='balanced'`.

**Why it fits the lane:** Refresh/Content Opportunity Scoring calls for a ranked action engine with reason codes — Random Forest's `predict_proba` output supports ranking by decline-confidence, and its feature importances support the reason-code / interpretation layer (Section 6).

**Feature list (January-window only):**
- `jan_clicks` — total January clicks
- `jan_impressions` — total January impressions
- `jan_avg_position` — average January search position
- `jan_ctr` — January click-through rate (`jan_clicks / jan_impressions`)

**Deliberately excluded:** March-window values of any kind (leakage), `trend_direction`/`trend_pct` (label-derived), `client_hash_id`/`content_hash_id` (used for grouping only, not as features).

**Target/proxy definition (one sentence):** The label is a three-class bucket — Declining (≤ -20%), Growing (≥ +20%), or Stable (between) — based on the percentage change in total clicks from January 2026 to March 2026, restricted to pages with at least 10 January clicks.

## 5. Evaluation

**Split:** Grouped by `client_hash_id` using `GroupShuffleSplit` (80/20, `random_state=42`), so no client's content appears in both train and test. This is necessary because pages from the same client are likely to share systematic patterns (industry, site structure, historical performance), and a random row-level split would leak client-level information across the split. The design is also time-aware in the sense that no data from the sealed final month (June 2026) was used anywhere in training or evaluation.

**Result (single split, seed=42):** Train: 8,074 rows / 15 clients. Test: 483 rows / 4 clients. Client overlap: 0 (confirmed).

| Method | Accuracy | Macro F1 |
|---|---|---|
| Naive baseline (majority class) | 0.663 | 0.266 |
| Heuristic baseline (position rule) | 0.120 | 0.108 |
| Random Forest (Jan-only features) | 0.329 | **0.305** |

Accuracy is misleading here: the naive baseline's 66.3% accuracy comes entirely from exploiting a test set that happened to skew 66% Declining (vs. 38% in the full labeled set) — an artifact of which 4 clients landed in the test split given only 19 total eligible clients. Macro F1, which weights all three classes equally, is reported as the primary metric.

**Stability check:** To address the small-client-count risk directly, the Random Forest was re-evaluated across 5 different train/test splits (seeds 0, 1, 42, 7, 99). Macro F1 ranged **0.284–0.354**, consistently above both baselines in every split — confirming the model's advantage is not an artifact of one favorable split.

**Error analysis:** The Random Forest correctly identifies 60% of Growing pages and 26% of Stable pages (recall), categories the naive baseline gets 0% on. Its main error mode is under-predicting Declining pages (29% recall on Declining, precision 0.66) — it is conservative about calling a page "Declining," which, combined with the confidence-based Refresh/Monitor split in Section 7, is a reasonable trade-off: pages it is *confident* are declining (proba ≥ 0.5) are flagged Refresh; the rest fall to Monitor rather than being ignored.

## 6. Interpretation

**Feature importances (Random Forest, Gini-based):**

| Feature | Importance |
|---|---|
| jan_clicks | 0.320 |
| jan_ctr | 0.247 |
| jan_impressions | 0.219 |
| jan_avg_position | 0.214 |

No single feature dominates — the four importances sit within a fairly narrow band (21–32%), meaning the model relies on a *combined* pattern across click volume, CTR, impressions, and position rather than any one dominant signal. This is consistent with what the reason codes show in practice (Section 7): many flagged pages don't trip a single obvious threshold, and are instead flagged on the combined signal pattern.

**Surprise / negative result:** The position-based heuristic (worse position → predict Declining) performed *worse than random guessing* (12% accuracy vs. ~33% chance across three classes). This is a genuine, reportable negative finding: raw January search position alone is a poor predictor of which direction a page's clicks will move over the following two months in this dataset — position needs to be combined with volume and CTR signals to be useful, which is exactly what the model does and the heuristic doesn't.

**Class-level pattern:** The model is more confident and accurate identifying Growing pages than Declining ones, and it is conservative (rather than aggressive) about calling a page Declining — it under-flags rather than over-flags, which shapes how the recommendation confidence threshold in Section 7 was set.

## 7. Recommendation

Using the Random Forest's predicted probabilities, all 483 test-set pages were ranked by decline-confidence (`proba_Declining`, descending) and mapped to an action:

- **Refresh** (16 pages) — predicted Declining with ≥50% confidence. Top priority for content review.
- **Monitor** (246 pages) — predicted Declining with <50% confidence, or predicted Stable. Revisit next window rather than act immediately.
- **Protect** (221 pages) — predicted Growing. Recommendation is to leave alone; an unnecessary edit could disrupt something that is currently working.

Each page also carries a **reason code** comparing its January signals against the typical (median) values for confirmed decliners in the training data — e.g., "position worse than typical decliner (5.8)" or "CTR below typical decliner (1.1%)." Where no single factor stands out, the code states this explicitly ("flagged on combined signal pattern, no single dominant factor") rather than forcing a misleading single-cause explanation.

**How a FlyRank editor would use this tomorrow:** Start at the top of the Refresh list (16 pages) — this is a short, manageable queue for one review cycle. For each page, read the reason code, check the page briefly, and decide to update, rewrite, or leave as-is. Treat Monitor as a "check again next month" list, not an action list. Treat Protect as a do-not-touch list unless there is a strong external reason.

**Confidence and limits, stated explicitly:** This is a directional, decision-support tool with macro F1 = 0.305 — a real but modest edge over baseline. It should narrow a large page list to a small, prioritized review queue; it should not be used to make unreviewed automatic changes to any single page, and it says nothing about *why* the ranking algorithm behaves the way it does — only that certain January signals are associated with subsequent click direction.

Full ranked list: `work/outputs/priority_queue.csv`.

## 8. Reproducibility

**To re-run from a fresh clone:**
1. Clone the repo: `git clone https://github.com/MuhammadAhmadIshtiaq/ml-internship-muhammadahmadishtiaq`
2. Open `work/notebooks/capstone.ipynb` in Google Colab (badge link at the top of the notebook).
3. Run all cells top to bottom (Runtime → Run all). When prompted, paste a valid Hugging Face **read** token for the gated `FlyRank/internship-warehouse` dataset (never hardcode the token in a cell).
4. Outputs are written to `work/outputs/`: `priority_queue.csv` and four chart PNGs (`chart_baseline_vs_model.png`, `chart_label_distribution.png`, `chart_sensitivity.png`, `chart_confusion_matrix.png`).

**Random seeds used:**
- Train/test split (primary): `random_state=42`
- Random Forest: `random_state=42`
- Sensitivity re-splits: seeds `[0, 1, 42, 7, 99]`

**Environment:** Google Colab default Python 3 runtime. Key packages: `duckdb`, `huggingface_hub`, `scikit-learn`, `pandas`, `numpy`, `matplotlib`, `seaborn` (installed via `%pip install` cells in the notebook — no separate `requirements.txt` was needed since Colab's base image covers the rest).

**Sealed evaluation status:** The final panel month (June 2026 / `_sample`) was **not** used anywhere in this analysis — development and evaluation both use the Jan–March mid-panel window only. No sealed-month holdout claim is made in this report; if a future iteration evaluates against June, that would require a separate, clearly labeled cell and a committed metrics file, per the reproducibility standard above.

## 9. Acknowledgments & data credit

Built on the FlyRank ML Internship dataset — https://flyrank.ai

---

> **Claims checklist before submitting:** observed / measured / directional / decision-support
> **Metrics vs. base rate:** report your task's base rate (majority-class %) next to any
> precision@K or accuracy — a high score can just be a high base rate. AUC / lift over
> baseline are the honest discrimination numbers.
> language everywhere · no causal claims without an experiment or causal design · no
> "predicted Google's algorithm" · no client-identifying details · numbers in this report
> match a fresh re-run.
