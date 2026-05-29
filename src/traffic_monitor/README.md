# Pass 2 — 48-hour traffic monitoring

Records the device's network behaviour over 48 hours of normal use, categorises every endpoint into one of four buckets (hardware_id / biometric / location / analytics), and produces per-bucket counts and cadence stats.

## Setup

- mitmproxy in transparent mode on the audit AP
- Vendor's pinning CA installed where TLS pinning is bypassable; otherwise log endpoint + payload size + timing only
- Device worn through one charge cycle + at least one overnight sleep session

## Bucket assignment

| Bucket | Trigger condition | Weight in Pass 3 |
|---|---|---|
| `hardware_id` | Endpoint payload includes `device_id` / `BLE_MAC` / `IMEI` / persistent UUID | 1.0 |
| `biometric` | Endpoint payload includes HR / SpO2 / sleep / activity / stress / body comp | 1.5 (Art. 9) |
| `location` | Endpoint payload includes lat/lon / IP-geo / SSID hash / dwell time | 1.2 |
| `analytics` | Endpoint matches known analytics SDK domain set | 0.8 |

A single endpoint can land in multiple buckets if it carries multiple types.

## Output

```json
{
  "audit_window_hours": 48,
  "device_state_log": "states.csv",
  "endpoints": [
    {
      "host": "api.example-vendor.com",
      "path_pattern": "/v2/data/hr_continuous",
      "buckets": ["biometric"],
      "payload_size_p50": 412,
      "payload_size_p95": 1840,
      "beacon_cadence_min": 5.0,
      "in_pass1": true,
      "vendor_declared": true,
      "non_eu_endpoint": false
    }
  ]
}
```

## Run

```bash
mitmdump -s record.py --set window_hours=48 --set output=pass2.json
```

The actual `record.py` runs inside AINode's audit pipeline. We publish the schema and bucket rules so any researcher with mitmproxy + 48 hours can reproduce.
