
# Chapter5  Service Monitoring & Editing (systemctl / journalctl)

When a firmware rollout goes sideways at 03:00, **systemd tooling** is your
fastest path to triage and remediation across hundreds of Linux-based
edge devices.

---

## 5.1  QuickReference Command Matrix

| Command | Purpose | Example |
|---------|---------|---------|
| `systemctl status UNIT` | Detailed status + last 10 log lines | `systemctl status videoedge` |
| `systemctl restart UNIT` | Graceful stop  start | Restart streaming cache |
| `systemctl reload UNIT` | Signal HUP / re-read config | Zerodowntime Nginx TLS reload |
| `systemctl enable UNIT` | Autostart at boot | Enable telemetry agent |
| `systemctl edit UNIT` | Dropin override file | Add Restart limits |
| `systemctl list-units --failed` | Show failed services | Nightly NOC cron |
| `journalctl -u UNIT` | View unit logs | Tail last 500 lines |
| `journalctl -S '10 min ago' -g ERR` | Time & grep filter | Find recent errors |
| `systemctl set-property` | Live cgroup tweak | Throttle CPU/Memory |
| `systemctl mask UNIT` | Prevent all starts | Block NetworkManager on CPE |

---

## 5.2  Reading Logs like a Pro

### Tail live with color

```bash
journalctl -u videoedge -f -o short-iso --no-pager
```

* `-f` follows live (CtrlC to exit)  
* `-o short-iso` ISO8601 timestampseasy diff between nodes

### Filter by PID & priority

```bash
pid=$(pgrep videoedge)
journalctl _PID=$pid -p warning --since -10m
```

---

## 5.3  Dropin Overrides (`systemctl edit`)

Add automatic restart & memory limit **without touching vendor .service**:

```ini
# /etc/systemd/system/videoedge.service.d/override.conf
[Service]
Restart=on-failure
RestartSec=5
MemoryHigh=2G
```

Reload unit files and restart gracefully:

```bash
systemctl daemon-reload
systemctl restart videoedge
```

Why override? Package upgrades wont overwrite `/etc/systemd/system/*.d`.

---

## 5.4  Remote systemctl via SSH

Adhoc restart of squid on an edge node without Ansible:

```bash
ssh -t root@edge-23 "systemctl restart squid"
```

Add `-o BatchMode=yes` for CI jobs (`ssh -oBatchMode=yes -T `).

**Bulk pattern** (parallel fanout):

```bash
parallel -j20 "ssh -oBatchMode=yes {} systemctl restart videoedge" ::: $(<edges.txt)
```

---

## 5.5  HealthAudit Script

`health_audit.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail
while read -r host; do
  status=$(ssh "$host" 'systemctl is-active videoedge || true')
  if [[ $status != active ]]; then
     echo "$host FAIL $status"
  fi
done <edges.txt
```

Run hourly via cron; output parsed by Prometheus textfile collector.

---

## 5.6  Live Resource Tweaks with `set-property`

Throttle CPU to 25% for soak tests:

```bash
systemctl set-property kafka CPUQuota=25%
```

Lift the cap:

```bash
systemctl set-property kafka CPUQuota=0
```

Memory protection (kill if cgroup >3GB):

```bash
systemctl set-property videoedge MemoryMax=3G
```

---

## 5.7  Kill vs Restart vs Reload  Decision Table

| Situation | Action | Reason |
|-----------|--------|--------|
| Config change, hotreload supported | `reload` | No service disruption |
| Binary upgrade, short outage OK | `restart` | Ensures clean state |
| Hang, unresponsive to TERM | `kill --signal=SIGKILL` then `start` | Force recovery |
| Unknown crash loop | `mask` then investigate | Prevent thrashing |

---

## 5.8  Automated FailedUnit PagerDuty Hook

```bash
failed=$(systemctl --failed --no-legend | wc -l)
if (( failed > 0 )); then
  pd-send "FAILED units on $(hostname): $(systemctl --failed --no-pager)"
fi
```

`pd-send` is a small Bash wrapper around PagerDuty EventsV2 API.

---

## 5.9  Exercises

1. Write a script that lists all units in `failed` **and** shows the last 15
   log lines for each.  
2. Create a dropin override that sets different
   `CPUQuota` per VLAN using `%i` instance templates.  
3. Benchmark `systemctl status` vs `journalctl -u` for tailing last 20 lines
   and explain why status reads from memory-cached journal.

---

> Next: **Chapter6  Remote Execution & File Transfer**.
