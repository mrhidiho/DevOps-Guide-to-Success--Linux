
# Chapter36  Advanced SNMPv3 & Trap Handling

SNMP isn't deadDOCSIS nodes, UPSPDUs, and legacy switches rely on it.
Implementing secure **SNMPv3 USM**, bulkwalks, and trap receivers in Bash
keeps polling agentless and under your control.

---

## 36.1  SNMPv3 USM Quick Recap

| Mode | Auth | Privacy | Example |
|------|------|---------|---------|
| `-l authNoPriv` | HMACSHA | none | ro user |
| `-l authPriv` | HMACSHA | AES128 | rw secure |
| `-l noAuthNoPriv` | none | none | legacy |

Create user on device:

```
snmp-server user netops auth sha myAuthPass priv aes 128 myPrivPass
```

---

## 36.2  Bulk Walk Script

```bash
snmpbulkwalk -v3 -l authPriv -u netops -A myAuthPass -X myPrivPass   -On $ip 1.3.6.1.2.1.1 |   awk -F': ' '{print $1,$2}' >walk.txt
```

Loop over nodes:

```bash
parallel -j100 ./walk.sh {} ::: $(<edges.txt)
```

---

## 36.3  Parallel Get with `xargs`

```bash
snmpget -v3 -l authPriv -u netops -A $AP -X $PP   $(printf '%s ' $(<oids.txt)) "$ip"
```

`oids.txt` includes modem SNR OIDs.

---

## 36.4  Building a Trap Receiver in Bash

```bash
snmptrapd -Lf /var/log/snmptrapd.log -f &
tail -F /var/log/snmptrapd.log | while read line; do
  if [[ $line =~ "linkDown" ]]; then
     curl -XPOST $SLACK -d "payload={"text":"$line"}"
  fi
done
```

For highvolume traps use `snmptt` to convert to syslog JSON.

---

## 36.5  Prometheus Exporter

Convert bulkwalk to metrics:

```bash
snmpbulkwalk ... ifInErrors |   awk -F': ' '{print "ifInErrors{ifIndex="$2"} "$3}'   >/var/lib/node_exporter/text/snmp.prom
```

Export every 60s via cron.

---

## 36.6  Exercises

1. Add AES256 instead of AES128 privacy to the walk script (device must
   support).  
2. Write a Bash script that listens on UDP162 with `nc -klu` and counts
   traps per OID.  
3. Generate a Grafana dashboard of top SNMP interface errors using your
   exporter metrics.

---
