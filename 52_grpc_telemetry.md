
### **52_grpc_telemetry.md**

```
# Chapter 52 – gRPC Telemetry Streaming & Time-Series Normalization

SNMP polling every 60 s can miss microbursts and adds CPU load to edge gear.  
Modern routers (IOS-XR, Junos, EOS) offer **gRPC dial-out telemetry** that
pushes high-frequency counters (100 ms – 1 s) to collectors.  
This chapter shows how to subscribe with `gnmic`, normalize JSON into Prometheus
remote-write, and alert on 5-second jitter.

## 52.1  Key Concepts

| Term | Meaning |
|------|---------|
| _gNMI_ | gRPC Network Management Interface (IETF) |
| _Dial-out_ | Device **pushes** telemetry to collector |
| Path | YANG XPath (e.g. `/interfaces/interface/state/counters`) |
| Encoding | JSON, GPB, or Protobuf-Any |

---

## 52.2  Quick `gnmic` Subscribe

```bash
gnmic -a 203.0.113.10:57400 \
  -u telem -p pass \
  subscribe \
  --path '/interfaces/interface[name=ge-0/0/0]/state/counters' \
  --stream-mode sample --sample-interval 1s \
  --encoding JSON \
  --log --debug
```

Output lines are JSON telemetry updates.

---

## **52.3**  **Converting to Prometheus Remote-Write**

**Create **gnmic.yaml**:**

```
subscriptions:
  sub1:
    paths:
    - /interfaces/interface/state/counters
    mode: sample
    sample-interval: 1s

outputs:
  - type: prometheus_remote_write
    url: http://promdex:19090/write
    batch-size: 100
    workers: 4
    sanitize-metric-names: true
```

Run:

```
gnmic -a 203.0.113.10:57400 -u telem -p pass --config gnmic.yaml
```

Device now streams straight into Prometheus via remote-write.

---

## **52.4**  **Normalization Rules**

in_octets** counter → **gnmi_in_octets_total

Apply **rate()** in PromQL:

```
rate(gnmi_in_octets_total{interface="ge-0/0/0"}[5s]) * 8
```

Yields near-real-time bits-per-second.

---

## **52.5**  **Alert on Jitter**

Recording rule (1-s window):

```
- record: link_bps_1s
  expr: rate(gnmi_in_octets_total[1s]) * 8
```

Alert:

```
- alert: LinkBpsJitter
  expr: abs_deriv(link_bps_1s[10s]) > 1e8
  for: 5s
  annotations:
    summary: "100 Mbps jitter on {{ $labels.interface }}"
```

---

## **52.6**  **Bash Health-Check Wrapper**

```
#!/usr/bin/env bash
collector=promdex:19090/-/ready
if ! curl -sf "$collector"; then
  echo "Prometheus remote-write down; disabling gnmic"
  pkill gnmic
fi
```

Systemd watches this every 30 s.

---

## **52.7**  **Security (TLS + mTLS)**

Generate device cert:

```
openssl req -new -x509 -days 365 -nodes \
  -subj "/CN=router1" -keyout router.key -out router.crt
```

gnmic.yaml**:**

```
tls-ca: /opt/ca.crt
tls-cert: /opt/collector.crt
tls-key: /opt/collector.key
insecure: false
```

Routers trust CA, collector validates device cert CN.

---

## **52.8**  **Exercises**

1. Subscribe to **/interfaces/interface/state/counters** for **all** interfaces and split into per-label metrics using **gnmic output processors**.
2. Benchmark CPU impact versus 10-second SNMP bulk walk by reading **/proc/stat** before/after.
3. Build a CI job that spins up a Juniper vqfx container, starts **gnmic**, and checks that Prometheus remote-write receives at least 100 samples in 30 s.

---

```
---

If you’d still prefer a ZIP once the runtime recovers, just say the word and I’ll attempt another package run.
```
