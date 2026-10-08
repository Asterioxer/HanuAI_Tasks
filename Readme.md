# HanuAi ML Assessment — Submission

This repository contains the complete submission for both tasks of the HanuAi
ML Assessment: web scraping + sentiment analysis (Task 1), and advanced EDA +
text mining (Task 2).

## Deliverables — mapped to the assignment brief

The assignment asks for exactly **3 deliverables per task** (script as
`.ipynb`, scraped/tagged data as Excel/CSV, and a report/doc) — 6 total.
Here's where each one is:

| # | Deliverable | File |
|---|---|---|
| 1 | Task 1 script (.ipynb) | [`Task1_combined.ipynb`](./Task1_combined.ipynb) |
| 2 | Task 1 Excel/CSV output | [`task1_sample_reviews_sentiment.csv`](./task1_sample_reviews_sentiment.csv) *(sample data — see warning above)* |
| 3 | Task 1 report/doc | [`Task1_Report.md`](./Task1_Report.md) |
| 4 | Task 2 script (.ipynb) | [`task2_eda_textmining.ipynb`](./task2_eda_textmining.ipynb) |
| 5 | Task 2 Excel/CSV output | [`task2_tagged_output.csv`](./task2_tagged_output.csv) |
| 6 | Task 2 report/doc | [`Task2_Report.md`](./Task2_Report.md) |

**Supporting files** (not separately required, kept for reference/transparency):

| File | What it is |
|---|---|
| `01_scrape_bestbuy_reviews.py` | Plain-Python source the scraping half of `Task1_combined.ipynb` was built from |
| `02_sentiment_analysis.py` | Plain-Python source the sentiment half of `Task1_combined.ipynb` was built from |
| `task2_eda_textmining.py` | Plain-Python source `task2_eda_textmining.ipynb` was built from |
| `task1_sample_reviews_raw.csv` | Pre-sentiment scraped-schema sample (input to the sentiment stage) |
| `task2_insights_summary.json` | Machine-readable version of the stats cited in `Task2_Report.md` |

A standalone presentation deck was **not** built — the assignment lists
"documentation/presentation to stakeholders" as an evaluation criterion, not
a separate deliverable; both `*_Report.md` files are written to serve that
stakeholder-facing role directly (see each report's structure below).

---

## Task 1 — Web Scraping & Sentiment Analysis

**`Task1_combined.ipynb`** runs in two parts, in order, in one notebook:

- **Part A (scraping):** Selenium-based scraper against a BestBuy Canada
  product page. Applies all 5 required sort filters (Most Helpful, Newest,
  Highest Rating, Lowest Rating, Most Relevant), paginates via "Show More",
  extracts the 7 required fields per review (primary key, title, text, date,
  rating, source, reviewer name), de-duplicates across sort passes, and
  checks `robots.txt` before running — it stops rather than scraping if
  disallowed.
- **Part B (sentiment):** VADER sentiment scoring + keyword-based aspect
  categorization (e.g. `['Battery Life (Neg)', 'Touch Controls / App (Pos)']`),
  matching the assignment's example tagging format. Runs on whatever Part A
  just produced; falls back to a previously saved CSV if Part A didn't run
  (e.g. no browser available) — this is exactly the path that was exercised
  for this submission.

**`Task1_Report.md`** covers, per the assignment's required structure:
scraping-challenge mitigations (anti-bot handling, CAPTCHA, pagination,
selector fragility) with a clear split between what's implemented vs. what's
discussed-but-not-built (and why), plus a stakeholder insights summary
(sentiment distribution, top satisfaction/dissatisfaction drivers,
recommendations) computed on the sample data, flagged as such throughout.

## Task 2 — Advanced EDA & Text Mining

**`task2_eda_textmining.ipynb`** runs against the full 1,000-row warranty/
service-event dataset (infotainment complaints across 4 makes, 23 models,
8 plants):

- **EDA:** missingness profile, duplicate check, build-to-claim lag
  distribution, make/model/plant breakdown.
- **Text mining:** rule-based entity extraction for all 6 target fields
  (Trigger, Failure Component, Failure Condition, Additional Context, Fix
  Component, Fix Condition) — keyword dictionaries built from the 5
  gold-labeled example rows and cross-validated against the dataset's coded
  classification fields, so mined tags land in the same label space the
  known-good rows use.
- **Issue-type categorization:** 7-way bucketing (Communication/Network,
  Electrical/Short, Software, Hardware, No Fault Found, Mechanical, Other)
  derived from the causal codes.
- **Clustering cross-check:** TF-IDF + KMeans over customer verbatims,
  independent of the rule-based tags — surfaced at least one genuine pattern
  (SD-card-removal complaints) the hand-built keyword rules hadn't targeted.

**`task2_eda_textmining.ipynb`'s output** is `task2_tagged_output.csv` (all
1,000 rows with mined entities, issue type, and cluster assignment) and
`task2_insights_summary.json` (the aggregate stats cited in the report).

**`Task2_Report.md`** covers, per the assignment's required structure: a
synthesis of the data and approach, stakeholder-facing insights (issue-type
distribution, top failure components/conditions, model/plant concentration,
recommended actions), and key learnings — including an explicit note on a
design correction made mid-build (an early version defaulted unmatched rows
to the most common tag, which looked complete but was hiding real gaps; the
fix was an honest "Unspecified" fallback instead).

---

## How both pipelines were validated

Every script here was actually executed, not just read through — this caught
real bugs a pure code review would have missed:

- Task 2's component-tagging first version defaulted everything unmatched to
  "Radio," inflating it to 95%+ of rows and burying the real signal; fixed to
  an honest "Unspecified" fallback.
- Converting Task 1's scripts to a single notebook initially split a
  `@dataclass` decorator onto a different cell than the class it decorates
  (silently breaking the dataclass if run top-to-bottom) and dropped a
  `pandas` import the merged notebook still needed — both caught by running
  the notebook end-to-end, not by inspection, and both fixed.
- Task 1's merged notebook was executed in full in this environment: the
  scrape cell correctly hit `robots.txt`'s disallow rule and stopped itself
  (rather than pushing through), its exception was caught cleanly, and the
  sentiment stage's CSV fallback ran correctly on the sample data — proving
  the two halves connect properly even though live scraping couldn't be
  exercised here.
