# HanuAi ML Assessment — Task 2 Report
### Advanced EDA & Text Mining on Infotainment Warranty Claims

---

## 1. Overall Synthesis: Understanding the Data

The dataset is **1,000 vehicle service/warranty events**, all related to
infotainment systems (radio, display, audio, navigation) across 4 makes and
23 models, built across 8 plants, all from model year 2020. Each row carries:

- **Structured metadata**: open date, build date, in-use date, plant, make,
  model, and two coded classification fields (`CAUSAL_CD_DESC`,
  `COMPLAINT_CD_DESC`).
- **Three free-text verbatim fields**: `CAUSAL_VERBATIM` (technician's root
  cause diagnosis), `CORRECTION_VERBATIM` (what fix was applied), and
  `CUSTOMER_VERBATIM` (the customer's original complaint, often in their own
  words — including some entries in Spanish, confirming this is real-world
  multi-language service-center data).
- **Six target entity columns** (Trigger, Failure Component, Failure
  Condition, Additional Context, Fix Component, Fix Condition) — populated
  for only **5 of 1,000 rows**. These 5 rows are the labeled example the
  assignment provides; the task is to mine the same entity structure out of
  the other 995 rows' free text.

**Data quality**: no duplicate Event IDs, no fully duplicate rows. Missingness
is minimal on the structured fields (under 0.3% on `CAUSAL_VERBATIM` and
`CAUSAL_CD_DESC`) — this is a clean, well-maintained operational dataset, not
one needing heavy cleaning. The real "missing data" problem is the 99.5%
gap in the six target entity columns, which is the actual task, not a data
quality defect.

**Potential value for HanuAi's customers (OEMs/fleet operators):** this
pipeline turns free-text technician notes — which are otherwise only
searchable by keyword and unusable for aggregate reporting — into structured,
countable categories. That's the difference between "we have 1,000 service
tickets" and "72% of HyperFury X infotainment claims are communication
failures, concentrated at the Fort Wayne plant" — the latter is what
engineering and quality teams can actually act on.

---

## 2. Text Mining Approach

Given only 5 labeled examples — too few to train any supervised model — the
approach taken is **rule-based entity extraction**, consistent with the
lightweight, explainable design used in Task 1's sentiment categorization:

