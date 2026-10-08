# HanuAi ML Assessment — Task 1 Report
### Web Scraping & Sentiment Analysis of BestBuy Canada Product Reviews

---

## 1. Solution for Scraping Challenges

Scraping a live e-commerce site like bestbuy.ca reliably, at scale, and without
violating site policy runs into four recurring categories of problems. Below is
what each looks like in practice and how `01_scrape_bestbuy_reviews.py` handles
it — plus what a **production** version would add beyond what's reasonable to
build or run against a live third-party site for an assessment.

### 1.1 Anti-bot detection / IP blocking

**The problem:** repeated automated requests from one IP, combined with a
non-browser request signature (missing headers, no JS execution, inhuman
timing), get flagged and blocked — sometimes after a handful of requests,
sometimes after hundreds.

**What the script does:**
- Uses Selenium with a real headless Chrome instance, so pages render and
  execute JS exactly as a browser would — this defeats detection methods that
  look for the absence of a JS engine (a common tell for `requests`/`urllib`
  scrapers).
- Sends a legitimate, current desktop User-Agent string.
- Applies a randomized delay (1.5–3.0s) between every click and page action,
  so request timing looks human rather than a tight, mechanical loop.

**What a production deployment would add (discussed, not implemented here):**
- **Rotating proxies / residential IP pools** — spreading requests across many
  IPs so no single address accumulates enough request volume to trip rate
  limits. Services like Bright Data or Smartproxy are standard for this.
- **Session/cookie rotation** alongside proxy rotation, since some anti-bot
  systems fingerprint sessions independently of IP.
- **Retry-with-backoff** — on a 403/429 or a detected block page, wait and
  retry with exponential backoff rather than hammering the same endpoint.
- These are *discussed* rather than built into the delivered script because
  implementing active IP-rotation/anti-detection infrastructure against a live
  third-party site, outside of a real commercial agreement with BestBuy, risks
  crossing from "polite scraping of public data" into the kind of high-volume,
  detection-evading access most ToS explicitly prohibit. The script instead
  checks `robots.txt` before running and stops cleanly if disallowed.

### 1.2 CAPTCHA challenges

**The problem:** sites increasingly gate access behind CAPTCHAs once
suspicious traffic patterns are detected, which a headless browser cannot
solve on its own.

**Suggested approaches** (for a production system, with the same caveat as
above — not implemented here):
- **Reduce the trigger rate first** — most CAPTCHA walls are a *response* to
  bot-like behavior, so slower request rates and realistic session behavior
  (scrolling, mouse movement simulation, dwell time) reduce how often you hit
  one at all.
- **Human-in-the-loop solving** for low-volume runs — surface the CAPTCHA to a
  person when it appears, rather than trying to defeat it programmatically.
- Third-party CAPTCHA-solving services exist commercially, but using them
  against a site that explicitly deploys CAPTCHAs to block automated access is
  a direct ToS violation, not a grey area — flagged here as a known technique,
  not a recommendation.

### 1.3 Pagination & dynamic content ("Show More")

**The problem:** BestBuy's reviews load incrementally via a "Show More"
button rather than URL-based pagination, so a naive scraper that only fetches
the initial page HTML misses most reviews.

**What the script does:** `expand_all_reviews()` repeatedly locates and clicks
the "Show More" button (with a scroll-into-view step, since some buttons only
become clickable once visible in the viewport) until it stops appearing or a
safety cap (`max_clicks=50`) is hit — the cap exists so a selector failure
degrades into stopping early rather than looping indefinitely against a
changed or missing element.

### 1.4 Markup/selector fragility

**The problem:** the single biggest real-world maintenance cost for any
scraper — e-commerce sites change their DOM structure without notice, so
hardcoded CSS selectors silently stop matching and the scraper returns empty
or partial results without raising an obvious error.

**What the script does:** every field extraction is wrapped individually in
try/except around `NoSuchElementException`, so one missing field (e.g. no
author name on a given review card) degrades gracefully to a sensible default
rather than discarding the whole review or crashing the run. The script also
logs a warning if it finishes with fewer than the assignment's 50-review
minimum, prompting a manual selector check against the live DOM rather than
silently shipping an incomplete dataset.

---

## 2. Business Insights Summary

