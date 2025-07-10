
# Chapter90  Hardware Telemetry: InBand ECC, RAS & `rasdaemon` to Prometheus

Silent DRAM errors and PCIe Correctable Errors (CE) can foreshadow cataclysmic
kernel panics. The Linux RAS stack**EDAC**, **APEI**, and **rasdaemon**logs
these events; exporting them to Prometheus lets you alert before an Sbit flips
into a hard crash.

_Last updated: 2025-07-10_

---

## 90.1  Kernel & Userspace Requirements

| Component | Version |
|-----------|---------|
| Kernel    | 5.10 with `CONFIG_EDAC_*`, `CONFIG_RAS` |
| rasdaemon | 0.8.0+ |
| `mcelog`  | optional (legacy) |

Install:

```bash
apt-get install -y rasdaemon edac-utils
systemctl enable --now rasdaemon
```

---

## 90.2  Verify InBand ECC Reporting

```bash
edac-util -v
# MC0: 0 Uncorrected Errors, 2 Corrected Errors
grep -r "EDAC" /var/log/kern.log | tail
```

PCIe AER:

```bash
dmesg | grep -i pcie | grep -i error
```

---

## 90.3  RASdaemon Database

`/var/lib/rasdaemon/ras-mc_event.db`

Query:

```bash
ras-mc-ctl --errors
ras-mc-ctl --summary
```

Event example:

```
ras: MC1:CE on channel1 DIMM_B0 (addr 0x123456) ...
```

---

## 90.4  Prometheus Exporter Script

`ras_exporter.sh`

```bash
#!/bin/bash
sqlite=/usr/bin/sqlite3
DB=/var/lib/rasdaemon/ras-mc_event.db

ce=$($sqlite $DB 'SELECT COUNT(*) FROM memory_errors WHERE severity="CE" AND time > datetime("now","-5 minutes");')
ue=$($sqlite $DB 'SELECT COUNT(*) FROM memory_errors WHERE severity="UE" AND time > datetime("now","-5 minutes");')
pcie=$($sqlite $DB 'SELECT COUNT(*) FROM pcie_errors WHERE severity="non-fatal" AND time > datetime("now","-5 minutes");')

cat <<EOF > /var/lib/node_exporter/text/ras.prom
memory_ce_5m {} } $ce
memory_ue_5m {} } $ue
pcie_ce_5m {} } $pcie
EOF
```

Run via systemd timer every minute.

Alerting rule:

```yaml
- alert: MemoryUE
  expr: memory_ue_5m > 0
  for: 0m
  labels: {severity: critical}
  annotations:
    summary: "Uncorrectable memory error on { $labels.instance }"
```

---

## 90.5  Enable Firmware First / APEI (Optional)

Add to GRUB:

```
acpi_enforce_resources=lax
```

Ensure `CONFIG_ACPI_APEI_EINJ=y` and load `einj` module to inject test errors:

```bash
modprobe einj
einj_dump
einj_execute_trigger 0x0F  # inject CE
```

`rasdaemon` logs synthetic CE for pipeline testing.

---

## 90.6  Case Study: RowHammer Precursors

* Noted spike to 50 CE/h on DIMM_B1.  
* Grafana dashboard flagged trend; replaced DIMM during change window.  
* Prevented later UE that would have rebooted CMTS chassis.

---

## 90.7  PCIe Advanced RAS

Enable Correctable Error Mask to filter noise:

```bash
setpci -s 5f:00.0 CAP_EXP+0x1c.w=0xffff
```

Monitor `pcie_errors` table for `symbol_status=bad_tlp`.

---

## 90.8  Prometheus Dashboard Highlights

* CE/UE by DIMM slot (label via `edac-util --dimms` + `ras-mc-ctl`).  
* Cumulative CE since boot vs CE5min burn rate.  
* PCIe CE over interface throughput to spot flaky optics.

---

## 90.9  Security / MultiTenant Note

`rasdaemon` database may reveal physical addresses. Restrict with:

```bash
chmod 0600 /var/lib/rasdaemon/ras-mc_event.db
```

---

## 90.10  Exercises

1. Use `einj` to inject 10 CEs and verify Prometheus alert fires once.  
2. Script automatic DIMM offlining with `echo offline > /sys/devices/system/memory/memoryX/state`
   when CE rate >100/h.  
3. Add Alertmanager webhook that opens Jira when UE detected with DIMM locator.

---
