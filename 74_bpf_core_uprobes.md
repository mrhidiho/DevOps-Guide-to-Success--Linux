
# Chapter74  eBPF CORE Uprobes for Go/Python VNFs

eBPF uprobes attach **LATENCY HISTOGRAMS** to userspace functions with **no
code change**.  CORE (Compile Once, Run Everywhere) uses BTF type info to
run the same ELF on any kernelperfect for instrumenting Go or Python VNFs in
the field.

_Last updated: 2025-07-10_

---

## 74.1  Why CORE Uprobes?

* **No kernel headers** on target device  
* ELF relocates to correct struct offsets via BTF  
* Attach to symbol names (`main.handlePkt`) or addresses  
* Nearzero overhead (< 0.5s per call)

---

## 74.2  Prerequisites

| Item | Version |
|------|---------|
| Kernel | 5.11 with CONFIG_DEBUG_INFO_BTF=y |
| libbpf | 1.4 |
| bpftool | built with `libbfd` for symbol lookup |

Install:

```bash
apt-get install -y clang-17 libbpf-dev bpftool
```

---

## 74.3  Build Simple Latency Histogram

`latency.bpf.c`

```c
#include "vmlinux.h"
#include <bpf/bpf_helpers.h>
#include <bpf/bpf_tracing.h>

struct {
  __uint(type, BPF_MAP_TYPE_HISTOGRAM);
  __uint(max_entries, 64);
  __type(key, u64);   /* slot */
  __type(value, u64); /* count */
} dist SEC(".maps");

struct start_t { u64 ts; };
struct {
  __uint(type, BPF_MAP_TYPE_HASH);
  __uint(max_entries, 1024);
  __type(key, u64);     /* pid */
  __type(value, struct start_t);
} start SEC(".maps");

/* uprobe on function entry */
SEC("uprobe/handlePkt_enter")
int BPF_KPROBE(pkt_enter)
{
    struct start_t st = { .ts = bpf_ktime_get_ns() };
    u64 pid = bpf_get_current_pid_tgid();
    bpf_map_update_elem(&start, &pid, &st, BPF_ANY);
    return 0;
}

/* uretprobe on function exit */
SEC("uretprobe/handlePkt_exit")
int BPF_KRETPROBE(pkt_exit)
{
    u64 pid = bpf_get_current_pid_tgid();
    struct start_t *st = bpf_map_lookup_elem(&start, &pid);
    if (!st) return 0;
    u64 delta = bpf_ktime_get_ns() - st->ts;
    u64 slot = bpf_log2l(delta / 1000);        /* s buckets */
    bpf_map_delete_elem(&start, &pid);
    bpf_map_inc(&dist, &slot);
    return 0;
}

char LICENSE[] SEC("license") = "Dual BSD/GPL";
```

Compile CORE:

```bash
clang -O2 -g -target bpf -D__TARGET_ARCH_x86 -c latency.bpf.c -o latency.bpf.o
```

---

## 74.4  Attaching with `bpftool`

Find binary + symbol:

```bash
bpftool btf dump file /usr/local/bin/vnf | grep handlePkt
```

Insert:

```bash
bpftool prog load latency.bpf.o "/sys/fs/bpf/latency" \
  pinmaps /sys/fs/bpf
bpftool cgroup attach $(cat /proc/$(pgrep vnf)/cgroup | cut -d: -f3) \
  prog pinned /sys/fs/bpf/latency \
  uprobe id /usr/local/bin/vnf:handlePkt \
  uretprobe id /usr/local/bin/vnf:handlePkt
```

---

## 74.5  Reading Histogram

```bash
bpftool map dump pinned /sys/fs/bpf/dist | sort -n
# Key=6 Value=432  2^6 s bucket
```

Prettyprint:

```bash
bpftool prog tracelog
```

---

## 74.6  Python Helper to Export Prometheus Buckets

`export_latency.py`

```python
from bcc import BPF
import time, prometheus_client as prom
b = BPF(src_file="latency.bpf.c")
b.attach_uprobe(name="/usr/local/bin/vnf", sym="handlePkt", fn_name="pkt_enter")
b.attach_uretprobe(name="/usr/local/bin/vnf", sym="handlePkt", fn_name="pkt_exit")

hist = b.get_table("dist")
PROM = prom.start_http_server(9105)

while True:
    time.sleep(5)
    buckets = hist.items_and_clear()
    for slot, count in buckets:
        prom.Gauge('vnf_pkt_latency_bucket', 's buckets',
                   ['le']).labels(le=str(2**slot.value)).set(count.value)
```

---

## 74.7  Go Binary With DWARF Stripped? Use BuildID

```bash
readelf -n /usr/local/bin/vnf | grep -A1 Build.ID
```

Then:

```bash
bpftool btf dump id $(cat /sys/kernel/btf/vmlinux) format raw | ...
```

CORE relocates types regardless of stripped symbols.

---

## 74.8  Production Guard

* Attach uprobes **only** on isolated CPUs (Chapter69 `taskset`).  
* Unload program on SIGSEGV via `bpftool prog detach`.  
* Use `sched_ext` (Chapter68) to schedule collector thread at nice 19.

---

## 74.9  RealWorld Win: Go gRPC Exporter

* p99 latency before: 1.4ms during VNF PPS peaks  
* After code tweak guided by eBPF histogram: 0.4ms p99, 0 drops

Tweaked `sync.Pool` size based on bucket spike at 128512s.

---

## 74.10  Exercises

1. Modify BPF program to record histogram perCPU map and print heatmap.  
2. Use CORE uprobes on CPythons `PyEval_EvalFrame` to measure time per
   networkautomation script call.  
3. Patch loader to pin maps in `/sys/fs/bpf/vnf_{pid}` and rotate filenames
   on binary upgrade.

---
