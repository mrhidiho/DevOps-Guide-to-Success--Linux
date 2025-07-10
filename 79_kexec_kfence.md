
# Chapter79  kexecBased Fast Reboot& `CONFIG_KFENCE` Crash Forensics

Long BIOS POST and firmware checks make cold boots painfully slow. **kexec**
jumps straight into a new kernel, rebooting in8s. Meanwhile **KFENCE**
(Kernel ElectricFence) samples heap allocations at runtime to catch elusive
useafterfree or buffer overflows _without_ heavyweight KASAN overhead.  
Together they slash outage time and surface memory bugs before production.

_Last updated: 2025-07-10_

---

## 79.1  Why Care?

| Reboot Method | EPYC7543P BoottoBoot | Dump Capture? |
|---------------|------------------------:|---------------|
| Cold (IPMI)   | 180s | No |
| `reboot` (normal) | 95s | kdumpcapable |
| **`kexec -e`** | **8s** | Yes (kdump) |

KFENCE adds <2% runtime overhead compared to KASANs >30%.

---

## 79.2  Install & Enable kexec

```bash
apt-get install -y kexec-tools
echo 'LOAD_KEXEC=true' | sudo tee -a /etc/default/kexec
systemctl enable kexec.target
```

Test manual jump:

```bash
kexec -l /boot/vmlinuz-$(uname -r) --initrd=/boot/initrd.img-$(uname -r) --reuse-cmdline
sync && kexec -e   # System reboots in seconds
```

---

## 79.3  Configure kdump Second Kernel

Add to GRUB:

```
crashkernel=512M
```

Rebuild GRUB; verify:

```bash
kdump-config test
```

Crash trigger:

```bash
echo c | sudo tee /proc/sysrq-trigger   # generates vmcore
```

---

## 79.4  Building Kernel with KFENCE

Enable in `.config`:

```
CONFIG_KFENCE=y
CONFIG_KFENCE_SAMPLE_INTERVAL=100
CONFIG_KFENCE_STATIC_KEYS=y
```

*Interval* = N allocations between samples (lower  more coverage).

Compile & install kernel 6.9+.

Boot dmesg excerpt:

```
KFENCE: initialized - using 255104 bytes for 255 objects at 0xffff9c2e95c74000-0xffff9c2e95cba800
```

---

## 79.5  Collecting KFENCE Reports

KFENCE writes to dmesg:

```
KFENCE: BUG in mlx5_core at drivers/net/...
invalid write of size 4 to 0xffff9c2e12a3b124
```

Log watcher to Slack:

```bash
journalctl -k -f | grep --line-buffered 'KFENCE: BUG'   | while read line; do ./slack_notify.sh "$line"; done
```

---

## 79.6  Nightly FastReboot Soak

Systemd unit `kfence-soak.service`:

```ini
[Service]
Type=oneshot
ExecStartPre=/sbin/kexec -l /boot/vmlinuz-kfence --initrd=/boot/initrd-kfence --reuse-cmdline
ExecStart=/bin/sleep 3600
ExecStartPost=/sbin/kexec -l /boot/vmlinuz-prod --initrd=/boot/initrd-prod --reuse-cmdline
ExecStartPost=/sbin/kexec -e
```

Timer:

```ini
OnCalendar=03:00
Persistent=true
```

If KFENCE BUG logged, script touches `/run/kfence_fail` and skips reload.

---

## 79.7  Prometheus Exporter for Reboot Delta

In `/usr/local/bin/reboot_exporter.sh`:

```bash
last=$(journalctl -q -b -1 -o short-unix | head -1 | cut -d' ' -f1)
echo "fast_reboot_seconds $(date +%s -d@$(date +%s))- $last"  > /var/lib/node_exporter/text/reboot.prom
```

Alert if reboot>60s during scheduled window.

---

## 79.8  Performance Impact

| Mode | DPDK PPS | p99 Lat (s) | PowerW |
|------|---------:|-------------:|--------:|
| Prod kernel | 9.2M | 41 | 215 |
| **KFENCE** | 9.0M | 43 | 217 |

Tiny perf hitacceptable during onehour soak.

---

## 79.9  Forensic Workflow after Crash

1. `kdump` saves `/var/crash//vmcore`.  
2. Script uploads core + `dmesg` + KFENCE logs to S3.  
3. CI pipeline runs `gdb vmlinux vmcore`  `crash`  autogenerate Jira ticket.

---

## 79.10  Exercises

1. Change `KFENCE_SAMPLE_INTERVAL` to10, observe PPS and BUG catchrate.  
2. Patch `flash_fw.sh` to run `kexec -p` (purge caches) before heavy copy.  
3. Use `perf probe` to extract stack of KFENCE invalid access and group similar sites.

---
