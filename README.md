## Stage A — Deterministic Rule-Based Risk Classification

Stage A is the first layer of the hybrid risk-classification pipeline. Its purpose is to classify clear and obvious risk events using deterministic rules before using more computationally expensive semantic or LLM-based methods.

### Input

For each GMRID event, Stage A primarily uses:

- `normalized_text` — combined and normalized Headline + Details
- `maritime_label` — additional maritime context metadata
- predefined risk rules containing:
  - strong phrases
  - individual keywords

The output ground-truth column `true_risks` is **not used as an input**. It is only used later for evaluation.

### Rule-Based Scoring

Each of the six risks is scored independently.

For a given risk:

- Strong phrase match = **+3 points**
- Keyword match = **+1 point**
- Relevant maritime metadata = **+1 point**

The maritime metadata bonus is applied only when textual evidence for that risk has already been detected.

The score for risk `r` can therefore be represented as:

**Score(r) = 3 × P(r) + K(r) + M(r)**

where:

- `P(r)` = number of matched strong phrases
- `K(r)` = number of matched keywords
- `M(r)` = maritime-context bonus (0 or 1)

### Example

Event text:

> "A piracy incident prompted a maritime advisory for vessels in the region."

Suppose the Maritime Security / Navigation rules detect:

- `"maritime advisory"` → strong phrase = +3
- `"piracy"` → keyword = +1
- `maritime_label = True` → metadata bonus = +1

Therefore:

**Score = 3 + 1 + 1 = 5**

### Classification Threshold

The initial Stage A threshold is:

**Score ≥ 3**

If a risk receives a score of at least 3, it becomes a Stage A prediction.

Since a single event may represent multiple risks, Stage A supports **multi-label classification**.

Example:

`[weather_disruption, port_closure]`

### Output

For every record, Stage A produces:

- predicted risk(s)
- score for each predicted risk
- matched strong phrases
- matched keywords
- metadata bonus, if applicable

Example output:

```python
{
    "predictions": ["maritime_security_navigation_disruption"],
    "evidence": {
        "maritime_security_navigation_disruption": {
            "score": 5,
            "matched_phrases": ["maritime advisory"],
            "matched_keywords": ["piracy"],
            "context_bonus": 1
        }
    }
}