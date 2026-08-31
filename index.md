
- **Author:*Abdullah shah*
- **Lane:*Refresh/Content Opportunity Scoring*
- **Repo:*https://github.com/shah833/Flyrank-ML-Internship*
- **Date:*30/8/2026*

## 0. Abstract

Publishing an article is just the first step, the real question is whether it keeps performing or starts to decline. This project identifies which pages are declining so they can be prioritized for review by an SEO content editor. Using FlyRank's content performance data, we compare a simple hand-written rule against a trained model to see which one flags declining pages more accurately. The trained model reached 62% precision on its top 50 picks, compared to 44% for the hand-written rule and 54% for a naive guess showing that combining several signals works better than relying on one. Agencies that manage articles for many clients at once, could use this to prioritize review work at scale instead of checking each client by hand.

## 1. Problem framing

The decision: which pages are declining and should be reviewed first?

Unit of analysis: one page.

Output: a ranked list of pages, ordered by how urgently each one needs review.

Action: an SEO content editor works through the list top to bottom, checking the highest-priority pages first.

A wrong call costs something either way. If a genuinely declining page gets missed, it keeps losing traffic. If a healthy page gets flagged by mistake, the editor wastes time reviewing something that didn't need it.

As a catalog grows to hundreds or thousands of pages across multiple clients, checking each one by hand isn't realistic, and a simple rule based on one signal misses too many real cases. This project tests whether a trained model can make this call more reliably than a hand-written rule.

## 2. Data safety

Data used: the starter dataset (content_refresh_anonymized.csv, 30,000 rows, one row per page) for both the baseline rule and the model, so the comparison is fair on same data for both. Earlier in the project, the bigger FlyRank warehouse table (fact_content_daily_performance, filtered to March 2026) was used to check the data could be trusted: no duplicate rows, the data really covered all of March, and a real gap found, only about 4% of rows had real engagement data, the rest were placeholders.

Columns excluded, and why: trend_direction and trend_pct are never used as features, since they're what the target is built from, using them would let the model see the answer instead of learning a real pattern.

