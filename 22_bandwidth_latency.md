
# Chapter22  Bandwidth & Latency Testing in Bash

Validating network performance after firmware or config changes is critical.
This chapter shows how to automate **ping, iperf3, TWAMP, and MTR** tests,
store results in CSV/Prometheus, and set thresholds for alerting.

---

## 22.1  ICMP Latency (ping)

### Basic Loop

```bash
while read -r ip; do
  rtt=$(ping -c4 -q $ip | awk -F/ 'END{print $5}')
  printf '%s,%s\n' "$ip" "$rtt" >>latency.csv
done <edges.txt
```

Adds median RTT column to CSV for trending.

### Prometheus Export

```bash
write_metric(){ printf 'latency_ms{ip="%s"} %s\n' "$1" "$2" >"/var/lib/node_exporter/text/lat_$1.prom"; }
```

---

## 22.2  iperf3 Throughput Tests

### Server

```bash
iperf3 -s -D  # runs as daemon
```

### Client Script

```bash
iperf_test(){
  local ip=$1
  res=$(iperf3 -J -c $ip -t 10)
  bw=$(echo "$res" | jq .end.sum_received.bits_per_second)
  echo "$ip,$bw" >>bw.csv
}
export -f iperf_test
parallel -j20 iperf_test ::: $(<edges.txt)
```

---

## 22.3  TWAMP Light (twamp) for OneWay Delay

Install tool:

```bash
apt-get install -y twamp
```

Run:

```bash
twamp-song -c $ip -i 100 -p 862 --format csv >>owd.csv
```

Parse:

```bash
awk -F, '{sum+=$7} END{print "avg_owd="sum/NR}'
```

---

## 22.4  MTR for Path Analysis

```bash
mtr -n -r -c 50 $ip --json | jq '.report.hubs[] | {hop,loss,avg}'
```

Diff against baseline route to detect MPLS changes.

---

## 22.5  Bandwidth Threshold Alert

```bash
awk -F, '$2<100e6{print $1,$2}' bw.csv >below_100M.txt
[[ -s below_100M.txt ]] && mail -s "Low BW!" noc@example.com <below_100M.txt
```

---

## 22.6  Grafana Panel Data

Convert CSV to Prometheus:

```bash
while IFS=, read -r ip bw; do
  echo "iperf_bps{ip=\"$ip\"} $bw"
done <bw.csv >/var/lib/node_exporter/text/iperf.prom
```

---

## 22.7  Exercises

1. Combine ping + iperf3 results into a single CSV (`ip,rtt,bw`) and sort by
   descending RTT.  
2. Write a TWAMP client wrapper that retries if packet loss >10%.  
3. Build a Bash script that runs `mtr` to **every default gateway** in
   devices.csv and alerts if any hop exceeds 50ms.

---
