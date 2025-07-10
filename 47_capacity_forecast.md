
# Chapter47  MachineLearningDriven Capacity Forecasting

Predict link saturation **before** customers feel congestion. This chapter
builds a pipelinepure Bash + one Python stepthat pulls bandwidth metrics
from InfluxDB/Prometheus, feeds them into a Prophet timeseries model, and
alerts when projected utilization crosses 80% within 90days.

_Last updated: 2025-07-10_

---

## 47.1  Data Extraction (Prometheus)

```bash
start=$(date -d '-180days' +%s)
end=$(date +%s)
step=3600  # hourly

curl -sG 'http://prom/api/v1/query_range' \
  --data-urlencode "query=rate(ifOutOctets{interface='edge0'}[5m])*8" \
  --data-urlencode "start=$start" --data-urlencode "end=$end" \
  --data-urlencode "step=$step" | jq -r '.data.result[0].values[] | @csv'   >bw.csv
```

Converts to `timestamp,bps`.

---

## 47.2  Prophet Forecast via Embedded Python

```bash
python - <<'PY'
import pandas as pd, sys, json, subprocess, os
from prophet import Prophet
df = pd.read_csv('bw.csv', names=['ds','y'])
df['ds'] = pd.to_datetime(df['ds'], unit='s')
m = Prophet(changepoint_prior_scale=0.05)
m.fit(df)
future = m.make_future_dataframe(periods=24*90, freq='H')
forecast = m.predict(future)
forecast[['ds','yhat_upper']].tail(24*90).to_csv('forecast.csv', index=False)
PY
```

Installs Prophet once in CI image.

---

## 47.3  Threshold Evaluation

```bash
alert_ts=$(awk -F, '$2>8e8 {print $1; exit}' forecast.csv)
if [[ -n $alert_ts ]]; then
  echo "Capacity will exceed 800Mbps on $alert_ts"
  slack_notify "Link edge0 forecast to hit 80% on $alert_ts"
fi
```

---

## 47.4  Grafana Forecast Panel

Upload `forecast.csv` to Loki or use Grafana CSV datasource; overlay actual
vsforecast.

---

## 47.5  Automation in GitLab CI

```yaml
forecast:
  image: python:3.12
  before_script:
    - pip install prophet pandas matplotlib
  script:
    - ./forecast.sh
  artifacts:
    paths: [forecast.csv]
```

Pipeline fails if forecast breach <30days (change gate).

---

## 47.6  Exercises

1. Modify script to pull multiple interfaces and output top 10 worst weeks
   to exhaust.  
2. Add seasonality (weekly traffic cycle) to Prophet model with
   `add_seasonality`.  
3. Generate an XLSX report (Chapter43) combining actual + forecast and push
   to Slack.

---
