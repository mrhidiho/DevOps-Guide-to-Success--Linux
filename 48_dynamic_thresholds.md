
# Chapter48  Dynamic Threshold Alerting with Prometheus Recording Rules

Static >80% CPU alarms create noise at 3AM yet miss genuine spikes at noon.
**Dynamic thresholds**rolling median and median absolute deviation (MAD)
adapt to diurnal ISP traffic patterns, making alerts both quieter _and_
smarter.

_Last updated: 2025-07-10_

---

## 48.1  Rolling MAD Concept

* **Median (`med`)**  center of last _N_ samples  
* **MAD**  median of `|x - med|` over window  
* **Alert** when current value exceeds `med + kMAD`  

For normallydistributed values, `k5`  99% confidence.

---

## 48.2  Prometheus Recording Rules

`dynamic.rules.yaml`:

```yaml
groups:
- name: dynamic-thresholds
  interval: 1m
  rules:
  - record: link_bw_med
    expr: quantile_over_time(0.5, rate(ifOutOctets{interface="edge0"}[5m]) * 8)[1h:]
  - record: link_bw_mad
    expr: quantile_over_time(
            0.5,
            abs(rate(ifOutOctets{interface="edge0"}[5m]) * 8 - link_bw_med))[1h:]
  - record: link_bw_dyn_thresh
    expr: link_bw_med + 5 * link_bw_mad
```

Prometheus now stores `link_bw_dyn_thresh` updated every minute.

---

## 48.3  Alertmanager Rule Using Dynamic Threshold

```yaml
- alert: DynamicHighBandwidth
  expr: rate(ifOutOctets{interface="edge0"}[5m]) * 8 > link_bw_dyn_thresh
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "Dynamic BW alert edge0"
    runbook: "https://kb.company/bw-dynamic"
```

---

## 48.4  Bash Script to AutoSilence During Maintenance

`auto_silence.sh`:

```bash
#!/usr/bin/env bash
if curl -sf http://noc/api/maintenance | jq -e '.active'; then
  amtool silence add -d 2h -c "Maintenance" matchers=alertname=DynamicHighBandwidth
fi
```

CI schedules via cron timer.

---

## 48.5  Grafana Visualization

Panel query:

```
rate(ifOutOctets{interface="edge0"}[5m])*8,
link_bw_dyn_thresh
```

Add alert state annotations for context.

---

## 48.6  Exercises

1. Modify recording rule to calculate MAD per `instance` label for multiedge
   panels.  
2. Script that increases multiplier `k` automatically during NFL SuperBowl
   based on calendar API.  
3. Compare falsepositive rate of static 90% threshold vs dynamic MAD over a
   week of logs.

---
