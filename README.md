# Clinical Information Extraction from ICU Admission Diagnoses with a Local LLM

Converting free-text admission diagnoses from MIMIC-III into structured clinical categories with Llama 3.1 running locally, then evaluating the output against guideline-based reference labels and a keyword-rule baseline.

![Python](https://img.shields.io/badge/Python-3.12-1a1a2e?style=flat-square&logo=python&logoColor=white)
![Llama](https://img.shields.io/badge/LLM-Llama%203.1%208B-0a3f6e?style=flat-square)
![Ollama](https://img.shields.io/badge/Runtime-Ollama%20(local)-2ecc71?style=flat-square)
![MIMIC-III](https://img.shields.io/badge/Data-MIMIC--III-e07b39?style=flat-square)

---

## Key Results

| Metric | Result |
|---|---|
| Valid structured outputs | **299 / 300 (99.7%)** |
| Agreement with reference labels, `body_system` (n=100) | **86%** (Cohen's κ 0.83) |
| Agreement with reference labels, `procedure_mentioned` (n=100) | **82%** (Cohen's κ 0.56) |
| Admission-weighted agreement | 88.8% body system, 92.5% procedure |
| Admissions labelled | 37,078 of 58,976 (62.9%) from the 300 most frequent phrases |
| Named procedures detected | 14 / 14 (errors were false positives, not misses) |

## The Problem

Admission diagnoses in electronic health records are typed as short free text:

```
CORONARY ARTERY DISEASE\CORONARY ARTERY BYPASS GRAFT /SDA
S/P FALL;TELEMETRY
UPPER GI BLEED
```

The field is abbreviated, inconsistent and often holds more than one condition. In MIMIC-III, 58,976 admissions use **15,614 distinct phrases**, and the same condition appears under many spellings (`GI BLEED`, `UPPER GI BLEED`, `GASTROINTESTINAL BLEED`). In this form the text cannot be grouped, counted or used as a model feature.

## What This Project Does

Each phrase is converted into two structured fields:

| Input | `body_system` | `procedure_mentioned` |
|---|---|---|
| `CORONARY ARTERY DISEASE\CORONARY ARTERY BYPASS GRAFT /SDA` | Heart | Yes |
| `UPPER GI BLEED` | Gut and Liver | No |
| `S/P FALL` | Injury | No |

**Approach**
- **Local LLM:** Llama 3.1 8B served through Ollama; the extraction step sends no data to external services
- **Prompting:** few-shot prompt with a closed label set, abbreviation definitions and JSON-only output at temperature 0
- **Validation:** raw responses retained; unparseable or out-of-schema answers recorded as extraction errors instead of being defaulted to "Other"
- **Efficiency:** extraction runs once per unique phrase, with a checkpoint every 50 phrases
- **Evaluation:** 100 randomly sampled phrases compared with reference labels produced with an external LLM from a written annotation guideline (without access to Llama's output) and reviewed by the author; per-category precision, recall and F1, Cohen's kappa, admission-weighted agreement, and a keyword-rule baseline

## Pipeline

```
AdmitDiagnosis (58,976 admissions)
        │
        ▼
Normalise text → 15,614 unique phrases → top 300 (62.9% of admissions)
        │
        ▼
Llama 3.1 8B via Ollama  ──  few-shot prompt, JSON output, temperature 0
        │
        ▼
Schema validation  ──  raw response + status kept for every phrase
        │
        ▼
Map labels back to 37,078 admissions
        │
        ▼
Reference labels (100 phrases)  →  agreement, metrics, baseline, error analysis
```

![Body system distribution](images/body_system_distribution.png)

## Keyword Baseline

| Method | Body system | Procedure |
|---|---|---|
| Llama 3.1 8B | 86% | 82% |
| Keyword rules | 95% | 100% |

The rules were written from the same guideline as the reference labels, with the sample visible, so their score is an optimistic upper bound. Two conclusions follow. The procedure field is essentially a keyword task, where rules are the simpler and more reliable tool. For body system, the rules break where wording varies (`STAB` inside `UNSTABLE ANGINA` → Injury, `CEREBRAL VASCULAR ACCIDENT` → Injury) and find no keyword for 33.7% of the 15,314 rarer phrases, where the LLM does not depend on a fixed word list.

## Error Analysis

26 of the 32 disagreements fall into five patterns, all caused by the prompt and label schema rather than by the model:

| Pattern | Disagreements | Example |
|---|---|---|
| `/SDA` with no named procedure not marked Unclear | 11 | `LIVER MASS/SDA` → No |
| Telemetry treated as a procedure | 6 | `PNEUMONIA;TELEMETRY` → Yes |
| Cancer assigned to the organ | 3 | `LUNG CANCER/SDA` → Lungs |
| Blood-vessel conditions misplaced | 3 | `CAROTID STENOSIS` → Injury |
| Altered mental status → Metabolic | 3 | `MENTAL STATUS CHANGES` |

## Repository Structure

```
├── admit_diagnosis_llm_extraction.ipynb   ← full pipeline, results and findings
├── images/
│   └── body_system_distribution.png
├── requirements.txt
├── .gitignore
└── README.md
```

Data files and outputs containing patient-level rows are excluded under the MIMIC data use agreement.

## How to Run

1. **Install Ollama** from [ollama.com](https://ollama.com) and download the model:
   ```
   ollama pull llama3.1:8b
   ```
2. **Install Python packages** (tested with Python 3.12):
   ```
   pip install -r requirements.txt
   ```
3. **Get the data:** download `mimic3c.csv` from [Kaggle](https://www.kaggle.com/datasets/drscarlat/mimic3c) and save the `hadm_id` and `AdmitDiagnosis` columns as `admit_diagnosis.csv` in the project folder.
4. **Run the notebook** with Ollama running:
   ```
   python -m notebook
   ```

The full extraction of 300 phrases takes about 50 minutes on CPU. Results are checkpointed to `outputs/llm_labels.csv`; when that file exists, the extraction step is skipped and the rest of the notebook runs in a few minutes. The evaluation section expects the reference workbook at `outputs/reference_labels.xlsx`.

## Limitations

- Only the 300 most frequent phrases were processed.
- Reference labels come from an LLM following a written guideline, reviewed by a non-clinician; agreement measures consistency with that guideline, not clinical accuracy.
- The 100-phrase sample is small, so per-category metrics are imprecise for rare categories.
- Admission diagnoses are short; full clinical notes contain far more information.

## Future Work

- Revised prompt adding the telemetry, `/SDA`, cancer and vascular rules, evaluated on the same reference set
- Clinician review of the reference sample
- Extraction on all 15,614 phrases, using keyword rules for `procedure_mentioned` and the LLM for `body_system`
- Application to full clinical notes (MIMIC-III NOTEEVENTS)

## Tech Stack

| Layer | Tools |
|---|---|
| LLM | Llama 3.1 8B, Ollama |
| Data | pandas, NumPy |
| Evaluation | scikit-learn, openpyxl |
| Visualisation | matplotlib |

---

**Data:** MIMIC-III Clinical Database (Johnson et al., 2016), aggregated version from Kaggle.
**Author:** Krishna K. Joshi · [GitHub](https://github.com/KrishnaKJoshi1)
