
# Chapter4  Text Processing DeepDive (sed, awk, grep, jq, diff)

Routers spit logs, modems expose JSON over HTTP, and switch configs live in
flat text.  This chapter arms you with **oneliners and multistep pipelines**
to transform, filter, and compare that data at scale.

---

## 4.1  sed  StreamEditor

### Option Reference

| Flag | Long | Purpose |
|------|------|---------|
| `-n` | `--quiet` | Suppress default print |
| `-i[SUF]` |  | Inplace edit (backup suffix optional) |
| `-e` |  | Add multiple scripts |
| `-r` | `--regexp-extended` | Use ERE (POSIXE) |
| `-s` |  | Treat multiple files as one |

### Common Commands

| Cmd | Effect | Example |
|-----|--------|---------|
| `s/old/new/g` | Global substitution | `sed -i 's/^debug=.*/debug=false/' app.conf` |
| `p` | Print pattern space | `sed -n '20,40p' syslog` |
| `d` | Delete line | `sed '/^#/d' config` |
| `a \text` | Append after match | `sed '/interface Gi0\/0/a \ standby 1 ip 10.0.0.1' sw.cfg` |

#### Multiline CTCP Correction

```
sed -e '/<CTCP_START>/,/<CTCP_END>/ s/old/new/' dump.txt
```

---

## 4.2  awk  The Communications SwissArmy Knife

### FieldSeparators

| FS | Use case |
|----|----------|
| `-F','` | CSV inventory |
| `-F'[: ]+'` | Syslog time prefixes |

### Builtin Vars

| Var | Meaning |
|-----|---------|
| `NR` | current record number |
| `NF` | number of fields |
| `FNR` | line number in current file |

### Aggregating CMTS Error Counters

```bash
snmpwalk -v2c -c pub $ip ifInErrors | \
  awk -F': ' '{sum+=$2} END{printf "errors=%d\n",sum}'
```

*Why not grep+cut?* awk avoids two forks and runs in ~40ms per 1000 OIDs.

### Multifile Join

```bash
awk -F, 'FNR==NR{v[$1]=$2; next}
         {print $0","v[$1]}' firmware.csv site_map.csv
```

Joins firmware targets with persite upgrade window in one pass.

---

## 4.3  grep & ripgrep

* For legacy BusyBox: `grep -E` supports basic ERE.  
* Modern servers: **ripgrep (`rg`)**  5 faster & respects `.gitignore`.

Scan massive TFTP tree for debug builds:

```bash
rg -tbin 'DEBUG=1' /tftpboot | tee debug_bins.txt
```

---

## 4.4  jq  JSON Processing for REST Routers

### Pull upstream/downstream SNR values

```bash
curl -s http://$ip/api/channels | \
  jq -r '.docsis[] | [.id,.snr] | @tsv' >snr.tsv
```

Combining with awk chart:

```bash
jq -r '.[]|.snr' snr.json | awk '{s+=$1; c++} END{print s/c}'   # avg SNR
```

---

## 4.5  diff & patch for Config Drift

1. **Extract running-config**:

```bash
ssh $sw 'show run' | sed 's/[[:space:]]\+$//' >run.cfg
```

2. Compare with golden:

```bash
diff -u golden.cfg run.cfg >drift.patch
```

3. If patch non-empty, flag remediation:

```bash
[[ -s drift.patch ]] && mail -s "Drift $sw" ops@isp.com <drift.patch
```

4. **Apply fix**:

```bash
patch -p0 /flash/startup.cfg <golden.patch
```

---

## 4.6  MultiTool Pipeline  Real Case

**Goal:** Extract modem MAC, SNR, and firmware from mixed JSON and SNMP,
output CSV for Grafana.

```bash
for ip in $(<modems.txt); do
  mac=$(snmpget -v2c -c pub $ip .1.3.6.1.2.1.2.2.1.6.2 | awk '{print $4}')
  snr=$(snmpget -v2c -c pub $ip 1.3.6.1.2.1.10.127.1.1.4.1.5.3 | awk '{print $4/10}')
  fw=$(curl -s http://$ip/api/info | jq -r .fw_version)
  printf '%s,%s,%s,%s\n' "$ip" "$mac" "$snr" "$fw"
done >metrics.csv
```

Feed into **Miller** for further joins:

```bash
mlr --csv sort -f snr metrics.csv | mlr --csv stats1 -a p95 -f snr
```

---

## 4.7  Exercises

1. Write a sed script that renumbers accesslists in Cisco config from `100`
   to `200` without touching comments.  
2. Use awk to produce a histogram of HTTP status codes from `edge.log`.  
3. Create a jq filter that turns nested modem channel JSON into a flat CSV
   with columns `id, freq, power`.  

---

> Next: **Chapter5  Service Monitoring & Editing**.

