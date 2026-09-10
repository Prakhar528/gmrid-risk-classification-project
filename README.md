# GMRID Risk Classification POC

This repository is a small, standalone feasibility proof of concept for a three-stage hybrid classifier for maritime and port disruption text from the Hugging Face `windchimeran/GMRID` dataset.

Only Stage A is implemented. Stage B (embeddings) and Stage C (RAG plus LLM adjudication) are intentionally out of scope for the current version.

## Six-risk taxonomy

- `weather_disruption`
- `natural_disaster`
- `port_operational_disruption`
- `port_closure`
- `labor_strike_disruption`
- `maritime_security_navigation_disruption`

The task is multi-label: an event may receive more than one risk candidate.

## Data and proxy ground truth

`notebooks/01_data_preparation.ipynb` downloads the original GMRID train and test splits. It builds `input_text` from the original headline plus details, normalizes the text, and derives `true_risks` **only** from exact tokens in the original GMRID `Category` field.

The Version 2 mapping adds previously omitted source categories:

- Weather: `Severe Winds`, `Hurricane`, `Typhoon`, `Hail`, `Tornado`
- Natural disaster: `Landslide`
- Labour: `Miscellaneous Strikes`, `Industrial Action`, `General Strike`, `Civil Service Strike`

Labels are emitted in the fixed six-risk taxonomy order. Predictions, article text, `Summarized_label`, and `maritime_label` are never used to construct ground truth.

After the correction, the POC contains 2,916 training records and 702 test records. The original filtered files contained 2,653 and 643 records, so 263 training records and 59 test records were restored.

### Known proxy-label limitations

These are category-derived proxy labels rather than a fresh annotation of the custom taxonomy:

- `Maritime Advisory` is broad and includes administrative, service, political, and operational events that do not necessarily describe security or navigation disruption.
- GMRID often labels the operational impact, such as `Port Closure`, without also labelling an explicit weather cause. A semantically valid weather prediction can consequently be counted as a false positive.
- Port closure and port-operational disruption overlap semantically, while the source categories are not consistently multi-label.
- Hazard and advisory categories do not always imply that disruption actually occurred.

Stage A V2 deliberately does not broaden maritime-security rules merely to reproduce noisy `Maritime Advisory` proxy labels.

## Stage A Version 1 baseline

Version 1 uses literal substring matching: +3 per strong phrase, +1 per keyword, an optional +1 maritime metadata bonus, and one global threshold of 3.

Its historical result on the original fixed 200-row development sample was micro precision 0.653, micro recall 0.152, and micro F1 0.246. Because Version 2 corrects the category mapping and changes dataset membership, this historical score is retained only as context and is not directly compared with the new metrics.

## Stage A Version 2

`notebooks/02_stage_a_rules.ipynb` implements deterministic contextual rules with:

- safe headline-plus-details input construction;
- lowercase, Unicode punctuation, and whitespace normalization;
- regex word-boundary matching;
- deduplication of overlapping evidence;
- risk-specific strong and supporting evidence;
- required textual context for closure, port-operation, labour, and maritime rules;
- `maritime_label` as supporting evidence only;
- thresholds of 3 for weather/natural disaster and 4 for context-dependent risks.

Every prediction contains the score for all risks, matched strong and supporting evidence, applied context bonuses, and a reason for predicting or rejecting each risk. Record status is `no_stage_a_prediction`, `stage_a_candidate_found`, or `multiple_stage_a_candidates`.

A Stage A candidate is not treated as a completely resolved multi-label record. Later stages may add or reject labels.

For the later cascade, `no_stage_a_prediction` records proceed to Stage B automatically. Records with one or more Stage A candidates retain those explainable candidates but may also proceed to Stage B when additional-label discovery is required; prediction coverage is therefore the percentage with at least one candidate, not the percentage conclusively completed by Stage A.

## Evaluation protocol

- **Development:** the fixed 200-row training sample with seed 42.
- **Validation:** a fixed 600-row sample from the remaining training records with seed 43.
- **Final test:** the original locked GMRID test split.

Rules and thresholds were finalized using development and validation only. The final test was evaluated once after freezing Version 2; no individual test errors were used for tuning.

### Validation comparison on identical records

| Version | Average | Precision | Recall | F1 |
|---|---:|---:|---:|---:|
| V1 | Micro | 0.686 | 0.168 | 0.269 |
| V2 | Micro | 0.669 | 0.612 | 0.639 |
| V1 | Macro | 0.821 | 0.194 | 0.263 |
| V2 | Macro | 0.758 | 0.676 | 0.643 |
| V1 | Weighted | 0.789 | 0.168 | 0.233 |
| V2 | Weighted | 0.783 | 0.612 | 0.627 |

Version 2 materially improves validation recall and F1, with a small reduction in aggregate precision. Prediction coverage rises from 25.8% to 75.7%. Exact-match ratio rises from 0.158 to 0.453, and Hamming loss falls from 0.164 to 0.125.

### Locked final-test result

| Average | Precision | Recall | F1 |
|---|---:|---:|---:|
| Micro | 0.646 | 0.625 | 0.635 |
| Macro | 0.730 | 0.679 | 0.625 |
| Weighted | 0.766 | 0.625 | 0.625 |

Final-test exact-match ratio is 0.457 and Hamming loss is 0.131. Stage A produces at least one candidate for 78.3% of records; 152 of 702 records (21.7%) receive no prediction.

The strongest final-test classes are natural disaster (F1 0.907), labour disruption (0.826), and port operations (0.697). Weather has high recall but low proxy-label precision (0.417/0.917), port closure remains affected by taxonomy overlap (F1 0.547), and maritime security/navigation remains deliberately conservative because `Maritime Advisory` is noisy (precision 0.917, recall 0.113).

## Reproducibility

Run the notebooks in order from the repository root or from the `notebooks` directory:

1. `notebooks/01_data_preparation.ipynb`
2. `notebooks/02_stage_a_rules.ipynb`

Both notebooks resolve the project root automatically and are suitable for a cloned Google Colab workspace. Required packages are listed in `requirements.txt`.

Stage A writes:

- `outputs/stage_a_v2_predictions.csv`
- `outputs/stage_a_error_analysis.csv`
- `outputs/stage_a_metrics_per_risk.csv`
- `outputs/stage_a_metrics_aggregate.csv`
- `outputs/stage_a_metrics_coverage.csv`

The error analysis contains complete development, validation, and final-test label errors with original categories, predictions, scores, matched evidence, context bonuses, and prediction explanations. Final-test errors are exported for record keeping but were not manually inspected or used for rule changes.
