# GMRID Risk Classification

A hybrid multi-label risk-classification pipeline built using the GMRID dataset. The project classifies events into six operational and maritime risk categories.

## Pipeline

1. **Data Preparation** — Cleans the data and creates the target risk labels.
2. **Stage A: Rules** — Classifies clear cases using deterministic rules.
3. **Stage B: Embeddings** — Uses `all-MiniLM-L6-v2` embeddings and logistic regression with learned thresholds.
4. **Stage C: RAG + LLM** — Routes uncertain cases to `Qwen2.5-3B-Instruct`, supported by three similar training examples retrieved using NumPy.

Stage C uses this fallback when adjudication fails:

1. Use Stage B.
2. Otherwise use Stage A.
3. Otherwise return `[]`.

## Results

| Method | Micro F1 | Macro F1 | Exact Match |
|---|---:|---:|---:|
| Stage A | 0.7634 | 0.6983 | 0.5665 |
| Stage B | 0.8765 | 0.8380 | 0.7448 |
| Final Pipeline | **0.8845** | **0.8383** | **0.7729** |

Stage C processed 194 of 819 test records. Only 100 routed training records were used for LLM evaluation.

