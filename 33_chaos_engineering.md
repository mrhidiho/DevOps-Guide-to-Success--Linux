
# Chapter33  Advanced Chaos Engineering in Bash

Practice failing components _before_ they fail you. This chapter shows how to
inject latency, packet loss, CPU hogs, and process crashes with **tc netem**,
**stressng**, and iptablesall orchestrated by Bash with automatic rollback.

---

## 33.1  Safety Nets

* Run chaos only in **nonpeak windows** (CSV `maintenance.csv`).  
* Implement a **killswitch file** (`/tmp/chaos_disable`) checked by every
  loop iteration.  
* Emit Prometheus metric `chaos_active 1` so NOC dashboards know.

---

## 33.2  Latency Injection with `tc netem`

```bash
tc qdisc add dev eth0 root netem delay 100ms 20ms distribution normal
```

* 100ms mean, 20ms stddev.  
Rollback:

```bash
tc qdisc del dev eth0 root
```

Automate:

```bash
for ip in $(<edges.txt); do
  ssh $ip "tc qdisc add dev eth0 root netem delay 80ms loss 1%" &
done; wait; sleep 300;  # 5 min chaos
parallel -j0 ssh {} "tc qdisc del dev eth0 root" ::: $(<edges.txt)
```

---

## 33.3  CPU & Memory Stress

```bash
stress-ng --cpu 4 --cpu-load 90 --timeout 120s &
stress-ng --vm 2 --vm-bytes 2G --timeout 120s &
wait
```

Write wrapper script that records `mpstat` before/after.

---

## 33.4  Random Process Kill Monkey

```bash
pids=( $(pgrep -u netops) )
victim=${pids[$RANDOM % ${#pids[@]}]}
kill -STOP $victim; sleep 60; kill -CONT $victim
```

Suspend instead of kill first to gauge impact.

---

## 33.5  Network Partition Simulation

```bash
iptables -I OUTPUT -d $peer_ip -j DROP
sleep 120
iptables -D OUTPUT -d $peer_ip -j DROP
```

For distributed DB testing (etcd, Mongo replica).

---

## 33.6  Chaos Controller Script

`chaos_run.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail
[[ -e /tmp/chaos_disable ]] && exit 0

case $1 in
  latency) tc qdisc add dev eth0 root netem delay 70ms ;;
  loss)    tc qdisc add dev eth0 root netem loss 5% ;;
  cpu)     stress-ng --cpu 4 --timeout 60s &;;
esac
sleep ${DURATION:-60}
./chaos_cleanup.sh $1
```

Cleanup script removes qdisc or kills stressng PID.

---

## 33.7  Integrating with CI/CD

* Run chaos after canary firmware passes functional tests.  
* Alert if `fw_success` drops below 95% during chaos window.

GitLab stage:

```yaml
chaos_test:
  script:
    - ./chaos_run.sh latency
```

---

## 33.8  Exercises

1. Write a Bash script that picks **10%** of nodes randomly for chaos and
   tags them in Prometheus with `chaos_group="treatment"`.  
2. Implement a check that rolls back chaos if packet loss exceeds 10% for
   more than 30seconds (uses `ping -I`).  
3. Extend `chaos_run.sh` to support disk fill (`fallocate -l 5G /tmp/fill`)
   and cleans up safely.

---
