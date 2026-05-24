# 🧠 Self-Healing Sentiment Analysis Pipeline

> An Apache Airflow pipeline that automatically detects, repairs, and analyzes Yelp reviews using a local LLM (Ollama) — no cloud required.

---

## 📌 Overview

This pipeline ingests raw Yelp review data, automatically heals malformed or problematic records, runs local sentiment inference via **Ollama (Llama 3.2)**, and produces a structured JSON report with health metrics — all orchestrated by **Apache Airflow 2.x (SDK-style DAGs)**.

The "self-healing" layer means the pipeline never crashes on bad data. Instead, it classifies each anomaly, applies a targeted fix, flags the record, and carries on.

---

## ✨ Features

- **Self-Healing Data Layer** — handles missing text, wrong types, empty strings, special-character-only content, and oversized reviews automatically
- **Local LLM Inference** — uses [Ollama](https://ollama.ai) to run `llama3.2` (or any compatible model) fully on-device
- **Retry Logic** — each review gets up to 3 inference attempts before graceful degradation
- **Degraded Mode** — if Ollama is unreachable, the pipeline continues and marks records as `degraded` instead of failing
- **Rich Reporting** — outputs per-run JSON with sentiment distribution, star-to-sentiment correlation, confidence scores, and healing stats
- **Health Status** — pipeline auto-classifies its own run as `HEALTHY`, `WARNING`, `DEGRADED`, or `CRITICAL`
- **Parameterized Runs** — batch size, offset, model name, and input file are all configurable at trigger time

---

## 🏗️ Architecture

```
┌──────────────┐     ┌──────────────┐     ┌──────────────────────┐
│  load_model  │     │ load_reviews │     │                      │
│  (Ollama)    │──┐  │  (from file) │──┐  │  diagnose_and_heal   │
└──────────────┘  │  └──────────────┘  │  │       _batch         │
                  └────────────────────┴─▶└──────────────────────┘
                                                      │
                                                      ▼
                                        ┌──────────────────────────┐
                                        │  batch_analyze_sentiment  │
                                        │       (Ollama LLM)        │
                                        └──────────────────────────┘
                                                      │
                                          ┌───────────┴──────────┐
                                          ▼                       ▼
                                 ┌─────────────────┐   ┌──────────────────────┐
                                 │ aggregate_results│   │ generate_health_report│
                                 │  (JSON output)  │   │   (pipeline status)   │
                                 └─────────────────┘   └──────────────────────┘
```

---

## 🔧 Tech Stack

| Component        | Technology              |
|-----------------|-------------------------|
| Orchestration   | Apache Airflow 2.x (SDK)|
| LLM Backend     | Ollama (`llama3.2`)      |
| Input Data      | Yelp Academic Dataset   |
| Language        | Python 3.10+            |
| Output Format   | JSON                    |

---



## 🚀 Getting Started

### 1. Clone the repository

```bash
git clone https://github.com/your-username/self-healing-pipeline.git
cd self-healing-pipeline
```

### 2. Install dependencies

```bash
pip install apache-airflow ollama
```

### 3. Pull the LLM model

```bash
ollama pull llama3.2
```

### 4. Set up environment variables

```bash
export PIPELINE_BASE_DIR=/path/to/your/project
export PIPELINE_INPUT_FILE=$PIPELINE_BASE_DIR/input/yelp_academic_dataset_review.json
export PIPELINE_OUTPUT_DIR=$PIPELINE_BASE_DIR/output
```

### 5. Place the DAG in Airflow's DAGs folder

```bash
cp dags/self_healing_pipeline.py $AIRFLOW_HOME/dags/
```

### 6. Trigger the pipeline

Via Airflow UI, or via CLI:

```bash
airflow dags trigger self_healing_pipeline \
  --conf '{"batch_size": 50, "offset": 0, "ollama_model": "llama3.2"}'
```

---

## 🩺 Self-Healing Logic

Each review is inspected and repaired before inference:

| Issue Detected           | Error Type               | Action Taken                  |
|--------------------------|--------------------------|-------------------------------|
| `text` is `None`         | `missing_text`           | Replaced with placeholder     |
| `text` is not a string   | `wrong_type`             | Converted to string           |
| `text` is blank/whitespace| `empty_text`            | Replaced with placeholder     |
| Text has no alphanumerics | `special_characters_only`| Replaced with `[Non-text content]` |
| Text exceeds max length  | `too_long`               | Truncated at 2000 chars       |
| No issues found          | —                        | Passed through as-is          |

All healed records are flagged with `was_healed: true`, `healing_action`, and `error_type` in the output.

---

## 📊 Output Format

Each run produces a timestamped JSON file in the output directory:

```
output/sentiment_analysis_summary_2025-01-15_14-30-00_Offset0.json
```

The report includes:

```json
{
  "run_info": { "timestamp": "...", "batch_size": 100, "offset": 0 },
  "totals": { "processed": 100, "success": 87, "healed": 12, "degraded": 1 },
  "rates": { "success_rate": 0.87, "healing_rate": 0.12, "degradation_rate": 0.01 },
  "sentiment_distribution": { "POSITIVE": 61, "NEGATIVE": 24, "NEUTRAL": 15 },
  "healing_statistics": { "truncated_text": 8, "filled_with_placeholder": 4 },
  "star_sentiment_correlation": { "5": { "POSITIVE": 40, "NEGATIVE": 2, "NEUTRAL": 5 } },
  "average_confidence": { "success": 0.88, "healed": 0.79, "degraded": 0.5 }
}
```

---

## 🚦 Pipeline Health Status

At the end of each run, the pipeline evaluates its own health:

| Status       | Condition                                      |
|--------------|------------------------------------------------|
| `HEALTHY`    | Low healing rate, zero degradation             |
| `WARNING`    | More than 50% of reviews required healing      |
| `DEGRADED`   | Some reviews fell back to neutral (Ollama issue)|
| `CRITICAL`   | More than 10% of reviews in degraded state     |

---

## 📁 Project Structure

```
self-healing-pipeline/
├── dags/
│   └── self_healing_pipeline.py   # Main DAG definition
├── input/
│   └── yelp_academic_dataset_review.json
├── output/
│   └── sentiment_analysis_summary_*.json
└── README.md
```

---
