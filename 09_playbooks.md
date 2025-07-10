
# Chapter9  Full Playbooks & Blue/Green Rollouts

This chapter glues together everything from chapters18 into **complete,
productionready scripts** for firmware rollout, configuration drift repair,
and emergency rollbackeach fully annotated.

---

## 9.1  FourPhase Firmware Rollout

| Phase | Goal | Tools |
|-------|------|-------|
| **Stage** | Copy firmware to /tmp | `scp`, parallel |
| **Verify** | SHA256 + GPG sign check | `sha256sum`, `openssl` |
| **Apply** | Flash in batches with health checks | `ssh`, `fwupdate` |
| **Rollback** | Autorevert on >2% fail | associative array tally |

### 9.1.1  Stage Script (`stage_fw.sh`)

```bash
#!/usr/bin/env bash
source lib/logging.sh
source lib/retry.sh
KEY=/opt/keys/edge.pem

mapfile -t edges <edges.txt
export KEY

stage_node() {
  local ip=$1
  retry 3 scp -q -i "$KEY" fw.bin admin@"$ip":/tmp/ || return 1
  ssh -i "$KEY" admin@"$ip" 'sha256sum /tmp/fw.bin' |       awk '{print $1"  '$ip'"}'
}

parallel --tag -j100 stage_node ::: "${edges[@]}" >stage.sha || exit 2
```

### 9.1.2  Verify Phase

```bash
local_sha=$(sha256sum fw.bin | cut -d' ' -f1)
err=$(awk '$1!="'$local_sha'"' stage.sha | wc -l)
(( err == 0 )) || { echo "Staging mismatch $err"; exit 2; }
```

Include OpenSSL signature verify from Chapter8.

### 9.1.3  Apply Phase (batch50)

```bash
apply_node(){ ssh -i $KEY admin@$1 'fwupdate apply /tmp/fw.bin && reboot'; }
export -f apply_node

cut -d, -f1 devices.csv | tail -n+2 |   parallel -j50 --joblog apply.log apply_node {}
```

### 9.1.4  Health Check & Rollback

```bash
grep -cE 'Exitval:[^0]' apply.log >fails.txt
total=$(wc -l <edges.txt); fail=$(wc -l <fails.txt)
if (( fail *100 / total > 2 )); then
   parallel -j30 ssh -i $KEY admin@{} 'fwupdate rollback' ::: $(<fails.txt)
fi
```

---

## 9.2  Config Drift Remediation Playbook

1. **Pull runningconfig**:

```bash
ssh $ip 'show run' | sed 's/[[:space:]]\+$//' >run.cfg
```

2. **Diff vs golden**:

```bash
diff -u golden.cfg run.cfg >drift.patch || true
```

3. **Autopatch** if safe:

```bash
if patch -R --dry-run <drift.patch &>/dev/null; then
   patch <drift.patch
   copy run start    # Cisco-like save
fi
```

4. **Commit logs to Git** for audit.

---

## 9.3  Blue / Green Streaming Encoder Upgrade

* Blue = `ffmpeg-stream@current.service`  
* Green = `ffmpeg-stream@next.service`

```bash
systemctl start ffmpeg-stream@next
sleep 5
if curl -s http://localhost:8080/health | grep -q OK; then
   systemctl stop ffmpeg-stream@current
   systemctl link ffmpeg-stream@next.service ffmpeg-stream@current.service
else
   systemctl stop ffmpeg-stream@next
fi
```

Symlink swap keeps config files pointing to generic current.

---

## 9.4  Emergency PortScan & Service Restart

```bash
#!/usr/bin/env bash
set -euo pipefail
while read -r ip; do
  if ! nc -z -w2 $ip 443; then
     ssh $ip systemctl restart nginx &
  fi
done <edges.txt; wait
```

---

## 9.5  Prometheus Metric Exporter in Pure Bash

`export_metrics.sh`:

```bash
#!/usr/bin/env bash
succ=$(grep -c 'Exitval: 0' apply.log)
fail=$(grep -vc 'Exitval: 0' apply.log)
{
  echo "fw_success $succ"
  echo "fw_fail $fail"
} > /var/lib/node_exporter/text/fw.prom
```

---

## 9.6  Exercise

1. Modify the fourphase rollout to read `reboot_window` from CSV and delay
   `apply` until inside window.  
2. Extend config drift script to open a Jira ticket via REST if diff >50lines.  
3. Adapt blue/green encoder to maintain **two** previous versions for fast
   rollback chain.

---

> Next: **Chapter10  Testing, Linting & CI/CD Integration**.
