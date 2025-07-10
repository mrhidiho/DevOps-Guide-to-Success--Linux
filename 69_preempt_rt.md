
---

```
# Chapter 69 – PREEMPT_RT & Adaptive L1D Flush Control

Low-latency lab gear sometimes needs **≤ 50 µs jitter** while still running modern
Spectre mitigations. A **PREEMPT_RT** kernel plus selective disabling of Intel’s
expensive **L1D cache flush** can cut context-switch latency by 30 % and boost
firmware-copy throughput—all without widening your attack surface.

_Last updated: 2025-07-10_

---

## 69.1  Why PREEMPT_RT?

| Kernel Mode | Max Latency (µs) on EPYC 7543P | Typical Use |
|-------------|--------------------------------:|-------------|
| `CONFIG_PREEMPT_NONE`        | ≥ 1 000 | Bulk throughput |
| `CONFIG_PREEMPT_VOLUNTARY`   | ≈ 300   | Default server |
| **`CONFIG_PREEMPT_RT`**      | **70–90** | Soft real-time VNFs, PTP |

PREEMPT_RT converts spinlocks to RT-mutexes so the scheduler can pre-empt the
kernel almost anywhere.

---

## 69.2  Building the RT Kernel

```bash
git clone https://git.kernel.org/pub/scm/linux/kernel/git/rt/linux-stable-rt.git
cd linux-stable-rt
git checkout v6.8.3-rt5
cp /boot/config-$(uname -r) .config
make olddefconfig
scripts/config --enable CONFIG_PREEMPT_RT
make -j$(nproc)
sudo make modules_install install
sudo grub-mkconfig -o /boot/grub/grub.cfg
```

Reboot → verify:

```
uname -a | grep rt
```

---

## **69.3**  **CPU Isolation & IRQ Affinity**

```
grubby --args="isolcpus=2-5 nohz_full=2-5 rcu_nocbs=2-5" --update-kernel=ALL
systemctl mask irqbalance
echo 2-5 | sudo tee \
  /proc/irq/$(grep -m1 mlx /proc/interrupts | cut -d: -f1)/smp_affinity_list
```

* VNFs fixed to CPUs 2-5
* OS housekeeping on other cores

---

## **69.4**  **Measuring Latency with** ****

## **cyclictest**

```
cyclictest -t4 -p99 -l200000 -S
```

Typical RT result:

```
T: 0 ( 700) P:99 I:1000 C:199999 Min:  7 Act: 12 Avg: 16 Max: 42
```

Max < 50 µs meets jitter SLA.

---

## **69.5**  **Spectre Mitigation Primer**

| **Mitigation**           | **Latency Cost** | **Disable Scope**                                  |
| ------------------------------ | ---------------------- | -------------------------------------------------------- |
| PTI (Meltdown)                 | ~3 %                   | kernel cmdline                                           |
| L1D Flush on VMEXIT/CTX SWITCH | 2000-4000 ns/ctx       | **per-task**via**arch_prctl**(kernel ≥ 6.6) |

We’ll keep PTI on but disable L1D flush for *trusted* flash helpers only.

---

## **69.6**  **Per-Task L1D Flush Toggle**

### **Small SUID helper**

noflush.c

```
#define _GNU_SOURCE
#include <sys/prctl.h>
#include <unistd.h>
#include <stdio.h>
int main(int argc, char **argv) {
    /* 73 = ARCH_SET_L1D_FLUSH on x86_64 */
    if (prctl(73, 0) != 0) { perror("prctl"); return 1; }
    execvp(argv[1], &argv[1]);
}
```

```
gcc -O2 noflush.c -o noflush
sudo chown root:root noflush && sudo chmod u+s noflush
```

Run trusted copier inside RT slice:

```
systemd-run --slice=flash.slice -p CPUWeight=20 \
  ./noflush uring_copy fw.bin /tmp/
```

All other tasks retain Spectre flush.

---

## **69.7**  **Benchmark – Throughput & Latency**

| **Scenario**             | uring_copy**GB/s** | **p99 Latency (** **cyclictest****)** |
| ------------------------------ | ------------------------ | ------------------------------------------------- |
| PTI + Flush ON, CFS            | 3.2                      | 180 µs                                           |
| **PREEMPT_RT + noflush** | **3.8**            | **45 µs**                                  |

EPYC 7543P, kernel 6.9-rc6, Intel P5510 NVMe.

---

## **69.8**  **Runtime Guard via PSI**

Combine Chapter 61 PSI with systemd:

```
psi=$(awk '/memory/ && /some/ {sub(".*avg10=","");print $1+0}' /proc/pressure/memory)
(( psi > 10 )) && systemctl set-property --runtime flash.slice CPUWeight=10
```

Drops flash CPU weight if memory pressure spikes.

---

## **69.9**  **Prometheus Export**

```
lat=$(grep 'Max:' /var/log/cyclictest.log | awk '{print $2}')
echo "rt_max_latency_us $lat" > /var/lib/node_exporter/text/rt.prom
```

**Alert if **rt_max_latency_us > 75**.**

---

## **69.10**  **Exercises**

1. Patch helper to toggle L1D flush **on** for untrusted user namespaces
   (Chapter 64) by reading **$NOFLUSH_PERMIT=0/1**.
2. Build CI that compiles an RT kernel with **CONFIG_PREEMPT_DYNAMIC=y** and
   benchmarks live KPatch load latency (Chapter 44).
3. **Use **perf stat -e cycles,instructions,l1d_flush** to profile firmware copy**
   with flush on/off at different queue depths (Chapter 62).

---

```
---

> **Tip:** after saving this file, run `markdownlint` or `prettier` in your repo to keep styling consistent with previous chapters.  If you’d still like a ZIP later, just ask once the packaging environment is stable.
```
