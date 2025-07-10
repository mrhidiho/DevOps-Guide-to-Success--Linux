
# Chapter 7 - CSV / TSV Workflows as Lightweight Databases

CSV inventories are simple, diff-friendly, and work on every jump-box-even when
SQLite or Postgres aren't available. This chapter shows how to treat CSV/TSV
files as mini-databases for fleet automation.

---

## 7.1  Why CSV Beats a Database (Sometimes)

| Advantage | Detail |
|-----------|--------|
| Zero infrastructure | No server, no creds-just a file in Git |
| Streamable | Process with `awk` without loading whole file |
| Human-readable | Open in Excel/Sheets for NOC hand-offs |
| Easy diff | Git shows line-level change history |

---

## 7.2  RFC-4180 Quirks You *Must* Handle

* Commas inside quotes: `"edge, lab",den,FW2025`  
* Embedded newlines in quoted fields.  
* Escaped quotes: `"fw""beta"""`.

For full compliance, use **csvkit** or **Miller**. For 90 % of cases,
setting `IFS=,` and using `read -r` is adequate.

---

## 7.3  Validating CSV Before Use

```bash
csvclean -q '"' devices.csv || { echo 'Bad CSV'; exit 1; }
head -1 devices.csv | grep -q '^ip,role,site,fw_target' || exit 2
```

`csvclean` ensures proper quoting; grep confirms header row.

---

## 7.4  Reading CSV Safely

```bash
while IFS=, read -r ip role site fw win; do
  [[ $ip == ip ]] && continue           # skip header
  printf '%s %-6s %-3s %s\n' "$ip" "$role" "$site" "$fw"
done < <(tail -n +2 devices.csv)
```

*Always* use `IFS=,` and `read -r` to keep backslashes literal.

---

## 7.5  `mapfile` / `readarray` for Fast Loads

```bash
mapfile -t ips < <(cut -d, -f1 devices.csv | tail -n+2)
echo "Loaded ${#ips[@]} IP addresses"
```

`mapfile` is written in C-much faster than loop-reads for >10 k rows.

---

## 7.6  Filtering & Joining with Miller (mlr)

### Filter by site

```bash
mlr --icsv --ocsv filter '$site=="den"' devices.csv >den.csv
```

### Outer join with upgrade targets

```bash
mlr --csv join -j ip --ul -f devices.csv upgrades.csv >join.csv
```

* `--ul` = unmapped-left: keeps devices with no upgrade row.

---

## 7.7  Aggregations with Awk

Average downstream SNR by /24:

```bash
awk -F'\t' '
  NR>1 {
    net=gensub(/(.*)\..*/, "\1", 1, $1)
    sum[net]+=$3; count[net]++
  }
  END {
    for(n in sum) printf "%s,%0.1f\n",n,sum[n]/count[n]
  }
' modem_metrics.tsv >snr_avg.csv
```

---

## 7.8  End-to-End Firmware Rollout Using CSV

```bash
# 1. Validate & filter night window
now=$(date +%H:%M)
while IFS=, read -r ip role site fw win; do
  [[ $ip == ip ]] && continue
  [[ $now > ${win%-*} && $now < ${win#*-} ]] || continue

  stage_fw "$ip" "firmware/$fw.bin" &
done <devices.csv
wait

# 2. Verify SHA and apply batch 50
cut -d, -f1 devices.csv | tail -n+2 |   parallel -j50 apply_fw {}
```

*Functions `stage_fw` and `apply_fw` are defined in Chapter 3 lib.*

---

## 7.9  Writing Results Back

```bash
printf 'ip,status\n' >results.csv
for ip in "${ips[@]}"; do
  printf '%s,%s\n' "$ip" "$(get_status "$ip")" >>results.csv
done
```

Use Miller to merge results into master inventory:

```bash
mlr --csv join -j ip --ul devices.csv results.csv >devices_new.csv
```

---

## 7.10  Exercises

1. Using Miller, split `devices.csv` into one file per `site`
   (`site_den.csv`, `site_kc.csv`, ).  
2. Write an awk script that detects duplicate IP addresses in the first
   column and prints them.  
3. Build a Bash function `csv_lookup ip column_name file` that returns the
   value of `column_name` for `ip` without loading the whole file into RAM.

---

> Next: **Chapter 8 - Cryptography & Secrets Management**.
