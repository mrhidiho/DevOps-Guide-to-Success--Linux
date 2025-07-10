
# Chapter11  Reference Tables & Quick CheatSheets

Keep this chapter open during oncallyoull find flag lookups for the 25
most common CLI tools used in networkdevice automation.

---

## 11.1  SSH / SCP / Rsync

### ssh Key Flags

| Flag | Meaning |
|------|---------|
| `-oBatchMode=yes` | Disable interactive auth |
| `-oStrictHostKeyChecking=accept-new` | Autoadd unknown hosts |
| `-oConnectTimeout=5` | Fail fast on dead node |
| `-tt` | Force TTY (sudo) |
| `-J jump` | ProxyJump |

### scp Common

| Flag | Purpose |
| `-C` | Compress |
| `-p` | Preserve times/perms |
| `-r` | Recursive |
| `-3` | Thirdparty copy via local |

### rsync Core

| Flag | Meaning |
| `-a` | Archive (perm, times, symlinks) |
| `--partial` | Keep partial downloads |
| `--bwlimit=1M` | Throttle |
| `--delete` | Mirror (danger!) |

---

## 11.2  sed Quick Reference

| Pattern | Description |
|---------|-------------|
| `s/foo/bar/` | Replace first |
| `s/foo/bar/g` | Replace all |
| `-n '5,10p'` | Print range 510 |
| `'/^#/d'` | Delete comments |
| `'/pattern/a \ NEW'` | Append after match |

---

## 11.3  awk OneLiners

| Task | Oneliner |
|------|-----------|
| Sum column2 | `awk '{s+=$2} END{print s}'` |
| Print unique field1 | `awk '!seen[$1]++'` |
| CSV select site==den | `awk -F, '$3=="den"'` |
| Histogram HTTP codes | `awk '{c[$9]++} END{for(k in c) print k,c[k]}'` |

---

## 11.4  journalctl Filters

| Flag | Example | Result |
|------|---------|--------|
| `-S '10 min ago'` | Logs since 10min |
| `-u nginx` | Unit filter |
| `-p warning` | Priority warning |
| `_PID=1234` | By PID |

---

## 11.5  diff / patch

| Command | Meaning |
|---------|---------|
| `diff -u a b >file.patch` | Unified diff |
| `patch -p1 <file.patch` | Apply |
| `diff -qr dir1 dir2` | Only list differing files |

---

## 11.6  jq Cheats

| Query | Output |
|-------|--------|
| `jq .fw_version` | Value of field |
| `jq -r '.interfaces[] | [.id,.snr] | @tsv'` | id + snr TSV |
| `jq 'del(.debug)'` | Remove key |

---

## 11.7  Netcat & Curl

| nc Task | Command |
|---------|---------|
| Port scan 80 | `nc -zv $ip 80` |
| Send file | `nc -q0 $ip 9999 <file` |

| curl Flag | Meaning |
|-----------|---------|
| `-s` | Silent |
| `-o file` | Output |
| `-u user:pass` | Basic auth |

---

## 11.8  Color Codes for Echo

| Name | Code |
|------|------|
| Red | `\e[31m` |
| Green | `\e[32m` |
| Yellow | `\e[33m` |
| Reset | `\e[0m` |

Usage:

```bash
echo -e "${RED}ERROR${NC} failed flash"
```

---

## 11.9  Signal Numbers

| Signal | Num | Typical use |
|--------|-----|-------------|
| SIGINT | 2 | CtrlC |
| SIGTERM | 15 | Graceful stop |
| SIGKILL | 9 | Force kill |
| SIGHUP | 1 | Reload config |

---

## 11.10  Time Formats

| Format | Example |
|--------|---------|
| ISO8601 | `date -u +%Y-%m-%dT%H:%M:%SZ` |
| Epoch | `date +%s` |
| Log stamp | `date "+%b %d %H:%M:%S"` |

---

## 11.11  Prometheus Textfile

Metric line:

```
fw_success 123
```

Place in `/var/lib/node_exporter/text/metrics.prom`

---

*End of quickreference cheatsheet.*
