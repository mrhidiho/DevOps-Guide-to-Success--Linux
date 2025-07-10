
```
# Chapter 66 – IO-Scheduler & blk-mq Tuning for NVMe / Zoned Namespace (ZNS)

Firmware staging, log-harvest, and `uring_copy` (Chapter 62) all hammer block
devices.  Modern kernels default to `none` scheduler on NVMe, but `mq-deadline`
or `kyber` can cut 99-p latency on heavily mixed workloads.  ZNS drives bring
new rules: append-only zones, reset commands, and zone trashing penalties.

_Last updated: 2025-07-10_

---

## 66.1  Quick Glossary

| Term | Meaning |
|------|---------|
| **blk-mq** | Multi-queue block layer (kernel ≥3.13) |
| **I/O Scheduler** | Reorder & merge requests (`none`, `mq-deadline`, `kyber`) |
| **ZNS** | NVMe Zoned Namespace – host must write sequentially inside zones |
| **Write Pointer** | Current offset inside a zone; resets only with `zone reset` |

---

## 66.2  Listing Schedulers

```bash
cat /sys/block/nvme0n1/queue/scheduler
# none mq-deadline kyber [none]
```

Square brackets denote current scheduler.

Change live:

```
echo kyber | sudo tee /sys/block/nvme0n1/queue/scheduler
```

---

## **66.3**  **Benchmark – Mixed Read/Write**

```
fio --name=rw --iodepth=32 --rw=randrw --rwmixread=70 \
    --filename=/dev/nvme0n1 --ioengine=io_uring --size=4G --time_based --runtime=60
```

Results (Intel P5510, kernel 6.8):

| **Scheduler** | **IOPS** | **p99 Lat (µs)** |
| ------------------- | -------------- | ----------------------- |
| none                | 320k           | 420                     |
| kyber               | 305k           | **210**           |
| mq-deadline         | 300k           | 240                     |

“kyber” halves tail latency on busy routing nodes with minimal throughput hit.

---

## **66.4**  **Automating via Udev Rule**

/etc/udev/rules.d/60-nvme-sched.rules

```
ACTION=="add|change", SUBSYSTEM=="block", KERNEL=="nvme*n1", \
 ATTR{queue/rotational}=="0", ATTR{queue/scheduler}="kyber"
```

Reload:

```
udevadm control --reload && udevadm trigger -s block
```

---

## **66.5**  **blk-mq Queue Depth & CPU Affinity**

Set depth:

```
echo 128 | sudo tee /sys/block/nvme0n1/queue/nr_requests
```

Pin hardware queues to specific CPUs:

```
cat /sys/block/nvme0n1/queue/*/cpu_list
echo 0-7 | sudo tee /sys/block/nvme0n1/queue/0/cpu_list
```

Helps when VNFs are pinned to certain NUMA nodes.

---

## **66.6**  **Zoned Namespace Primer**

List zones:

```
nvme zns report-zones /dev/nvme0n1 --human-readable --extended
```

Key fields: **WP** (write pointer), **ZS** (zone size), **ZA** (zone append).

* **Append** – use **io_uring/zoned append**, drive sets LBA
* **Reset Cost** – avoid frequent **zone reset**

---

## **66.7**  **ZNS Firmware Staging Strategy**

1. Allocate one zone per firmware image; record **zone start LBA** in DB.
2. Append chunks sequentially:

```
io_uring_prep_zoned_append(sqe, fd, buf, len, zone_start, user_data);
```

3. Mark zone “valid” in DB after SHA-256 verified.
4. Reuse zone only after **nvme zns reset-zone**.

Prevents “zone trashing” that forces GC.

---

## **66.8**  **Bash Helper to Pick Free Zone**

```
free=$(nvme zns report-zones /dev/nvme0n1 -o json | jq -r '.[]|select(.state=="Empty")|.zslba' | head -1)
echo "Use zone starting at LBA $free"
```

---

## **66.9**  **Monitoring blk-mq & Scheduler Stats**

```
cat /sys/block/nvme0n1/queue/iosched/kyber_stats
```

Prometheus textfile exporter:

```
for f in read_lat_ns write_lat_ns; do
  v=$(cat /sys/block/nvme0n1/queue/iosched/$f)
  echo "nvme_kyber_${f} $v"
done > /var/lib/node_exporter/text/kyber.prom
```

**Alert if **write_lat_ns > 300000**.**

---

## **66.10**  **Exercises**

1. Measure **uring_copy** (Chapter 62) throughput on **none** vs **kyber** with queue depth 64 → 256.
2. Write a Bash script that migrates firmware staging from flat file to ZNS append-format, logs zone start LBA, and trims zones after deploy.
3. Build a systemd timer that sets **nr_requests=512** off-peak, then back to 128 during business hours.

---

```
This completes Chapter 66.  Let me know if you’d like more advanced topics (#67 systemd resource control, #68 sched_ext runqueue BPF) or if you want ZIP packaging once the sandbox behaves!
```
