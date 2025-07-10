
```
# Chapter 61 – cgroups v2 & Pressure-Stall Information (PSI)

When a firmware rollout script stalls, is it CPU contention, I/O backlog, or
memory reclaim?  **cgroups v2** gives you fine-grained resource governance
(CPU, memory, I/O, pids) in a _single unified hierarchy_, while **PSI
(Pressure-Stall Information)** exposes real-time “pressure” metrics that
predict brownouts _before_ they hit customers.

_Last updated: 2025-07-10_

---

## 61.1  Why cgroups v2?

* **Unified tree** – all controllers (cpu, io, memory) on one path  
* **Weight-based CPU & I/O** – no more juggling `cpu.shares` vs `cfs_quota_us`  
* **Delegation** – unprivileged users can manage sub-trees without root  
* **PSI** – `/proc/pressure/{cpu,memory,io}` for per-resource stall stats

> **Firmware Rollout Tie-In:** Run flashing scripts in a low-weight slice so
> BGP, gRPC collectors, and Chaos Mesh never starve.

---

## 61.2  Enabling cgroups v2 (systemd)

Boot loader kernel arg:
```

systemd.unified_cgroup_hierarchy=1

```
Verify:

```bash
stat -c %T /sys/fs/cgroup      # → cgroup2fs
```

**systemd --version** ≥ 245 auto-mounts unified tree under **/sys/fs/cgroup**.

---

## **61.3**  **Creating a Custom Slice**

```
sudo systemd-run \
     --unit=flash.slice \
     --slice=system.slice \
     -p CPUWeight=50 \
     -p MemoryHigh=2G \
     -p IOWeight=100 \
     bash -c './flash_fw.sh devices.csv'
```

* **CPUWeight=50** (default=100) → half CPU share
* **MemoryHigh=2G** soft limit—process throttles when exceeded
* **IOWeight=100** relative to other slices (10-10000)

**Verify:**

```
systemctl status flash.slice
cat /sys/fs/cgroup/system.slice/flash.slice/cpu.stat
```

---

## **61.4**  **On-The-Fly Adjustment via** ****

## **systemctl**

```
systemctl set-property flash.slice CPUWeight=20
```

Useful when Prometheus alerts on high run-queue latency during rollout.

---

## **61.5**  **Delegating to Unprivileged NetOps User**

```
sudo loginctl enable-linger netops
sudo mkdir -p /sys/fs/cgroup/user.slice/user-1001.slice/user@1001.service/mycg
sudo chown -R netops: /sys/fs/cgroup/.../mycg
```

**Now **netops** can **echo $$ > mycg/cgroup.procs** to self-assign.**

---

## **61.6**  **PSI Basics**

Three files:

```
/proc/pressure/cpu
/proc/pressure/memory
/proc/pressure/io
```

Sample:

```
some avg10=1.50 avg60=1.30 avg300=1.10 total=345120
full avg10=0.40 ...
```

* **some** – time *some* tasks were stalled
* **full** – time the *entire* system stalled

**avg10** → average percentage over 10 s window.

---

## **61.7**  **Prometheus Exporter (Textfile)**

```
for r in cpu memory io; do
  awk '/avg10/ {gsub("avg10=",""); print "psi_"r"_some_avg10 " $2}' \
      /proc/pressure/$r
done \
  > /var/lib/node_exporter/text/psi.prom
```

Grafana dashboard plots spikes during flash bursts.

---

## **61.8**  **Alert Rule Example**

```
alert: HighIODelay
expr: psi_io_some_avg10 > 10
for: 30s
labels:
  severity: critical
annotations:
  summary: "I/O pressure ≥10 % on {{ $labels.instance }}"
  runbook: "https://kb.isp.io/psi"
```

**Rollback script listens for this alert and **systemctl set-property flash.slice IOWeight=1**.**

---

## **61.9**  **Real-World Case Study (Edge Firmware)**

| **Step**                | **PSI cpu avg10** | **PSI io avg10** | **Actions**           |
| ----------------------------- | ----------------------- | ---------------------- | --------------------------- |
| Pre-flash baseline            | 0.2 %                   | 0.1 %                  | —                          |
| Copy firmware to /tmp (tmpfs) | 0.3 %                   | 0.2 %                  | normal                      |
| **Writes to eMMC**      | 1.1 %                   | **15.4 %**       | Alert, auto-reduce IOWeight |
| Verification                  | 0.4 %                   | 0.3 %                  | raised weight back          |

Saving 1-minute of brownout prevented BGP dampening on core routers.

---

## **61.10**  **Advanced:** ****

## **memory.reclaim**

For one-shot reclamation:

```
echo 536870912 > /sys/fs/cgroup/flash.slice/memory.reclaim   # reclaim 512 MB
```

Useful before starting large gzip extracts.

---

## **61.11**  **Lab: Measuring PSI vs** ****

## **ioping**

1. Create a 2 GB file on NVMe.
2. **Run **ioping -D -c 5000 file** under default slice vs **CPUWeight=20**.**
3. Plot **psi_io_some_avg10** every second; note lower spikes with throttled
   slice.

---

## **61.12**  **Exercises**

1. Write a Bash loop that bumps **CPUWeight** by +10 every 30 s until
   psi_cpu_some_avg10** stays below 5 %.**
2. Move Prometheus Node Exporter into its own slice with **MemoryMax=256M**
   and confirm it never OOMs while you run **stress-ng --vm**.
3. Use **systemd-run --uid=netops --scope** to launch a single Netmiko bulk
   task in delegated slice and compare PSI to system slice run.

---

```
---

Copy this into your repo; if you’d like a ZIP when packaging stabilizes, let me know!
```
