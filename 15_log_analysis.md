
# Chapter15  Log Analysis & Intrusion Indicators

Logs are your first forensic artifact when a router starts beaconing or a
jumpbox shows a bruteforce spike.  This chapter turns Bash into a miniSIEM:
tailing, grepping, and summarizing suspicious activity across thousands of
nodesfeeding alerts into Slack or Prometheus.

---

## 15.1  Log Sources in Communications Equipment

| Device | Log Facility | Retrieval |
|--------|--------------|-----------|
| Linuxbased CPE | systemdjournal (`journalctl`), `/var/log` | SSH / rsyslog |
| Cisco/Juniper switch | `show log` / syslog over UDP | Expect / SNMP |
| CMTS | Remote syslog | Central syslogd |

---

## 15.2  journalctl Filters for Security Events

### Failed SSH Auth

```bash
journalctl -u ssh --since -1h -g "Failed password"
```

### Kernel Alerts (potential exploit)

```bash
journalctl -k -p alert --since yesterday
```

### Service Exec via sudo

```bash
journalctl _COMM=sudo -g "COMMAND=" -S '24h ago'
```

---

## 15.3  Building an Auth Failure Counter

`auth_fail.sh`:

```bash
#!/usr/bin/env bash
count=$(journalctl -u ssh -S -1h -g "Failed password" | wc -l)
echo "ssh_failed_auth{host="$(hostname)"} $count"   > /var/lib/node_exporter/text/auth.prom
```

Prometheus alert:

```
- alert: SSHBruteForce
  expr: sum(ssh_failed_auth[5m]) > 20
  for: 2m
  labels: {severity: critical}
```

---

## 15.4  Syslog Pipeline for Switches

Expect script to dump last 200 lines:

```expect
spawn ssh admin@$ip
expect "#" { send "show log | last 200\r" }
expect "#" { send "exit\r" }
```

Parse for STP reconvergence or portflap:

```bash
grep -E "(STP.*re-rooted|%LINK-3-UPDOWN)" logs/$ip.log   | mail -s "Link flap $ip" noc@example.com
```

---

## 15.5  Suspicious Commands Detection

Audit interactive shells:

```bash
journalctl _COMM=bash -g "wget|curl|nc" -S -6h
```

List users issuing raw socket commands:

```bash
journalctl _COMM=bash -o short-unix |   awk '$0~/nc /{print $3}' | sort | uniq -c
```

---

## 15.6  Fail2ban Style with Pure Bash

```bash
banlist=/var/tmp/ban.txt
journalctl -u ssh -S -5m -g "Failed password" |   awk '{print $(NF-3)}' | sort | uniq -c | while read -r n ip; do
     if (( n > 5 )); then
        echo "banning $ip"
        iptables -I INPUT -s $ip -j DROP
        echo "$ip $(date +%s)" >>$banlist
     fi
done
```

Cleanup cron to unban after 24h.

---

## 15.7  Central Log Collection to JumpBox

Push critical journal fields:

```bash
journalctl -u ssh -o json --since -1m |   jq -c '{ts:.__REALTIME_TIMESTAMP,msg:.MESSAGE,host:.SYSLOG_IDENTIFIER}'   | curl -s -XPOST -H "Content-Type: application/json"         http://loghost:9200/ssh/_bulk --data-binary @-
```

Lightweight alternative to full Beats/Fluentd.

---

## 15.8  Ring Buffer Live Tail

Preview logs across 50 nodes in realtime:

```bash
parallel -j0 --tag ssh {} "journalctl -f -n0 -u ssh" ::: $(<edges.txt)
```

CtrlC aborts all tails via parallel.

---

## 15.9  Exercises

1. Write a script that tallies **sudo** invocations per user per day and mails
   top5 users to security@example.com.  
2. Implement a systemd unit + timer that runs `auth_fail.sh` every 2minutes.  
3. Extend fail2ban Bash to restore DROP rules after reboot (persist with
   `iptables-save | tee /etc/iptables/rules.v4`).

---

> **Appendix:** Integrating with ElasticSIEM and Grafana Loki (see repo
`extras/`). 
