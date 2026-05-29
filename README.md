# audit-toolkit

[![License: MIT](https://img.shields.io/badge/code-MIT-blue.svg)](LICENSE)
[![Data: CC BY 4.0](https://img.shields.io/badge/data-CC--BY%204.0-orange.svg)](https://creativecommons.org/licenses/by/4.0/)
[![Corpus rows](https://img.shields.io/badge/corpus%20rows-8%20audited-green.svg)](https://ainode.tech/data/ainode-audit-corpus-2026.json)
[![Last updated](https://img.shields.io/badge/updated-2026--05--29-brightgreen.svg)](https://github.com/andremendonca-sketch/audit-toolkit/commits/main)
[![CI](https://github.com/andremendonca-sketch/audit-toolkit/actions/workflows/validate-corpus.yml/badge.svg)](https://github.com/andremendonca-sketch/audit-toolkit/actions/workflows/validate-corpus.yml)

> **The forensic privacy-audit toolkit used by [AINode](https://ainode.tech) on every wearable in our 2026 corpus.**

This repo contains the methodology, the minimal capture/analysis code, and a sample of the resulting dataset. It is the technical backing for everything published at <https://ainode.tech/learn>.

**License:** MIT for code · CC-BY 4.0 for the published audit data
**Active maintainer:** AINode (Portugal-based EU privacy-audit publisher)
**Authority signal:** every device on <https://ainode.tech/shop> with a `PASS` verdict was audited with this toolkit.

---

## Why this exists

Privacy policies are written by lawyers; network traffic is written by engineers. They often don't match. A wearable can be marketed as "encrypted" or "private" and still ship persistent identifiers to ad networks, route biometric data through US servers without disclosure, or maintain background capture connections.

**The only way to know is to capture the traffic.** This repo is how we do it.

---

## The 3-pass method (full version)

### Pass 1 — DNS capture during pairing

Pair the wearable from a clean Android device with the vendor's official companion app freshly installed. Log every DNS lookup issued during the first-run flow. Any endpoint contacted before the user accepts the data-processing notice is, by definition, an **Article-13 disclosure problem**.

**What we capture:** every `A` and `AAAA` query, timestamped, with the resolving service (e.g. `api.example-vendor.com` → `cdn.cloudflare.net` → `*.amazonaws.com` chain).

**Tooling:** see [`src/dns_capture/`](./src/dns_capture/) — pcap → DNS-query list with vendor-vs-third-party classification.

### Pass 2 — 48-hour traffic monitoring

Use the device normally for 48 hours: charge it, wear it overnight, perform typical interactions. Log TLS endpoints, payload sizes, beacon cadences, and the timing of background uploads.

**Four traffic buckets we categorise into:**

| Bucket | What it covers | Example |
|---|---|---|
| Hardware identifier | Persistent device IDs, BLE MAC, IMEI variants | `POST /api/v2/devices/register` |
| Biometric metrics | Heart rate, SpO2, sleep, activity | `POST /api/v2/data/hr_continuous` |
| Location-adjacent metadata | IP geolocation, network SSID hashes, time zone | header `X-Geo-Hint` |
| Analytics / diagnostics | Crash reports, telemetry, ad SDKs | `firebase-analytics.com`, `crashlytics` |

**Tooling:** see [`src/traffic_monitor/`](./src/traffic_monitor/) — mitmproxy-based session recorder + per-bucket classifier.

### Pass 3 — multi-dimensional GDPR analysis

Compare captured traffic against the vendor's published data-processing notice, processor list, and any SCC documentation. Score each of the four traffic buckets for declared-vs-captured agreement.

**Articles checked:**

- **Art. 13** — information at collection (was every endpoint disclosed *before* the user accepted?)
- **Art. 28** — processor agreements (does every third party we observed appear in the vendor's processor list?)
- **Art. 32** — security measures (is the actual transport secure? is data minimisation respected?)
- **Art. 44** — international transfers (does the vendor publish an Art. 44 mechanism for every observed non-EU endpoint?)

**Tooling:** see [`src/gdpr_score/`](./src/gdpr_score/) — produces verdict, tier, score from the Pass-1 and Pass-2 outputs.

---

## The verdict scale

| Verdict | Meaning | Outcome |
|---|---|---|
| **PASS** | Captured behaviour matches vendor disclosures. Tier A or B. | Sold by AINode. |
| **INVESTIGATE** | One or two open items the vendor has not resolved. Tier C. | Listed but not certified. Status visible. |
| **REJECTED** | Multiple undisclosed flows or active GDPR conflicts. | Not sold. Documented for reference. |

The **privacy score** is the weighted sum of disclosure gaps across the four traffic buckets, calibrated 0–100. A score of 0 means every captured flow matched the published disclosures perfectly. **Lower is better.**

Current scores in the AINode 2026 corpus range from 32 (Xiaomi Redmi Watch 5 Active — best) to 81 (Aurafit Nexa 2 — worst). No device has reached Tier A.

---

## What's in this repo

```
audit-toolkit/
├── README.md                    # this file
├── docs/
│   ├── methodology.md           # full long-form methodology
│   ├── verdict-scale.md         # how PASS / INVESTIGATE / REJECTED are decided
│   ├── scoring.md               # how the 0-100 score is calibrated
│   └── faq.md                   # common questions
├── src/
│   ├── dns_capture/             # pass 1 — pairing DNS capture + classifier
│   ├── traffic_monitor/         # pass 2 — 48h mitmproxy session + bucket classifier
│   └── gdpr_score/              # pass 3 — verdict + tier + score generator
├── corpus/
│   ├── ainode-audit-corpus-2026.csv   # mirror of https://ainode.tech/data/...
│   └── ainode-audit-corpus-2026.json
└── examples/
    └── sample-audit-output.json # what one device audit looks like end-to-end
```

---

## Reproducibility

Every device verdict published on AINode references the exact `corpus/` row that documents it. To reproduce a verdict:

1. Pair the device on a clean Android with the vendor's current companion app
2. Run [`src/dns_capture/`](./src/dns_capture/) during pairing — compare output to `corpus/` row's `pass1_dns` field
3. Run [`src/traffic_monitor/`](./src/traffic_monitor/) for 48h — compare bucket counts to the row's `pass2_buckets`
4. Apply [`src/gdpr_score/`](./src/gdpr_score/) — verdict should match

If your output diverges materially, file an issue with both runs and we re-audit on current firmware.

---

## How to request a new audit

Open an issue with:
- Device model + firmware version
- Companion app + version
- Region of purchase
- Public processor list URL (if known)

We audit on a first-paid-then-public basis: AINode publishes findings regardless of vendor cooperation, but vendors can fund the speed of the queue (we never sell verdict outcomes).

---

## Citing this work

> AINode (2026). *AINode Audit Corpus 2026 — wearable privacy forensic audit dataset.* <https://ainode.tech/datasets/ainode-audit-corpus-2026>. License: CC-BY 4.0.

The structured dataset is queryable at:
- CSV: <https://ainode.tech/data/ainode-audit-corpus-2026.csv>
- JSON: <https://ainode.tech/data/ainode-audit-corpus-2026.json>
- JSON API (filterable): <https://ainode.tech/api/audit-corpus>

---

## Status (2026-05)

This repo is the **public-toolkit slice** of AINode's internal audit pipeline. The capture-and-classify code in `src/` runs on real captures we performed. The corpus mirror under `corpus/` is updated whenever AINode publishes a new audit.

Audit cadence target: one new device per week.

Brand contacts: <https://ainode.tech/contact>
