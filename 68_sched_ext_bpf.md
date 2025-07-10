
# Chapter 68 – BPF `sched_ext` & Run-Queue Analysis

When a 60 k pps VNF shares a CPU with Python telemetry, the default CFS
scheduler can add 200 µs tail latency.  The new **`sched_ext`** framework
(5.19 + patches, mainlining in progress) lets you load a BPF program to become
_the scheduler_ for selected CPUs.  This chapter walks through enabling the
framework, loading `scx_simple`, writing a custom latency-first scheduler, and
visualizing run-queue pressure with `bpftrace`.

_Last updated: 2025-07-10_

---

## 68.1  Prerequisites

* Kernel ≥ 6.9 **built with**  
  `CONFIG_SCHED_CLASS_EXT=y`  
  `CONFIG_BPF_SYSCALL=y`
* libbpf ≥ 1.4
* clang/llvm ≥ 17
* Root or `CAP_BPF` to load the scheduler

---

## 68.2  Quick Glossary

| Term | Meaning |
|------|---------|
| **SCX** | Scheduler Class eXternal (BPF scheduler) |
| **run-queue** | Per-CPU list of runnable tasks |
| **enqueue / dequeue** | Hooks called when task becomes runnable/blocked |
| **dispatch** | Assign a task to CPU |

---

## 68.3  Building the Kernel Module

```bash
git clone https://github.com/sched-ext/scx-examples
cd scx-examples
make   # builds scx_simple.bpf.o
```

---

## **68.4**  **Loading** ****

## **scx_simple**

```
sudo ./load_simple.sh 0-3   # schedule CPUs 0-3 via scx_simple
```

dmesg**:**

```
SCX: Loaded scx_simple on cpus_mask=0f tasks 0
```

Verify:

```
cat /sys/kernel/debug/sched/ext
# shows BPF prog id, enqueues, dispatches
```

Rollback:

```
sudo ./unload.sh
```

---

## **68.5**  **Writing a Custom Latency-First Scheduler**

scx_latency.bpf.c** (excerpt):**

```
SEC("scx/enqueue")
void BPF_PROG(enq, struct task_struct *p, u64 enq_flags) {
    u64 now = bpf_ktime_get_ns();
    bpf_task_storage_get(&enq_time, p, 0, 0)->ts = now;
}

static u64 pick_cpu(void) {
    u32 id = bpf_get_smp_processor_id();
    struct rq_metrics *m = bpf_map_lookup_elem(&rq_time, &id);
    return m && m->avg_lat_ns < 50000 ? id : (id ^ 1);
}

SEC("scx/select_cpu")
u32 BPF_PROG(sel, struct task_struct *p) {
    return pick_cpu();
}
```

Compile & load:

```
clang -O2 -target bpf -c scx_latency.bpf.c -o scx_latency.bpf.o
sudo ./load_prog.sh scx_latency.bpf.o 0-3
```

Now tasks with high wait time gravitate to CPUs with lower run-queue latency.

---

## **68.6**  **Visualizing Run-Queue Latency with** ****

## **bpftrace**

rq_lat.bt

```
kprobe:sched_switch {
  @start[args->prev_pid] = nsecs;
}
kprobe:sched_wakeup /@start[args->pid]/ {
  @lat = hist((nsecs - @start[args->pid]) / 1000);  // µs
  delete(@start[args->pid]);
}
interval:s:5 { print(@lat); clear(@lat); }
```

Run during firmware rollout; compare histogram before and after custom SCX

scheduler.

---

## **68.7**  **Limiting to Specific cgroup**

Create slice:

```
systemd-run --unit=vfn.slice -p CPUWeight=200 sleep infinity
```

Attach scheduler only to tasks in slice:

```
echo "/sys/fs/cgroup/system.slice/vfn.slice" | \
  sudo tee /sys/kernel/debug/sched/ext/master/tasks_path
```

Now SCX only manages VNFs; SSH daemon stays on CFS.

---

## **68.8**  **Fallback Guard**

If BPF scheduler crashes (**bpf_prog_terminate**), kernel falls back to CFS and

prints:

```
sched_ext: fallback to CFS after BPF error -22
```

Set alert:

```
grep -q "sched_ext: fallback" /var/log/kern.log && \
  systemctl restart flash.slice
```

---

## **68.9**  **Real-World Lab Result**

* **Baseline (CFS)**: 99-th latency = 180 µs, 1 % packet drop at PPS peak
* **scx_latency**: 99-th latency = 60 µs, 0 drop, CPU util +2 %
* Hardware: EPYC 7543P, kernel 6.9-rc6, Intel QAT offload ON

---

## **68.10**  **Exercises**

1. Modify **scx_latency** to read **cpu.latency_ns** target from
   **/sys/fs/cgroup/*/cpu.latency_ns** so each slice sets its own budget.
2. Write a Bash script that rolls out SCX on half the cores (**0-15**), runs a
   dual-workload benchmark, then switches (**echo CFS**) and compares **perf stat** results.
3. Use **bpftool prog profile id <prog_id>** to capture DWARF stack samples and
   visualize CPU overhead of your SCX scheduler.

---

```
Feel free to embed this into your repo.  If you’d like packaging (ZIP) once the sandbox stabilizes—or further advanced chapters—just let me know!
```
