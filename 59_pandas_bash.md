
```
# Chapter 59 – pandas + Bash for Large CSV & Log Crunching

`awk` and `sort -k` get you far, but once daily ISP logs hit hundreds of
megabytes, a single `pandas` block—embedded inside Bash—can calculate joins,
rolling windows, and percentiles in seconds with readable code.

_Last updated: 2025-07-10_

---

## 59.1 Pattern: Bash → Here-Doc → pandas

```bash
python - <<'PY'
import pandas as pd, sys
df = pd.read_csv('bw.csv', names=['ts','bps'])
# 5-minute rolling 95th-percentile
df['ts'] = pd.to_datetime(df['ts'], unit='s')
pct95 = df.set_index('ts')['bps'].rolling('5min').quantile(0.95)
pct95.to_csv('bw_95p.csv')
PY
echo "Done."
```

Bash continues after Python block.

---

## **59.2 Zero-Copy via** ****

## **stdin**

## **/**

## **stdout**

```
zcat big_logs.csv.gz | \
python - <<'PY'
import pandas as pd, sys, io
df = pd.read_csv(sys.stdin, names=['ip','url','ms'], usecols=[0,2,4])
print(df.groupby('ip')['ms'].mean().to_csv(index=True))
PY | sort -t, -k2 -nr | head
```

No temp files; stream in, stream out.

---

## **59.3 Memory-Efficient Tricks**

```
df = pd.read_csv('logs.csv',
                 dtype={'site_id':'category','status':'int16'},
                 parse_dates=['ts'],
                 chunksize=5_000_000)
for chunk in df:
    ...
```

* **chunksize** = iterate generator, constant RAM
* **category** turns strings → int codes.

---

## **59.4 Writing Feather / Parquet for Bash**

```
df.to_feather('snr.feather')
```

**feather** loads 10× faster than CSV.

**Bash uses **python -c "import pandas as pd,sys; print(pd.read_feather(sys.argv[1]).head())" snr.feather**.**

---

## **59.5 Example: Join Modem SNR with Outage Window**

```
snr = pd.read_csv('snr_daily.csv')           # mac,snr
out = pd.read_csv('outage.csv')              # mac,reason,start,end
merged = snr.merge(out[['mac','reason']], on='mac', how='left')
print(merged.groupby('reason')['snr'].mean())
```

Single line replaces multi-step **join | awk**.

---

## **59.6 CLI Wrappers (**

## **pdpipe**

## ** /** ****

## **petl**

## **)**

* **petl** – SQL-like: python -m petl select table.csv --where "snr>30"
* **pandas-cli** – pandas some.csv --describe**.**

Great for one-liners inside Bash loops.

---

## **59.7 Prometheus Export**

Generate histogram buckets:

```
hist = pd.cut(df['ms'], bins=[0,50,100,200,500,1000]).value_counts().sort_index()
for lbl,val in hist.items():
    b = lbl.right
    print(f"api_latency_bucket{{le=\"{b}\"}} {val}")
```

**Redirect to **/var/lib/node_exporter/text/api_latency.prom**.**

---

## **59.8 Exercises**

1. Convert daily SNMP CSV (1 GB) into Parquet partitioned by **date=**; query last hour in <2 s.
2. Build a Bash script that pipes **zgrep** output into a pandas block to compute per-site 99th RTT and prints top 10 sites.
3. Use **pd.merge_asof** to join modem SNR samples with telemetry spikes within ±30 s window; output suspect MACs for Expect auto-session.

---

```
Let me know if you want additional hybrid chapters or a packaging retry once the sandbox issues clear up.
```
