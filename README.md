# SDAIA Drug Safety Pipeline
### Trusted Lakehouse for Drug Side-Effect Monitoring

> This project was completed as the final project of the **SDAIA Academy** training program **"Modern Data Engineering for AI Systems"**.

---

## 1. Project Overview

An end-to-end data pipeline that ingests **100,000 drug side-effect reports**, processes them with **Apache Spark**, validates them with **4 data quality checks**, and uses a **PASS/FAIL Quality Gate** to decide where each batch goes:

- **PASS** → stored as trusted data in **Delta Lake** (Silver layer)
- **FAIL** → sent to **Quarantine** together with the reason

Only trusted data is used to build the final outputs: **analytics** (Gold layer) and a **RAG assistant** that answers drug-safety questions.

---

## 2. Problem Description

Health authorities receive thousands of reports about drug side effects. If these reports contain errors such as **duplicated reports, invalid ages, missing drug names, or impossible dosages**, any analysis built on them gives a **wrong picture of drug safety**: a safe drug may look dangerous, or a dangerous drug may look safe.

**Goal:** make sure no bad data ever reaches the analytics or the AI assistant.

**Business Impact:**
- Duplicated reports inflate the risk of a drug (one case counted 300 times).
- Missing drug names make a report useless for safety monitoring.
- Invalid values (negative ages or dosages) distort averages and comparisons.

---

## 3. Data Source

| Item | Details |
|---|---|
| Source | Kaggle – Drug Side Effects 100K dataset |
| File | `drug_side_effects_100k_dataset.csv` |
| Size | 100,000 rows × 16 columns |
| Type | Synthetic patient-level reports (no real patient data) |
| Coverage | 10 drugs (about 10,000 reports each) |

**Data Dictionary** (the 8 columns used in the pipeline):

| Column | Type | Description |
|---|---|---|
| `patient_id` | String | Unique report / patient ID (e.g. PT-100000) |
| `age` | Integer | Patient age (18–90 in the source data) |
| `gender` | String | Male / Female |
| `drug_name` | String | Drug taken (e.g. Insulin, Paracetamol) |
| `dosage_mg` | Integer | Dosage in milligrams |
| `side_effect` | String | Reported side effect (e.g. Nausea, Rash) |
| `severity` | String | Mild / Moderate / Severe |
| `outcome` | String | Recovered / Recovering / Hospitalized / Fatal |

---

## 4. Workflow / Architecture

![Architecture](images/architecture.png)

| Layer | Content |
|---|---|
| **Bronze** | Raw data exactly as ingested (Delta Lake) |
| **Silver** | Clean, validated, trusted data – only batches that PASS the gate (Delta Lake) |
| **Gold** | Business-ready aggregated tables for analytics and RAG (Delta Lake) |
| **Quarantine** | Batches that FAIL the gate, with `failed_checks` and `quarantined_at` (Delta Lake) |

**Pipeline steps:**
1. **Ingestion** – batch read of the CSV with Spark using an enforced schema; raw data saved to Bronze.
2. **Transformation** – keep the 8 relevant columns, trim spaces, standardize text casing, add `ingested_at`.
3. **Data Quality** – run 4 quality checks.
4. **Quality Gate** – PASS → Silver, FAIL → Quarantine.
5. **Gold** – aggregate trusted data into analytics tables.
6. **Outputs** – analytics chart + RAG assistant.

---

## 5. Data Quality Checks

All rules are stored in one `RULES` dictionary, so they can be changed in one place.

| Check | Rule | Why it matters |
|---|---|---|
| **Completeness** | `patient_id`, `drug_name`, `side_effect`, `severity` must not be null or empty | A report without a drug name cannot be used for drug safety |
| **Uniqueness** | `patient_id` must not repeat | Duplicates inflate the number of cases for a drug |
| **Validity** | `age` between 0–120; `gender` and `severity` from allowed values | Impossible values distort averages and counts |
| **Accuracy / Business Rule** | `dosage_mg` > 0 | A zero or negative dosage is medically impossible |

**Quality Gate:** a batch **PASSES** only if **every** check has an error rate **≤ 5%**. Otherwise the **whole batch** is sent to Quarantine with the names of the failed checks.

---

## 6. AI / RAG and Analytics Output

### Analytics (Gold layer)
Two Gold tables built **only from trusted Silver data**:
- `drug_safety_summary` – total reports, severe cases (%), and hospitalized/fatal cases (%) per drug
- `top_side_effects` – the top 3 side effects for each drug

### RAG Assistant
| Stage | Implementation |
|---|---|
| Chunking | Each Gold row becomes one text chunk (10 drug chunks + 1 comparison chunk) |
| Embeddings | `sentence-transformers` – `all-MiniLM-L6-v2` |
| Vector Database | FAISS (`IndexFlatIP`, cosine similarity) |
| Retrieval | Top 2 most similar chunks per question |
| LLM | `Qwen2.5-0.5B-Instruct` (free, runs inside Colab, no API key) – instructed to answer **only** from the retrieved context |