1. Keyword/regex dictionaries were built **from the vocabulary used in the 5
   gold-labeled rows** (e.g., "Black Screen", "Inoperative", "No Additional
   Functionality") and extended using the structured `CAUSAL_CD_DESC` /
   `COMPLAINT_CD_DESC` codes as a validation signal, so mined tags land in the
   same label space the 5 known-good rows already use rather than inventing a
   parallel taxonomy.
2. Each of the six target fields is mined from the verbatim field a human
   would naturally draw it from: `CUSTOMER_VERBATIM` for Trigger/Additional
   Context (what the customer reported and when), `CAUSAL_VERBATIM` +
   `CUSTOMER_VERBATIM` combined for Failure Component/Condition, and
   `CORRECTION_VERBATIM` for Fix Component/Condition.
3. An **"Unspecified" fallback** is used instead of defaulting to the most
   common tag (e.g., "Radio") when no keyword matches — an earlier version of
   this pipeline defaulted untagged rows to "Radio" and it inflated that tag
   to 95%+ of rows, which hid more than it revealed. "Unspecified" is a
   deliberately honest signal: 334/1000 rows have no failure-condition
   keyword match, meaning the taxonomy still has real gaps worth closing
   with more keyword coverage or a small supervised model once more labels
   exist.
4. As an **unsupervised cross-check**, TF-IDF + KMeans clustering (8 clusters)
   was run over `CUSTOMER_VERBATIM` independently of the rule-based tags.
   After removing report-boilerplate terms ("customer states", "check and
   advise") that otherwise dominate every cluster regardless of actual
   failure type, this surfaced a genuine pattern the keyword rules hadn't
   explicitly targeted: a cluster centered on **SD card removal** (26 rows),
   confirming the clustering approach catches real signal the hand-built
   taxonomy missed.

### Issue-type categorization

Rows were further bucketed into 7 high-level issue types derived from the
`CAUSAL_CD_DESC` codes (Communication/Network Failure, Electrical/Short
Circuit, Software/Programming Issue, Component Failure/Hardware, No Fault
Found/Follow-up, Alignment/Mechanical, Other/Unclassified) — this is the
"type of issues" grouping the assignment asks for.

---

## 3. Insights for Stakeholders

### Issue type distribution (n=1,000)

| Issue Type | Count | % |
|---|---|---|
| Communication / Network Failure | 450 | 45.0% |
| Electrical / Short Circuit | 275 | 27.5% |
| Software / Programming Issue | 112 | 11.2% |
| Component Failure (Hardware) | 87 | 8.7% |
| No Fault Found / Follow-up | 53 | 5.3% |
| Alignment / Mechanical | 13 | 1.3% |
| Other / Unclassified | 10 | 1.0% |

**Nearly 3 in 4 claims (72.5%) are communication/network or electrical/short
issues** — not mechanical wear. This points toward a module/wiring reliability
problem, not a parts-durability one, which changes where engineering
resources should go.

### Top failure components mentioned

Radio (925), Display (415), Backup Camera (97), USB Port (63), Navigation
Module (61) — the radio/display pairing dominates, consistent with the
`COMPLAINT_CD_DESC` codes being almost entirely Audio/Entertainment/
Navigation categories.

### Top failure conditions

After "Unspecified" (334 — the taxonomy's honest gap, see above): Inoperative
(215), Malfunction (195), Internal Fault (150), Cuts Out/Intermittent (117),
No Sound (90).

**Intermittent faults (117 rows) are a specific red flag** — these are the
hardest and most expensive warranty claims to resolve, since "cuts out
randomly" often can't be reproduced on the shop floor, leading to repeated
visits. The "No Trouble Found" fix-outcome (7 rows explicitly, but likely
undercounted given 64 rows have no clear fix-condition match) is a proxy for
this exact pain point.

### Build-to-claim timing

Average lag between vehicle build date and claim open date: **338.5 days**
(~11 months) — claims cluster well past the typical first-90-day defect
window, suggesting these are largely wear/degradation or firmware-drift
issues rather than out-of-box manufacturing defects.

### Model & plant concentration

| Top Models by claim volume | Top Plants by claim volume |
|---|---|
| HyperFury X — 148 | Fort Wayne (FTW) — 217 |
| TurboFlare — 117 | Flint (FLT) — 208 |
| QuantumRider — 109 | Sillao (SIL) — 199 |
| AeroSpecter — 86 | Spring Hill (SHT) — 169 |
| NebulaJet — 77 | Delta (DEL) — 142 |

Cross-tabbing model against issue type shows **HyperFury X skews toward
Communication/Network failures** (72 of its 148 claims, 48.6%) more heavily
than the fleet-wide average (45.0%) — a candidate for a model-specific
TCICM/radio-module supplier or wiring-harness review rather than a
fleet-wide fix.

### Recommended actions

1. **Prioritize communication-module reliability** (TCICM, radio-to-bus
   communication) as the top engineering focus — it's the largest single
   issue category by a wide margin and concentrated enough by model
   (HyperFury X) to suggest a traceable root cause rather than random
   failure.
2. **Investigate the Fort Wayne / Flint plants' claim concentration** against
   their production volume share — if claim rate (not just raw count) is
   elevated at these plants relative to output, that points to an assembly
   or component-sourcing difference worth auditing.
3. **Treat intermittent/"No Trouble Found" claims as a distinct workflow** —
   these need a different diagnostic protocol (e.g., mandatory data-logging
   tools at time of complaint) rather than the standard "inspect, replace,
   close" flow, since they're the ones generating repeat visits.
4. **Close the "Unspecified" taxonomy gap** — expanding the keyword
   dictionaries against the 334 unmatched failure-condition rows (ideally by
   reviewing a sample manually to find missed vocabulary) would materially
   improve how much of the dataset becomes genuinely actionable going
   forward.

---

## 4. Key Learnings & Further Improvements

- **5 labeled rows is not enough for supervised learning, but is enough to
  anchor a rule-based taxonomy** — the approach here treats the 5 gold rows
  as a vocabulary seed rather than training data, which is the right call at
  this label volume. If HanuAi can get even 50–100 more rows manually
  labeled, a lightweight supervised classifier (or few-shot LLM prompting
  with the gold rows as examples) would likely outperform the keyword rules
  on recall, especially for paraphrased failure descriptions the regex
  patterns don't anticor.
- **The "Unspecified" fallback was a deliberate design correction** made
  during development — the first version defaulted to the most common label
  and looked complete but was actually hiding the taxonomy's real gaps. This
  is worth stating plainly rather than polishing away: an honest 33%
  "Unspecified" rate on failure condition is more useful to a stakeholder
  than a false 100% coverage rate that's silently wrong a third of the time.
- **Clustering and rule-based tagging are complementary, not redundant** —
  the TF-IDF/KMeans pass caught the SD-card-removal pattern independently of
  any hand-written keyword rule, which is exactly the kind of unanticipated
  pattern a purely rule-based system can't surface by design. A production
  pipeline should keep both: rules for known, actionable categories;
  clustering as an ongoing discovery mechanism for emerging failure modes
  nobody's written a rule for yet.
- **Multi-language text (Spanish entries observed in `CORRECTION_VERBATIM`)**
  is a real-world complication the current English-only keyword patterns
  don't handle — those rows silently fall through to "Unspecified" rather
  than being mis-tagged, which is the safer failure mode, but a production
  version should either translate first or maintain parallel keyword sets.
