# CheXpert Plus — Comprehensive Exploratory Data Analysis

> **Source notebooks:** `chexpert_plus_core_table_eda.ipynb`, `chexpert_plus_eda_updated.ipynb`, `chexpert_plus_findings_eda.ipynb`, `chexpert_plus_impression_eda.ipynb`, `chexpert_plus_report_eda.ipynb`
> **Dataset:** CheXpert Plus (Stanford AIMI, Chambon et al., 2024)
> **Paper:** [arXiv:2405.19538](https://arxiv.org/abs/2405.19538) | **DOI:** [10.57761/fzna-pm76](https://doi.org/10.57761/fzna-pm76) | **Platform:** [Stanford Redivis](https://stanford.redivis.com/datasets/5yyj-1a9f6ap0x?v=1.0)
> **Project Goal:** RAG-based radiology report generation system

---

## Table of Contents

1. [Dataset Overview](#1-dataset-overview)
2. [Schema & Column Reference](#2-schema--column-reference)
3. [Data Splits & Patient Leakage](#3-data-splits--patient-leakage)
4. [Imaging Metadata](#4-imaging-metadata)
5. [Patient & Study Hierarchy](#5-patient--study-hierarchy)
6. [Missingness & Data Quality](#6-missingness--data-quality)
7. [Report Section Analysis](#7-report-section-analysis)
8. [Patient Demographics](#8-patient-demographics)
9. [Clinical & Administrative Metadata](#9-clinical--administrative-metadata)
10. [Pathology Labels — Findings (`findings.csv`)](#10-pathology-labels--findings-findingscsv)
11. [Pathology Labels — Impression (`impression.csv`)](#11-pathology-labels--impression-impressioncsv)
12. [Pathology Labels — Full Report (`report.csv`)](#12-pathology-labels--full-report-reportcsv)
13. [Cross-Source Label Comparison](#13-cross-source-label-comparison)
14. [Label Co-occurrence](#14-label-co-occurrence)
15. [Report Structure Templates](#15-report-structure-templates)
16. [De-identification Notes](#16-de-identification-notes)
17. [RAG Design Implications](#17-rag-design-implications)

---

## 1. Dataset Overview

| Property | Value |
|---|---|
| **Table name** | `df_chexpert_plus_240401` |
| **Table ID** | `bavj-4x2exxtvq` (Stanford AIMI / Redivis v1.0) |
| **Total rows (image records)** | **223,462** |
| **Total columns** | **27** |
| **Unique patients** | **64,725** |
| **Unique studies** | **187,711** |
| **File size** | ~409.5 MB (CSV) / 422.3 MB in-memory |
| **Schema validation** | ✅ All 27 expected columns present, zero extras |
| **Duplicate image paths** | **0** |
| **Duplicate full rows** | **0** |
| **Exact duplicate reports** | **4** (2 boilerplate texts appearing 2× each) |

CheXpert Plus extends the original CheXpert dataset with full de-identified radiology reports (free text), structured section parsing, and rich patient-level demographics and administrative metadata. Labels are extracted using the CheXpert NLP labeler (Irvin et al., 2019) from three report sources: `findings`, `impression`, and the full `report`.

---

## 2. Schema & Column Reference

### 2.1 Column Groups

| Group | Columns |
|---|---|
| **Image paths** | `path_to_image`, `path_to_dcm` |
| **View metadata** | `frontal_lateral`, `ap_pa` |
| **Patient identifier** | `deid_patient_id`, `patient_report_date_order` |
| **Free-text report** | `report`, `section_narrative`, `section_clinical_history`, `section_history`, `section_comparison`, `section_technique`, `section_procedure_comments`, `section_findings`, `section_impression`, `section_end_of_impression`, `section_summary`, `section_accession_number` |
| **Demographics** | `age`, `sex`, `race`, `ethnicity` |
| **Administrative** | `interpreter_needed`, `insurance_type`, `recent_bmi`, `deceased` |
| **Dataset** | `split` |

### 2.2 Data Types

| Type | Count | Variables |
|---|---|---|
| **string / object** | 24 | All text, categorical, path, and identifier columns |
| **float64** | 2 | `age`, `recent_bmi` |
| **int64** | 1 | `patient_report_date_order` |

### 2.3 Image Path Structure

All image paths follow the regex pattern:
```
(?P<split>[^/]+)/patient(?P<patient>\d+)/study(?P<study>\d+)/view(?P<view>\d+)_(?P<orientation>frontal|lateral).jpg
```
- **Parsing success rate:** 100% (0 / 223,462 paths failed to parse)
- **Patient ID consistency check:** 0 mismatches between parsed ID and `deid_patient_id`
- **Split consistency check:** 0 mismatches between path-derived split and `split` column

---

## 3. Data Splits & Patient Leakage

### 3.1 Split Distribution

| Split | Image Records | Studies | Patients | % Records |
|---|---|---|---|---|
| **train** | 223,228 | 187,511 | 64,525 | **99.90%** |
| **valid** | 234 | 200 | 200 | **0.10%** |
| **test** | 0 | 0 | 0 | *(not in this table)* |
| **Total** | 223,462 | 187,711 | 64,725 | 100.00% |

> The official validation set contains **234 expert-annotated images** — a very small held-out set. Downstream projects should create a stratified patient-level train/val/test split from the training data.

### 3.2 Leakage Analysis

- **Patients appearing in > 1 split:** **0 / 64,725 (0.00%)**
- Train and validation sets are **strictly disjoint at the patient level**
- Path-derived split matches the `split` column in **100%** of all rows

---

## 4. Imaging Metadata

### 4.1 View Orientation (`frontal_lateral`)

| Orientation | Count | Percentage |
|---|---|---|
| **Frontal** | 191,071 | 85.50% |
| **Lateral** | 32,391 | 14.50% |
| Missing | 0 | 0.00% |

### 4.2 Radiographic Projection (`ap_pa`)

| Projection | Count | % All | % of Non-Null |
|---|---|---|---|
| **AP** (Anteroposterior) | 161,622 | 72.33% | 84.59% |
| **PA** (Posteroanterior) | 29,432 | 13.17% | 15.40% |
| LL (Lateral Left) | 16 | 0.01% | 0.01% |
| RL (Lateral Right) | 1 | 0.00% | 0.00% |
| **NaN** (all Lateral images) | 32,391 | 14.50% | — |

### 4.3 Frontal × Projection Cross-tabulation

| | AP | PA | LL | RL | NaN | **Total** |
|---|---|---|---|---|---|---|
| **Frontal** | 161,622 | 29,432 | 16 | 1 | 0 | **191,071** |
| **Lateral** | 0 | 0 | 0 | 0 | 32,391 | **32,391** |
| **Total** | 161,622 | 29,432 | 16 | 1 | 32,391 | **223,462** |

> All Lateral images have `NaN` for `ap_pa` by design. The 17 non-standard values (LL/RL) within frontal rows are rare anomalies.

---

## 5. Patient & Study Hierarchy

### 5.1 Summary Counts

| Level | Count |
|---|---|
| Total image records | 223,462 |
| Unique studies | 187,711 |
| Unique patients | 64,725 |
| Average images per study | 1.19 |
| Average studies per patient | 2.90 |

### 5.2 Images per Patient

| Statistic | Value |
|---|---|
| Count | 64,725 patients |
| Mean | **3.45** images |
| Std Dev | 4.65 |
| Min | 1 |
| Q1 (25%) | 1 |
| Median | 2 |
| Q3 (75%) | 4 |
| **Max** | **92** |

- Patients with only **1 study:** 22,692
- Patients with **> 10 studies:** 3,521
- Most-studied patient (`patient28746`): **92 records**

### 5.3 Images per Study

| Statistic | Value |
|---|---|
| Count | 187,711 studies |
| Mean | 1.19 |
| Std Dev | 0.42 |
| Median | 1 |
| Max | 3 |

| Views per Study | Studies | Percentage |
|---|---|---|
| 1 (single-view) | 153,738 | **81.90%** |
| 2 (two-view) | 32,195 | 17.15% |
| 3 (three-view) | 1,778 | 0.95% |

### 5.4 Longitudinal Study Order (`patient_report_date_order`)

| Statistic | Value |
|---|---|
| Mean | 5.36 |
| Std Dev | 7.42 |
| Min | 1 |
| Q1 | 1 |
| Median | 3 |
| Q3 | 6 |
| Max | 92 |

> `patient_report_date_order` provides a chronological ordering of each patient's encounters. Combined with `deid_patient_id`, this enables longitudinal modeling.

---

## 6. Missingness & Data Quality

### 6.1 Column-level Missing Values (all 27 columns)

| Column | Non-Null Count | Missing Count | Missing % |
|---|---|---|---|
| `section_technique` | 10,732 | 212,730 | **95.20%** |
| `section_procedure_comments` | 25,669 | 197,793 | **88.51%** |
| `section_history` | 38,192 | 185,270 | **82.91%** |
| `section_findings` | 59,469 | 163,993 | **73.39%** |
| `section_end_of_impression` | 73,738 | 149,724 | **67.00%** |
| `section_clinical_history` | 138,866 | 84,596 | **37.86%** |
| `recent_bmi` | 164,985 | 58,477 | **26.17%** |
| `section_summary` | 189,648 | 33,814 | **15.13%** |
| `ap_pa` | 191,071 | 32,391 | **14.50%** |
| `section_comparison` | 213,125 | 10,337 | **4.63%** |
| `age` | 223,184 | 278 | 0.12% |
| `section_impression` | 223,317 | 145 | 0.06% |
| `section_narrative` | 223,434 | 28 | 0.01% |
| *All other 14 columns* | 223,462 | 0 | **0.00%** |

> **Critical insight:** `section_findings` is absent in **73.4%** of records, while `section_impression` is present in **99.94%** — a fundamental structural characteristic of Stanford radiology report templates.

### 6.2 Trivially Short / Whitespace-Only Values

| Section | Whitespace-only count |
|---|---|
| `section_end_of_impression` | 68,768 (most common: `\n` appearing 63,362 times) |
| `section_narrative` | 5,526 |
| `section_findings` | 1,407 |
| `section_summary` | 692 |

### 6.3 Data Integrity

- **Physiologically implausible BMI (< 10 or > 80):** 41 rows (0.02% of non-null)
- **Age HIPAA top-coding:** Ages ≥ 89 are capped at 89. The **89-bucket** contains **9,518 rows (4.26%)**.
- **Report text structural integrity:** `section_findings` and `section_impression` match 100% inside the full `report` string (sampled verification).

---

## 7. Report Section Analysis

### 7.1 Section Availability (non-empty text present)

| Section | Records Present | Availability % |
|---|---|---|
| `report` | 223,462 | **100.00%** |
| `section_accession_number` | 223,462 | **100.00%** |
| `section_impression` | 223,296 | **99.93%** |
| `section_narrative` | 217,922 | 97.52% |
| `section_comparison` | 213,073 | 95.35% |
| `section_summary` | 189,648 | 84.87% |
| `section_clinical_history` | 138,860 | 62.14% |
| `section_findings` | 59,441 | **26.60%** |
| `section_history` | 38,191 | 17.09% |
| `section_procedure_comments` | 25,669 | 11.49% |
| `section_technique` | 10,732 | 4.80% |
| `section_end_of_impression` | 7,392 | 3.31% |

### 7.2 Text Length Statistics (Word Counts)

| Section | N Present | Mean | Median | P90 | P99 | Max | Mean Chars |
|---|---|---|---|---|---|---|---|
| `report` | 223,462 | 117.4 | 110 | 164 | 261 | **828** | 842 |
| `section_findings` | 59,469 | 60.8 | 54 | 104 | 187 | 509 | 429 |
| `section_impression` | 223,317 | 41.8 | 38 | 69 | 112 | 425 | 303 |
| `section_summary` | 189,648 | 16.3 | 19 | 28 | 30 | 325 | 120 |
| `section_technique` | 10,732 | 7.0 | 7 | 9 | 22 | 257 | 44 |
| `section_clinical_history` | 138,866 | 6.9 | 6 | 11 | 20 | 133 | 49 |
| `section_history` | 38,192 | 6.6 | 6 | 11 | 19 | 199 | 46 |
| `section_narrative` | 223,434 | 5.7 | 5 | 9 | 19 | 146 | 41 |
| `section_procedure_comments` | 25,669 | 5.4 | 5 | 5 | 17 | 94 | 32 |
| `section_comparison` | 213,125 | 2.8 | 1 | 7 | 14 | 225 | 23 |
| `section_end_of_impression` | 73,738 | 1.6 | 0 | 1 | 24 | 337 | 12 |

### 7.3 Token Count Statistics (tiktoken `cl100k_base`)

| Section | Mean Tokens | Median Tokens | P90 | P95 | P99 | Max | % > 256 | % > 512 |
|---|---|---|---|---|---|---|---|---|
| `report` | **245.6** | 233 | 328 | 375 | 498 | 1,464 | **34.2%** | 0.85% |
| `section_findings` | 102.6 | 90 | 174 | 215 | 313 | 760 | 2.4% | 0.05% |
| `section_impression` | 99.3 | 92 | 162 | 189 | 255 | 849 | 0.96% | 0.01% |

> Full reports average ~246 tokens. Over 34% exceed 256 tokens but only 0.85% exceed the 512-token context window — single-chunk indexing is viable for the vast majority.

---

## 8. Patient Demographics

### 8.1 Age Distribution

| Statistic | Value |
|---|---|
| Non-null count | 223,184 (missing: 278, **0.12%**) |
| Mean | **60.39 years** |
| Std Dev | 17.77 |
| Min | 0 |
| Q1 (25%) | 49 |
| **Median** | **62 years** |
| Q3 (75%) | 74 |
| Max | **89** *(HIPAA-capped)* |

**Age Group Breakdown:**

| Age Group | Count | Notes |
|---|---|---|
| 0–17 | 3 | Near-zero pediatric |
| 18–29 | 15,358 | |
| 30–44 | 27,123 | |
| 45–59 | 57,950 | |
| **60–74** | **68,353** | **Largest group** |
| 75+ | 54,397 | |

> Dataset is strongly skewed toward **older adults** (60–74 is the most represented age group). Pediatric data is negligible.

### 8.2 Sex Distribution

| Sex | Count | Percentage |
|---|---|---|
| **Male** | 132,482 | **59.29%** |
| Female | 90,701 | 40.59% |
| Unknown | 279 | 0.12% |

### 8.3 Race Distribution

| Race | Count | Percentage |
|---|---|---|
| **White** | 126,669 | **56.68%** |
| Other | 31,561 | 14.12% |
| Unknown | 25,370 | 11.35% |
| Asian | 23,719 | 10.61% |
| Black | 12,062 | 5.40% |
| Pacific Islander | 3,180 | 1.42% |
| Native American | 547 | 0.24% |
| Patient Refused | 354 | 0.16% |

> White patients account for over half the dataset. ~11.4% have unknown race, limiting demographic fairness analyses.

### 8.4 Ethnicity Distribution

| Ethnicity | Count | Percentage |
|---|---|---|
| **Non-Hispanic/Non-Latino** | 169,886 | **76.02%** |
| Hispanic/Latino | 28,222 | 12.63% |
| Unknown | 24,947 | 11.16% |
| Patient Refused | 407 | 0.18% |

### 8.5 Recent BMI

| Statistic | Value |
|---|---|
| Non-null count | 164,985 (missing: 58,477 = **26.17%**) |
| Mean | 26.73 kg/m² |
| Std Dev | 6.51 |
| Min | 0.00 *(anomaly/placeholder)* |
| Q1 | 22.30 |
| **Median** | **25.70** |
| Q3 | 30.00 |
| Max | 50.00 *(likely capped)* |
| Implausible values (< 10 or > 80) | 41 rows |

> Median BMI of 25.7 sits at the upper-normal boundary (overweight threshold: 25). The 26% missing rate limits BMI's utility as a consistent covariate.

---

## 9. Clinical & Administrative Metadata

### 9.1 Interpreter Needed

| Value | Count | Percentage |
|---|---|---|
| **No** | 165,937 | **74.26%** |
| Unknown | 39,191 | 17.54% |
| Yes | 18,334 | 8.20% |

### 9.2 Insurance Type

| Insurance | Count | Percentage |
|---|---|---|
| **Medicare** | 107,202 | **47.97%** |
| Private Insurance | 49,074 | 21.96% |
| Unknown | 41,244 | 18.46% |
| Medicaid | 21,499 | 9.62% |
| Other | 4,443 | 1.99% |

> Medicare dominates at 48%, consistent with the elderly skew in the age distribution. This reflects a primarily inpatient/older-adult hospital population.

### 9.3 Deceased Status

| Status | Count | Percentage |
|---|---|---|
| **No** | 132,037 | **59.09%** |
| Yes | 91,029 | **40.74%** |
| Unknown | 396 | 0.18% |

> ~41% mortality reflects longitudinal follow-up data — many patients have since died during the observation period. This serves as a **proxy for clinical severity**.

---

## 10. Pathology Labels — Findings (`findings.csv`)

> **Source:** Labels extracted by the CheXpert NLP labeler from `section_findings` only.
> **Shape:** 223,462 rows × 15 columns (1 path + 14 labels).
> **Coverage caveat:** `section_findings` is present in only **26.6%** of records — most "Not Mentioned" labels arise from section absence, not clinical absence.

**Label Encoding:** `1.0` = Positive | `0.0` = Negative | `-1.0` = Uncertain | `NaN` = Not Mentioned

### 10.1 Full Label Distribution

| Pathology | Positive (1.0) | Pos % | Negative (0.0) | Neg % | Uncertain (-1.0) | Unc % | Not Mentioned (NaN) | NaN % |
|---|---|---|---|---|---|---|---|---|
| **No Finding** | 167,933 | 75.15% | 0 | 0.00% | 0 | 0.00% | 55,529 | 24.85% |
| **Lung Opacity** | 32,358 | 14.48% | 1,488 | 0.67% | 105 | 0.05% | 189,511 | 84.81% |
| **Support Devices** | 31,710 | 14.19% | 377 | 0.17% | 1 | 0.00% | 191,374 | 85.64% |
| **Pleural Effusion** | 24,511 | 10.97% | 10,586 | 4.74% | 2,281 | 1.02% | 186,084 | 83.27% |
| **Edema** | 12,148 | 5.44% | 4,443 | 1.99% | 2,687 | 1.20% | 204,184 | 91.37% |
| **Cardiomegaly** | 10,971 | 4.91% | 11,261 | 5.04% | 1,554 | 0.70% | 199,676 | 89.36% |
| **Atelectasis** | 9,136 | 4.09% | 167 | 0.07% | 8,371 | 3.75% | 205,788 | 92.09% |
| **Pneumothorax** | 6,391 | 2.86% | 18,577 | 8.31% | 702 | 0.31% | 197,792 | 88.51% |
| **Consolidation** | 4,159 | 1.86% | 7,083 | 3.17% | 5,938 | 2.66% | 206,282 | 92.31% |
| **Enlarged Cardiomediastinum** | 4,130 | 1.85% | 12,928 | 5.79% | 6,909 | 3.09% | 199,495 | 89.27% |
| **Lung Lesion** | 3,490 | 1.56% | 329 | 0.15% | 459 | 0.21% | 219,184 | 98.09% |
| **Fracture** | 3,266 | 1.46% | 2,750 | 1.23% | 148 | 0.07% | 217,298 | 97.24% |
| **Pleural Other** | 1,663 | 0.74% | 13 | 0.01% | 759 | 0.34% | 221,027 | 98.91% |
| **Pneumonia** | 915 | 0.41% | 381 | 0.17% | 3,702 | 1.66% | 218,464 | 97.76% |

### 10.2 Class Imbalance Summary (Findings)

| Pathology | Pos % | Neg % | Pos/Neg Ratio |
|---|---|---|---|
| Pleural Other | 0.74% | 0.01% | **127.9×** |
| Support Devices | 14.19% | 0.17% | **84.1×** |
| Atelectasis | 4.09% | 0.07% | 54.7× |
| Lung Opacity | 14.48% | 0.67% | 21.75× |
| Lung Lesion | 1.56% | 0.15% | 10.6× |
| Pneumothorax | 2.86% | 8.31% | **0.34×** ← more negatives |
| Enlarged CM | 1.85% | 5.79% | **0.32×** ← more negatives |

### 10.3 Multi-label Statistics (Findings)

- Mean positive labels per row: **0.648** (std: 1.299)
- Median: 0 (75th percentile: 0; max: 8)
- No Finding = 1.0 with co-occurring positive pathology: **1,068 rows (0.64% contradiction rate)**

### 10.4 High-Uncertainty Labels (Findings)

| Pathology | Uncertain count | Uncertain % | vs. Positive count |
|---|---|---|---|
| Pneumonia | 3,702 | 1.66% | **4.05× more uncertain than positive** |
| Enlarged Cardiomediastinum | 6,909 | 3.09% | **1.67× more uncertain than positive** |
| Consolidation | 5,938 | 2.66% | **1.43× more uncertain than positive** |
| Atelectasis | 8,371 | 3.75% | ≈ 1:1 ratio with positive |

---

## 11. Pathology Labels — Impression (`impression.csv`)

> **Source:** Labels extracted from `section_impression` (present in 99.93% of records).
> **Shape:** 223,462 rows × 15 columns.
> **Note:** Far higher positive rates than findings labels — impression summarizes the entire study.

### 11.1 Full Label Distribution

| Pathology | Positive (1.0) | Pos % | Negative (0.0) | Neg % | Uncertain (-1.0) | Unc % | Not Mentioned (NaN) | NaN % |
|---|---|---|---|---|---|---|---|---|
| **Support Devices** | 115,891 | **51.86%** | 3,978 | 1.78% | 54 | 0.02% | 103,539 | 46.33% |
| **Lung Opacity** | 102,950 | **46.07%** | 5,081 | 2.27% | 330 | 0.15% | 115,101 | 51.51% |
| **Pleural Effusion** | 89,267 | **39.95%** | 36,290 | 16.24% | 7,614 | 3.41% | 90,291 | 40.41% |
| **Edema** | 53,011 | 23.72% | 21,229 | 9.50% | 12,217 | 5.47% | 137,005 | 61.31% |
| **Atelectasis** | 33,851 | 15.15% | 727 | 0.33% | 34,401 | 15.39% | 154,483 | 69.13% |
| **Cardiomegaly** | 30,558 | 13.67% | 16,160 | 7.23% | 3,921 | 1.75% | 172,823 | 77.34% |
| **No Finding** | 21,259 | 9.51% | 0 | 0.00% | 0 | 0.00% | 202,203 | 90.49% |
| **Pneumothorax** | 17,879 | 8.00% | 58,631 | 26.24% | 2,418 | 1.08% | 144,534 | 64.68% |
| **Consolidation** | 13,702 | 6.13% | 31,452 | 14.07% | 26,627 | 11.92% | 151,681 | 67.88% |
| **Lung Lesion** | 9,368 | 4.19% | 1,349 | 0.60% | 1,557 | 0.70% | 211,188 | 94.51% |
| **Fracture** | 8,706 | 3.90% | 3,927 | 1.76% | 494 | 0.22% | 210,335 | 94.13% |
| **Enlarged Cardiomediastinum** | 7,559 | 3.38% | 22,522 | 10.08% | 15,021 | 6.72% | 178,360 | 79.82% |
| **Pneumonia** | 4,847 | 2.17% | 3,322 | 1.49% | 19,367 | 8.67% | 195,926 | 87.68% |
| **Pleural Other** | 3,957 | 1.77% | 134 | 0.06% | 2,724 | 1.22% | 216,647 | 96.95% |

### 11.2 Multi-label Statistics (Impression)

- Mean positive labels per row: **2.20** (std: 1.34) — vs. 0.65 in findings
- Median: 2 | Q3: 3 | Max: 8
- No Finding = 1.0 with co-occurring positive pathology: **8,207 rows (38.60% inconsistency rate)**

### 11.3 Impression — Top Co-occurrences

| Label Pair | Co-occurrence count |
|---|---|
| Lung Opacity + Support Devices | **60,154** |
| Pleural Effusion + Support Devices | **55,946** |
| Lung Opacity + Pleural Effusion | **53,216** |
| Edema + Support Devices | **34,994** |
| Edema + Pleural Effusion | **27,426** |
| Lung Opacity + Edema | **27,205** |
| Atelectasis + Support Devices | **20,535** |

---

## 12. Pathology Labels — Full Report (`report.csv`)

> **Source:** Labels extracted from the complete `report` text field (all sections combined).
> **Shape:** 223,462 rows × 15 columns.
> **Highest positive rates** of the three sources — captures mentions from all sections.

### 12.1 Full Label Distribution

| Pathology | Positive (1.0) | Pos % | Negative (0.0) | Neg % | Uncertain (-1.0) | Unc % | Not Mentioned (NaN) | NaN % | Pos/Neg Ratio |
|---|---|---|---|---|---|---|---|---|---|
| **Support Devices** | 133,282 | **59.64%** | 2,279 | 1.02% | 34 | 0.02% | 87,867 | 39.32% | 58.5× |
| **Lung Opacity** | 110,580 | **49.48%** | 6,485 | 2.90% | 550 | 0.25% | 105,847 | 47.37% | 17.1× |
| **Pleural Effusion** | 94,076 | **42.10%** | 43,956 | 19.67% | 7,472 | 3.34% | 77,958 | 34.89% | 2.14× |
| **Edema** | 54,255 | 24.28% | 23,071 | 10.32% | 13,032 | 5.83% | 133,104 | 59.56% | 2.35× |
| **Atelectasis** | 38,283 | 17.13% | 745 | 0.33% | 35,267 | 15.78% | 149,167 | 66.75% | 51.4× |
| **Cardiomegaly** | 36,172 | 16.19% | 25,996 | 11.63% | 7,023 | 3.14% | 154,271 | 69.04% | 1.39× |
| **Pneumothorax** | 19,136 | 8.56% | 67,218 | 30.08% | 2,878 | 1.29% | 134,230 | 60.07% | 0.28× |
| **Consolidation** | 14,709 | 6.58% | 33,262 | 14.88% | 28,101 | 12.58% | 147,390 | 65.96% | 0.44× |
| **Lung Lesion** | 13,149 | 5.88% | 1,473 | 0.66% | 1,361 | 0.61% | 207,479 | 92.85% | 8.93× |
| **No Finding** | 13,053 | 5.84% | 0 | 0.00% | 0 | 0.00% | 210,409 | 94.16% | — |
| **Fracture** | 11,143 | 4.99% | 4,594 | 2.06% | 550 | 0.25% | 207,175 | 92.71% | 2.43× |
| **Enlarged Cardiomediastinum** | 9,141 | 4.09% | 32,721 | 14.64% | 19,327 | 8.65% | 162,273 | 72.62% | 0.28× |
| **Pneumonia** | 5,963 | 2.67% | 4,720 | 2.11% | 23,045 | 10.31% | 189,734 | 84.91% | 1.26× |
| **Pleural Other** | 5,760 | 2.58% | 50 | 0.02% | 2,971 | 1.33% | 214,681 | 96.07% | 115.2× |

### 12.2 Multi-label Statistics (Full Report)

- Mean positive labels per row: **2.44** (std: 1.42) — highest of all three sources
- Median: 3 | Q3: 3 | Max: 8
- No Finding = 1.0 with co-occurring positive pathology: **6,042 rows (46.29% inconsistency rate)**

### 12.3 Report — Top Co-occurrences

| Label Pair | Co-occurrence count |
|---|---|
| Lung Opacity + Support Devices | **73,742** |
| Pleural Effusion + Support Devices | **68,154** |
| Lung Opacity + Pleural Effusion | **59,751** |
| Edema + Support Devices | **41,137** |
| Edema + Lung Opacity | **30,563** |
| Edema + Pleural Effusion | **30,166** |
| Atelectasis + Support Devices | **26,648** |

---

## 13. Cross-Source Label Comparison

### 13.1 Positive Rates Across All Three Sources

| Pathology | Findings % | Impression % | Report % | Key Observation |
|---|---|---|---|---|
| **Support Devices** | 14.19% | 51.86% | **59.64%** | Often in narrative/technique only |
| **Lung Opacity** | 14.48% | 46.07% | **49.48%** | Dominates impression + report |
| **Pleural Effusion** | 10.97% | 39.95% | **42.10%** | High concordance imp ↔ report |
| **Edema** | 5.44% | 23.72% | **24.28%** | Nearly identical imp vs. report |
| **Atelectasis** | 4.09% | 15.15% | **17.13%** | High uncertainty across all sources |
| **Cardiomegaly** | 4.91% | 13.67% | **16.19%** | +5,614 mentions in full report |
| **Lung Lesion** | 1.56% | 4.19% | **5.88%** | Incidental; +40% in full report |
| **Fracture** | 1.46% | 3.90% | **4.99%** | Bone finding; +28% in full report |
| **No Finding** | **75.15%** | 9.51% | 5.84% | Driven by 73.4% findings absence |
| **Pneumonia** | 0.41% | 2.17% | 2.67% | High uncertainty (8-10%) all sources |

### 13.2 Root Cause: Section Sparsity

The dramatic difference in positive rates between `findings.csv` and `impression.csv`/`report.csv` is entirely explained by the fact that `section_findings` is absent in **73.4%** of records. When the findings section is blank, the NLP labeler records `No Finding = 1.0` or `NaN` for all pathologies, inflating the "No Finding" positive rate to 75%.

**Impact on Incidental Findings:**
- `Fracture`, `Lung Lesion`, and `Cardiomegaly` — chronic/anatomical findings often mentioned only in the narrative or findings section — are **28–40% more prevalent** in `report.csv` than `impression.csv`.
- This means `impression.csv` under-represents chronic/incidental pathologies while `report.csv` captures the most complete picture.

---

## 14. Label Co-occurrence

### 14.1 Findings Co-occurrence Matrix (Positive × Positive)

| | Enl.CM | Cardi. | Lung Op | Lung L | Edema | Consol | Pneum | Atel. | Pneumo | Pleur.Eff | Pleur.O | Fract | Supp.Dev |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| **Lung Opacity** | 2,460 | 5,989 | 32,358 | 1,178 | 7,384 | 1,535 | 663 | 4,478 | 3,650 | 16,348 | 846 | 1,564 | **19,514** |
| **Pleural Effusion** | 1,849 | 5,044 | 16,348 | 1,348 | 6,649 | 2,211 | 301 | 4,558 | 2,930 | 24,511 | 533 | 1,123 | 16,080 |
| **Support Devices** | 2,105 | 5,859 | 19,514 | 1,223 | 8,999 | 2,451 | 333 | 5,742 | 4,791 | 16,080 | 562 | 1,251 | 31,710 |
| **Cardiomegaly** | 922 | 10,971 | 5,989 | 342 | 3,983 | 696 | 127 | 1,551 | 488 | 5,044 | 202 | 458 | 5,859 |
| **Edema** | 1,096 | 3,983 | 7,384 | 254 | 12,148 | 814 | 129 | 1,983 | 687 | 6,649 | 110 | 388 | 8,999 |
| **Atelectasis** | 499 | 1,551 | 4,478 | 435 | 1,983 | 577 | 72 | 9,136 | 1,337 | 4,558 | 177 | 538 | 5,742 |
| **Pneumothorax** | 120 | 488 | 3,650 | 398 | 687 | 314 | 31 | 1,337 | 6,391 | 2,930 | 123 | 396 | 4,791 |

**Strongest clinical clusters (findings):**

| Pair | Count | Clinical interpretation |
|---|---|---|
| Lung Opacity + Support Devices | 19,514 | ICU / intubated patients |
| Lung Opacity + Pleural Effusion | 16,348 | Common comorbidity |
| Pleural Effusion + Support Devices | 16,080 | Critically ill patients |
| Edema + Support Devices | 8,999 | Congestive/cardiac ICU |
| Lung Opacity + Edema | 7,384 | Pulmonary edema cluster |
| Cardiomegaly + Support Devices | 5,859 | Cardiac failure patients |
| Pneumothorax + Support Devices | 4,791 | Post-procedure / trauma |

### 14.2 Longitudinal Patient Complexity

- **Findings:** Most complex patients exhibited up to **11 distinct positive pathologies** across all studies
- **Impression:** Most complex patients: up to **13 distinct positive pathologies** (patients `08813`, `22920`)
- **Report:** Most complex patients: up to **13 distinct positive pathologies** (patients `08549`, `30494`, `07632`)

---

## 15. Report Structure Templates

The top structural templates (by section combination present):

| Rank | Template | Count | % |
|---|---|---|---|
| 1 | `narrative, clinical_history, comparison, impression, summary` | **60,750** | 27.19% |
| 2 | `narrative, history, comparison, impression, summary` | 26,739 | 11.97% |
| 3 | `narrative, clinical_history, comparison, procedure_comments, findings, impression` | 25,397 | 11.37% |
| 4 | `narrative, clinical_history, comparison, impression, end_of_impression, summary` | 22,020 | 9.85% |
| 5 | `narrative, comparison, impression, end_of_impression, summary` | 20,057 | 8.98% |
| 6 | `narrative, comparison, impression, summary` | 10,347 | 4.63% |
| 7 | `narrative, clinical_history, comparison, findings, impression, summary` | 9,342 | 4.18% |
| 8 | `narrative, clinical_history, comparison, findings, impression, end_of_impression, summary` | 6,080 | 2.72% |
| 9 | `narrative, comparison, findings, impression, end_of_impression, summary` | 5,475 | 2.45% |
| 10 | `narrative, clinical_history, comparison, technique, impression, summary` | 3,667 | 1.64% |

> **Templates #1, 2, 4, 5, 6 have no `findings` section** — explaining why 73.4% of records lack `section_findings`.
> **`impression` appears in virtually all templates** and is the most reliable extraction target.
> `section_technique` and `section_comparison` are near-constant boilerplate when present — low NLP value.

---

## 16. De-identification Notes

CheXpert Plus reports were processed by Stanford's de-identification pipeline, removing/replacing approximately **1 million PHI spans** across the corpus.

| De-id artifact | Details |
|---|---|
| **Underscore placeholders** (`___`) | 4.19% of reports (9,363 rows) contain `___` runs |
| **MIMIC-style `[** **]`** | 0% — not used |
| **Angle-bracket `<ENTITY>`** | 0% — not used |
| **Anonymization notice** | All `section_accession_number` values contain: *"This report has been anonymized. All dates are offset from the actual dates by a fixed interval associated with this patient ID."* |
| **Date offsets** | All dates shifted by a patient-specific random offset — temporal ordering within a patient is preserved |
| **Age capping** | Ages > 89 capped at 89 per HIPAA Safe Harbor (affects ~9,518 rows, 4.26%) |

> **For embedding/RAG pipelines:** Collapse de-id placeholders to a single normalized token (e.g., `[DEID]`) before building a retrieval index to prevent false semantic similarity from placeholder strings.

---

## 17. RAG Design Implications

Based on comprehensive EDA findings, the following design decisions are recommended for a RAG-based radiology report generation system:

| Decision | Recommendation | Rationale from EDA |
|---|---|---|
| **Retrieval unit** | `section_findings` + `section_impression` (fall back to full `report`) | Impression has 99.93% coverage; findings has 26.6% |
| **Corpus scope** | `train` split only, after exact-duplicate removal | 4 exact duplicates identified; no leakage to validate |
| **Chunking** | Single chunk per report for most records | Mean 246 tokens; only 0.85% exceed 512 tokens |
| **Text normalization** | Collapse de-id placeholders to `[DEID]` | 4.19% of reports contain `___` placeholder runs |
| **Excluded fields** | `section_technique`, `section_comparison` | Near-constant boilerplate; keep as metadata, not embeddings |
| **Image-to-text key** | Parse `patient_id/study_id/view_id` from `path_to_image` | 100% parseable; zero path integrity failures |
| **View policy** | Decide frontal-only vs. all-views | 85.5% frontal; lateral adds 14.5% (different content) |
| **Fairness evaluation** | Stratify by `race`, `sex`, `insurance_type`, `interpreter_needed` | Significant demographic skew identified |
| **Leakage guard** | Verified — zero cross-split patient overlap | 0 / 64,725 patients appear in multiple splits |
| **Label source** | Prefer `report.csv` for highest recall; `impression.csv` for precision | Report captures 17–40% more positive labels for incidental findings |

---

## Summary Statistics Snapshot

| Metric | Value |
|---|---|
| Total image records | **223,462** |
| Unique patients | **64,725** |
| Unique studies | **187,711** |
| Train / Valid split | 223,228 / 234 |
| Cross-split patient leakage | **0** |
| Frontal images | 191,071 (85.5%) |
| AP frontal | 161,622 (72.3%) |
| Median patient age | **62 years** |
| Male patients | 59.3% |
| White race | 56.7% |
| Medicare insurance | 48.0% |
| Patients deceased | 40.7% |
| BMI missing | 26.2% |
| `section_impression` coverage | **99.93%** |
| `section_findings` coverage | **26.60%** |
| Mean full report word count | 117.4 words |
| Mean full report token count | 245.6 tokens (cl100k_base) |
| Reports > 512 tokens | 0.85% |
| Most common pathology (findings) | No Finding (75.15%) |
| Most common true pathology (findings) | Lung Opacity (14.5%), Support Devices (14.2%) |
| Most common pathology (report) | Support Devices (59.6%), Lung Opacity (49.5%) |
| Exact duplicate reports | 4 |
| De-id placeholder reports (`___`) | 9,363 (4.19%) |

---

*Generated from: `chexpert_plus_core_table_eda.ipynb`, `chexpert_plus_eda_updated.ipynb`, `chexpert_plus_findings_eda.ipynb`, `chexpert_plus_impression_eda.ipynb`, `chexpert_plus_report_eda.ipynb`, and 13 exported summary CSVs.*
*Reference: Chambon et al., "CheXpert Plus: Augmenting a Large Chest X-ray Dataset with Text Radiology Reports, Patient Demographics and Additional Image Formats," 2024.*
