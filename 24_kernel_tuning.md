
# Chapter24  Kernel & sysctl Performance Tuning

When 5Gbps of DOCSIS traffic suddenly drops to 500Mbps after a firmware
push, the culprit is often a kernel buffer or offload flagnot your Bash
logic. This chapter arms you with **sysctl**, **ethtool**, and IRQ tuning
recipes to squeeze the most out of router and server kernels.

---

## 24.1  Key TCP sysctl Parameters

| Setting | Default | Optimized | Why change? |
|---------|---------|-----------|-------------|
| `net.core.rmem_max` | 212kB | 16MB | Raise receive buffer for highlat links |
| `net.core.wmem_max` | 212kB | 16MB | Raise send buffer |
| `net.ipv4.tcp_window_scaling` | `1` |  | Ensure enabled for >64kB windows |
| `net.ipv4.tcp_mtu_probing` | `0` | `1` | Auto MTU discovery behind tunnels |
| `net.ipv4.tcp_congestion_control` | `cubic` | `bbr` | BBR improves throughput on long RTT |

Apply live:

```bash
sysctl -w net.core.rmem_max=$((16*1024*1024))
```

Persist in `/etc/sysctl.d/99-net.conf`.

---

## 24.2  IRQ Affinity & RSS

Check current mapping:

```bash
grep . /proc/irq/*/smp_affinity_list
```

Set NIC IRQs to CPU03, storage to 47:

```bash
echo 0-3 | tee /proc/irq/$(grep eth0 -l /proc/irq/*/action)/smp_affinity_list
```

Automate in Bash loop using `lscpu | awk` to detect NUMA nodes.

---

## 24.3  ethtool Offload Flags

List:

```bash
ethtool -k eth0
```

Disable LRO/GRO for latencysensitive traffic:

```bash
ethtool -K eth0 gro off lro off
```

Verify with `sar -n DEV 1` throughput numbers before/after.

---

## 24.4  Tuning Receive Packet Steering (RPS)

Set CPU mask:

```bash
echo f > /sys/class/net/eth0/queues/rx-0/rps_cpus   # CPUs 03
```

Adjust flow entries:

```bash
echo 32768 > /proc/sys/net/core/rps_sock_flow_entries
```

---

## 24.5  Benchmark Script

`bench_rtt_bw.sh`:

```bash
before=$(iperf3 -c $srv -t5 -J | jq .end.sum_received.bits_per_second)
apply_tuning
after=$(iperf3 -c $srv -t5 -J | jq .end.sum_received.bits_per_second)
printf 'BW %s -> %s (%.1f%%)\n' $before $after   $(awk "BEGIN{print ($after-$before)*100/$before}")
```

Logs improvement to CSV.

---

## 24.6  Stress Testing with stressng

```bash
stress-ng --cpu 8 --io 4 --vm 2 --vm-bytes 1G --timeout 60s
```

Monitor with `vmstat 1` and `dstat`.

---

## 24.7  Automating sysctl Across Fleet

```bash
for ip in $(<edges.txt); do
  ssh $ip "sysctl -w net.ipv4.tcp_rmem='4096 87380 16777216'" &
done; wait
```

Use Ansible `sysctl` module for idempotence.

---

## 24.8  Exercises

1. Script that detects TCP retransmissions via `/proc/net/netstat` and boosts
   `net.ipv4.tcp_retries2` accordingly.  
2. Write a Bash loop that toggles BBR and records throughput deltas in CSV.  
3. Build a systemd service that sets IRQ affinity on boot, reading layout
   from a JSON file per hardware SKU.

---
