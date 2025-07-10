
# Chapter40  systemd Timers & Template Units for Scheduled Jobs

Cron is portable but lacks dependency control, structured logging, and
parallel execution limits. systemd timers provide native scheduling with
journal logs, process isolation, and templated units for pernode operations.

---

## 40.1  Basic Timer Anatomy

`/etc/systemd/system/backup.service`:

```
[Unit]
Description=Config backup

[Service]
Type=oneshot
ExecStart=/usr/local/sbin/backup.sh
```

`/etc/systemd/system/backup.timer`:

```
[Timer]
OnCalendar=*-*-* 02:00:00
RandomizedDelaySec=15m

[Install]
WantedBy=timers.target
```

Enable:

```bash
systemctl enable --now backup.timer
```

---

## 40.2  Viewing & Debugging

```bash
systemctl list-timers --all
journalctl -u backup.service
```

---

## 40.3  Template Units for PerHost Jobs

`flash@.service`:

```
[Service]
Type=oneshot
ExecStart=/usr/local/bin/flash_fw %i
```

`flash@edge-1.timer`:

```
[Timer]
OnCalendar=Sat 03:00
Unit=flash@edge-1.service
```

Enable multiple:

```bash
systemctl enable flash@edge-{1..10}.timer
```

---

## 40.4  Conditionals & Safeguards

Run only if AC power:

```
ConditionACPower=true
```

Prevent overlap (BusyBox systems):

```
ExecStartPre=/usr/bin/flock /var/run/flash.lock -n true
```

---

## 40.5  Time Zone & Clock Drift

Use `systemd-timesyncd` and `OnCalendar=UTC` to avoid local DST surprises.

---

## 40.6  Exercises

1. Convert your cronbased `health_audit.sh` to systemd timer; include
   `OnBootSec=10min`.  
2. Write templated timer that stages firmware to every node whose hostname
   matches `%H`.  
3. Use `systemd-run --on-active=5m` to schedule a oneoff rollback task when
   deployment fails.

---
