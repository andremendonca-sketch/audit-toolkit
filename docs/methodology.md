# Full methodology

This document is the long-form version of [the README's 3-pass overview](../README.md#the-3-pass-method-full-version). It exists for researchers and journalists who need to reproduce or cite AINode audits without ambiguity.

## Capture environment

| Component | Choice | Rationale |
|---|---|---|
| Phone | Pixel 7a, factory reset before each audit | Stock Android, no GAPPS shimming, well-instrumented at the network layer |
| Android version | latest stable at audit time (pinned in `corpus/` row) | Reproducible behaviour |
| Network | dedicated 2.4 GHz AP routed through mitmproxy | All traffic visible; no ISP DNS interference |
| Time | audit window logged in UTC | Reproducible across timezones |
| Apps installed | only the vendor's companion app + Android system | No analytics-SDK cross-contamination |

The Pixel is wiped between audits via factory reset to prevent residual identifiers from the previous device from polluting the new audit.

## Pass 1 — DNS capture during pairing

### What happens

1. Phone is powered on after factory reset
2. Phone joins the audit AP (DNS pointed at our resolver)
3. Vendor's companion app is installed fresh from Play Store
4. App is opened, account created if required, device paired
5. DNS log is captured from the moment the app is launched until the first successful "device ready" UI state

### What we record

For every DNS query:
- `timestamp` (ms since pairing start)
- `query` (FQDN)
- `record_type` (A / AAAA / TXT)
- `cname_chain` (resolved CNAME path)
- `final_ip_region` (geo-resolved country of final A/AAAA)
- `classification` — one of:
  - `vendor` (matches the vendor's published domain set)
  - `vendor_undeclared` (vendor-owned but not in their public processor list)
  - `processor_declared` (third party present in vendor's published processor list)
  - `processor_undeclared` (third party NOT in vendor's processor list — **this is the Art. 28 problem**)

### How we judge "before the user accepted"

We capture the UI state at each DNS query timestamp. Any query issued before the user actively tapped through the data-processing notice screen is flagged. Annotations: `before_consent: true|false`.

This is the **Art. 13 disclosure gap**. Vendors typically argue that consent isn't required for "core service functionality" — we publish the queries and let the reader judge.

## Pass 2 — 48-hour traffic monitoring

### Why 48h

Most wearables run a daily sync cycle (sleep summary push, firmware-check beacon, ad-SDK heartbeat). 24h would miss the day-2 patterns; 72h doesn't add new endpoints in practice. 48h is the minimum to observe two daily cycles plus the overnight sleep flow.

### What happens

1. After Pass 1, the device is worn normally for 48 hours
2. All traffic flows through mitmproxy with the vendor's pinned cert installed where needed
3. TLS payloads are decrypted where pinning permits; otherwise we log endpoint + payload size + timing
4. Background uploads are timestamped against device state (charging? worn? overnight?)

### What we record

Per session:
- Endpoint (host + path-pattern, query stripped)
- Payload size buckets
- Beacon cadence (median + p95 interval)
- Whether the endpoint appeared in Pass 1
- Bucket assignment (hardware_id / biometric / location / analytics)

### Bucket assignment rules

- **hardware_id**: any endpoint receiving `device_id`, `BLE MAC`, `IMEI`, `advertising_id`, or persistent UUID
- **biometric**: any endpoint receiving HR, SpO2, sleep, activity, stress, body composition
- **location**: any endpoint receiving lat/lon, IP-derived geo, network SSID hashes, dwell-time per region
- **analytics**: Firebase, Crashlytics, Adjust, AppsFlyer, Branch, Segment, Mixpanel, custom ad SDKs

A single endpoint can land in multiple buckets if it receives multiple types.

## Pass 3 — multi-dimensional GDPR analysis

### Per-bucket scoring

For each bucket, score the **disclosure gap** as:

- `0` — every captured flow in this bucket was declared in the vendor's processor list, with a matching Art. 28 agreement, an Art. 13 notice covering it, an Art. 44 mechanism for any non-EU endpoint, and Art. 32 transport security
- `10` — minor formal gap (e.g. CCT not the latest version)
- `25` — partial gap (e.g. one undeclared endpoint, low payload volume)
- `40` — major gap (e.g. one undeclared endpoint with high-cadence beacons)
- `60` — multiple undeclared endpoints, multiple Art. articles affected
- `80+` — vendor's published position materially misrepresents the captured behaviour

The four bucket scores are weighted:

- Biometric: ×1.5 (Art. 9 special category)
- Hardware ID: ×1.0
- Location: ×1.2
- Analytics: ×0.8

Final score = round(sum / total_weight × 100 / 80) — capped at 100.

### Verdict thresholds

| Score range | Verdict | Tier |
|---|---|---|
| 0–25 | PASS | A |
| 26–45 | PASS | B |
| 46–65 | INVESTIGATE | C |
| 66–80 | INVESTIGATE | D |
| 81–100 | REJECTED | — |

### What triggers a hard REJECTED

Any of the following, regardless of score:

- Pass-1 query to a non-vendor domain *before* the data-processing notice was displayed (Art. 13 hard fail)
- Pass-2 endpoint receiving biometric data with no Art. 44 mechanism for a non-EU destination (Art. 44 hard fail)
- Vendor's published processor list explicitly contradicted by captured behaviour (bad-faith disclosure)

## Re-audits

A device is re-audited when:

- Vendor publishes a major firmware or app update
- Vendor publicly contests the verdict and demonstrates remediation
- Six months elapse since the last audit (re-audit cycle)

Old verdicts are preserved in `corpus/` as version history. The currently displayed verdict at <https://ainode.tech> is always the most recent.

## What this methodology does NOT do

- Side-channel power analysis or RF / BLE-level inspection (not in scope; we audit the network behaviour as seen from a normal user's phone)
- Source-code review (we do not have it; we audit observed behaviour)
- iOS-side comparison (Android only; iOS audits are on our 2027 roadmap)
- Rooting or modifying the device firmware

## Cited authorities

- Regulation (EU) 2016/679 — GDPR
- Art. 29 Working Party Opinion 8/2014 on connected devices
- CNIL — *Compteurs communicants : recommandations* (2024 wearable extension)
- BayLDA — guidance on continuous biometric capture (2024)
- Garante per la Protezione dei Dati Personali — provvedimenti su dispositivi indossabili

Full citation set is in [`/corpus/citations.json`](../corpus/citations.json).
