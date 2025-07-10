# Chapter51  Advanced BGP FlowSpec & RealTime Blackholing

**BGP FlowSpec** lets an ISP push granular ACLs (match + action) across the
backbone within secondsideal for DDoS scrub or instant RTBH (RemoteTriggered
BlackHole) mitigation. This chapter shows how to craft FlowSpec NLRI, publish
with **ExaBGP**, and autoexpire rules via Bash.

_Last updated: 2025-07-10_

---

## 51.1  FlowSpec Rule Anatomy

| Component | Example | Purpose |
|-----------|---------|---------|
| Match     | `dst 203.0.113.55/32` | Destination IP |
|           | `protocol = udp`      | Layer4 proto 17 |
| Action    | `discard`             | Drop packet |
| Optional  | `redirect-to-nexthop 192.0.2.254` | RTBH to sinkhole |

FlowSpec is carried in MPBGP AFI 1 / SAFI 133.

---

## 51.2  ExaBGP Minimal Config

`exabgp.conf`:

```
neighbor 10.0.0.1 {{
  router-id 10.0.0.2;
  local-address 10.0.0.2;
  local-as 65055;
  peer-as 65055;
}}
process send-fs {{
   encoder json;
   run /opt/scripts/fspipe.sh;
}}
```

ExaBGP pipes FlowSpec announcements from `fspipe.sh` STDOUT.

---

## 51.3  Bash Generator for RTBH

```bash
#!/usr/bin/env bash
ip=$1 ttl=${{2:-3600}}
cat <<EOF
announce flow route {{ dst ${{ip}}/32 protocol ip; discard; next-hop self; community [ 65535:666 ]; }} meta {{ name "AUTO-RTBH" timeout ${{ttl}}; }}
EOF
```

Usage:

```bash
./rtbh.sh 203.0.113.55 1800 | nc -q1 localhost 5000
```

---

## 51.4  Automatic Feed Ingestion

```bash
tail -F /var/log/ids/alerts.log |   awk '/DDoS/ {{print $NF}}' | while read vip; do
    ./rtbh.sh ${{vip}} 300 | nc -q1 localhost 5000
  done
```

Autoblackholes IPs flagged by IDS for 5minutes.

---

## 51.5  Expiry & GarbageCollection

ExaBGP can autowithdraw on `meta timeout`. For routers without,
schedule Bash cleanup:

```bash
exabgpcli show adj-rib out extensive |   jq -r '.neighbor.message[] | select(.nlri.flow.action=="discard") | .nlri'   > current.json
```

---

## 51.6  Validation on Juniper / Cisco

*Juniper:*

```bash
show route flow terse | match 203.0.113.55
```

*Cisco (XR):*

```bash
show bgp ipv4 flowspec | include 203.0.113.55
```

---

## 51.7  Prometheus Metrics

```bash
exabgpcli show adj-rib out | grep 'discard' | wc -l |   xargs -I{{}} echo "flowspec_active_rules {{}} "   > /var/lib/node_exporter/text/flowspec.prom
```

---

## 51.8  Exercises

1. Extend `rtbh.sh` to accept `/24` prefixes and push ratelimit (`rate 100`) action instead of discard.  
2. Write a cron job that withdraws any FlowSpec rule older than 30minutes by parsing ExaBGP `history` directory.  
3. Build CI tests using `vagrant + FRRouting` to assert FlowSpec NLRI propagates and traffic drops via `pktgen`.  

---
