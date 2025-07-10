
# Chapter76  Chrony PTP with NIC Hardware Timestamping & Ocelot FD

Achieving **submicrosecond time sync** across a lab rack is critical for
latencysensitive tests (e.g., `sched_ext` profiling). This chapter shows how
to enable NIC hardware Tx/Rx timestamps, configure **Chrony** for PTP, and
verify offset drift under load.

_Last updated: 2025-07-10_

---

## 76.1  Hardware & Firmware Prereqs

| NIC | Driver | PTP Support |
|-----|--------|-------------|
| Intel i225/i226 | `igc` | IEEE1588v2 |
| Mellanox ConnectX6 DX | `mlx5` | HW timestamp, QP771 |
| Microchip Ocelot FD PHY | `ocelot-ptp` | HW timestamping, FD Sync |

Check:

```bash
ethtool -T eth0 | grep hardware
```

Should print `hardware-transmit hardware-receive`.

---

## 76.2  Configuring PTP Hardware Clock (PHC)

```bash
sudo ptp4l -i eth0 -m -S -2 &
```

* `-S` uses system clock if PHC absent.  
* Wait until `offset` < 1 s.

---

## 76.3  Chrony with PHC2SYS

`/etc/chrony/chrony.conf`

```
refclock PHC /dev/ptp0 poll 2 dpoll -2 offset 0
makestep 0.1 3
rtcsync
hwtimestamp eth0
log measurements statistics tracking
```

Start services:

```bash
systemctl enable --now chronyd phc2sys
systemctl status chronyd
```

---

## 76.4  Verifying Offset

```bash
chronyc tracking
# Root dispersion: 0.4s
```

Live watch:

```bash
watch -n1 'chronyc sourcestats | tail -1'
```

Expect RMS offset < 800ns.

---

## 76.5  Network Load Test

Generate 10Gb traffic:

```bash
iperf3 -c peer -b 10G -t 60 -P 4
```

Plot offset:

```bash
chronyc -a 'measurements np 20' | grep ^PHC > offset.log
```

Python quick graph (Chapter59 pandas).

---

## 76.6  Ocelot FD Sync (Layer2)

Configure on switch:

```
ptp enable
ptp transparent
ptp forward non-ptp
```

Linux host attach:

```bash
ip link add name ptp0 type ocelot-ptp
ip link set ptp0 up
```

Check:

```bash
ptp4l -i ptp0 -s -m
```

Offsets typically < 100ns on coax backplane.

---

## 76.7  Prometheus Export

Node Exporter textfile:

```bash
off=$(chronyc tracking | awk '/System time offset/ {print $4*1000}')
echo "ptp_offset_ns $off" \
  > /var/lib/node_exporter/text/ptp.prom
```

Alert if `ptp_offset_ns > 1000`.

---

## 76.8  RealWorld Win

Before PTP: lab routers timestamp logs with 3ms skew; crossbox latency
measurements noisy. After Chrony+PHC: skew 350ns; jitter graphs matched
hardware traffic generator within 2%.

---

## 76.9  Exercises

1. Attach two hosts via direct DAC; run PTP boundary clock, measure offset at
   1Gb vs 25Gb.  
2. Configure `chrony.conf` `filter 2` and measure convergence time vs default.  
3. Use `hwstamp_ctl` to toggle one-step PTP and observe chrony offset drift.

---
