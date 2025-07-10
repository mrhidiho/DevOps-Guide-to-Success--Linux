
# Chapter73  btrfs Subvolume Rollback & qgroup Quotas for Firmware Images

`btrfs` snapshots make firmware upgrades atomic and *instantly* reversible,
while **qgroups** track perdevice disk quotaseven when images deduplicate
via CoW (copyonwrite).  This chapter shows a rollback workflow, quota
alerts, and performance tuning for NVMe.

_Last updated: 2025-07-10_

---

## 73.1  Filesystem Layout

```
/srv/firmware
  @live    current images
  @prev    previous (autorollback target)
  @staging incoming uploads
```

Create subvolumes:

```bash
btrfs subvolume create /srv/firmware/@live
btrfs subvolume create /srv/firmware/@prev
btrfs subvolume create /srv/firmware/@staging
```

Enable quotas:

```bash
btrfs quota enable /srv/firmware
```

---

## 73.2  Assign qgroup per Subvolume

```bash
btrfs qgroup assign /srv/firmware/@live 0/5
btrfs qgroup limit 10G 0/5
```

* Limits `@live` to 10GiB physical space; snapshots dont doublecount blocks.

List usage:

```bash
btrfs qgroup show --human-readable /srv/firmware
```

---

## 73.3  Atomic Upgrade & Rollback

### Upgrade

```bash
rsync -a --delete fw_new/ /srv/firmware/@staging/
btrfs subvolume snapshot -r /srv/firmware/@live /srv/firmware/@backup_$(date +%F)
btrfs subvolume delete /srv/firmware/@prev
mv /srv/firmware/@live /srv/firmware/@prev
mv /srv/firmware/@staging /srv/firmware/@live
```

### Rollback (failure)

```bash
mv /srv/firmware/@live /srv/firmware/@failed_$(date +%s)
mv /srv/firmware/@prev /srv/firmware/@live
systemctl restart flash.slice
```

All moves are metadataonly (CoW), <200ms on NVMe.

---

## 73.4  Scheduled Snapshot Rotation

Systemd timer:

```ini
[Unit]      # /etc/systemd/system/firmware-snap.service
[Service]
Type=oneshot
ExecStart=/usr/local/sbin/snap_fw.sh
```

`snap_fw.sh`:

```bash
btrfs subvolume snapshot -r /srv/firmware/@live \
  /srv/firmware/@snap_$(date +%F_%T)
btrfs subvolume delete $(ls -1t /srv/firmware/@snap_* | tail -n +31)
```

Keeps last 30 daily snapshots.

Timer:

```ini
[Timer]
OnCalendar=03:00
Persistent=true
```

---

## 73.5  Quota Alerting

Prometheus textfile exporter:

```bash
usage=$(btrfs qgroup show -F /srv/firmware 0/5 | awk 'NR==2{print $3}')
echo "fw_live_bytes $usage" \
 > /var/lib/node_exporter/text/fw_qgroup.prom
```

Alert: `fw_live_bytes > 9.5*1024*1024*1024`.

---

## 73.6  Performance Tuning

* **Mount options:** `compress=zstd:3,noatime,ssd,discard=async`  
* **Disable CoW for logs:**

```bash
chattr +C /srv/firmware/@live/logs
```

* **Defrag weekly (not snapshots):**

```bash
btrfs filesystem defrag -r -clzo /srv/firmware/@live
```

---

## 73.7  RealWorld Incident Averted

Firmware 2025Q3 failed regression; rollout script detected MD5 mismatch,
`mv`rollback executed in 0.23s, CMTS rebooted from previous image, customers
unaffected.

---

## 73.8  Exercises

1. Write a Bash wrapper that checks `btrfs scrub status` and refuses upgrade
   if any uncorrectable errors exist.  
2. Benchmark restore time: snapshot rollback vs copying 10GB across NVMe.  
3. Implement perPOP qgroup (0/10, 0/11) and generate Grafana pie chart
   showing usage.
---
