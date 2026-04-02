# 🦈 Shark Attacks — Data Preparation Pipeline

> **Data and Information Quality** · A.Y. 2025/26 · Politecnico di Milano  
> Dataset nr. 3 · **Score: 4 / 4** ✅

**Authors:** Devis Nishku (270011) · Andrea Lancini (259408)

---

## Overview

This project implements a complete **data preparation and quality assessment pipeline** on the [Global Shark Attacks dataset](https://www.sharkattackfile.net/), carried out for the *Data and Information Quality* course of the M.Sc. in Computer Science and Engineering at Politecnico di Milano.

The pipeline covers every stage of the data quality lifecycle — from raw profiling and quality measurement, through systematic cleaning and standardisation, to a final post-treatment quality re-assessment — producing a cleaned, analysis-ready CSV.

---

## Pipeline Stages

### 1. Data Loading & Basic Profiling
- Dataset loaded directly from a public GitHub URL as a CSV (~6 000+ rows, 24 columns)
- Dimensionality, data types, and exact-duplicate analysis
- Per-column statistics: count, null rate, distinct values, duplicates, max-frequency value

### 2. Data Exploration
- Investigation of the three redundant `Case Number` columns and their inconsistencies
- Analysis of `href` vs `href formula` vs `pdf` relationships (56 diverging cases identified)
- Correlation analysis on numerical features (`Year` vs `original order`)

### 3. Initial Data Quality Assessment
Measured **before** any transformation:

| Dimension | Approach |
|---|---|
| **Completeness** | Non-null cells / total cells |
| **Syntactic Accuracy** | Regex and domain checks on `Case Number`, `Age`, `Sex`, `Fatal (Y/N)` |
| **Semantic Accuracy** | Not feasible — no external ground truth available |
| **Timeliness** | Not feasible — no record timestamps available |
| **Consistency** | Integrity rules: `CaseNumber` columns agree · `fatal/injury` co-occurrence · `Date.year == Year` · `Name` gender tag implies `Sex` value |
| **Uniqueness / Distinctness / Constancy** | Computed column-wise |

Automated full profiling generated via **ydata-profiling**.

### 4. Data Transformations

**Column renaming** — snake_case across the board; `Case Number.2` promoted to canonical `case_number` (most accurate of the three).

**Feature dropping:**
- `age`, `time`, `species` — ≥ 44% missing values, imputation infeasible
- `case_number_1`, `case_number_2` — mostly duplicate of `case_number`
- `original_order` — high correlation with `year` (redundant)
- `href`, `href_formula` — redundant with `pdf` (one edge-case value recovered first)
- `location` — too granular and noisy; `country` + `area` sufficient

**Standardisation** applied column by column:

| Column | Transformation |
|---|---|
| `case_number` | Strip, lowercase, remove double dots and inconsistent separators |
| `date` | Parsed to `YYYY-MM-DD`; month/day placeholders tracked via boolean flags `true_month` / `true_day` |
| `type` | Lowercase, strip; `"boat"` → `"boating"` |
| `country` | Uppercase; split ambiguous dual-country entries; historical/alternate names mapped to modern equivalents (e.g. `BURMA` → `MYANMAR`) |
| `area` | Lowercase, strip; mojibake characters fixed (`Maranh�o` → `Maranhao`, etc.) |
| `fatal` | Domain mapped to `{y, n, nan}`; junk values (`#VALUE!`, `2017`, `F`) → `nan` |
| `sex` | Domain mapped to `{m, f, nan}`; keyboard-adjacent typo `N` → `m` handled |
| `activity` | Keyword-regex categorised into `{surfing, swimming, fishing, diving, boating, other, unknown}` |
| `name` | Lowercase, whitespace normalised; generic descriptors (e.g. *"male victim"*) nullified |
| `injury` | Categorised into `{lacerations, amputation, minor injury, no injury, fatal, unclassified}` |
| `source` | Lowercase, strip; parenthetical dates removed; delimiters normalised |
| `pdf` | Lowercase; underscores → hyphens; `.pdf` extension removed (redundant) |

### 5. Missing Value Imputation

| Column | Strategy |
|---|---|
| `date` | Reconstructed from `year` (Jan 1 of that year; flags set accordingly) |
| `year` | Extracted from `date` |
| `country` / `area` | Cross-filled when one is available |
| `sex` | Predicted via **gender_guesser** on the first name token |
| `fatal` + `injury` | `'u'` / `'unclassified'` when both are simultaneously null |
| `type`, `activity`, `name`, `source` | Filled with `'unknown'` |

### 6. Outlier Detection
- Boxplot analysis on `date` and `year`
- 124 rows with `year == 0` detected (likely default value) and corrected by extracting the year from the `date` field

### 7. Duplicate Detection

**Exact matching** — no exact duplicates found after cleaning.

**Record Linkage** (approximate / fuzzy):
- Candidate pair generation via two complementary strategies:
  - *Sorted Neighbourhood* on `date` (window = 11)
  - *Blocking* on `country` + `area`
- Pair comparison using Jaro-Winkler on Soundex-encoded `name`, plus string similarity on `date`, `injury`, `activity`, `type`, `sex`, `country`, `area`
- Pairs scoring ≥ 11 (out of a maximum 13) flagged as duplicates; the row with more unknown/placeholder values is dropped

### 8. Final Data Quality Assessment
All quality dimensions from Stage 3 re-evaluated on the cleaned dataset, providing a quantitative before/after comparison.  
Cleaned dataset exported to `shark_attacks_cleaned.csv`.

---

## Tech Stack

| Tool | Purpose |
|---|---|
| Python 3 | Core language |
| pandas · numpy | Data manipulation |
| seaborn · matplotlib | Visualisation |
| missingno | Missing value visualisation |
| ydata-profiling | Automated profiling reports |
| recordlinkage | Candidate pair indexing & comparison for deduplication |
| gender_guesser | Name-based sex imputation |

---

## How to Run

The notebook was developed in **Google Colab**. To reproduce:

1. Open `DIQ_2025_26_shark_attacks_nishku_lancini.ipynb` in Colab (or any Jupyter environment).
2. Run all cells in order. The dataset is fetched automatically from GitHub — no manual download required.
3. The cleaned output will be saved locally as `shark_attacks_cleaned.csv` (or to your Google Drive if that cell is enabled).

> **Note:** The association rules cell and the ydata-profiling cells are computationally heavy. The `run_algorithm` flag in the association rules cell is set to `False` by default.

---

*Politecnico di Milano — M.Sc. Computer Science and Engineering*
