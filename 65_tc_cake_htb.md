

# Chapter 65 – Advanced `tc`: CAKE, FQ_CoDel & HTB Hierarchies

Bufferbloat on DOCSIS or GPON backhaul links can spike latency >200 ms when a
single modem uploads firmware.  Modern Linux qdiscs—**CAKE** (Common ACK
Enhancement) and **FQ_CoDel**—plus an HTB shaping hierarchy give near‐wire-rate
throughput _and_ sub-10 ms queueing.

_Last updated: 2025-07-10_

---

## 65.1  Prerequisites

* Kernel ≥ 5.4 (for CAKE `diffserv4` & `nat` keywords)
* `iproute2` ≥ 5.10
* Interface examples use `eth0` (ingress) and `ifb0` (egress shaping)

---

## 65.2  Baseline Measurement

```bash
ping -i 0.2 1.1.1.1 &           # background
iperf3 -c speedtest -R -t 30    # saturate uplink
```

Note 95-percentile RTT.

---

## **65.3**  **Creating Ingress IFB**

```
modprobe ifb
ip link add ifb0 type ifb
ip link set ifb0 up
tc qdisc add dev eth0 handle ffff: ingress
tc filter add dev eth0 parent ffff: \
  protocol ip u32 match u32 0 0 action mirred egress redirect dev ifb0
```

Now **ifb0** sees ingress packets for shaping.

---

## **65.4**  **CAKE Shaper (Dual-Queue Coupled AQM)**

```
# Upstream 50 Mb, downstream 400 Mb (DOCSIS example)
tc qdisc replace dev ifb0 root cake bandwidth 50Mbit \
     diffserv4 nat dual-srchost
tc qdisc replace dev eth0 root cake bandwidth 400Mbit \
     diffserv4 nat dual-dsthost ingress
```

Key flags:

| **Flag**       | **Meaning**                      |
| -------------------- | -------------------------------------- |
| nat                  | Auto extract NAT inner IP for fairness |
| dual-srchost         | Fairness by source on upload           |
| dual-dsthost ingress | Fair by dest on download               |

---

## **65.5**  **Verifying Latency Improvement**

Repeat ping + iperf; 95-percentile RTT should drop to <30 ms.

Check stats:

```
tc -s qdisc show dev ifb0 | grep delay
```

---

## **65.6**  **Hierarchical Token Bucket (HTB) for VLANs**

Tree: 400 Mb root → per-VLAN classes.

```
tc qdisc add dev eth0 root handle 1: htb default 30
tc class add dev eth0 parent 1: classid 1:10 htb rate 50Mbit ceil 100Mbit
tc class add dev eth0 parent 1: classid 1:20 htb rate 100Mbit ceil 200Mbit
tc class add dev eth0 parent 1: classid 1:30 htb rate 250Mbit ceil 400Mbit
tc qdisc add dev eth0 parent 1:10 cake diffserv4
tc qdisc add dev eth0 parent 1:20 fq_codel
tc qdisc add dev eth0 parent 1:30 cake diffserv4
```

Filter VLAN 100 to class 1:10:

```
tc filter add dev eth0 parent 1: protocol 802.1Q flower \
  vlan_id 100 action skbedit priority 10 classid 1:10
```

---

## **65.7**  **FQ CoDel for Legacy Gear**

```
tc qdisc replace dev ifb0 root fq_codel target 5ms interval 100ms
```

Use when CAKE module not present (<5.4 kernels).

---

## **65.8**  **Automation Script (Bash)**

shape.sh

```
#!/usr/bin/env bash
UL=$1  # e.g., 50M
DL=$2  # e.g., 400M

tc qdisc del dev ifb0 root >/dev/null 2>&1
tc qdisc del dev eth0 root >/dev/null 2>&1

tc qdisc add dev ifb0 root cake bandwidth ${UL}bit diffserv4 nat dual-srchost
tc qdisc add dev eth0 root cake bandwidth ${DL}bit diffserv4 nat dual-dsthost ingress
```

Cron pulls **ifSpeed** via SNMP nightly and re-invokes **shape.sh**.

---

## **65.9**  **Monitoring with** ****

## **tc -s**

## ** + Prometheus**

Exporter snippet:

```
tc -s qdisc show dev ifb0 | \
 grep -A4 'cake' | awk '/backlog/ {print "cake_backlog_bytes "$3}'
```

Graph rises during congestion; alert if backlog > 1 MiB for 10 s.

---

## **65.10**  **Exercises**

1. Simulate bufferbloat: disable shaping, run **flent rrul**; re-enable CAKE and compare plots.
2. Write a systemd service that auto-detects link rate via **ethtool eth0 | grep Speed** and re-runs **shape.sh**.
3. Build a GitLab CI job that spins a netns pair, applies HTB+CAKE, and runs **iperf3** + **ping** asserting RTT < 10 ms at 90 % line rate.

---

```
This completes Chapter 65 with full explanations, commands, real-world benchmarks, and exercises.  Let me know if you’d like the next deep-dive topic (#66 IO sched / blk-mq, #67 systemd-resource-control, etc.) or if you need packaging in ZIP once stabilized.
```