![RAG Assistant](images/rag_assistant.png)

---

## 7. Results

| Batch | Rows | Result | Destination |
|---|---|---|---|
| `batch_001_original` | 100,000 | ✅ **PASS** (all checks 0.00%) | Silver (Delta Lake) |
| `batch_002_corrupted` | 2,300 | ❌ **FAIL** | Quarantine |

**Failing case** – a corrupted batch created on purpose (invalid ages, missing drug names, negative dosages, 300 duplicates):

| Check | Error rate | Status |
|---|---|---|
| Completeness | 8.83% | ❌ |
| Uniqueness | 13.04% | ❌ |
| Validity | 9.30% | ❌ |
| Accuracy (dosage > 0) | 8.83% | ❌ |

![Quality Gate FAIL](images/quality_gate_fail.png)

**Verification:** Silver contains exactly 100,000 trusted rows and Quarantine contains 2,300 rows – **not a single corrupted row reached the trusted data**. Delta Lake history provides an audit trail of every write.

**Analytics findings (in this dataset):**
- Highest severe side-effect rate: **Amoxicillin (8.54%)**; lowest: **Amlodipine (7.37%)**
- Highest hospitalized/fatal rate: **Lisinopril (12.22%)**
- Example top side effects: Insulin → Sweating, Weight Gain, Hypoglycemia; Paracetamol → Rash, Liver Toxicity, Nausea

**RAG results:** the assistant answered all test questions correctly from trusted data, e.g. *"How many Paracetamol cases ended in hospitalization or death?"* → **1,188**.

**Lesson learned:** the small LLM first answered the comparison question incorrectly even though the retrieved context was correct. Rewriting the comparison chunk to state the highest and lowest drugs explicitly fixed it, showing that **chunk quality directly affects answer quality**.

### Requirements Checklist

| SDAIA Requirement | Where in the project |
|---|---|
| Data Source & Ingestion (batch) | Step 2 – Spark CSV read with enforced schema → Bronze |
| Spark processing | Steps 2–7 (PySpark) |
| Transformation raw → clean | Step 3 – column selection, trimming, standardization |
| At least 3 quality checks | Step 4 – Completeness, Uniqueness, Validity, Accuracy (4 checks) |
| PASS/FAIL Quality Gate | Step 5 – `quality_gate()` with 5% threshold |
| Quarantine for failed data | Steps 5–6 – Quarantine Delta table with reasons |
| One passing + one failing case | Step 5 (PASS) and Step 6 (FAIL) |
| Delta Lake storage | Bronze, Silver, Gold, Quarantine – all Delta tables |
| AI/RAG or Analytics output | Step 7 (Analytics) + Step 8 (RAG) |
| Architecture diagram | `images/architecture.png` |

---

## 8. Technologies Used

| Technology | Role |
|---|---|
| **Google Colab** | Development environment |
| **PySpark 3.5.3** | Ingestion, transformation, quality checks, aggregation |
| **Delta Lake 3.2.1** | Bronze / Silver / Gold / Quarantine storage with history |
| **sentence-transformers** | Text embeddings for RAG |
| **FAISS** | Vector database and similarity search |
| **Hugging Face Transformers (Qwen2.5-0.5B-Instruct)** | LLM for answer generation |
| **Matplotlib / Pandas** | Visualization |

---

## 9. How to Run the Project

1. Open `SDAIA_Drug_Safety_Pipeline.ipynb` in [Google Colab](https://colab.research.google.com).
2. Upload `data/drug_side_effects_100k_dataset.csv` (the notebook's upload cell or the Files panel). It must be at `/content/drug_side_effects_100k_dataset.csv`.
3. Run all cells in order (**Runtime → Run all**).
4. The pipeline will create the Delta tables under `/content/lakehouse/` (bronze, silver, gold, quarantine), print the PASS and FAIL gate results, show the analytics chart, and answer the RAG questions.

> Note: the pip warning about `dataproc-spark-connect` can be ignored. The first RAG run downloads the embedding model and the LLM (~1–2 minutes).

---

## 10. Future Improvements

- **Streaming ingestion** (Kafka / Spark Structured Streaming) to validate reports as they arrive.
- **Orchestration** with Airflow to run the pipeline on a schedule.
- **Row-level quarantine** to keep valid rows from a failed batch and isolate only bad rows.
- **Alerts** (email / Slack) when a batch is quarantined.
- **Larger LLM** and a simple chat interface (e.g. Streamlit) for the RAG assistant.

---

## 11. Limitations & Disclaimer

- The dataset is **synthetic** and evenly distributed, so differences between drugs are small. Findings describe **this dataset only** and are not medical conclusions.
- The failing batch was created on purpose to demonstrate the Quality Gate.
- This project is for **educational purposes only** and does not replace advice from a doctor or pharmacist.

---
## 12. GitHub Repository

- **SDAIA Academy (GitHub):** 🔗 https://github.com/SDAIAAcademy
- **SDAIA Official Website:**  https://sdaia.gov.sa
