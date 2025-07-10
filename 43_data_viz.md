
# Chapter43  Data Visualization & Reporting from Bash

Readable graphs turn raw metrics into compelling insights for leadership and
postmortems. Create plots and spreadsheets directly from Bash CI jobs.

---

## 43.1  gnuplot CSV Quick Plot

```bash
gnuplot -e "set datafile separator ','; set terminal png;
            set output 'bw.png'; plot 'bw.csv' using 2 with lines title 'bps'"
```

Attach `bw.png` as CI artifact.

---

## 43.2  termplot for Inline Graphs

```bash
echo "1 10\n2 80\n3 40" | termplot
```

ASCII graph in Slack messages via backticks.

---

## 43.3  Bash  Python Matplotlib via HereDoc

```bash
python - <<'PY'
import pandas as pd, matplotlib.pyplot as plt, sys
df = pd.read_csv('latency.csv', names=['ip','rtt'])
df.plot(kind='bar', x='ip', y='rtt')
plt.tight_layout(); plt.savefig('latency.png')
PY
```

---

## 43.4  excel generation with `csv2xlsx`

```bash
csv2xlsx latency.csv latency.xlsx
```

CI artifacts downloadable for managers.

---

## 43.5  Prometheus  Grafana Snapshot

```bash
curl -s -H "Authorization: Bearer $TOKEN"   -XPOST https://grafana/api/snapshots   -d '{"dashboard":{"id":17},"expires":"3600"}'
```

Returns public link for incident report.

---

## 43.6  Exercises

1. Script that groups latency CSV by site and outputs stacked area graph with
   Matplotlib.  
2. Use gnuplot to plot firmware success % over time using `fw_success.prom`
   metrics.  
3. Build a Bash tool that posts `latency.png` to Slack file upload API.

---
