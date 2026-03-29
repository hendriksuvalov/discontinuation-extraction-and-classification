# Article Workflow (A10/C10 Medication Stop Analysis)

This repository contains a two-stage workflow for antidiabetics (ATC A10) and statins (ATC C10).

NB! In order to ensure sensitive data not leaking, these files have been heavily modified to remove output cells, API keys and paths. In order to replicate the workflow, you have to add them according to your own setup.

## 1) Required input files

Place these source CSVs in the same working directory as the notebooks:

- a10_switches_original.csv
- c10_switches_original.csv

Minimum required columns:

- id: unique row identifier
- anamnesis: free-text clinical note (Estonian)

Recommended additional metadata columns (kept when merging):

- patient_id
- patient_age
- pat_sex
- any other fields you want to carry through

## 2) Stage 1: Extract stop event + reason + drug

Notebooks:

- diabetes_12months_specific_ATC.ipynb (uses a10_switches_original.csv)
- statins_12months_specific_ATC.ipynb (uses c10_switches_original.csv)

What happens:

- Reads input notes from anamnesis.
- Runs LLM extraction with guided JSON output.
- Produces one JSON-like result per row with:
  - drug_stop_phrase
  - reason_for_stopping
  - drug_name

Outputs:

- a10_switch_stops.csv
- c10_switch_stops.csv

Output schema:

- id
- result (serialized JSON string containing the 3 fields above)

## 3) Stage 2A: Classify why the medication was stopped

Notebooks:

- a10_switches_why_stopped.ipynb
- c10_switches_why_stopped.ipynb

Inputs consumed:

- Stage 1 outputs (a10_switch_stops.csv or c10_switch_stops.csv)
- Original source CSV (a10_switches_original.csv or c10_switches_original.csv)

What happens:

- Parses result JSON into columns.
- Merges parsed output with original rows.
- Filters to rows with non-empty reason_for_stopping.
- Filters by drug-name keyword lists for target ATC group.
- Classifies reason_for_stopping into one category:
  - Adverse reactions
  - Treatment success
  - Treatment inefficacy
  - Contraindication
  - Non-medical reasons
  - Other
  - Indeterminate

Outputs:

- a10_switches_why_stopped.csv
- c10_switches_why_stopped.csv

Output schema:

- Input (the extracted reason_for_stopping text)
- Category

## 4) Stage 2B: Classify who stopped the medication

Notebooks:

- a10_switches_who_stopped.ipynb
- c10_switches_who_stopped.ipynb

Inputs consumed:

- Stage 1 outputs (a10_switch_stops.csv or c10_switch_stops.csv)
- Original source CSV (a10_switches_original.csv or c10_switches_original.csv)

What happens:

- Parses result JSON into columns.
- Merges parsed output with original rows.
- Filters to rows with non-empty reason_for_stopping.
- Filters by drug-name keyword lists for target ATC group.
- Classifies who made discontinuation decision using discontinuation phrase + anamnesis context:
  - Doctor
  - Patient
  - Unspecified

Outputs:

- a10_switches_who_stopped.csv
- c10_switches_who_stopped.csv

Output schema:

- Input (the extracted drug_stop_phrase text)
- Category

## 5) Practical run order

1. Run diabetes_12months_specific_ATC.ipynb -> creates a10_switch_stops.csv
2. Run statins_12months_specific_ATC.ipynb -> creates c10_switch_stops.csv
3. Run a10_switches_why_stopped.ipynb and a10_switches_who_stopped.ipynb
4. Run c10_switches_why_stopped.ipynb and c10_switches_who_stopped.ipynb

## 6) Prerequisites

- GPU-capable environment recommended (vLLM usage)
- Hugging Face access token configured in notebooks
- Model availability for: neuralmagic/Meta-Llama-3.1-70B-Instruct-quantized.w4a16
- Python packages used in notebooks include:
  - vllm (v0.8.3)
  - transformers (v4.51.0)
  - torch (v2.6.0)
  - pandas
  - pydantic
  - huggingface_hub

## 7) Notes


- Some notebooks include absolute paths in older cells; if needed, change them to local relative paths before running.
- The *_gpt.ipynb notebooks require Azure resources setup along with the API key. They operate exactly like the "\*_why_stopped.ipynb" files.
