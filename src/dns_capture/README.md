# Pass 1 — DNS capture during pairing

Captures every DNS query issued by the vendor's companion app during the pairing flow, classifies each as `vendor` / `vendor_undeclared` / `processor_declared` / `processor_undeclared`, and flags queries that happened **before** the user accepted the data-processing notice.

## Inputs

- A `.pcap` from the audit AP covering the pairing window
- A `vendor_disclosures.json` for the vendor (their published processor list + declared endpoint set)

## Output

A per-query JSON log:

```json
[
  {
    "ts_ms": 0,
    "query": "config.example-vendor.com",
    "record_type": "A",
    "cname_chain": ["config.example-vendor.com", "edge.cloudfront.net"],
    "final_ip_region": "US",
    "classification": "vendor",
    "before_consent": true
  },
  {
    "ts_ms": 142,
    "query": "firebase-settings.crashlytics.com",
    "record_type": "A",
    "cname_chain": ["firebase-settings.crashlytics.com"],
    "final_ip_region": "US",
    "classification": "processor_undeclared",
    "before_consent": true
  }
]
```

## Run

```bash
python3 classify.py \
  --pcap pairing.pcap \
  --vendor-disclosures vendor_disclosures.json \
  --consent-screenshot consent_tap_ts.txt \
  --output pass1_dns.json
```

## Code

`classify.py` is intentionally short — most of the work is in the disclosure-matching rules. The classifier uses public suffix list + the vendor's declared domain set; anything outside is flagged.

The actual Python implementation lives in AINode's internal pipeline; we publish the rule-set and schema so audits are reproducible from your own captures.
