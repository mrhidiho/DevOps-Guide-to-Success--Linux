
# Chapter 63 – Transparent Huge Pages, Memory Compaction & `MADV_COLLAPSE`

Large (2 MiB) pages cut TLB misses on packet-hungry VNFs, yet mis-tuned THP
can steal CPU during compaction or inflate memory on logging pods.  
This chapter shows you when to enable THP, how to trigger compaction with
`madvise(MADV_COLLAPSE)`, and how to confirm latency wins with `perf stat`.

_Last updated: 2025-07-10_

---

## 63.1  Transparent Huge Pages (THP) Modes

```text
/sys/kernel/mm/transparent_hugepage/enabled
[always] madvise never
```

* **always** – kernel tries to promote any anon 4 k page to 2 MiB
* **madvise** – only if process calls madvise(..., MADV_HUGEPAGE)
* **never** – disable THP

 *ISP Practice* : leave “always” on heavy packet-processing VNFs, but switch to

“madvise” on memory-fragmented log servers.

---

## **63.2**  **Checking THP Hit Rate**

```
grep thp_ /proc/vmstat | column -t
```

Key fields:

| **vmstat key** | **Meaning**           |
| -------------------- | --------------------------- |
| thp_fault_alloc      | THP allocated on page-fault |
| thp_collapse_alloc   | Compaction collapsed pages  |
| thp_split            | THP broken back to 4 k      |

**Aim for **thp_split / thp_fault_alloc < 0.05**.**

---

## **63.3**  **Bash Script to Switch Modes**

```
#!/usr/bin/env bash
set -e
echo madvise | sudo tee /sys/kernel/mm/transparent_hugepage/enabled
echo 10 > /proc/sys/vm/compact_memory   # one-shot compaction
```

Run during change window; confirm with **grep thp_** delta.

---

## **63.4 ** ****

## **MADV_COLLAPSE**

## ** for User-Space Control**

**Linux 6.7 adds **madvise(addr, len, MADV_COLLAPSE)**.**

Python demo (requires **ctypes**):

```
import ctypes, mmap, os
libc = ctypes.CDLL("libc.so.6")
page = mmap.mmap(-1, 2<<20)          # 2 MiB
# fill 2 MiB with data
page.write(b"\0"* (2<<20))
MADV_COLLAPSE = 25
libc.madvise(ctypes.c_void_p(ctypes.addressof(ctypes.c_char.from_buffer(page))),
             ctypes.c_size_t(2<<20), MADV_COLLAPSE)
```

Kernel collapses populated 4 k chunks into a single THP—no global compaction.

---

## **63.5**  **Measuring Benefit with** ****

## **perf stat**

```
perf stat -e dTLB-load-misses,cycles \
  ./vnf_benchmark --duration 30
```

Compare **cycles/TLB-miss** before vs after enabling THP; 10-20 % drop typical

on Intel Ice Lake.

---

## **63.6**  **Latency Pitfall: Compaction Stalls**

/proc/pressure/memory** (Chapter 61):**

```
some avg10=8.00 ...
full avg10=0.40 ...
```

If **some** spikes >10 % during firmware rollout, switch from **always** → **madvise**.

Automated guard:

```
if (( $(awk '/memory/ && /some/ {sub(".*avg10=",""); print $1+0}' /proc/pressure/memory) > 10 )); then
  echo madvise | sudo tee /sys/kernel/mm/transparent_hugepage/enabled
fi
```

---

## **63.7**  **Container Context (cgroups v2)**

**MemoryMax=4G** + THP can OOM quickly.**  **Use:

```
systemctl set-property --runtime flash.slice MemoryHigh=3G
```

**MemoryHigh** throttles reclaim, preventing cgroup from OOM-killing neighbors.

---

## **63.8**  **HugeTLBFS for DPDK**

Allocate explicit hugepages:

```
echo 1024 > /proc/sys/vm/nr_hugepages   # 2 MiB × 1024 = 2 GiB
mount -t hugetlbfs none /dev/hugepages
```

DPDK selects these for DMA; small control-plane processes still use THP.

---

## **63.9**  **Real-World Case Study (vCMTS Burst)**

* Before: THP “always”; compaction CPU spike to 120 % during nightly burst, 3 %
  packet loss.
* After: switch to “madvise”, explicit **madvise(MADV_HUGEPAGE)** in vCMTS
  process; compaction CPU 10 %, zero loss.

---

## **63.10**  **Exercises**

1. Write a Bash wrapper that toggles THP to **never** during ISSU (Chapter 49)
   and returns to previous mode post-upgrade.
2. Compare 99-percentile p99 latency of a log-parsing Python job with THP on/off using **hyperfine**.
3. Use **bpftrace** to count **do_huge_pmd_anonymous_page** events per second and
   plot vs throughput.

---

```
Let me know if you’d like more deep-dive chapters (e.g., #64 namespaces, #65 advanced `tc` shaping) or once the sandbox is stable, I can bundle these into ZIP files again.
```
