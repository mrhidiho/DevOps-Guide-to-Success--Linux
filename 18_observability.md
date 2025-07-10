
# Chapter18  Observability: Metrics, Tracing & Alerting

You cant fix what you cant see.  This chapter turns Bash scripts into firstclass
observability citizensexporting Prometheus metrics, emitting OpenTelemetry
traces, and wiring alert rules for firmware rollouts and network tests.

---

## 18.1  Prometheus Textfile Exporter

### Directory Layout

```
/var/lib/node_exporter/text/
  fw_success.prom
  ssh_fail.prom
```

Each file contains one or more lines:

```
fw_success  120
fw_fail     3
```

Node exporter scrapes the directory every 5seconds.

### Bash Metric Writer

```bash
write_metric(){
  local k=$1 v=$2
  printf '%s %s\n' "$k" "$v" >"/var/lib/node_exporter/text/$k.prom"
}
```

Call within your rollout script:

```bash
((success++)); write_metric fw_success $success
```

---

## 18.2  Latency & Bandwidth Probes

`probe_latency.sh`:

```bash
rtt=$(ping -c4 -q $ip | awk -F/ 'END{print $5}')
write_metric "edge_latency_ms{ip=\"$ip\"}" $rtt
```

`probe_bw.sh` using iperf3 JSON:

```bash
bw=$(iperf3 -c $ip -t5 -J | jq .end.sum_received.bits_per_second)
write_metric "edge_bw_bps{ip=\"$ip\"}" $bw
```

---

## 18.3  Alertmanager Rules

`firmware.yaml`:

```yaml
- alert: FirmwareFailures
  expr: increase(fw_fail[10m]) > 0
  for: 5m
  labels:
    severity: critical
  annotations:
    summary: Firmware failures on {{ $labels.instance }}
```

---

## 18.4  Pushgateway for Ephemeral Jobs

For batch script that terminates before Prometheus scrape:

```bash
cat <<EOF | curl --data-binary @- http://push:9091/metrics/job/fw_roll/ip/$ip
# TYPE fw_status gauge
fw_status{ip="$ip"} $status
EOF
```

Clear:

```bash
curl -X DELETE http://push:9091/metrics/job/fw_roll/ip/$ip
```

---

## 18.5  OpenTelemetry Bash Tracing

Install `otel-cli`:

```bash
curl -sSL https://github.com/equinix-labs/otel-cli/releases/download/v0.8.0/otel-cli_0.8.0_Linux_x86_64.tar.gz | tar xz
```

Wrap stages:

```bash
otel span --name "stage_fw" -- ./stage_fw.sh
otel span --name "apply_fw" -- ./apply_fw.sh
```

Exporter endpoint:

```bash
export OTEL_EXPORTER_OTLP_ENDPOINT=http://collector:4318
```

Traces appear in Grafana Tempo  Grafana.

---

## 18.6  Grafana Dashboards

Key panels:

| Panel | Query |
|-------|-------|
| Firmware success rate | `(fw_success)/(fw_success+fw_fail)` |
| Avg Latency by POP | `avg(edge_latency_ms) by (site)` |
| Open Ports over Time | `sum(open_ports_total) by (port)` |

---

## 18.7  Bash Helper Library (`lib/metrics.sh`)

```bash
metric(){
  local name=$1 value=$2 labels=$3
  printf '%s%s %s\n' "$name" "$labels" "$value" >"/tmp/$name.prom.$$"
  mv "/tmp/$name.prom.$$" "/var/lib/node_exporter/text/$name.prom"
}
# Usage:
metric "ssh_failed_total" 5 "{host=\"$host\"}"
```

Handles atomic write via temp + mv.

---

## 18.8  Alert Testing with amtool

```bash
amtool alert add test_alert severity=critical instance=lab description="test"
```

Silence:

```bash
amtool silence add matchers=alertname=test_alert -d 1h -c "testing"
```

---

## 18.9  Exercises

1. Build a Bash script that traces each firmware stage (stage, verify, apply)
   using `otel-cli`, linking spans with `--parent-id`.  
2. Write a Pushgateway wrapper that expires metrics after 24h
   (`curl -X DELETE `).  
3. Create a Grafana alert that triggers if average bandwidth drop
   (`edge_bw_bps`) exceeds 30% in the last hour.

---