Leakage risks considered: the main risk was trend_direction/trend_pct leaking into the model as features, since both are label-derived. This was avoided by excluding them entirely from the feature list. client_id and content_id are pseudonymous IDs, used only for grouping (keeping each client's pages together during the train/test split), never fed to the model as features.

Client-identifying details: no client names, URLs, or private query text appear anywhere in work/, only pseudonymous IDs, used for grouping only.

## 3. Baseline

The rule: A page is eligible for review if it has at least 80 impressions in the last 90 days and hasn't been updated in over 2 months. Among eligible pages, one is flagged if its click-through rate is at least 85% below the average for pages at the same search position. This threshold was picked by testing a few numbers (40%, 65%, 85%, 95%) and checking how many pages got flagged each time, lower numbers flagged way too many pages (52% at 40%), and going past 85% barely changed anything (30% flagged at 85%, 27% at 95%).

Why it's a fair comparison: the rule and the model are tested on the exact same data and the exact same metric, the same held-out test set of 6,163 pages, scored with Precision@50.

Its numbers: out of 8,252 eligible pages, the rule flagged 2,453 (about 30%). On the test set, it scored a Precision@50 of 0.44 (22 out of the top 50 were genuinely declining)which is actually below the 54% base rate, meaning a plain guess would have done better than this rule alone.

## 4. Model / analysis

Method, and why it fits: Random Forest. It was picked over Gradient Boosting because Gradient Boosting builds trees one after another, each fixing the last one's mistakes, which makes it easier to overfit without careful tuning. Random Forest builds many trees independently and averages their votes, which is safer for this model.

Feature list: avg_position, ctr, competition_level, impressions_90d, and content_type. The text columns (content_type, competition_level) were converted into one-hot encoding for model. Missing competition_level values (2,610 rows) were labeled "Unknown" instead of dropped or guessed.

Left out on purpose: trend_pct and trend_direction, since they define the target and using them as features would let the model see the answer instead of learning it. client_id and content_id are used only for grouping, never as features.

Target, in one sentence: whether a page is declining (trend_direction == "down") versus not declining (everything else grouped together), matching the baseline's simple flag/no-flag structure so the two can be fairly compared.

## 5. Evaluation

Split: grouped by client. Every page from one client stays entirely in either the training set or the test set, never split between both. A random split would let the model quietly learn a client's specific patterns during training, then get an unfairly easy time on that same client's other pages during testing, that wouldn't reflect how it performs on a genuinely new client. This gave 23,837 training rows and 6,163 test rows.

Metric: Precision@50 out of the top 50 pages each system ranks as most urgent, how many are actually declining? This matters because a reviewer would start from the top of the list, not read through everything at once. The base rate here is 54% (16,262 of 30,000 pages are declining), meaning a lazy "always guess declining" approach would already be right just over half the time.

Model vs baseline, same test set: the baseline scored 0.44 (22 of 50 correct), actually below the 54% base rate. The model scored 0.62 (31 of 50 correct)which is clearly ahead of both the baseline and the base rate.

What the errors look like: the baseline's mistakes trace back to relying on one signal (CTR) that turned out to be weak on its own. The model's errors were more spread out and harder to pin to one cause, but it still made real, meaningful use of signals the baseline ignored, mainly how many impressions a page had.

## 6. Interpretation

What the model found: impressions_90d was by far the most useful signal (importance score 0.041), followed by avg_position (0.019). ctr — the one signal the whole baseline rule was built on, barely mattered (0.001). Content type and competition level added almost nothing, and a couple even slightly hurt the model.

The surprise: the baseline rule leans entirely on CTR, but the model found CTR nearly useless on its own. This matches something already noticed while building the baseline, pages with zero CTR all scored identically until impressions were added as a tiebreaker to tell strong evidence apart from thin evidence. The model reached the same conclusion independently.

What this means: a well-understood "CTR alone isn't enough" is a real, useful finding, not a failure. It suggests the baseline rule may be leaning on the wrong signal, and a future version should weigh impressions more heavily.

## 7. Recommendation

What the output supports: a ranked list of pages, ordered by how urgently each one needs review, not a diagnosis of every possible problem. An SEO content editor would start at the top of the list and work down, since the top pages have the strongest evidence of being genuinely declining.

How an editor would use it tomorrow: pull the ranked list, review the top pages first, and use their own judgment on each one rather than treating the list as a final verdict. Since the current rule only detects one problem type (low CTR vs. peers), every flagged page from the baseline carries the same label, snippet_review so the editor should still check what's actually wrong before acting.

Confidence: moderate. The model (Precision@50 = 0.62) clearly beats both the baseline (0.44) and a naive guess (0.54), so it's the more reliable of the two systems. But it's built on a small set of features and hasn't been tested outside March 2026.

Limits: the system currently only flags one type of problem. Detecting other issues — like stale content or weak engagement — would need additional rules or model logic, which is a reasonable next step but outside what this version does.

## 8. Reproducibility

1) git clone https://github.com/shah833/Flyrank-ML-Internship
2) pip install -r requirements.txt
3) Run the notebooks in order: w03_data_contract.ipynb (verifies the warehouse data) → w04_baseline_score.ipynb (builds the baseline rule, writes work/outputs/baseline_action_score.csv) to w05_model.ipynb (trains the model and produces the comparison numbers)

Random seed: random_state=42, used for both the train/test split and the Random Forest model — re-running produces the same split and the same results every time.

Environment: standard packages already listed in requirements.txt (pandas, scikit-learn, duckdb) — no extra packages needed beyond what's already there.

Evaluation is checkable, not just claimed: the test set was held out using a grouped client split before any training happened, and the model was only scored on it once, after training. The exact code that builds this split lives in w05_model.ipynb, and the resulting numbers are saved as a committed receipt at work/outputs/w05_model_results.json so the "evaluated once, on unseen clients" claim can be checked directly from the repo, not just taken on faith.

## 9. Acknowledgments & data credit

Built on the FlyRank ML Internship dataset - [flyrank.ai](https://flyrank.ai)
