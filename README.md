# Charlson Comorbidity Index (CCI) Pipeline

> End-to-end CCI calculation pipeline built on the OMOP Common Data Model — covering mock patient data generation, drug-condition temporal filtering, ICD-10 mapping, and comorbidity scoring.

---

## Table of Contents

- [Overview](#overview)
- [What is CCI?](#what-is-cci)
- [Dataset](#dataset)
- [Project Structure](#project-structure)
- [Pipeline](#pipeline)
- [Key Logic](#key-logic)
- [Visualizations](#visualizations)
- [Installation](#installation)
- [Usage](#usage)
- [Limitations](#limitations)
- [Future Work](#future-work)

---

## Overview

This project implements a **Charlson Comorbidity Index (CCI)** scoring pipeline using the **OMOP Common Data Model (CDM)** structure. Since real patient data is not available, a realistic mock dataset is generated using the `Faker` library, then processed through standard OMOP-aligned tables to produce per-patient comorbidity scores.

The pipeline follows clinical logic: drug exposure must precede the associated diagnosis, ensuring temporal validity before any comorbidity is counted.

---

## What is CCI?

The **Charlson Comorbidity Index** is a weighted scoring system used in clinical research and healthcare analytics to estimate a patient's **10-year mortality risk** based on the presence of specific chronic conditions.

Each condition carries a weight (1–6), and the total score reflects overall disease burden:

| Score | Estimated 10-Year Survival |
|---|---|
| 0 | ~98% |
| 1–2 | ~90% |
| 3–4 | ~77% |
| 5+ | ~21% |

Conditions include myocardial infarction, diabetes, dementia, metastatic cancer, renal disease, and 13 others — all mapped via ICD-10 codes.

---

## Dataset

All data is synthetically generated. No real patient records are used.

| Table | Records | Description |
|---|---|---|
| `PERSON` | 100 | Synthetic patients with demographics |
| `OBSERVATION_PERIOD` | 100 | One observation period per patient |
| `DRUG_EXPOSURE` | 500 | Drug exposure records across 20 drug types |
| `CONDITION_OCCURRENCE` | 500 | Condition records across 20 condition types |

ICD-10 codes used for CCI mapping:

```
I21, I428, I70, G45, F00, I278, M05, K25, B18,
E100, E102, G041, I120, C00, I850, C77, B20, N0P3, N00, N3G
```

---

## Project Structure

```
├── CCI.ipynb          # Main notebook — full pipeline
├── README.md
└── requirements.txt
```

---

## Pipeline

```
Mock Data Generation (Faker)
   │
   ├── PERSON table          (100 patients, demographics)
   ├── OBSERVATION_PERIOD    (start/end dates per patient)
   ├── DRUG_EXPOSURE         (500 records, 20 drug types)
   └── CONDITION_OCCURRENCE  (500 records, 20 condition types)
         │
         ▼
First Exposure Identification
   └── Per patient per drug → keep only earliest exposure date
         │
         ▼
Temporal Filtering
   └── Keep only conditions that occurred BEFORE drug exposure date
         │
         ▼
ICD-10 Mapping
   └── condition_concept_id → ICD-10 code via lookup table
         │
         ▼
CCI Scoring (comorbidipy)
   └── Charlson score per patient
         │
         ▼
Visualization
```

---

## Key Logic

### 1. First Exposure Identification

For each patient and drug type, only the **earliest drug exposure record** is retained. This prevents double-counting when a patient has multiple records for the same drug.

```python
dat_drug_first['min_drug_exposure_start_date'] = (
    dat_drug_first
    .groupby(['person_id', 'drug_concept_id'])['drug_exposure_start_date']
    .transform(min)
)
```

### 2. Temporal Filtering

A condition is only counted toward CCI if it was diagnosed **on or before** the date of first drug exposure. This enforces the clinical assumption that pre-existing conditions drive comorbidity — not new diagnoses.

```python
dat_drug_and_cond_match2['drug_on_or_after_condition'] = (
    dat_drug_and_cond_match2['condition_start_date']
    <= dat_drug_and_cond_match2['drug_exposure_start_date']
).astype(int)
```

### 3. CCI Calculation

```python
from comorbidipy import comorbidity

charlson_scores = comorbidity(
    df=dat_condition_lookup,
    id='condition_concept_id',
    code='code',
    score='charlson',
    icd='icd10',
    variant='quan',
    weighting='quan',
    assign0=True
)
```

---

## Visualizations

The notebook includes the following plots:

| Plot | Description |
|---|---|
| Birth Year Distribution | Age spread across the synthetic cohort |
| Top 10 Drugs by Exposure | Most frequently prescribed drug types |
| Conditions per Patient | Distribution of unique condition counts |
| First Exposure Date Distribution | Timeline of drug exposure starts |
| Observation Period Lengths | Days between observation start and end |
| Top 10 Conditions | Most frequently occurring condition types |
| Drug–Condition Co-occurrence Heatmap | Which drugs and conditions appear together |
| Age at First Exposure (Boxplot) | Age distribution per drug type |

---

## Installation

```bash
pip install comorbidipy
pip install pyomop
pip install pandas numpy faker matplotlib seaborn sqlalchemy
```

---

## Usage

Run the notebook top to bottom in a standard Jupyter or Google Colab environment. No external data files are required — all data is generated within the notebook.

```bash
jupyter notebook CCI.ipynb
```

---

## Limitations

- All patient data is **synthetic** — results are not clinically validated
- The mock ICD-10 codes are illustrative; some (e.g. `N0P3`, `N3G`) are not valid real-world codes
- The condition-to-drug matching logic uses concept ID equality as a proxy — in production, a proper OMOP vocabulary mapping would be required
- `pyomop` is used for the CDM connection layer only; no live OMOP database is connected

---

## Future Work

- [ ] Connect to a real OMOP CDM instance (e.g. via PostgreSQL or Databricks)
- [ ] Replace mock ICD-10 codes with valid SNOMED-to-ICD mappings from the OMOP vocabulary
- [ ] Add Elixhauser Comorbidity Score as an alternative to CCI
- [ ] Build a patient-level summary table with age-adjusted CCI scores
- [ ] Extend to survival analysis using CCI as a covariate

---

## Tech Stack

![Python](https://img.shields.io/badge/Python-3.10-blue?style=flat-square&logo=python&logoColor=white)
![Jupyter](https://img.shields.io/badge/Jupyter-Notebook-orange?style=flat-square&logo=jupyter&logoColor=white)
![Pandas](https://img.shields.io/badge/Pandas-150458?style=flat-square&logo=pandas&logoColor=white)

---

## References

- [Charlson et al. (1987)](https://www.sciencedirect.com/science/article/pii/0021968187901718) — Original CCI publication
- [OMOP Common Data Model](https://ohdsi.github.io/CommonDataModel/) — OHDSI CDM documentation
- [comorbidipy](https://github.com/llorencburgas/comorbidipy) — Python comorbidity scoring library
- [Quan et al. (2005)](https://pubmed.ncbi.nlm.nih.gov/16224307/) — Updated ICD-10 CCI coding algorithm

---

## License

This project is for academic and research purposes.
