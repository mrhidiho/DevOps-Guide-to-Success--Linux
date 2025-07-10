
```
# Chapter 67 – systemd Resource-Control for Run-Time Guardrails

`systemd` isn’t just a service manager—it’s a front-end to cgroups v2.  
Drop-in properties such as **CPUWeight**, **MemoryHigh**, and **IPAccounting**
let you cage runaway Bash loops, rate-limit firmware downloads, and generate
per-service network usage—_with no kernel-tuning scripts_.

_Last updated: 2025-07-10_

---

## 67.1  Property Cheat-Sheet

| Resource | Knob | Range / Unit | Notes |
|----------|------|--------------|-------|
| CPU | `CPUWeight=` | 1 – 10 000 (default = 100) | Relative share within parent slice |
| Mem | `MemoryHigh=` | Bytes / `%` | Soft limit, throttles instead of OOM |
| Mem | `MemoryMax=` | Bytes / `%` | Hard limit; OOM-kills inside cgroup |
| I/O | `IOWeight=` | 1 – 10 000 | blk-mq weight; cf. Chapter 66 |
| PIDs | `TasksMax=` | Integer | Prevent fork bombs |
| Net | `IPAccounting=` | yes / no | Per-unit RX/TX counters |
| Net | `IPAddressDeny=` | CIDR list | eBPF LPM filter for outbound traffic |

---

## 67.2  Drop-In Unit Overrides

```bash
sudo systemctl edit flash.service
```

/etc/systemd/system/flash.service.d/override.conf

```
[Service]
CPUWeight=20
MemoryHigh=2G
MemoryMax=3G
IOWeight=50
IPAccounting=yes
IPAddressDeny=10.0.0.0/8 192.168.0.0/16
```

Reload & restart:

```
systemctl daemon-reload
systemctl restart flash
```

---

## **67.3**  **One-Shot CLI (No Unit Needed)**

```
systemd-run --unit=measure.slice -p CPUWeight=10 \
            -p MemoryHigh=1G -p TasksMax=128 \
            bash stress_test.sh
```

Command exits, slice is auto-cleaned.

---

## **67.4**  **Viewing Live Stats**

```
systemctl status flash.service
systemctl show -p CPUUsageNSec -p IPIngressBytes -p IPEgressBytes flash
```

Or read cgroup files:

```
cat /sys/fs/cgroup/system.slice/flash.service/cpu.stat
cat /sys/fs/cgroup/.../ip_statistics
```

---

## **67.5**  **Alerting with Prometheus Node Exporter**

Enable **--collector.systemd** flag; metrics appear:

```
systemd_unit_cpu_seconds_total{unit="flash.service"}  42
systemd_unit_ip_ingress_bytes_total{unit="flash.service"}  2.3e+08
```

Alert if **cpu_seconds_total** > 600 over 5 min (runaway loop).

---

## **67.6**  **Dynamic Adjustment via** ****

## **systemctl set-property**

```
# drop CPU to 5 % if memory pressure high
systemctl set-property --runtime flash.service CPUWeight=5
```

**--runtime** avoids writing to disk; reverts on reboot.

Automated guard (Bash):

```
psi=$(awk '/memory/ && /some/ {sub(".*avg10=",""); print $1+0}' /proc/pressure/memory)
(( psi > 10 )) && systemctl set-property --runtime flash.service CPUWeight=10
```

Combines Chapter 61 PSI with systemd throttling.

---

## **67.7**  **Per-Slice Accounting Example**

Create site slices:

```
systemctl set-property --runtime --slice edge-den.slice CPUWeight=50
systemctl set-property --runtime --slice edge-lax.slice CPUWeight=50
```

Move service:

```
systemctl set-property flash@edge01.service Slice=edge-den.slice
```

CPUWeight applies across **all** services in slice, enabling fair sharing by

POP.

---

## **67.8**  **Limiting Firmware Upload Bandwidth**

kernel >= 5.15 supports **IPIngressMaxBytesPerSec** (systemd 255):

```
[Service]
IPIngressMax=20M
IPEgressMax=20M
```

Fallback: use **tc cake** (Chapter 65) at slice’s network namespace.

---

## **67.9**  **Real-World Win: Preventing Telemetry Starvation**

* Before: firmware cron job on CMTS spiked CPU; gNMI exporter missed 30 s
  samples, Grafana gaps triggered false alerts.
* After: **CPUWeight=50**, exporter slice **CPUWeight=200** → exporter always wins
  scheduling, zero gaps during rollout.

---

## **67.10**  **Exercises**

1. Create a transient slice via **systemd-run** that sets **TasksAccounting=yes**
   and kills script if forks > 50. Verify with **forkstat**.
2. Build a GitLab CI step that runs functional tests inside
   **–p MemoryMax=512M**; fail pipeline if OOM killer invoked.
3. Write a Python script (**systemd_dbus.py**) that listens for
   **PropertiesChanged** D-Bus signals to auto-scale **IOWeight** when
   **io.cost.qos** latency crosses 2 ms.

---

```
Let me know if you want a deep-dive Chapter 68 (sched_ext BPF schedulers) or a ZIP once packaging issues clear!
```
