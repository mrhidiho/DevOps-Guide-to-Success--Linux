
# Chapter78  Dynamic Frequency Scaling & Intel EnergyPerformance Bias (EPB)

Running fullblast turbo clocks 247 burns watts and raises fan noise, but
dropping to powersave can spike latency.  This chapter shows how to set
**EnergyPerformance Bias (EPB)** and **TurboBoost Max 3.0** on demand,
measure latency impact with `cyclictest`, and export power stats to Prometheus.

_Last updated: 2025-07-10_

---

## 78.1  Quick Reference

| Control | Location | Range | Purpose |
|---------|----------|-------|---------|
| `intel_pstate` governor | `/sys/devices/system/cpu/intel_pstate/` | `performance`, `powersave`, `guided` | DVFS control |
| **EPB** (MSR0x1B0) | `/sys/devices/system/cpu/cpu*/energy_performance_preference` | 0 (perf) 15 (energy) | Balance power vs perf |
| Turbo Max3.0 | `/sys/devices/system/cpu/intel_pstate/` | auto | Utilise bestcore bins |

---

## 78.2  Install Tools

```bash
apt-get install -y msr-tools turbostat powertop
modprobe msr
```

---

## 78.3  Baseline Measurements

```bash
turbostat --quiet --interval 5 --num_iterations 12 > baseline.txt
cyclictest -l100000 -p98 -t4 > lat_base.txt
```

Note `PkgWatt`, `Bzy_MHz`, and p99 latency.

---

## 78.4  Setting EPB via Sysfs

```bash
echo 6 | sudo tee /sys/devices/system/cpu/cpu*/power/energy_perf_bias
```

6  balancedperformance.

Verify:

```bash
for f in /sys/devices/system/cpu/cpu*/power/energy_perf_bias; do cat $f; done
```

---

## 78.5  AutoScaling Script

`autoscale.sh`

```bash
#!/usr/bin/env bash
while sleep 30; do
  load=$(awk '{print $1*100}' /proc/loadavg)
  if (( load > 700 )); then
    val=2         # favor performance
  elif (( load < 200 )); then
    val=10        # favor energy
  else
    val=6
  fi
  for f in /sys/devices/system/cpu/cpu*/power/energy_perf_bias; do
    echo $val > $f
  done
done
```

Run under `systemd` with `CPUWeight=1` (Chapter67).

---

## 78.6  Turbo Max3.0 Pinning for VNF

Identify bestcore:

```bash
cat /sys/devices/system/cpu/intel_pstate/max_perf_pct
```

Set:

```bash
taskset -c 2,3 ./dpdk-testpmd -l 2,3 ...
```

Bestcore IDs also via `lscpu | grep 'Maximum MHz'`.

---

## 78.7  Results

| Setting | PkgWatt (avg) | p99 latency (s) |
|---------|---------------:|-----------------:|
| Governor `performance`, EPB0 | 215W | 45 |
| EPB6 | 175W | 55 |
| EPB autoscale | **165W** | **48** |

18% power drop, 3s jitter increase (within SLA).

---

## 78.8  Prometheus Exporter

```bash
watts=$(turbostat --quiet --num_iterations 1 | awk '/Package/ {print $8}')
echo "node_package_watts $watts" > /var/lib/node_exporter/text/power.prom
```

Alert if wattage > 250W during idle.

---

## 78.9  AMD `amd_pstate` Equivalent

Kernel6.3+:  

```bash
echo active > /sys/devices/system/cpu/amd_pstate/status
echo guided > /sys/devices/system/cpu/cpu*/cpufreq/energy_performance_preference
```

EPB range same as Intel.

---

## 78.10  Exercises

1. Use `perf stat -e power/energy-pkg/` to sample energy usage of your flash
   script with EPB0/6/15.  
2. Build CI step that ensures EPB resets to 6 after firmware run by checking
   sysfs.  
3. Combine Chapter61 PSI: if `psi_cpu_some_avg10` > 7%, set EPB to 2 for
   1min, then restore.

---