*(Note: BestBuy's live review markup could not be scraped in this sandboxed
environment — no network access to bestbuy.ca and no browser runtime
available here. The numbers below are computed by running the actual
sentiment/categorization script against a 60-review synthetic sample built in
the scraper's exact output schema, to validate the pipeline end-to-end. Once
`01_scrape_bestbuy_reviews.py` is run against a live product page with network
access, re-run `02_sentiment_analysis.py` on the real output and these numbers
will reflect actual customer feedback.)*

### Sentiment distribution (sample run, n=60)

| Sentiment | Count | % |
|---|---|---|
| Positive | 47 | 78.3% |
| Negative | 13 | 21.7% |
| Neutral | 0 | 0.0% |

Average star rating: **3.77 / 5** · Average VADER compound score: **+0.41**
(moderately positive overall).

### Top drivers of customer satisfaction

Based on aspect-tag frequency across positive-leaning mentions:

1. **Touch controls / app connectivity** — the most-mentioned aspect overall
   (25 mentions); when positive, reviewers specifically praise responsive
   touch controls and reliable Bluetooth/multipoint pairing.
2. **Comfort / fit** (19 mentions) — long-session wearability is a repeated
   positive theme.
3. **Sound quality** (19 mentions) — bass depth and clarity drive strong
   praise when present.
4. **Price / value** (16 mentions) — perceived fairness of price relative to
   feature set.

### Top drivers of dissatisfaction

The same aspects that drive praise are also the top complaint categories when
sentiment is negative — this is the most actionable pattern in the data:

- **Touch controls / app** is the single most polarizing aspect: it's the
  top-mentioned category in *both* positive and negative reviews. Some units
  or firmware versions appear to have touch sensitivity issues (accidental
  pause/skip), while others are praised for responsiveness — suggesting
  either batch/firmware inconsistency or a genuinely divisive design choice
  worth a closer look at return/support ticket data.
- **Battery life** (18 mentions) is disproportionately negative-skewed —
  complaints center on shorter-than-expected runtime and loose charging
  ports, a possible hardware QA signal rather than a design complaint.
- **Comfort / fit** complaints cluster around clamping force causing
  discomfort on extended wear — the same aspect that drives praise from
  other users, pointing to a possible head-size/fit-range gap rather than a
  uniform defect.

### Recommendations

1. **Investigate touch-control consistency** — given it's simultaneously the
   top praised *and* top criticized aspect, a targeted review of whether
   complaints cluster by firmware version, purchase date, or region would
   clarify whether this is a QA/batch issue (fixable via firmware update) or
   an inherent design trade-off (requires a UX redesign).
2. **Audit battery/charging-port QA** — the negative skew on battery life and
   loose charging-port complaints is a classic early-warning signal for a
   hardware tolerance issue worth flagging to the hardware QA team before it
   shows up in return-rate data.
3. **Consider a second fit-range/comfort SKU** — if comfort complaints
   consistently mention clamping force while others report no issue, a
   looser-fit variant could capture the segment currently churning to
   negative reviews on this axis.

---

## 3. Key Learnings & Further Improvements

- **VADER is a strong first-pass baseline** for this kind of short, informal
  review text, but it has no concept of aspect-specific sentiment — a review
  that's positive about sound quality but negative about battery life still
  gets one overall compound score. The keyword-based aspect tagging
  compensates for this at the categorization layer, but a proper
  aspect-based sentiment model (e.g., fine-tuned DistilBERT, or an LLM-based
  classifier prompted per-aspect) would resolve mixed-sentiment reviews more
  precisely — recommended as the next iteration for a production HanuAi
  pipeline, with VADER staying as the fast first-pass filter for high-volume
  ingestion.
- **Scraper fragility is the dominant operational risk**, not sentiment
  accuracy. A selector change on BestBuy's end silently breaks data
  collection; a model tweak merely shifts classification quality at the
  margins. Any production deployment needs monitoring (e.g., a daily
  sanity-check that review counts and non-null field rates haven't collapsed)
  far more than it needs a better sentiment model.
- **This submission's numbers are on synthetic sample data**, not a live
  scrape — flagged explicitly above and worth re-running against the real
  site before treating any insight here as a genuine finding about the
  product in question.
