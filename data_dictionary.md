# Data Dictionary & Inference Logic

**Rakamin × Medika Nusantara — Workforce Intelligence PoC**
*Companion document to `dataset_A_messy.csv` and `dataset_B_clean.csv`*

---

## 1. The two datasets

| Dataset | Rows | Represents |
|---|---|---|
| `dataset_A_messy.csv` | 215 | Raw extract from Medika's 4 accessible source systems (HRIS, ATS, LMS, Performance). Payroll is off-limits per Day 30 discovery — `payroll_id` is blank for all rows. The 15 extra rows beyond 200 are **duplicate-ish** records: same employees appearing twice across systems with slight variations (different IDs, name typos, conflicting dates) because identity has not been resolved. |
| `dataset_B_clean.csv` | 200 | The same 200 unique employees after the contextualization layer has run: identities resolved to a canonical `employee_id`, job titles normalized to a taxonomy, skills inferred from training and role priors with confidence + provenance, performance ratings normalized to a 1–5 scale, and records flagged for human review where appropriate. |

Both datasets describe **the same 200 underlying employees**. The diff between A and B is the entire value proposition of Rakamin's data layer.

---

## 2. Why Dataset A looks the way it does

The brief calls for "enterprise reality" messiness. Each item below maps to a specific line of the brief and the realistic source of that mess in a real Indonesian healthcare-distribution company:

| Mess in Dataset A | Real-world source |
|---|---|
| **47% of records have empty `skills_raw`** | HR rarely re-enters skills in HRIS after hire; skills sit in resumes (ATS) and never sync. Brief target: 40%+. |
| **Job titles wildly inconsistent** (e.g., `Senior Sales`, `Sales 2`, `Sr. Medical Rep`, `Senior Medical Representative` — same role) | 90 branches each created their own conventions. There is no enforced taxonomy in SuccessFactors. |
| **Date formats vary** (`2024-05-08`, `24/01/2021`, `06/07/1986`, `21-10-1972`) | Different branches hire admins who use Excel locale defaults; HRIS accepts free-text dates. |
| **15 duplicate-ish rows** — same person appears twice with different IDs | An employee was hired through ATS (candidate_391), got an HRIS record only after onboarding, and the link was never made. Or they were rehired post-resignation and got a second candidate ID. |
| **Different IDs per system** (`EMP-2938` / `candidate_918` / `lms-name@…` / `perf_7920`) | Each system was bought separately over years; nobody invested in a shared identity service. |
| **Conflicting `start_date_hris` vs `start_date_ats`** | Contract-conversion dates differ from offer-acceptance dates; nobody reconciles. |
| **Performance scales differ by branch** (`numeric_5` / `letter` / `descriptive` / `percentage`) | The custom internal performance tool was deployed differently per branch — a known issue called out in the brief. |
| **22% of perf ratings are blank** | Some employees have <1 year tenure or were on leave during the rating cycle; some branches simply skipped that year. |
| **Trainings_count occasionally disagrees with `trainings_completed` length** | Manual data entry; nobody noticed. |
| **20% missing ATS IDs** | Internal hires (transfers, promotions) never went through ATS. |
| **15% missing LMS IDs** | Some employees never logged into Moodle (drivers, certain field roles). |
| **No `payroll_id` column at all** | The Day-30 Oracle privacy-policy ruling means payroll is never extracted. We remove the column from the schema rather than carrying it as 100% blank — a real ETL pipeline would not waste a column on a system it has been told not to access. |

**The "82%-but-feels-wrong" failure mode is encoded into Dataset A.** Train an attrition model on this data and it will memorize spurious patterns (e.g., "perf_history_years = 0" looking like a churn signal when it actually means "employee is too new to have data") — exactly the trap the case warns about.

---

## 3. Contextualization logic — how Dataset B is produced

The contextualization layer is the IP that Rakamin reuses across every client. It's structured as four sub-services. Each one adds a column or two to the final clean record.

### 3.1 Identity Resolution Service

**Goal:** Assign a single canonical `employee_id` per unique person. Alias all source-system IDs to it.

**Method:**
1. **Anchor on HRIS** as the source-of-truth spine. SAP `EMP-XXXX` is canonical.
2. **Block** candidate matches by `(branch_code, role_family)` to keep comparison tractable.
3. **Score** each candidate pair using a weighted combination of:
   - Full name fuzzy match (Levenshtein distance, normalized) — 35%
   - Date of birth exact match — 25%
   - Email match (when present) — 20%
   - Start date within ±90 days — 10%
   - Role family match — 10%
4. **Decide:**
   - Score ≥ 0.95 → auto-merge (Tier 1)
   - 0.70 ≤ score < 0.95 → flag for human review with side-by-side diff (Tier 2)
   - Score < 0.70 → keep separate (Tier 3)

