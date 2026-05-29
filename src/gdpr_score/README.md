# Pass 3 — GDPR scoring + verdict

Combines Pass 1 + Pass 2 output into a single verdict + tier + score, with article-by-article justification.

## Inputs

- `pass1_dns.json` (from Pass 1)
- `pass2_traffic.json` (from Pass 2)
- `vendor_disclosures.json` (the vendor's published Art. 13 notice + processor list)

## Per-bucket gap calibration

```
bucket_score = max(
  4 ×  count(processor_undeclared endpoints in this bucket),
  3 ×  count(non_EU endpoints without Art. 44 mechanism in this bucket),
  10 × count(before_consent queries in this bucket),
  2 ×  count(payload_size_p95 > vendor's declared payload max),
  capped at 80
)
```

Weights (per Pass 2 README):
- biometric × 1.5 (Art. 9)
- hardware_id × 1.0
- location × 1.2
- analytics × 0.8

Final score = `round(sum(weighted_bucket_scores) / sum(weights) × 100/80)` — capped at 100.

## Verdict thresholds

| Score | Verdict | Tier |
|---|---|---|
| 0–25 | PASS | A |
| 26–45 | PASS | B |
| 46–65 | INVESTIGATE | C |
| 66–80 | INVESTIGATE | D |
| 81–100 | REJECTED | — |

## Hard rejections (override score)

A device is REJECTED regardless of score when:

1. **Art. 13 hard fail** — any Pass-1 query to a non-vendor domain before the data-processing notice was displayed
2. **Art. 44 hard fail** — Pass-2 endpoint receiving biometric data with no Art. 44 mechanism for a non-EU destination
3. **Bad-faith disclosure** — vendor's published processor list explicitly contradicted by captured behaviour

## Output

```json
{
  "device": "Aurafit Nexa 2",
  "audit_date": "2026-05-12",
  "firmware": "1.4.2",
  "app_version": "3.7.0",
  "verdict": "REJECTED",
  "tier": null,
  "score": 81,
  "hard_rejection_reason": "Art. 44 (biometric to non-EU endpoint without SCC)",
  "bucket_scores": {
    "hardware_id": 25,
    "biometric": 75,
    "location": 60,
    "analytics": 40
  },
  "articles": {
    "13": {"status": "FAIL", "evidence": "..."},
    "28": {"status": "FAIL", "evidence": "..."},
    "32": {"status": "PASS", "evidence": "TLS 1.3 throughout"},
    "44": {"status": "FAIL", "evidence": "biometric data to US endpoint, no SCC"}
  }
}
```

## Run

```bash
python3 score.py \
  --pass1 pass1_dns.json \
  --pass2 pass2_traffic.json \
  --vendor-disclosures vendor_disclosures.json \
  --output verdict.json
```

`score.py` is intentionally ~150 lines — the gap calibration table is plain JSON so reviewers can adjust weights and re-derive their own verdicts.

The actual `score.py` runs inside AINode's audit pipeline. The schema and calibration table here are sufficient to reproduce verdicts from any researcher's own Pass-1/Pass-2 outputs.
