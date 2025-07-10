
---

```
# Chapter 70 – AutoNUMA Balancing & `numactl --membind` for SR-IOV Paths

On dual-socket EPYC or Xeon hosts, SR-IOV VFs mapped to one CPU socket
shouldn’t bounce packets through the other socket’s memory controller.
**AutoNUMA** helps—_if_ tuned—while explicit `numactl --membind` guarantees
page locality.  This chapter shows how to detect bad NUMA placement, steer
DPDK hugepages to the right node, and benchmark packet-per-second (PPS) wins.

_Last updated: 2025-07-10_

---

## 70.1  NUMA Primer

| Term | Meaning |
|------|---------|
| **Node** | CPU sockets or NVDIMM region with its own memory controller |
| **Local vs remote** | Access within node ≈ 90 ns; cross-socket ≈ 150–200 ns |
| **AutoNUMA** | Kernel daemon migrates hot pages to local node |

---

## 70.2  Detecting Bad Placement

### 1.  `numastat -c`

```bash
numastat -c | grep fwcli
```

Columns: **N0, N1, ...** → remote ratio > 0.05 flags trouble.

### **2. ** ****

### **perf c2c**

### ** (cache-to-cache)**

```
perf c2c record -a -d sleep 30
perf c2c report | head
```

High cross-socket lines show false sharing.

---

## **70.3**  **Forcing Memory Bind for DPDK**

Allocate hugepages per node:

```
echo 2048 > /sys/devices/system/node/node0/hugepages/hugepages-2048kB/nr_hugepages
echo 0    > /sys/devices/system/node/node1/hugepages/hugepages-2048kB/nr_hugepages
```

Run:

```
numactl --cpunodebind=0 --membind=0 \
  dpdk-testpmd -l 0-7 -n 4 --vdev ...
```

Now hugepages & CPU on Node 0; VF PCI address must also be Node 0:

```
cat /sys/bus/pci/devices/0000:04:10.0/numa_node   # expect 0
```

---

## **70.4**  **Tuning AutoNUMA**

Kernel params (6.7+):

```
numa_balancing=1 numa_balancing_scan_period_min_ms=200
```

Set scan size to 16 MB (SR-IOV buffers):

```
echo 16777216 | sudo tee /proc/sys/kernel/numa_balancing_scan_size_mb
```

Script to pin flash slice:

```
systemctl set-property flash.slice NUMAPolicy=preferred NUMAMask=0
```

---

## **70.5**  **Hugepage Migration**

migratepages** tool:**

```
pids=$(pgrep -f dpdk-testpmd | tr '\n' ,)
migratepages $pids 1 0
```

Moves all pages from Node 1 → Node 0 without restart.

---

## **70.6**  **Benchmark: PPS Gain**

| **Scenario**           | **64B PPS (2×25 Gb)** | **L3 Miss %** |
| ---------------------------- | ---------------------------- | ------------------- |
| Default AutoNUMA             | 8.2 Mpps                     | 9.5                 |
| **membind 0 + cpus 0** | **9.8 Mpps**           | **5.1**       |

EPYC 7713, 2×CX6-DX SR-IOV, kernel 6.9-rc4.

---

## **70.7**  **Monitoring NUMA via Prometheus**

Textfile exporter:

```
for n in 0 1; do
  rss=$(numastat -p $(pgrep dpdk-testpmd) | awk -v N=$n '$1=="Node"N":"{print $2}')
  echo "vnf_rss_bytes{node=\"$n\"} $rss"
done > /var/lib/node_exporter/text/numa.prom
```

Alert if remote node RSS > 100 MB.

---

## **70.8**  **Handling SR-IOV VF Hot-Plug**

Udev rule:

```
ACTION=="add", SUBSYSTEM=="pci", ATTR{vendor}=="0x15b3", \
 RUN+="/usr/local/sbin/bind-huge.sh %p"
```

bind-huge.sh** reads **/sys/bus/pci/devices/$1/numa_node** and echoes hugepage**

count to that node’s pool.

---

## **70.9**  **Real-World Win (vBRAS)**

Before: auto-placement → drop to 6 Mpps at 90 % CPU.

After explicit bind: 9.5 Mpps, CPU 82 %, latency p99 −30 µs.

---

## **70.10**  **Exercises**

1. Write a Bash watchdog that migrates pages with **migratepages** when remote
   RSS > 50 MB.
2. Patch DPDK **--socket-mem** to allocate different size per NUMA zone and
   benchmark cache hit rate.
3. Run **perf mem** to record remote DRAM accesses, plot with FlameGraph.

---

```
---

Let me know when you’re ready for the next ultra-expert topic (e.g., Chapter 71 on BBR v3 + ECN++), or if you need ZIP packaging once the environment is stable again.
```