**Output column:** `identity_resolution_confidence` (0.0–0.99). `systems_matched` ("3/4" means we resolved this employee across 3 of 4 accessible systems — payroll is excluded from the denominator since it's off-limits).

**Honest failure mode:** Common Indonesian names (e.g., "Ahmad Pratama") with similar DOBs are the hardest case. For those we never auto-merge; we always route to human review.

### 3.2 Standardization Layer

**Goal:** Normalize messy job titles and performance scales to a single taxonomy.

**Method — job titles:**
- Maintained internally: a base **healthcare-distribution role taxonomy** (11 canonical roles, e.g., `Medical Representative — Senior`).
- Each canonical role has a list of known variants (e.g., `Sales 2`, `Sr. Medical Rep`, `Senior Medical Representative` all map to `Medical Representative — Senior`).
- Lookup is exact-match first, then fuzzy match (>0.85 similarity) → if still no match, route to human review and add to taxonomy.

**Method — performance scales:**
- We map every scale to a 1–5 numeric:
  - `letter` (A→5, B→4, C→3, D→2, E→1)
  - `descriptive` (Excellent→5 … Poor→1)
  - `percentage` (≥90→5, ≥80→4, ≥70→3, ≥60→2, else 1)
  - `numeric_5` → as-is

**Output columns:** `role_normalized`, `role_family`, `performance_rating_normalized`, `performance_confidence`.

### 3.3 Skill Inference Cascade

**Goal:** Fill the 40%+ missing skill fields without making things up.

**Cascade (each step runs only if the previous step did not yield a skill):**

| Step | Source | Confidence range | Code in `skill_sources` |
|---|---|---|---|
| 1 | Explicit `skills_raw` field | 0.92 – 0.99 | `EXPLICIT` |
| 2 | Training history (Moodle) — each completed course maps to one or more skills | 0.78 – 0.92 | `TRAINING` |
| 3 | Role priors — if the employee has fewer than 3 skills after Steps 1–2, fill from the canonical role's core skill set | 0.55 – 0.72 | `ROLE_PRIOR` |
| 4 | (Future) Performance review text NLP — extract skill mentions from free-text review comments | 0.65 – 0.85 | `REVIEW_NLP` *(stubbed in PoC, not used in Dataset B)* |
| 5 | Insufficient data | — | Skill is **not** added; record is flagged for human review |

**Why this order:** Explicit > behavioral signal (training completion) > role-based prior. We never use a lower-confidence source to override a higher-confidence one.

**Why we cap at role priors:** Without this, employees with no training records and no explicit skills would have an empty skill profile, which would silently break downstream models. Role priors give us a defensible floor with low confidence — and the low confidence is what triggers a human review flag, so the user always sees that this is a soft inference, not ground truth.

**Output columns (three parallel lists, position-aligned):**

| Column | Format | Example |
|---|---|---|
| `skills` | semicolon-separated skill names | `Negotiation; B2B Sales; Pharmaceutical Product Knowledge; Customer Relationship Management` |
| `skill_sources` | semicolon-separated source codes | `EXPLICIT; EXPLICIT; ROLE_PRIOR; ROLE_PRIOR` |
| `skill_confidences` | semicolon-separated decimals (0.0–0.99) | `0.99; 0.98; 0.56; 0.67` |

To recover *which* training a `TRAINING`-tagged skill came from, the prototype joins against the `trainings_completed` column from Dataset A and the `TRAINING_TO_SKILL` mapping. Keeping the source detail out of the CSV keeps the file scannable in Excel.

Plus aggregates: `skill_count`, `avg_skill_confidence`.

### 3.4 Quality & Review Layer

**Goal:** Tell the user how much they should trust each record.

**Method:**

`data_quality_score` = mean of four components, each 0–1:
1. Identity resolution confidence
2. Average skill confidence (or 0.5 if no skills)
3. Performance confidence (or 0.5 if no rating)
4. Performance history depth (1.0 if ≥2 years, else 0.6)

**Human review flag (`needs_human_review = yes`)** triggered if any of:
- Identity confidence < 0.80
- Average skill confidence < 0.65 (when skills exist)
- Missing performance data despite tenure > 1 year
- Overall `data_quality_score < 0.70`

`review_reasons` is a semicolon-separated list of which conditions tripped.

**Why the threshold is 0.80 / 0.65 / 0.70:** These are deliberately conservative for the PoC. In the live system we would tune them per use case (an attrition model would tolerate lower data quality than a compensation decision).

---

## 4. What this dataset is for

The prototype loads both datasets and shows them side-by-side feeding the **Reskilling Recommendation Engine**. The argument the prototype makes:

1. With Dataset A, the recommender either (a) refuses to predict for the 47% with no skills, or (b) recommends generic training that wastes budget.
2. With Dataset B, every employee gets a recommendation tagged with a confidence band. Low-confidence recommendations are visibly flagged so the HR manager knows which ones to verify.
3. The 30% of employees flagged for human review in Dataset B are not failures — they're **the product** demonstrating its honesty. Showing "we don't know" beats hallucinating.

This is the literal embodiment of Rakamin's core thesis: *"the gap isn't the tool, it's the data layer underneath."*
