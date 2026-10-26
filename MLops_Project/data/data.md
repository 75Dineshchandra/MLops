# Diabetes 130-US Hospitals for Years 1999-2008

Source: [UCI Machine Learning Repository](https://archive.ics.uci.edu/dataset/296) · DOI: [10.24432/C5230J](https://doi.org/10.24432/C5230J) · License: CC BY 4.0

## Overview

This dataset represents ten years (1999-2008) of clinical care at 130 US hospitals and integrated delivery networks. Each row is a hospital encounter for a patient diagnosed with diabetes who underwent laboratory tests, received medications, and stayed up to 14 days.

**Goal:** predict whether a patient will be readmitted to the hospital within 30 days of discharge.

**Why it matters:** despite high-quality evidence that preventive and therapeutic interventions improve outcomes for diabetic patients, many patients do not receive them. This can be partly attributed to arbitrary diabetes management in hospital settings that fail to attend to glycemic control. Poor diabetes care raises hospital costs (through readmissions) and affects patient morbidity and mortality through diabetes-related complications.

## Dataset at a Glance

| Property | Value |
|---|---|
| Characteristics | Multivariate |
| Subject area | Health and Medicine |
| Associated tasks | Classification, Clustering |
| Feature types | Categorical, Integer |
| Instances | 101,766 |
| Features | 47 (plus 2 ID columns and 1 target) |
| Missing values | Yes |
| Donated | 5/2/2014 |
| Creators | John Clore, Krzysztof Cios, Jon DeShazo, Beata Strack |

## Inclusion Criteria

Encounters were extracted from the database if they met all of the following:

1. Inpatient encounter (a hospital admission).
2. Diabetic encounter: any kind of diabetes was entered as a diagnosis.
3. Length of stay of at least 1 day and at most 14 days.
4. Laboratory tests were performed during the encounter.
5. Medications were administered during the encounter.

## Files

| File | Size | Description |
|---|---|---|
| `diabetic_data.csv` | 18.3 MB | Main dataset |
| `IDS_mapping.csv` | 2.5 KB | Maps the integer IDs for `admission_type_id`, `discharge_disposition_id`, and `admission_source_id` to their meanings |

## Data Dictionary

### Identifiers

| Variable | Description | Missing |
|---|---|---|
| `encounter_id` | Unique identifier of an encounter | No |
| `patient_nbr` | Unique identifier of a patient | No |

### Demographics

| Variable | Type | Description | Missing |
|---|---|---|---|
| `race` | Categorical | Caucasian, Asian, African American, Hispanic, other | Yes |
| `gender` | Categorical | Male, female, unknown/invalid | No |
| `age` | Categorical | 10-year intervals: [0, 10), [10, 20), ..., [90, 100) | No |
| `weight` | Categorical | Weight in pounds | Yes |

### Admission and Hospital Stay

| Variable | Type | Description | Missing |
|---|---|---|---|
| `admission_type_id` | Categorical | Integer ID for 9 distinct values (e.g. emergency, urgent, elective, newborn, not available) | No |
| `discharge_disposition_id` | Categorical | Integer ID for 29 distinct values (e.g. discharged to home, expired, not available) | No |
| `admission_source_id` | Categorical | Integer ID for 21 distinct values (e.g. physician referral, emergency room, transfer from a hospital) | No |
| `time_in_hospital` | Integer | Days between admission and discharge | No |
| `payer_code` | Categorical | Integer ID for 23 distinct values (e.g. Blue Cross/Blue Shield, Medicare, self-pay) | Yes |
| `medical_specialty` | Categorical | Specialty of the admitting physician, 84 distinct values (e.g. cardiology, internal medicine, family/general practice, surgeon) | Yes |

### Utilization and Procedures

| Variable | Type | Description | Missing |
|---|---|---|---|
| `num_lab_procedures` | Integer | Number of lab tests performed during the encounter | No |
| `num_procedures` | Integer | Number of procedures (other than lab tests) performed during the encounter | No |
| `num_medications` | Integer | Number of distinct generic names administered during the encounter | No |
| `number_outpatient` | Integer | Outpatient visits in the year preceding the encounter | No |
| `number_emergency` | Integer | Emergency visits in the year preceding the encounter | No |
| `number_inpatient` | Integer | Inpatient visits in the year preceding the encounter | No |

### Diagnoses

Diagnoses are coded as the first three digits of the ICD-9 code.

| Variable | Type | Description | Missing |
|---|---|---|---|
| `diag_1` | Categorical | Primary diagnosis (848 distinct values) | Yes |
| `diag_2` | Categorical | Secondary diagnosis (923 distinct values) | Yes |
| `diag_3` | Categorical | Additional secondary diagnosis (954 distinct values) | Yes |
| `number_diagnoses` | Integer | Number of diagnoses entered into the system | No |

### Lab Results

| Variable | Type | Description | Missing |
|---|---|---|---|
| `max_glu_serum` | Categorical | Glucose serum test result: `>200`, `>300`, `normal`, or `none` if not measured | No |
| `A1Cresult` | Categorical | HbA1c result: `>8` (greater than 8%), `>7` (greater than 7% but less than 8%), `normal` (less than 7%), or `none` if not measured | No |

### Diabetic Medications

Each of the following columns is categorical with no missing values and uses the same four values:

- `up`: dosage was increased during the encounter
- `down`: dosage was decreased
- `steady`: dosage did not change
- `no`: the drug was not prescribed

Columns (23 total):

`metformin`, `repaglinide`, `nateglinide`, `chlorpropamide`, `glimepiride`, `acetohexamide`, `glipizide`, `glyburide`, `tolbutamide`, `pioglitazone`, `rosiglitazone`, `acarbose`, `miglitol`, `troglitazone`, `tolazamide`, `examide`, `citoglipton`, `insulin`, `glyburide-metformin`, `glipizide-metformin`, `glimepiride-pioglitazone`, `metformin-rosiglitazone`, `metformin-pioglitazone`

### Medication Summary

| Variable | Type | Description | Missing |
|---|---|---|---|
| `change` | Categorical | Whether there was a change in diabetic medications (dosage or generic name): `change` or `no change` | No |
| `diabetesMed` | Categorical | Whether any diabetic medication was prescribed: `yes` or `no` | No |

### Target

| Variable | Type | Description | Missing |
|---|---|---|---|
| `readmitted` | Categorical | Days to inpatient readmission: `<30` (readmitted in less than 30 days), `>30` (readmitted after more than 30 days), `No` (no record of readmission) | No |

## Quick Start

Install the `ucimlrepo` package and load the data directly:

```bash
pip install ucimlrepo
```

```python
from ucimlrepo import fetch_ucirepo

# fetch dataset
diabetes = fetch_ucirepo(id=296)

# data (as pandas dataframes)
X = diabetes.data.features
y = diabetes.data.targets

# metadata and variable information
print(diabetes.metadata)
print(diabetes.variables)
```

Alternatively, load the downloaded CSV:

```python
import pandas as pd

df = pd.read_csv("diabetic_data.csv")
ids = pd.read_csv("IDS_mapping.csv")
```

## Recommended Data Splits

There is no official recommendation. A standard train/test split can be used. For model selection, a three-way holdout split (train/validation/test) is suggested.

## Notes and Considerations

- **Missing values:** `race`, `weight`, `payer_code`, `medical_specialty`, `diag_1`, `diag_2`, and `diag_3` contain missing values.
- **Sensitive data:** the dataset contains patient age, gender, and race. Consider fairness and privacy implications when modeling and reporting results.
- **Repeat patients:** `patient_nbr` identifies patients, and a patient may appear in multiple encounters. Consider grouping by patient when splitting to avoid leakage between train and test sets.
- **ID columns:** `admission_type_id`, `discharge_disposition_id`, and `admission_source_id` are integer codes; use `IDS_mapping.csv` to decode them.
- **Target framing:** `readmitted` has three classes. For the original task (30-day readmission), it is commonly collapsed to a binary label (`<30` vs. everything else).

## Citation

If you use this dataset, please cite:

> Beata Strack, Jonathan P. DeShazo, Chris Gennings, Juan L. Olmo, Sebastian Ventura, Krzysztof J. Cios, and John N. Clore, "Impact of HbA1c Measurement on Hospital Readmission Rates: Analysis of 70,000 Clinical Database Patient Records," *BioMed Research International*, vol. 2014, Article ID 781670, 11 pages, 2014. http://www.hindawi.com/journals/bmri/2014/781670/

Dataset citation:

> Clore, J., Cios, K., DeShazo, J., & Strack, B. (2014). *Diabetes 130-US Hospitals for Years 1999-2008* [Dataset]. UCI Machine Learning Repository. https://doi.org/10.24432/C5230J

## License

This dataset is licensed under a [Creative Commons Attribution 4.0 International (CC BY 4.0)](https://creativecommons.org/licenses/by/4.0/) license, which allows sharing and adaptation for any purpose, provided appropriate credit is given.