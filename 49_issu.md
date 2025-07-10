
# Chapter49  InService Software Upgrade(ISSU) on Routers

**ISSU** upgrades redundant control planes (RP/SCB) or dualsupervisor
chassis _without_ dropping BGP peers or customer traffic.  While vendor CLI
varies, the orchestration patterns are the sameBash can automate health
checks, switchover timing, and autoabort if SLA is threatened.

_Last updated: 2025-07-10_

---

## 49.1  ISSU HighLevel Flow

```
Stage image  Prechecks  Load standby supervisor  Switchover  Sync lines  Commit  Postchecks
```

| Phase | Risk | Mitigation |
|-------|------|-----------|
| Stage | none | Copy image via mgmt VRF |
| Load | standby crash | Monitor `show redundancy` |
| Switchover | 200ms routing reconverge | Graceful BGP timers |
| Commit | Linecard reload | PerLC health script |

---

## 49.2  Example (Cisco ASR9K CLI)

```bash
ssh $rp "admin install add source ftp://fw@10.0.0.1/asr9k-mini-x64-7.7.2.tar"
ssh $rp "admin install activate issu asr9k-mini-x64-7.7.2 synchronous"
ssh $rp "show install active summary"
```

Bash automation waits for `System Ready` before proceeding.

---

## 49.3  Bash ISSU Orchestrator

`issu_run.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail
rp1=$1; rp2=$2; image=$3

health() { ssh $1 "show redundancy summary | inc READY"; }

echo "Stage image on standby"
ssh $rp2 "copy tftp://10.0.0.1/$image harddisk:/"

echo "Activate on standby"
ssh $rp2 "admin install add harddisk:/$image activate issu"

echo "Pre-checks"
health $rp1 || exit 2
health $rp2 || exit 2

echo "Switchover"
ssh $rp1 "redundancy switchover"
sleep 60

echo "Post-checks"
health $rp1 || { echo 'FAIL'; exit 2; }
ssh $rp1 "show bgp summary" | grep -q 'Estab'
```

Aborts if BGP not Established after 60s.

---

## 49.4  CommitConfirm for Juniper

```bash
request system software add jinstall-22.3R1.tgz no-copy
request system reboot in-service-upgrade
request system commit confirm 10
```

Bash watchdog:

```bash
sleep 600
ssh $sw "show system commit" | grep -q 'Confirmed' ||     ssh $sw "request system rollback"
```

---

## 49.5  LineCard Health Verification

```bash
ssh $rp "show platform | include UNPOWERED|FAILED" && abort=1
ssh $rp "show interfaces terse | match down" | wc -l
```

Trigger autorollback if `abort` flagged.

---

## 49.6  Prometheus & Grafana During ISSU

* Record rule `issu_state{chassis}` set to `pre`, `switchover`, `post`.  
* Grafana alert suppress rules when `issu_state!="post"`.

---

## 49.7  Exercises

1. Modify `issu_run.sh` to stagger ISSU across POPs using CSV window column.  
2. Script that measures BGP flap time (`clear bgp` + timestamp) before and
   after ISSU to ensure no regression.  
3. Collect `show redundancy history` logs into Git after each ISSU for audit.

---
